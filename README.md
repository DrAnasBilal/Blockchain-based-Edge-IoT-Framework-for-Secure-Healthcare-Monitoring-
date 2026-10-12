# Blockchain-based Edge-IoT Framework for Secure Healthcare Monitoring and Predictive Emergency Response

Reference implementation of the framework described in the manuscript *"Blockchain-based Edge-IoT
Framework for Secure Healthcare Monitoring and Predictive Emergency Response Using Hyperledger
Fabric"*. Sensor payloads are encrypted at the edge and stored off-chain (IPFS); only hashes,
identifiers, consent grants and audit events go on a permissioned Hyperledger Fabric ledger; a
CNN-LSTM split across edge and cloud predicts emergencies in a rolling 60-minute horizon.

> **Research prototype, not a medical device.** See [Limitations](#limitations-and-differences-from-the-paper).

## Architecture

```
 sensors ──mTLS──► EDGE GATEWAY (Raspberry Pi)                         CLOUD / MEDICAL SERVER
                   1 causal Kalman + min-max      ┌──── 6x6 grid ──► edge CNN (Conv64→Conv128→MaxPool)
                   2 clinical-rule fast path ─────┤                         │ 1152-d features
                   3 AES-256-GCM encrypt          │                         ▼
                   4 SHA-256 of ciphertext        │                  LSTM(128)→FC(64)→softmax  ──► risk ≥ τ ?
                   │            │                 │                                                   │
          ciphertext▼            ▼ hash,CID,keyId │                          Emergency gateway (WebSocket)
              IPFS (private)   HYPERLEDGER FABRIC ◄── emergency events ◄──────────────┘      │
                               • ACL / consent / revocation                      clinician confirms or
                               • anchored hashes (append-only)                   timeout → fallback path
                               • PDC: AES keys (never on shared state)
                               • wrapped-key release + audit log, break-glass
```

## Where each part of the paper lives

| Paper | Code |
|---|---|
| 3.2 aggregation, Kalman, min-max, 6x6 grid | `iomt/preprocessing.py` |
| 3.3 on-chain access control, Algorithms 2–3 | `chaincode/healthcare/healthcare.go`, `iomt/ledger.py` (`MockLedger`, `FabricCLILedger`) |
| 3.4 off-chain storage, Algorithms 4–5 | `iomt/storage.py`, `iomt/edge_gateway.py`, `iomt/physician_client.py` |
| 3.5–3.6 CNN-LSTM, edge/cloud split, Table 2/5 | `iomt/model.py`, `iomt/predictor.py`, `iomt/train.py` |
| 3.8 AES-GCM, 96-bit nonces, tag, 24 h / 10k-tx key rotation | `iomt/crypto.py` |
| 3.9 datasets, patient-wise + chronological splits, leakage protocol, threshold τ | `iomt/data.py`, `iomt/metrics.py`, `iomt/train.py` |
| 4.1 emergency definition and thresholds | `iomt/emergency_rules.py` |
| 4.2 consent, revocation, least privilege, break-glass, admins cannot read keys | chaincode + `MockLedger` |
| 4.3 certificate lifecycle (mTLS, private CA, OCSP) | Fabric CA + uvicorn mTLS flags (below) |
| Fig. 3 emergency path, clinician confirmation, timeout monitor, fallback | `iomt/alerts.py`, `iomt/cloud_server.py` |
| 5.1–5.4 metrics, CIs, ablations, attack simulation | `iomt/metrics.py`, `scripts/benchmark.py`, `scripts/simulate_attacks.py` |

## Repository layout

```
iomt/                 python package (edge, cloud, crypto, ledger clients, model)
chaincode/healthcare/ Go smart contract + private data collection config
docker/               IPFS (Kubo v0.22.0) compose file
scripts/              demo, benchmark, attack simulation, Fabric deploy/enrol helpers
tests/                30 unit/integration tests (no external services needed)
```

## Quick start (no Docker, no Fabric, no PyTorch)

```bash
python -m venv .venv && source .venv/bin/activate
pip install numpy pandas scikit-learn cryptography requests      # core
python -m unittest discover -s tests -t . -v                      # 30 tests
python -m scripts.run_demo                                        # full flow, in-process
python -m scripts.simulate_attacks --n 500                        # 500 randomised attacks
python -m scripts.benchmark --records 2000 --reps 5
```

The demo uses `MockLedger` (a hash-chained Python re-implementation of the chaincode rules),
in-memory storage and a heuristic risk model, and shows: streaming → early warning before the
glucose threshold → encrypted anchoring → consent-gated decryption → tamper detection →
revocation → fallback escalation.

## Train the CNN-LSTM

```bash
pip install torch
python -m iomt.train --patients 500 --epochs 100 --out models/run1            # synthetic cohort
python -m iomt.train --diabd data/diabd.csv --patients 5288 --out models/diabd # DiaBD-seeded
```

Protocol (Sec. 3.9): causal Kalman → **patient-level** 70/15/15 split → min-max fitted on training
patients only → per-patient windows (no window crosses a patient boundary) → Adam (lr 1e-3,
exponential decay to 0.1× over training), batch 32, ≤100 epochs, early stopping → τ chosen on the
**validation** cohort by Youden's J (ties by F1) → test set evaluated **once**, with 1,000-resample
bootstrap 95% CIs. Outputs `model.pt` and `meta.json` (τ, scaler, metrics).

## Run the cloud tier and edge gateway

```bash
IOMT_MODEL_DIR=models/run1 uvicorn iomt.cloud_server:app --port 8000
# mTLS (Sec. 4.3): add --ssl-keyfile server.key --ssl-certfile server.crt --ssl-ca-certs ca.crt --ssl-cert-reqs 2
```

Endpoints: `POST /v1/infer`, `POST /v1/alerts`, `POST /v1/alerts/{id}/confirm`,
`WS /ws/clinician?channel=primary|fallback`. Unconfirmed alerts escalate to the `fallback` channel after
`IOMT_CONFIRM_TIMEOUT_S` (default 30 s).

```python
from iomt.cloud_client import HttpCloud
from iomt.edge_gateway import EdgeGateway
from iomt.predictor import TorchPredictor
from iomt.storage import IPFSStorage

pred = TorchPredictor("models/run1")
gw = EdgeGateway("edge-gw-1", ledger, IPFSStorage("http://127.0.0.1:5001"), pred,
                 HttpCloud("https://cloud:8000", cert=("edge.crt", "edge.key"), verify="ca.crt"))
result = gw.ingest("pat-001", ts, np.array([glucose, sys, dia, pulse, spo2, bmi]))
```

## Real IPFS and Hyperledger Fabric

```bash
docker compose -f docker/docker-compose.yml up -d                  # IPFS (Kubo v0.22.0)
FABRIC_SAMPLES=~/fabric-samples scripts/deploy_fabric.sh           # test-network + chaincode 'healthcc'
FABRIC_SAMPLES=~/fabric-samples scripts/fabric_enroll.sh           # identities with uid/role attributes
```

The chaincode reads the caller's X.509 attributes `uid` and `role`
(`admin | edge | service | patient | physician`). Use one `FabricCLILedger` per identity:

```python
from iomt.ledger import FabricCLILedger
env = {"CORE_PEER_TLS_ENABLED": "true", "CORE_PEER_LOCALMSPID": "Org1MSP",
       "CORE_PEER_MSPCONFIGPATH": ".../users/edge-gw-1@org1.example.com/msp",
       "CORE_PEER_ADDRESS": "localhost:7051", "CORE_PEER_TLS_ROOTCERT_FILE": ".../peer0.org1.../tls/ca.crt",
       "FABRIC_CFG_PATH": "~/fabric-samples/config"}
ledger = FabricCLILedger(env, orderer_ca=".../orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem",
                         peers=[("localhost:7051", ".../peer0.org1.../tls/ca.crt")])
```

## Security design (as implemented)

* AES-256-GCM, 96-bit nonce = 32-bit CSPRNG prefix ‖ 64-bit counter (unique per key), 128-bit tag;
  AAD binds ciphertext to `patient|record_id`.
* Key per patient, rotated every 24 h or 10,000 encryptions; retired keys kept for audit.
* Keys live in a Fabric **private data collection** (`PDCPatientKeys`); they are released only
  RSA-OAEP-wrapped to the requesting physician's public key, after a consent check, a freshness
  check (±60 s) and a nonce replay check; every attempt (granted or denied) is logged on-chain.
* Admins and edge nodes cannot call `RequestKey`; only patients grant/revoke consent for their own data.
* Anchors are append-only; retrieval re-hashes the IPFS blob and compares with the anchored SHA-256
  before decrypting.
* Break-glass: reason required, 1-hour grant, flagged for retrospective review.
* Edge fast path: a threshold hit raises an alert and an on-chain emergency event even when the cloud is down.

## Limitations and differences from the paper

Please read before comparing numbers.

1. **Reported results are not reproduced by this repo as shipped.** Latency (142.5 ms), 845 TPS,
   97.10 % accuracy etc. depend on a real Fabric network (3 orgs, 4 peers, 3 Raft orderers),
   private IPFS cluster and the paper's data. `benchmark.py` defaults to mock back-ends and only
   measures the Python edge pipeline.
2. **DiaBD and PIMA are cross-sectional** (one row per person). The paper's time-series setup
   (30 days, 10 time points, 60-min horizon, 1,060 emergency events) cannot come from them directly,
   so `data.py` *simulates* trajectories (optionally seeded with DiaBD baselines). Metrics from
   simulated data should be described as such. Replace `data.py` loaders with real longitudinal CGM/BP/SpO₂
   data to make clinical claims.
3. **"CID = SHA-256(ciphertext)" (Algorithm 4)** is not an IPFS CID. The code anchors both the SHA-256
   and the IPFS-returned CID.
4. **Table 2 layer sizes are inconsistent** (FC input 128, output-layer input 64). Implemented as
   LSTM(128) → FC(64) → softmax(2).
5. **Pulse thresholds** for tachy/bradycardia are not specified numerically; 40/150 bpm are assumed.
   The hyperglycaemia rule uses glucose > 350 mg/dL without the paper's "fast-changing trend" clause.
6. **Fabric topology**: `deploy_fabric.sh` uses `test-network` (2 orgs, 1 orderer). The paper's
   3-org / 3-orderer EtcdRaft layout needs a custom `configtx.yaml` and compose files (not included).
   Endorsement policy `OR(Org1.peer, Org2.peer)` follows the paper; note that `RequestKey` uses
   randomised RSA-OAEP, so it must not require multiple endorsers.
7. **Go chaincode and `FabricCLILedger` were written without a Go toolchain/Fabric network available
   and are untested** - run `go mod tidy && go vet ./...` in `chaincode/healthcare`, then smoke-test the
   functions with `peer chaincode invoke`. The Python code (crypto, pipeline, `MockLedger`,
   alerts) is covered by the unit tests; `model.py`/`train.py`/`cloud_server.py` need `torch`/`fastapi`
   and were not executed here.
8. Fabric authenticates callers via X.509 identity, so the paper's explicit "physician signature"
   field is replaced by the transaction signature itself.
9. Not implemented: homomorphic/ABE variants, OCSP stapling, power-consumption measurement, pod
   autoscaling experiments (Fig. 6–7), multi-cluster consensus.

## License / citation

Add your license and cite the manuscript when using this code.
