# TraceTag

Offline-first, tamper-evident shipment evidence for the SIH prototype.

**All source is ordinary editable text.** Open this folder in VS Code or your preferred editor. The backend, UI, database schema, contract, configuration, and firmware contract are directly editable.

The current sensor plan uses **MPU6050 motion, MQ-6 air-monitoring gas response and DHT11 temperature/humidity**, with other demo inputs simulated. The version 2 data format protects explicit source labels alongside every reading. See [Sensor setup and firmware contract](docs/SENSOR-SETUP.md). The simulator now sends clearly labelled v2 data; existing v1 history and checkpoints remain supported unchanged.

## Quick start on this Windows machine

From PowerShell in this folder:

```powershell
.\scripts\setup.ps1
.\scripts\start-demo.ps1
```

Open http://127.0.0.1:8000. The setup installs only project dependencies. It uses an installed Python/Node toolchain or the existing bundled runtime on this computer. On another Windows PC, install Python 3.12+, Node.js 22+, and pnpm first.

The launcher starts an authenticated TLS MQTT broker on loopback port 8883, local Anvil, deploys/reuses the contract, and starts the backend with the built React UI. It uses data/demo.db with append-only evidence guards; deliberate corruption endpoints are disabled. Sign in using your private account from data/security/access.json. See [Security controls and ESP32 requirements](docs/SECURITY.md).

Generate simulated readings:

```powershell
.\.venv\Scripts\python.exe -m scripts.mock_device --journal data/demo-simulator.db --generate 24 --historical --drop-first-ack
```

This writes simulated records durably, publishes them through MQTT, deliberately ignores the first application ACK, and retries safely. The dashboard will show 24 stored records, rather than 25. Run it again to append more records; sequence numbers persist.

In **Evidence**, click **Verify evidence**, then anchor a range (initially 1–24). Local hash-chain VALID and independently anchored VERIFIED are separate states. The UI labels the network **Local Anvil / simulated EVM**.

Stop the services:

```powershell
.\scripts\stop-demo.ps1
```

The launcher records only its own process IDs/start times. Database and simulator history are preserved. Anvil is a local demonstration chain; a reset or loss of its state can make old checkpoints unavailable.

If the saved Anvil state cannot be read, the launcher stops before overwriting it. Use `scripts/start-demo.ps1 -SkipBlockchain` to run the dashboard and authenticated MQTT while investigating recovery. Existing checkpoint verification will report unavailable. This does not repair, reset or recreate blockchain history.

## Edit and run the UI

```powershell
pnpm --dir frontend dev
```

Vite serves the editable UI at http://127.0.0.1:5173 and proxies API requests to port 8000. Edit frontend/src/App.tsx and frontend/src/styles.css. For the built dashboard:

```powershell
pnpm --dir frontend build
```

Refresh the browser after a build. A running backend serves these updated assets.

## Edit and run the backend manually

Copy .env.example to .env if it does not exist. Keep configuration changes there.
Stop the launcher before taking over its ports. Start your broker and Anvil separately, then:

```powershell
.\.venv\Scripts\python.exe -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000 --reload
```

Use a single API process: do not add multiple workers sharing the same MQTT client ID.
API access requires a session login; see docs/SECURITY.md. Interactive API docs are disabled in secured mode.

## Offline/reboot/replay rehearsal

Generate offline history in one simulator process, then replay it in a new process:

```powershell
.\.venv\Scripts\python.exe -m scripts.mock_device --journal data/demo-simulator.db --generate 5 --historical --offline
.\.venv\Scripts\python.exe -m scripts.mock_device --journal data/demo-simulator.db --drop-first-ack
```

Use --unsynced with --generate to model unknown UTC after an offline reboot. Such records remain in history but are not plotted at a made-up UTC time.

The simulator stores before sending; it retains unacknowledged rows across restarts. Its SQLite journal is a software analogue of the proposed LittleFS journal. It does not establish that ESP32 flash writes survive power loss.

After 30 seconds without records the device is STALE; after 90 seconds it is OFFLINE. A connected broker alone does not make a device LIVE. Delayed-arrival markers are inferred from sample/receive times, not proof of the cause of a delay. The disconnected device's growing queue is unknown remotely.

## Actual hardware and Mosquitto

For the agreed two-laptop arrangement, see [Two-laptop setup](docs/TWO-LAPTOP-SETUP.md). The ESP32 connects directly over Wi-Fi to the broker on this software laptop; the other laptop handles firmware upload and serial monitoring.

The normal hardware path uses Mosquitto. This machine's automated integration test uses the bundled Aedes development broker instead; Mosquitto has not been installed or physically tested.

For native Mosquitto, create data/mosquitto and run from this project root:

```powershell
mosquitto -c config/mosquitto.conf -v
```

Alternatively, with Docker Desktop installed:

```powershell
docker compose up -d
```

Before starting Mosquitto or Docker, provision its password file and a certificate for the current laptop LAN IP as described in docs/SECURITY.md. Both brokers use authenticated TLS port 8883. Do not run them together on the same port. Set the ESP32's broker address to the laptop LAN IP. The development broker is loopback-only and cannot accept the ESP32.

Use docs/PROTOCOL.md, docs/record.schema.json and docs/test-vectors.json as the firmware handoff. Verify the exact hashes in both C++ and Python before hardware integration.
Physical acceptance tests: valid sensing, durable journal before publish, offline reboot, persistent sequence/ACK cursor, oldest-first replay, ACK loss, tamper switch and impact event.

## Local contract tools

```powershell
pnpm compile:contract
.\node_modules\@foundry-rs\anvil-win32-amd64\bin\anvil.exe --host 127.0.0.1 --chain-id 31337
# In another terminal:
.\.venv\Scripts\python.exe -m scripts.deploy_anchor
```

Copy the printed ANCHOR_CONTRACT and ANCHOR_ACCOUNT into .env and restart the manually launched backend. The demo launcher handles these values itself. This package includes the Windows Anvil binary; Docker or a native Foundry installation is an alternative.

## Tests

```powershell
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m scripts.verify_security_mqtt
pnpm --dir frontend build
pnpm compile:contract
# With the live demo seeded and anchored:
.\.venv\Scripts\python.exe -m scripts.verify_local_demo
```

The last command verifies live Anvil, copies the demo DB into a temporary test directory, changes historical content, then recomputes the entire local chain. Both manipulations must fail against the existing independent checkpoint. It does not corrupt the live demo.

For a deliberate historical edit, use scripts/start-demo.ps1 -AllowTamperingDemo and an admin login. It uses the separate data/tamper-demo.db; use a separate simulator journal too. The normal database remains protected.

## Files and scope

- backend/: FastAPI, MQTT, SQLite, verification, Merkle, Anvil and tests.
- frontend/: React/TypeScript source and CSS.
- contracts/: Solidity source; generated compiler artifacts live under out/.
- scripts/: setup, start/stop, simulator, compilation, deployment and smoke checks.
- config/: editable broker configuration.
- docs/: architecture, device protocol, generated schema and fixed test vectors.
- data/: local runtime databases, simulator queue, chain state and logs (ignored by version control).

Implemented scope is one device and shipment, with local viewer/operator/admin accounts, TLS MQTT, per-device HMACs, authenticated ACKs, storage guards and audit events. There is no provisioning UI, public-chain deployment, firmware flashing, real custody transfer workflow or QR page. See docs/SECURITY.md for security limits. The database retains a custody table for future work but does not fabricate custody events. Raw telemetry stays off-chain. Sensors can be inaccurate or spoofed; hash chains and checkpoints detect history changes, not physical truth.

See docs/VALIDATION.md for the checks actually run and remaining hardware work.

## Official references

- FastAPI lifecycle: https://fastapi.tiangolo.com/advanced/events/
- Paho MQTT callbacks: https://eclipse.dev/paho/files/paho.mqtt.python/html/client.html
- Vite: https://vite.dev/guide/
