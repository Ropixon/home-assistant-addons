## HA app 0.9.6 — sensor table editing, 2026-10-30

- Renaming sensors in the *Paired sensors* table: after one name, a click into
  another row's name jumped back to the previous row (typing went there) and a
  draft was lost at the next refresh. Leaving a field saves it (change) while
  the focus moves; the panel then disabled every input while the command ran,
  so the next field refused the focus. The 1 s refresh also moved every row
  (append), which detached the field being edited. Fields now stay enabled
  (their command waits until the gateway is free), rows move only when the
  order changes, and the field being edited keeps its draft. The same applies
  to the settings fields of the sensor cards.

## HA app 0.9.5 — RopixNet device management, 2026-10-30

- The HA gateway and sensor pages link to RopixNet device management through
  the actual Supervisor app ID. Pairing and sensor configuration entities stay
  enabled with the same IDs/topics for automations, but become hidden advanced
  configuration controls when the management link is available. Measurements
  remain visible. Without a resolved app link, the MQTT controls remain visible.
- The panel has a localized return link to the preceding HA frontend page, or
  to the HA device list for a direct ingress visit. Standalone local panels do
  not show an HA return link. The sidebar entry is named RopixNet.
- Pairing instructions address one sensor, appear only while pairing is active,
  and say when to release the button (LED flashing rapidly); a separate tip
  explains the short press. Results survive window closure; automatic pairing
  no longer displays artificial progress percentages or stale "pairing
  started" confirmations.
- A paired sensor that reconnected while pairing was open (session lost,
  HS1 on the pairing channel) showed as pairing in the panel (80 %) and, when
  its key exchange timed out, as *Pairing sensor … failed · 0 % · The gateway
  refused the request*. Only an approved pairing attempt now counts as
  pairing progress, result or failure; a reconnect logs `RECONNECT FAILED`
  and the sensor keeps its pairing and retries. The failure reason shown is
  the attempt of the sensor named, never another sensor's.
- *paired* = the session is committed and the sensor stored (manual approval:
  100 %). It is also shown for 15 s while the window stays open, then the
  window searches again. Nothing after it (descriptor, first values) holds it.
- *Paired sensors*: one row per sensor (name renamed in place, ID, online,
  values, signal, battery, last contact). A sensor that reports lights up,
  a pressed button highlights its row and brings it into view (no jump to
  the settings card); a click on a row opens the sensor's settings. On a
  phone the ID and last contact (then signal and battery) columns hide.
- A different dongle plugged in for the old one: *Check the dongle again*
  accepted only the saved dongle's UID, so a replacement was never chosen
  (and the background retry then showed *disconnected*); only the manual port
  choice worked. A complete scan with exactly one compatible dongle now takes
  it, saves it and logs `DONGLE CHANGED: old -> new`. With two or more the
  choice stays with the user. The background retry still reopens only the
  saved dongle.

## HA app 0.9.4 — the last values survive a restart, 2026-10-27

- After a restart of the app, sensors showed *Unavailable* in HA until their
  next report (up to the reporting interval) and their values were cleared.
  The gateway now keeps each sensor's last report in `gateway.db` (one row per
  sensor, replaced by the next report) and shows it again after a start:
  HA and the panel keep the last values, and a sensor is offline only when
  its last report is older than the stale timeout (3× its interval), counted
  from when the report arrived. Removing a sensor deletes its kept report.

## HA app 0.9.3 — pairing without approval, BUTTON report, sensor names, 2026-10-26

- A setting changed back while the previous change was still on its way
  (30 → 40 → 30) ended at 40: the gateway dropped the second wish because it
  equalled the old confirmed value. It is now kept and sent after the first
  transaction (also after BUSY, CONFLICT or a timeout).
- BUSY from a sensor no longer means "nothing committed": a sensor whose
  settings write landed while its read-back failed answers BUSY too. The
  gateway marks the values unknown and reads the configuration back
  (CONFIG_GET) before trying again.
- BUTTON: a sensor's report flagged BUTTON (a short press on a paired
  sensor, trigger bit 5) highlights it in the panel for 2 s and scrolls it
  into view; *Last button press* keeps naming the sensor pressed
  last. HA: gateway sensor *Last button press* (the sensor's name) and a
  per-sensor *Last button press* timestamp; the event log shows `BUTTON: <ID>`.
- Sensor names: *Sensor name* in each sensor card (up to 40 characters, kept
  by the gateway, removed with the sensor) is the HA device name.
- Panel title *RopixNet – Device management*.
- Pairing without approval (ADR-011): the open window pairs every sensor in
  pairing mode by itself (RSSI filter, capacity limit). Several at once: at
  most 4 attempts run, the others are offered at their next announcement; an
  OFFER is repeated at most every 2 s, so announcing sensors do not use up the
  gateway's output budget (ACKs and HS2 were dropped with 10 sensors). The
  panel lists the sensors paired in this window instead of the choice and
  *Pair sensor*; HA: *Pairing 2: Sensor* and *Pairing 3: Approve* are
  deleted, *Pairing 4: Close* is *Pairing 2: Close*. CLI `--manual-approval`
  keeps the per-ID approval (development).

## HA app 0.9.2 — Signal quality removed, 2026-10-24

- The sensor diagnostic **Signal quality** (Good/Fair/Weak) is gone; **Receive
  quality** (%) is the one rating, next to **Signal strength** (RSSI) and
  **Signal SNR**. At every start the gateway publishes an empty retained
  discovery config for the old entity (`<sensor>_signal`), so Home Assistant
  deletes it instead of keeping it unavailable. The panel's five bars are
  unchanged (from Receive quality). `limits.signal_quality` and its thresholds
  are removed; the diagnostic JSON no longer has the `signal` word.
- Dongle firmware and capabilities: when a dongle is attached (and at every
  panel check) the gateway logs `DONGLE: fw v1.3.0; hw 868 TCXO v0.0; caps …;
  SNR yes|no` from `AT+IDENTIFY` and `AT+CAPS`. Home Assistant gets the gateway
  diagnostic **Dongle firmware** (attributes hardware, UID, caps, SNR support);
  the panel shows `FW … · SNR ano/ne` next to the dongle port. Unknown fields
  (older firmware) are reported as unknown, never as an error.
- USB replug: **Check the dongle again** answered "The gateway is communicating…" for as
  long as a sensor was reconnecting (its handshake counted as busy), although
  the gateway had already found the dongle again. The check never takes the
  port, so it is no longer blocked: a running dongle is re-verified on its open
  port (IDENTIFY, CAPS); a dead one is marked disconnected and searched for
  again. Choosing another dongle or creating the network still waits for
  pairing. Unplugging is detected by the serial reader (status *disconnected*)
  and the same port is retried every 5 s, the radio settings applied again.
- Panel: an error that already says to try again no longer gets a second
  "You can try again".
- **Last seen** no longer fills the sensor's Activity (HA logbook): the
  timestamp entity changed at every contact. The last contact is the attribute
  `last_seen` of **Sensor connectivity** (its state changes only on offline /
  back online); the old entity is deleted in HA at startup.
- The panel speaks English and Czech: the language of Home Assistant (the HA
  frontend around the ingress panel: its page language, the user's saved
  language), else the browser; Czech for `cs`, English otherwise. All texts in
  `static/i18n.json` (the same keys in both, checked by tests); MQTT entity
  names stay English.
- Sensor form: *Reporting* has *Report at least every* (max report interval)
  and, greyed with "set by the sensor", min report interval and measurement
  interval (read-only in the TH10 descriptor; in HA they were already
  read-only diagnostic sensors); *Connection* at the bottom has *Check in
  every* with what it does (an empty POLL when nothing was sent, settings
  pickup, offline after 3× without contact).
- Pairing diagnostics: frames the gateway drops on purpose are logged, at most
  once a minute per sensor and reason: `IGNORED: <id>; key exchange from a
  sensor that is not paired here …; pair it again` (a sensor that kept a
  pairing the gateway does not have, e.g. after an interrupted handshake or a
  removal, reconnects in vain), an announcement below the pairing RSSI filter,
  while pairing is closed, or with the gateway full. Handshake replies (OFFER,
  HS2, SESSION_ACCEPT, SESSION_COMMIT) log how long after the request they
  went on air (`on air N ms after RX`, host times) to diagnose lost HS2.
- Voltage entities show two decimals (`suggested_display_precision` 2):
  3.25 V instead of "3 V". Note: the TH10 firmware sends the battery in mV
  through a 10k/10k divider placeholder (`SENSORS_BATTERY_DIVIDER_*_OHM`,
  "set the resistors of the board"), so a board without that divider reports
  twice its battery voltage (6.5 V for 3.25 V); the gateway shows what the
  sensor sends.

## HA app 0.9.1 — release build fixed, 2026-10-24

- The 0.9.0 release run stopped in its test job on Linux (images were not
  built). Three tests that start a real gateway and sensor process still gave
  the sensor `--frequency`, which the sensor has refused since the gateway
  assigns the operating radio settings; the sensor exited at once and the
  gateway waited. The tests now set the channel on the gateway only.
- The TH10 firmware header check compares with `limits.MIN_RX_WINDOW_MS` and
  is skipped where the firmware tree is not checked out (CI). No gateway
  behaviour change.

## HA app 0.9.0 — prebuilt images, official app repository, 2026-10-24

- Distribution the official way: GitHub Actions (`.github/workflows/ha-app.yml`)
  tests, builds aarch64 and amd64 images and pushes them to
  `ghcr.io/<owner>/{arch}-addon-ropixon_gateway:<version>`; Home Assistant only
  pulls them (no build on the device; a Raspberry Pi 3B could not build).
- A public app repository (`repository.yaml` + `ropixon_gateway/` metadata,
  no source) is exported by `tools/build_ha_repository.py --public` and
  updated by the workflow; the source repository may stay private.
- `build_ha_repository.py --image …` / `--local` sets or removes `image:`;
  the tag must equal `ropixon.__version__`. Release steps:
  `home-assistant/RELEASING.md`. No gateway behaviour change.

## HA app 0.8.2 — a reinstall keeps everything, 2026-10-24

- The whole gateway state (identity and keys, paired sensors with pinned keys,
  products, confirmed settings, the wizard's network) moves from `/data`
  (deleted on uninstall) to the app's public configuration folder `/config`
  (`map: addon_config`, host `/addon_configs/<repo>_ropixon_gateway`, kept on
  uninstall unless the user deletes it). After a reinstall the same gateway,
  sensors and HA entities continue; no new devices.
- One module decides (`ropixon/state_dir.py`): valid `/config` wins; else a
  verified SQLite-backup copy from `/data` (kept as a fallback for this
  version); else a new gateway. A damaged kept database stops the app with a
  Czech log message instead of creating a new identity; the option
  `start_as_new_gateway` keeps the damaged file and starts new. `Store`
  refuses to invent an identity for kept state (`--existing-state`).
- State folder 0700, database 0600 in the container; docs on where data
  lives, reinstalling, starting from scratch, and the Samba visibility of keys.

## HA app 0.8.1 — old gateways in Home Assistant, 2026-10-23

- After a reinstall the gateway has a new identity and HA kept the old
  gateway (and sensor) devices from retained discovery. After every MQTT
  connection (and on *Hledat znovu*) the gateway now collects retained
  discovery of other Ropixon gateways for 3 s (at most 4000 topics, malformed
  payloads ignored) and lists those that are not online, with their entities
  and the sensors only they had.
- Web panel **Old Home Assistant records**: *Remove…* → *Confirm removal* deletes exactly that gateway's discovery configs and the
  `ropixon/<id>/…` state topics they name (empty retained payloads), logged in
  Gateway events. Nothing automatic; our own and online gateways are never
  listed. Codes `mqtt_unavailable`, `stale_unknown`.
- Docs: why duplicates appear after a reinstall; update or restore a backup
  instead of uninstalling.

## HA app 0.8.0 — graphical installation wizard, 2026-10-23

- Current product renamed to Ropixon TH10 across firmware adapters/build
  names, gateway profiles (both descriptor revisions), HA metadata, tests and
  active product docs. Product ID 0x00010001, descriptor revisions, wire format,
  identity and MQTT unique IDs stay unchanged. The physical checkout path and
  historical evidence retain their old names; old CMake caches are adopted.
- Czech four-step ingress wizard: verified USB dongle, recommended/manual
  radio settings, separate HARD/SOFT choice, summary and explicit creation.
- Bounded serial auto-detection with IDENTIFY/HOST_V1 validation; zero/one/many
  devices, failure causes, explicit selection and stable identity persistence.
  The panel stays alive without radio; probes never transmit or configure RF.
- Readback-confirmed radio configuration and SQLite completion. Gate pairing
  in the node, panel and MQTT; preserve legacy identity/peers/profile and lock
  unsupported network edits. Incomplete installs remain incomplete on restart.
- Keep Supervisor MQTT/hardware settings separate with Czech option labels;
  preserve optional legacy bootstrap inputs. Rebuild HA context and wheelhouse.

## HA app 0.7.2 — receive quality percentage, 2026-10-23

- Add a separate per-sensor Receive quality diagnostic (0–100 %) and five-bar
  panel display, preserving RSSI, SNR and the legacy enum IDs.
- Average eight authenticated per-packet scores using captured RX SF/BW,
  normalized RSSI/SNR minimum and an explicitly documented sensitivity model.
  RSSI-only is marked as an estimate; unknown profiles remain unknown.
- Keep the last percentage between packets; age/staleness and connectivity
  are separate. Reset history on a new authenticated session or removal.
- Remove RX window from HA options/schema and ignore legacy saved choices.
  Gateway policy defaults internally to 1000 ms; TH1 minimum is 200 ms.
  Existing whole-packet airtime extension stays; Python radio runtime follows
  the same policy-floor behavior. Older TH1 firmware with a 2000 ms minimum
  requires a firmware update for a 1000 ms policy. No wire/version mechanism.
- Rebuild the HA source context, wheel and hashed dependency lock.

## HA app 0.7.1 — TH1 battery voltage, 2026-10-23

- Gateway profile TH1 descriptor revision 2, the same tables as the TH1
  firmware with battery measurement: channel 4 VOLTAGE (u32, 0.001 V = mV),
  its report Δ (1..10000 mV, factory 100 mV) and report on change (factory
  off). Revision 1 stays for sensors with older firmware.
- HA: **Battery voltage** (V, device class voltage, measurement, Sensors
  card); settings **Battery: Δ** and **Battery: on change** (sorted after the
  four buttons). Web panel: *Battery voltage*. A channel role in the product
  profile (`DeviceSpec.names`, `controller.CHANNEL_ROLES`) names it.
- A sensor whose firmware changes from revision 1 to 2 is identified again
  (DESCRIBE), new entities appear, existing unique IDs stay.
- No wire-format or database change; sensor firmware not changed here.

## HA app 0.7.0 — sensor settings in the panel, link quality, final device pages, 2026-10-22

- Web panel: **Sensors and their settings**, per paired sensor: readings
  (Temperature 1, Temperature 2, Humidity), link quality (bars, dBm), last contact,
  the settings form (*Reporting*: check-in, max/min report interval; *Changes*:
  per channel Δ and report-on-change, *Δ both temperatures*), *Apply changes* /
  *Discard changes* / *Factory values*, a Czech status line, and *Remove sensor* with a confirmation. The same controller drafts as HA; fields show the
  draft highlighted next to the confirmed value. Server-side validation with
  codes (`invalid_value`, `not_writable`, `product_unknown`, `sensor_busy`).
- Link quality: RSSI and SNR of the last authenticated uplinks of each sensor
  (8 in RAM, forged or unauthenticated frames do not count); HA diagnostics
  **Signal strength** (dBm), **Signal SNR** (dB), **Signal quality** (Good /
  Fair / Weak; with SNR: good = SNR ≥ 0 and RSSI ≥ −100, fair = SNR ≥ −5 or
  RSSI ≥ −115; RSSI only: −100 / −115 dBm; SF7), **Last seen**.
- Dongle AT: `+RX:len,hex,RSSI:x` and the new `+RX:len,hex,RSSI:x,SNR:y`
  (integer dB, decimal accepted, −40..40); SNR carried to the node, the radio
  log and HA.
- Gateway device page, final: Controls **Pairing 1: Open**, **Pairing 2:
  Sensor**, **Pairing 3: Approve**, **Pairing 4: Close**; Sensors **Pairing
  status** (attributes `remaining_s`, `progress`); Configuration **Pairing
  duration**; Diagnostic **Gateway events**. All other gateway entities are
  deleted from HA (generic legacy list, including the 0.6.1 button
  *Refresh devices in HA* = `rediscover`); their command topics still work.
- Sensor device page: the configuration card sorts (ICU, cs and en) as
  **1 Apply changes**, **2 Discard changes**, **3 Factory defaults**,
  **4 Remove sensor**, then per channel (*Humidity: on change*, *Humidity: Δ*,
  *Temp 1: …*, *Temps: Δ (both)*), then *Timing: check-in*, *Timing: max
  report*. Device info: model (TH1) and hardware (product, revision).
- One version: `ropixon.__version__` (0.7.0) is the package version
  (pyproject dynamic), the HA device `sw_version`, and is written by the build
  into `config.yaml` and the Dockerfile. Unique IDs unchanged.
- No wire-format, sensor firmware or database change.

## HA app 0.6.9 — clearer gateway device page, 2026-10-22

- Controls: the pairing steps in order, **1 Open pairing**, **2 Found sensor**,
  **3 Approve sensor**, **4 Close pairing**. Sensors: **Pairing status** only
  (time left and progress in its attributes). Configuration: **Pairing
  window** only.
- Diagnostic: *Gateway events*; disabled by default for new installations:
  *Pairing progress*, *Pairing time left*, *Republish to HA* (was *Republish
  to Home Assistant*), *Approve sensor by ID*. All stay in the web panel.
- Removal: **Remove sensor** (configuration) on each sensor's device page,
  topic `sensor/<id>/command/remove` = PRESS, the same controller path,
  checks and codes. The gateway-level *Sensor to remove* select and *Remove
  selected sensor* button are deleted from HA (their commands still work).
- Unique IDs unchanged. No wire-format, sensor firmware or database change.

## HA app 0.6.8 — channel names without channel numbers, 2026-10-22

- A channel is named by its quantity, numbered only when a sensor has more of
  it, by its order (TH1: Temperature 1, Temperature 2, Humidity; Δ Humidity,
  On change: Humidity). Internal channel IDs remain in topics, unique IDs,
  the protocol and diagnostics; entity IDs are unchanged.
- One rule, `controller.channel_label`. A product profile may name channels
  (`DeviceSpec.names`, e.g. probe vs. air temperature); TH1 has none yet.
- No wire-format, sensor firmware or database change.

## HA app 0.6.7 — sensor settings card, 2026-10-22

- Short entity names that fit an HA row (≤ 21 characters): **Apply changes**
  (was *Send changes to sensor*; sorts first on the configuration card),
  **Check-in interval**, **Δ Temperature 1/2**, **Δ Temperature (both)** (the
  group entity: both temperatures at once), **Δ Humidity 3**, **On change:
  Temp 1/Temp 2/Humidity 3**, **Max/Min report interval**. Entity IDs
  (unique_id) unchanged.
- New button **Load factory defaults**: drafts the product's factory values
  (as after a new pairing) and the gateway's default check-in, only where
  they differ; the user reviews them and presses Apply. Unknown product:
  refused with code `no_factory_defaults`. **Discard changes** only drops
  unsent edits.
- No wire-format, sensor firmware or database change.

## HA app 0.6.6 — panel "LoRa gateway management", 2026-10-22

- New HA sidebar panel **LoRa gateway** (HA Ingress, port 8099) using the design: add a sensor (saved pairing duration, start/stop, status by protocol
  phase with the real time left, found sensor with RSSI, explicit pairing,
  manual ID), two-step removal with the full ID, *Refresh devices in Home Assistant*, diagnostics and the last gateway events. Czech, light/dark,
  phone layout.
- Each panel action has a request ID and waits for the gateway's own result
  and the state after it; one action at a time; disabled while offline. Only
  the ingress proxy and localhost may connect.
- MQTT: `command/republish` and the **Republish to Home Assistant** button
  send every discovery topic again. CLI `--panel-port`.
- Review fixes: a pairing attempt per sensor (approved → handshake → paired /
  failed with a code, attempt ID) in the status, so a result shows also while
  the window stays open and for several sensors; a repeated approval of a
  running attempt is idempotent (no new OFFER); the panel blocks pairing of a
  sensor with a running attempt, *End pairing* stays available; request
  timeouts (6 s state, 10 s command; *Result unverified* + immediate refresh),
  one poll at a time, offline after 10 s without a fresh state; rejected
  commands carry stable codes shown in Czech, the English text stays in
  diagnostics and in the MQTT command result.
- Internal restructuring, no behaviour change: `controller.py`
  (GatewayController: commands, selections, drafts, results, events, one
  state snapshot for MQTT and the panel), `mqtt.py` only renders HA
  discovery, `errors.py` (codes set where a request is refused, no text
  matching), panel page in `static/` (index.html, panel.css, panel.js; in the
  wheel). The panel uses `pair_new`: the gateway refuses an already paired ID.
- No wire-format, sensor firmware or database change.

## HA app 0.6.5 — pairing and settings from the dashboard, 2026-10-21

- Pairing in HA: **Open pairing** / **Close pairing**, **Pairing window**
  (60/120/300 s or *Until closed*, the last only when the sensors operate on
  the pairing channel), **Found sensor** select (one found ID is selected
  automatically), **Approve selected sensor**. **Pairing status**, **Pairing
  progress** (%) and **Pairing time left** follow searching → found →
  approved → key exchange → paired / failed (with the reason), result shown
  for 5 minutes. **Gateway events**: the last 15 log lines without
  REPORT/POLL. Topics `ropixon/<gateway>/pairing` and `.../events`.
- Removal by a **Sensor to remove** select and **Remove selected sensor**;
  the old *Remove sensor (ID)* text entity is deleted from HA.
- Sensor settings are a form: a field only drafts the value (validated at
  once); **Send changes to sensor** sends every device-model change in ONE
  transaction plus the check-in interval, **Discard changes** drops them.
  **Settings status** per sensor: not sent / waiting for the next contact /
  confirmed / rejected / no answer. Fields show draft, else requested, else
  confirmed values (`confirmed`, `not_sent`, `waiting` in `device_config`).
- All entity names are English; existing entities keep their entity IDs.
- Gateway: `open_join(None)` (until closed), `close_join()`,
  `pairing_status()`, logs `PAIRING FAILED` on a rejected HS3 or an expired
  handshake; console `join open|close`. No wire-format, sensor firmware or
  database change.

## HA app 0.6.3 — configurable pairing RSSI filter, 2026-10-21

- Build fix: ship a prebuilt gateway wheel and binary dependencies for Linux
  ARM64/amd64, Python 3.12. Install from a hash-pinned local wheelhouse without
  PyPI access or an isolated setuptools build on Raspberry. The base image
  still needs to be cached or downloadable. Generate release wheels on the
  workstation using tools/build_ha_repository.py.


- App option `pairing_rssi_filter` defaults to true (existing -45 dBm threshold).
  Set false and restart to show valid pairing announcements at any RSSI.
- CLI: `--no-pairing-rssi-filter`. The join log reports the effective policy.
- An open join window and explicit ID approval remain required. No wire-format,
  sensor firmware, identity, session or database format change.

## HA app 0.6.2 — upgrade package, 2026-10-21

- Current gateway/device-model sources, multiple sensors, configuration readback
  and pairing/radio-settings fixes.
- Preserve bandwidth options saved by app 0.6.1: 7/8/9 are interpreted as
  125/250/500 kHz at the HA entrypoint. New configuration uses kHz.
- Keep the same app slug and /data/radio-state path. Database format 3 migrates
  to format 4; back up app data before upgrading. Do not uninstall the app.
- A local build/update is required. Hardware acceptance is separate from the
  software checks; this is not a claim of a stable protocol release.

## Device model (profile 3)

- Snapshot REPORT of every channel; the gateway learns the product from
  DESCRIBE page 0 and takes its settings from its own product profile
  (`device.PRODUCTS`, TH1). An unknown product shows its values and has no
  settings. The gateway reads the configuration back and keeps desired / pending / confirmed / unknown
  values. One atomic CONFIG_SET per change, confirmed only by a matching status
  or a readback; a lost status is never resent blindly.
- HA: one entity per channel (temperature, humidity, ...), maximum interval,
  delta and change reporting per channel, "delta of all temperatures" as one
  transaction; fixed parameters (sampling, minimum spacing) as diagnostics.
  Only confirmed values are shown.
- Reference sensor: `SensorNode(device=DeviceModel())`.

## Several sensors per gateway

- The gateway serves up to 256 sensors (`--max-sensors` / HA `max_sensors`,
  1..256): one session, request queue and reported state per sensor; DATA
  routed by the sensor's short address; at most 4 handshakes in progress.
- The limit counts paired and approved sensors; when full, candidates are not
  listed and approval is rejected.
- `remove <ID>` / HA **Remove sensor (ID)** forgets a sensor; its MQTT topics
  are cleared and HA removes the device. The last 64 removals are stored and
  cleared again after every broker connection (also after a restart).
- A handshake in progress holds its place in the limit, also after its
  approval expired.
- Console commands for a sensor take its ID: `period <ID> 15000`; new
  `sensors`. HA checkin topic per sensor.
- Channel load over the last hour (HA diagnostics), warning above 10 %.
- Reported sensor state cached in memory: an HA snapshot reads no database
  per sensor (256 sensors: ~15 ms on a PC).
- The join window stays open after a pairing when the operating settings are
  the pairing channel (further sensors may join).
- Database format 4: per-sensor state; a format 3 database (one sensor) is
  migrated automatically.
- Reference sensor ignores other sensors' ACCEPT / COMMIT (shared channel).
- Pairing again: a paired sensor in pairing mode (its button) is a candidate
  again, also after `remove`, and may move to another gateway; the candidate of
  a completed pairing is used up. Reference sensor: `start(pair_again=True)`.
- A sensor announcing with an installation code is reported (log
  `CANDIDATE NEEDS INSTALL CODE`, HA `code_required`) instead of being
  silently ignored; pairing stays button-only.

<a id="radiova-nastaveni-parovaci-kanal-a-provozni-nastaveni"></a>
## RESULT UNSUPPORTED

- A sensor that does not recognize a command responds RESULT UNSUPPORTED;
  gateway immediately ends it as UNSUPPORTED and HA reports it (previously
  it waited for expiry).

## Radio settings: pairing channel and operating settings

- Pairing always uses the fixed regional pairing channel (EU868: 869.525 MHz,
  SF7, 125 kHz, CR 4/5). Gateway assigns operating settings in SESSION_ACCEPT;
  the sensor confirms them in READY and switches after COMMIT.
- New `region` and `bw` options, with BW in kHz (previously AT index); the
  service retunes radio by state. `join` with a paired sensor opens a recovery window.
- ACCEPT/READY format change: older sensors and gateways cannot pair together.

<a id="jedno-announce-pro-kandidata-parovani"></a>
## One ANNOUNCE for a pairing candidate

- Gateway offers a candidate for approval after one valid nearby ANNOUNCE.
- Removed the three-messages-in-5-s condition, incompatible with TH1's 5 s listening.
- Candidate remains valid in the current open join window; a new window clears
  old candidates. RSSI checks and explicit ID approval remain.

<a id="rozpracovane-zmeny"></a>
# Work in progress

- CONFIG_SET/STATUS through REPORT and POLL; no need to wait for a first measurement.
- HA numeric checkin entity in seconds, displaying the actual confirmed value.
- Sensor availability through POLL separated from temperature age.
- READY supplies actual revision and recovery of pending configuration results.
- Expired configuration requests are finalized regularly in tick() (EXPIRED /
  UNKNOWN), reported in HA, and no longer block further settings while the sensor sleeps.

# 0.6.0

- Optional MQTT bridge, HA Discovery for temperature/voltage, device/diagnostics.
- MQTT pairing button and explicit sensor ID approval, candidate freshness,
  bounded command queue, ignored retained replay and expired requests.
- MQTT sockets in a separate thread; outages do not block radio, latest-state coalescing.
- Gateway LWT availability, sensor timeout, stable IDs, HA/broker restart recovery.
- Separate HA OS app build context for aarch64/amd64, PC and Pi guide.
- Existing wire-v3, dongle firmware, and pairing databases remain compatible.
- Actual local MQTT broker in tests; first HA OS installation still awaits testing.
- Full automated suite: 169 tests passed; original protocol tests preserved.

# 0.5.0

- Independent gateway and sensor processes: each owns only one port and local DB.
- Asynchronous radio pairing with explicit ID approval, grants, pins, deadlines,
  exact-byte retries, and bounded cache. Proximity mode, no PSK downgrade.
- Independent measurement generation, SOFT/HARD contacts, ACK and command-result verification.
- Gateway CLI console, OS DB locks, Windows launchers, Linux/Raspberry Pi OS guide.
- Preserved wire-v3 and existing proximity bench database compatibility.
- 22 new tests including two OS processes with their own UART ports/DBs and
  restart; 156 passed in total. New processes not yet physically verified on dongles.

# 0.4.2

- TX_DONE accepts exact +TX:DONE and TX:DONE lines. Actual dongle firmware
  v1.2.0 sent the second form, as established by raw UART log.
- Windows log confirms complete configuration of both dongles and one
  50-byte COM7 → COM5 RF transfer, RSSI −35 dBm. Full pairing not yet verified.
- Regression for both TX_DONE formats; 134 tests passed.

# 0.4.1

- Fixed AT parameter names: PREAMBLE_SIZE and IQ_INV for TX/RX according to
  actual firmware table. 0.4.0 sent nonexistent PREAMBLE/IQ commands.
- Command rejection now prints both command name and dongle response.
- Emulator rejects unknown queries/setters; added wrong-name regressions.
- 132 tests passed; the user's physical test has not yet passed radio configuration.

# 0.4.0

- Serial AT driver for two Ropixon dongles: RX during AT responses, verified
  RF settings, separate TX_DONE, bounded queues, stop on ambiguity.
- CLI ports/identify/bench; one PC, two USB ports, Python sensor, actual RF.
- Pairing, encrypted measurements, ACK, and stored-identity recovery over new transport.
- Real clocks, dongle RSSI, conservative firmware-airtime-based timeouts.
- Required HOST_V1 AT contract documented; the user maintains firmware changes.
- 23 new tests; 128 passed in total. No physical hardware test, no wire-v3 change.

# 0.3.0

- Wire-v3: DATA 14 B, application base 5 B, REPORT with 8-byte data totals 43 B, ACK 41 B.
- Separate long bootstrap and authenticated short address/context assignment.
- u32 counter, full header authentication, preserved 16-byte ChaChaPoly tag.
- Transactions only for operation types; removed redundant lengths and per-frame RX times.
- Policy format 2 with immediate RX and persistent timeout; CONFIG tag 000A / rxwindow.
- Bounded ingress, discovery, handshake attempts, responses, and replay ACK.
- Separate decoders; transport accepts unknown application profiles/schemas.
- Atomic migration of old policy/journal, preserving deduplication and identities.
- Compact-wire, malicious-input, alias, timeout, and migration tests.

Radio v1/v2 frames are rejected. Physical dongle, HA, mesh, and MCU firmware
remain unimplemented. Historical specification 0.2 is retained in docs/history.

# 0.2.0

- Separate MID network profile and SOFT/HARD sensor delivery policy.
- Reserved FAST/EXTREME, not yet enabled.
- Acknowledged CONFIG_SET for mode and check contacts.
- Durable bootstrap approval, gateway pin before HS3, and SESSION_COMMIT.
- Step, discovery, overall-attempt limits, backoff, and bounded recovery.
- Recovery after message loss and restart between persistence checkpoints.
- Virtual tick scheduler, important reporting, availability checks.
- Wire version 02 and new Noise prologue. Wire-v1 rejected.
- Updated specification 0.2 and 75 automated tests.

Fully paired 0.1 simulation databases load with identity/configuration preserved.
Original incomplete pairing without recovery approval is not automatically
resumed for security reasons. The protocol remains in development; physical
radio and HA are not connected.
