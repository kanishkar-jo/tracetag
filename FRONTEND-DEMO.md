# TraceTag frontend technology demonstration

Open http://127.0.0.1:8000, sign in with your existing account and select **Technology demo** (the default page).

All demonstration values live only in browser memory. Leaving this page or refreshing resets them. Nothing in this demo submits records, edits stored evidence or creates blockchain transactions.

## Physical readings

- Temperature and humidity display only existing version-2 or version-3 records explicitly tagged `dht11`. Missing, simulated and legacy readings are not filled in. Old physical readings are labelled stale.
- MQ-4 and MQ-5 display authenticated v3 physical ADC readings, or **Awaiting sensor** when missing. No MQ-6 value is relabelled and no gas reading is fabricated. See `docs/ESP32-WIFI.md` for the connection steps.
- The original Operations, Shipment and Evidence pages retain recorded history, including the earlier sensor formats.

## Presentation walkthrough

1. **Journey:** choose a commodity, then Play or Next sample. One step represents one illustrated minute. Playback is accelerated and stops after 80 samples.
2. **Connectivity:** choose Network outage, generate two samples, hand off to Truck B, then restore the link. Watch packets move from local queue to relay to acknowledged delivery. No physical relay/network operation occurs.
3. **Security:** rotate the demo key, edit a demo block, verify to reveal a mismatch, then restore and verify again. AES/HKDF and independent ledger nodes are storyboards; the check is a simulated outcome, not cryptography.
4. **Intelligence:** select Traffic delay or Rough handling. Change commodity to see different proposed gas modules, then acknowledge the demo alert. Gas indices and risk outcomes are illustrative, not concentrations, AQI, validated spoilage decisions or shelf-life predictions.
5. **Energy:** switch between daylight/night and toggle adaptive operation. Values are illustrative; physical solar hardware, a validated power budget and sleep control are outside this frontend.
6. **Passport:** preview stakeholder views, scan the QR for a demo JSON summary, export a report or create a new browser-only batch. The QR is not a hosted public passport. Export also displays copyable JSON if the browser does not download files.

## Editable source

- `frontend/src/TechnologyDemo.tsx`: panels, scenario presentation, physical input filtering, QR and report.
- `frontend/src/demoModel.ts`: browser-only sample, relay, key and tamper state.
- `frontend/src/technology-demo.css`: styles and responsive layouts.
- `frontend/src/App.tsx`: navigation and integration with existing sensor reads.

The QR component is `qrcode.react`. Build from `frontend` using `pnpm run build`.
The existing local launcher can run with `scripts/start-demo.ps1 -NoBrowser -SkipBlockchain`; this preserves the saved blockchain state while starting the dashboard and broker.

Existing login and account roles remain in force. The stakeholder selector changes the demonstration presentation only.
