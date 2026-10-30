<a id="ropixon-06-home-assistant-bez-preinstalace"></a>
# Ropixon 0.8 — Home Assistant gateway and installation wizard

This is the first experimental MQTT integration. It does not replace your
existing Home Assistant installation or configuration. Flow: the sensor
communicates by radio with a USB dongle; the Python gateway performs pairing,
verification, and decryption; a decoder converts content to values; the MQTT
broker passes values to the existing HA MQTT integration.

The gateway serves up to **256 sensors** (option `max_sensors`, 1..256); each
appears in HA as its own device under the gateway. Temperature and voltage come
from the reference Python sensor; a physical sensor is the next step. RF
firmware is not changed here. Neither HACS nor a custom `custom_components`
module is required. Operation can remain local.

<a id="nejrychlejsi-test-se-stavajicim-raspberry-pi"></a>
## Quickest test with an existing Raspberry Pi

For now leave both dongles in the PC. The PC gateway can connect over your
home network to an MQTT broker on Raspberry Pi. You need not move the dongle
or pairing state immediately.

1. In HA, run an MQTT broker (for example the **Mosquitto broker** app/add-on)
   and connect the **MQTT** integration to it. If already in use, no second
   broker is needed. Give the gateway a valid MQTT account. For official
   Mosquitto, create a separate local non-admin HA user; do not name it
   `homeassistant` or `addons`.
2. Extract project 0.6 into a new folder. To retain existing pairing, copy
   the entire `radio-state` folder from the previous version while both
   processes are stopped. Run `INSTALL_HA.cmd` or the command below.
   Existing radio tests still work.
3. Copy `examples/mqtt.example.json` to project-root `mqtt.json`. Fill in
   Raspberry Pi IP and MQTT credentials. Do not use hostname `core-mosquitto`
   from a PC; that works between HA OS apps. Do not share the password file.
4. In PowerShell at the project root, run:

```powershell
.\.venv\Scripts\python.exe -m pip install -e ".[ha]"
.\.venv\Scripts\python.exe -u -m ropixon.service gateway --port COM5 --region EU868 --state .\radio-state --mqtt-config .\mqtt.json
```

5. Start the sensor in a second window as before:

```powershell
.\.venv\Scripts\python.exe -u -m ropixon.service sensor --port COM7 --region EU868 --state .\radio-state --interval-ms 5000
```

For an unpaired sensor add `--pair` and follow the procedure below. Verify
actual COM ports. Expect `RADIO READY`, then `MQTT connected`, and `REPORT`
after receiving the sensor. MQTT may be temporarily unavailable while radio
continues. Do not change known-working radio parameters for the first test.

<a id="instalace-z-repozitare"></a>
## Installing from the app repository (app 0.9.0+, recommended)

Settings → Apps → Store → ⋮ → Repositories → add the Ropixon app repository
URL (see `home-assistant/RELEASING.md`) → **Ropixon LoRa Gateway** → Install.
Home Assistant pulls a prebuilt image (about 70 MB); nothing is built on the
device, so a Raspberry Pi 3B (1 GB RAM) is enough. Updates arrive the same way
(Check for updates → Update). Requires 64-bit Home Assistant OS (aarch64) or a
PC (amd64). The steps below (copying the folder over Samba) remain for local
development: Home Assistant then builds the image on the device, which is slow
and on a Pi 3B may not finish.

<a id="instalace-primo-do-home-assistant-os"></a>
## Installing directly on Home Assistant OS

For HA OS with apps/add-ons (64-bit/aarch64 Raspberry Pi or amd64). For HA
Container, use the same Python service or a separate container alongside HA;
Home Assistant Container has no app store. Reinstallation is not needed there either.

1. Through Samba share or SSH, copy the **entire**
   `home-assistant/ha_repository/ropixon_gateway` folder to `/addons/ropixon_gateway`
   on HA OS. A prepared release bundle includes `wheelhouse` with the gateway
   and all Python dependencies (Linux ARM64/amd64, Python 3.12). When using a
   source checkout, first run `python gateway/Ropixon_LoRa_prototyp_v0.6.0/tools/build_ha_repository.py`
   on the workstation. Its Python environment needs pip, setuptools>=68 and
   wheel; package downloads use official PyPI. Copy the resulting entire app
   folder, including `wheelhouse`. The container installs only hash-pinned
   local wheels; it does not download Python/build dependencies. The base
   Python image must still be cached or reachable.
2. Refresh the app/add-on store list (three-dot menu → check for updates).
   Open **Ropixon LoRa Gateway** under local apps and install. The first build
   downloads a Python image and libraries and may take time.
3. Leave `serial_port: auto`. Connect a compatible Ropixon USB LoRa dongle to
   the HA host; the panel also opens when no dongle is attached. No COM port
   guessing is needed. `baud` normally stays 230400.
4. Set the MQTT broker, account and TLS options in the HA application's
   Configuration tab. For local Mosquitto use `core-mosquitto:1883`. Changes to
   these Supervisor-owned options require restarting the app. The panel does
   not write HA credentials or pretend to save Supervisor configuration.
5. Start the app and open **RopixNet** in the sidebar. The installation
   wizard has four steps: **Connection → Radio network → Acknowledgement → Finish**.
   Leave the recommended EU868 profile, or choose frequency/SF/BW/CR under
   **Custom settings**. HARD confirms each measurement; SOFT uses periodic
   confirmed contacts and does not confirm every measurement. This choice is
   independent of the radio parameters.
6. Check the summary and press **Create network and save**. The backend validates
   the complete radio channel and power against EU868, configures the dongle
   with readback, then commits the network to the kept state (`/config/radio-state/gateway.db`, see below).
   Only a confirmed backend result enables **Add sensor**. Pairing remains
   blocked server-side (panel, MQTT and node API) before network completion or
   while the radio is unavailable. No pairing starts automatically.

### Dongle detection and recovery

Detection enumerates serial ports with pySerial on Linux/Windows, deduplicates
Linux `/dev/serial/by-id` aliases, and probes at most eight ports for a maximum
five-second scan budget (each AT command has a 300 ms timeout). A probe only
opens the port, issues `AT+IDENTIFY` and `AT+CAPS`, and closes it. It sends no
radio configuration, TX, reset or flash command. DTR/RTS start disabled; POSIX
exclusive mode avoids sharing a port with another owner. USB VID/PID alone is
never accepted as proof of compatibility.

One compatible result is selected automatically after a complete scan. More
results require an explicit choice. A truncated scan never chooses the first
result arbitrarily. **Znovu vyhledat dongle** repeats the bounded scan; an
advanced dropdown permits a targeted test of an enumerated port. No response,
incompatible interface/firmware, unavailable/busy port and missing hardware
have separate explanations. Serial identification verifies the device
and host interface, not antenna operation or an RF link. Standard pairing is
the subsequent practical radio exchange; no separate RF test is provided.

The selected device's host-reported UID and stable path are persisted. Runtime
and probes share one owner thread and never open the same port concurrently.
A live runtime is reused by the scan action. After disconnect, pairing closes,
pending radio responses are discarded and the panel stays available. Retry of
the selected path is paced at five seconds and rechecks UID. If the path moves
(for example a Windows COM change), **Znovu vyhledat** finds the saved UID again.
A replacement or ambiguous result requires a deliberate choice. The recovery
card remains available even after the network has been created.

### Persistence and existing installations

The network profile, region, power and delivery default are authoritative in
the gateway database after completion, not in browser storage. An unfinished
installation stays unfinished after restart. Existing databases with no wizard
marker are adopted without resetting identity, keys, peer pins or history.
An authenticated sensor's saved operating PHY takes precedence over stale HA
defaults. Conflicting stored sensor PHYs fail closed instead of guessing.
Legacy radio options remain optional migration inputs in the HA schema; new
installs do not ask for them. Changing them after network creation does not
retune the saved network. The supported panel intentionally has no edit/reset
network action or coordinated over-the-air retuning.

RX policy remains internal at 1000 ms, with no user field. TH10 minimum is
200 ms; old firmware with a 2000 ms minimum needs an update for a 1000 ms policy.
Whole-packet airtime extension remains. A provisioned sensor keeps its existing
saved policy on reconnect. Startup, MQTT options, app backups and hardware
access stay managed by HA; the panel neither restarts HA nor changes credentials.

**Moving an already paired gateway from PC:** a new empty Pi database creates
a different identity that the old sensor correctly rejects. First verify HA
using the PC gateway if desired. Later migration must transfer the original
`gateway.db` to the stopped app's data directory. Do not automatically erase
sensor keys. Automatic app database import is not part of this version.
Do not run two copies of the same identity.

<a id="panel-sprava-lora-brany"></a>
## Panel "LoRa gateway management" (app 0.6.6+)

**Language (app 0.9.2+).** The panel is in English or Czech, after the
language of Home Assistant: the HA frontend around the panel (its page
language, the user's saved language), else the browser; Czech for `cs`,
English for everything else. All texts are in `ropixon/static/i18n.json`
(`en`, `cs`, the same keys). The names below use the English interface. Czech translations remain
available; MQTT entity names stay English (MQTT discovery has no translations).

The app version (Device info: *Firmware* of the gateway) is one number,
`ropixon.__version__`; the build writes it into `config.yaml` and the Dockerfile.

The primary place for pairing, removing and configuring a sensor is
**RopixNet – Device management**. Open **RopixNet** in the HA sidebar (HA Ingress;
if missing, enable *Show in sidebar* on the app page, or use *Open web UI*).
The gateway and sensor device pages in HA also link to this panel through
their configuration-page link (`configuration_url`). The app resolves its full
installation ID using Supervisor `/addons/self/info`; it does not hard-code a
repository prefix or expose a token in the link.

Use HA device pages for measurements, history, areas, dashboards and automations.
The duplicate MQTT configuration entities are hidden by default and categorized
as configuration, while retaining their IDs, command topics and enabled state
for existing automations and advanced use. Existing user visibility choices in
HA remain authoritative. If the app cannot resolve its own Supervisor ID, it
leaves those controls visible rather than advertising an invalid link.

The panel's **← Back to Home Assistant** link returns to the preceding HA page
when HA supplies a valid frontend history entry. A direct ingress visit returns
to **Settings → Devices & services → Devices**. Local standalone panel runs
without HA do not display a misleading HA return link.

Choose **Add sensor**, start pairing, then follow step 2 for one sensor:
hold its button about 3 seconds until the LED flashes rapidly, then release it.
Step 2 is shown only while pairing is active. Newly paired sensors stay listed
after it ends. Automatic pairing displays the remaining time and protocol
status without an artificial percentage. Short-press identification and naming
are explained with the paired sensor cards. The panel also contains removal
with confirmation, integration maintenance, gateway diagnostics and sensor
settings. On a phone the cards stack; light and dark mode follow the browser.

The panel talks to the gateway process directly, not through MQTT: each action
carries its own request ID and the page shows the gateway's result for that
action (*Refused: …* with the reason, or success) together with the state
after it. One action runs at a time; a pairing is followed per sensor until
it is paired or failed (also while the window stays open), and a sensor with
a running pairing cannot be approved twice. Without a fresh gateway state for
10 s the page shows *Gateway unreachable* and disables all controls; a command
without an answer in 10 s shows *Result unverified* and reloads the state.
Rejections are worded in the panel language from stable codes; the gateway's English text
is under *Last gateway message*. A typed ID is sent only by *Pair by ID*
(normalized to lower case; an invalid or already paired ID is refused by the
page). Only the Supervisor's ingress proxy may connect (HA signs the user in);
port 8099 is not published on the network. The MQTT entities below stay for
automations and other dashboards; their names stay English.

Below, **Sensors and their settings** lists every paired sensor: readings
(Temperature 1, Temperature 2, Humidity), the link quality (bars and dBm), last contact,
and its settings form: *Reporting* (*Report at least every*: the sensor sends its values at least this
often even if nothing changed; then the greyed values the sensor fixes, min
report interval and measurement interval, "set by the sensor"), *Changes* (per channel its Δ threshold and the
report-on-change switch; *Δ both temperatures* for both) and, at the bottom,
*Connection* (*Check in every*: the longest time without a confirmed contact; with nothing to send the
sensor sends an empty POLL, picks up new settings, and is shown offline after
3× this interval without contact; longer saves battery, changes apply later).
Each card starts with **Sensor name**: a name of up to 40 characters (e.g.
*Cellar*), kept by the gateway and used as the HA device name; empty shows the
sensor ID. A short press of the button on a paired sensor sends a report
flagged BUTTON: its card is highlighted for 2 s with *Button pressed* and
scrolled into view, and *Last button press* above the list keeps
naming the sensor pressed last (*Show sensor* jumps to it). This is how to
tell which sensor is which after pairing several, and name them. In HA: the
gateway's **Last button press** (state = the sensor's name, attributes `id`,
`at`) and each sensor's **Last button press** (timestamp), both retained; a
gateway restart keeps the last ones. In HA the fixed values are read-only diagnostic sensors (*Sampling interval*,
*Min report interval*); the editable ones are *Timing: max report* and
*Timing: check-in*. A changed field is highlighted as a draft
with the value confirmed by the sensor beside it; *Apply changes* sends,
*Discard changes* drops, *Factory values* drafts the factory values. The drafts
are the same as in HA. *Remove sensor* at the bottom asks for a confirmation.

For a PC run: `--panel-port 8099` serves the same page on
`http://127.0.0.1:8099/` (localhost only).

<a id="stare-brany"></a>
## Where the gateway keeps its data; reinstalling (app 0.8.2+)

The whole state of the gateway is one database, `radio-state/gateway.db` in the
app's **public configuration folder**: `/config` inside the app, on the host
`/addon_configs/<repository>_ropixon_gateway/` (also visible over the Samba
share *addon_configs*). It holds the gateway identity and keys, the paired
sensors with their pinned keys, their products and confirmed settings, their
names, the last report of each sensor (one row per sensor, replaced by the
next report; app 0.9.4+) and the network from the installation wizard. After
a restart of the app the last values stay shown in HA and in the panel; a
sensor is shown offline only when its last report is older than 3× its
reporting interval, counted from when it arrived, not from the restart. The
history itself is kept by Home Assistant (recorder). The HA device and every entity's unique
ID start with the gateway identity.

- **Update:** nothing changes.
- **Uninstall and install again:** HA deletes the app's private `/data`, but
  keeps the public configuration folder unless you tick the option to delete
  the app's data too (Supervisor uninstall `remove_config`, default off). The
  new installation finds the database and continues as before: the same
  gateway, the same sensors and entities, no wizard, no new devices in HA.
- **Update from 0.8.1 or older:** at the first start the database is copied
  from `/data/radio-state` to `/config/radio-state` (SQLite backup, verified,
  the identity must match); the old copy stays in `/data` for this version. If
  both exist, `/config` wins. The app log says which was used (`STATE: config`,
  `migrated` or `fresh`).
- **Damaged database:** the app does not start and does not create a new
  gateway (all sensors would lose it); the log explains it in Czech. Restore
  the folder from an HA backup, or set **Start as a new gateway**
  (`start_as_new_gateway`) in the app configuration: the damaged file is kept
  next to it as `gateway.db.damaged-<time>` and a new gateway starts (all
  sensors must be paired again).
- **Really start from scratch:** uninstall with deleting the app's data, or
  delete `/addon_configs/<repository>_ropixon_gateway/radio-state` while the
  app is stopped. Then remove the old gateway from HA with the panel's cleanup
  below.

The database holds private keys: inside the container the folder is 0700 and
the file 0600, but anyone with access to the Samba share *addon_configs* (or
an HA backup) can read them. Requires an HA Supervisor with the
`addon_config` folder (2023.11 or newer).

### Old gateway devices in Home Assistant

Before 0.8.2 an uninstall deleted the identity: the next installation created a
new gateway, so HA showed a new *Ropixon Gateway* (and sensors paired again got
new entities) while the old gateway's discovery stayed retained on the MQTT
broker. To clean up: after every MQTT connection the gateway looks for retained
discovery of *other* Ropixon gateways whose availability is not `online`. The
web panel lists them under **Old Home Assistant records** (ID, number of
entities, state, the sensors only that gateway had). **Remove…** →
**Confirm removal** publishes empty retained payloads to exactly that
gateway's discovery configs and the `ropixon/<id>/…` state topics they name, so
HA deletes its entities and devices (possibly after an HA restart). Nothing is
deleted automatically, the current gateway and online gateways are never
listed, and a removal is logged in *Gateway events*. The scan reads at most
4000 retained topics for 3 s; malformed or foreign payloads are ignored.

<a id="parovani-z-ha"></a>
## Pairing from HA (MQTT entities)

Find **Ropixon Gateway** under **Settings → Devices & services → MQTT**. HA
cannot group entities inside a device card, so each name starts with its group,
and HA sorts by name (checked with Czech and English collation). App 0.7.0+
publishes these gateway entities (the table shows their categories with the management link available):

| Card | Entity | What it does |
|---|---|---|
| Configuration (advanced, hidden) | **Pairing 1: Open** | opens the pairing window for the saved duration |
| Configuration (advanced, hidden) | **Pairing 2: Close** | closes the window early (a timed window also closes by itself) |
| Diagnostic | **Pairing status** | searching / found / approved / exchanging keys / paired / failed; attributes `remaining_s` (time left, none for *Until closed*), `progress` (20 → 100 %), `found`, `rssi`, `error`, `window_paired` (the sensors paired in this window) |
| Diagnostic | **Last button press** | the sensor whose button was pressed last (its name, else ID); attributes `id`, `at` |
| Configuration (advanced, hidden) | **Pairing duration** | `60 s`, `120 s`, `300 s` or `Until closed` (only when the sensors operate on the pairing channel) |
| Diagnostic | **Gateway events** | the latest gateway log line; attribute `events`: the last 15 lines |
| Diagnostic | **Dongle firmware** | firmware of the attached USB dongle (e.g. `v1.3.0`, from `AT+IDENTIFY`); attributes `hardware`, `uid`, `caps` (`AT+CAPS` tokens), `snr` (true when the firmware reports packet SNR, token `RX_SNR`); unknown while no dongle is attached (app 0.9.2+) |

The window pairs every sensor in pairing mode by itself (ADR-011), so the
choice and approval entities of earlier versions (*Pairing 2: Sensor*,
*Pairing 3: Approve*) are deleted; *Pairing 4: Close* is now *Pairing 2:
Close* (same entity ID). A gateway started with `--manual-approval` (CLI,
development) still publishes them. Earlier gateway entities (*Approve sensor by ID*, *Republish to HA*, *Pairing
progress*, *Pairing time left*, *Sensors to pair*, *Last command result*,
*Pairing window state*, *Paired sensors*, *Channel load*, the removal select and
button, and the 0.6.1 button *Refresh devices in HA*) are deleted from HA by an
empty retained discovery config at every connection. Their MQTT command topics
still work for automations, the values are in the topic `ropixon/<gateway>/gateway`,
and the web panel shows all of them. Entity IDs (unique IDs) never change, so
cards and automations keep working; HA keeps the entity ID it created first.
HA may apply a changed name or card (entity category) of an existing entity
only after a restart of Home Assistant or a reload of the MQTT integration.

Advanced MQTT controls (the primary user flow is the RopixNet panel above):

1. Optionally choose **Pairing duration**. `Until closed` is refused with
   *Pairing cannot stay open: the sensors operate on another channel* when the
   operating settings differ from the pairing channel (an open window holds the
   radio on the pairing channel).
2. Press **Pairing 1: Open**. **Pairing status** shows *Searching for sensors*.
3. Put the sensor into pairing mode (TH10: hold the button 3 s; the reference
   sensor: start it with `--pair`) near the gateway. One valid ANNOUNCE in the
   open window is sufficient (RSSI at least −45 dBm when `pairing_rssi_filter`
   is enabled; see below); the gateway pairs it without approval.
4. The status follows *Approved &lt;ID&gt;: waiting for the sensor* →
   *Exchanging keys with &lt;ID&gt;* → *Paired &lt;ID&gt;*, or *Failed &lt;ID&gt;:
   &lt;reason&gt;*. Put more sensors into pairing mode while the window is
   open: they pair one after another (at most 4 at once; the others are
   offered at their next announcement, every 5 s on TH10). The panel lists
   them under *Newly paired*.
5. Press **Pairing 2: Close** when done (or let the time run out). A window
   with operating settings other than the pairing channel ends automatically
   when a sensor completes pairing (one radio: the new sensor's reports come
   on its operating channel); pair several at once with the recommended
   settings, where both are the same channel.
6. Which sensor is which: a short press of the button on a paired sensor
   highlights it in the panel; name it there (*Sensor name*).

Ready-to-paste dashboard card *Pairing* (Dashboard → Edit → Add card → Manual).
The entity IDs are those HA creates for the names above; an installation from
before 0.7.0 keeps its older IDs (check them in Settings → Entities):

```yaml
type: vertical-stack
cards:
  - type: entities
    title: Pairing
    entities:
      - entity: sensor.ropixon_gateway_pairing_status
        name: Status
      - type: attribute
        entity: sensor.ropixon_gateway_pairing_status
        attribute: remaining_s
        name: Time left
        suffix: s
      - type: attribute
        entity: sensor.ropixon_gateway_pairing_status
        attribute: progress
        name: Progress
        suffix: "%"
      - type: section
        label: Steps
      - entity: button.ropixon_gateway_pairing_1_open
        name: 1 Open pairing
      - entity: button.ropixon_gateway_pairing_2_close   # pairing_4_close on an older installation
        name: 2 Close pairing
      - entity: sensor.ropixon_gateway_last_button_press
        name: Last button press
      - type: section
        label: Settings
      - entity: select.ropixon_gateway_pairing_duration
        name: Pairing duration
  - type: markdown
    title: Gateway events
    content: >-
      {% for line in state_attr('sensor.ropixon_gateway_gateway_events', 'events') or [] %}
      - {{ line }}
      {% endfor %}
```

Pairing is by button in the open window, without approval (ADR-011; TOFU:
the window, the RSSI filter and the capacity limit decide, RSSI does not prove
a sensor's identity). A sensor manufactured with an
installation code (XXpsk0) cannot pair with this gateway yet: it is not a
candidate; the log shows `CANDIDATE NEEDS INSTALL CODE: <ID>` and the gateway
state lists it under `code_required` (the gateway state topic and the web panel).
The protocol and the sensor firmware already support installation codes;
only entering the code in the gateway is missing.

Pairing checks the open window and, by default, RSSI of the latest
announcement (at least -45 dBm). In app 0.6.3+, set `pairing_rssi_filter:
false` in the app configuration and restart it to pair valid announcements at
any received signal strength. The join window, protocol validation and
capacity limit still apply. The `JOIN OPEN` log shows whether
the filter is enabled. CLI equivalent: `--no-pairing-rssi-filter` (gateway
only); console: `join [seconds|open|close]` (`open`: until closed). Omitted
configuration defaults to filtering enabled; this changes discovery
admission, not the wire format. Opening a new window clears previous
candidates. Commands use QoS0, are not retained, and do not cross a disconnected
MQTT session. The gateway ignores retained command replays. Messages queued
for more than 10 s are rejected. Give MQTT account access only to trusted clients.

On restart, an already paired sensor restores a secured session without
pairing again. Several sensors may complete their handshakes at once (at most
4 in progress; the others are offered a place at a later announcement). With
the gateway full (`max_sensors` paired) no further sensor is paired. A sensor
holds its place from its offer until pairing completes, also when the offer
expires during its handshake.

<a id="odebrani-cidla"></a>
## Removing a sensor

On the sensor's device page press **4 Remove sensor** (configuration card,
right after the three settings buttons; the press cannot be undone and HA does
not ask). The web panel removes with a confirmation step. The gateway forgets
the sensor completely (pinned key, address, state, queued requests) and clears
its MQTT topics, so HA removes its device and entities.
The gateway remembers the last 64 removed sensors and clears their topics again
after every broker connection, also after a restart: a removal while the
broker is offline does not leave a ghost device in HA.
The sensor is not notified. To pair it again (here or with another gateway),
put it into pairing mode (its button) while a join window is open: a paired
sensor announcing again pairs again as well (log `CANDIDATE: <ID> (paired,
pairs again)`). It replaces its stored gateway only
when the new pairing is accepted; otherwise it returns to the previous one.
The console equivalent is `remove <ID>`.

<a id="model-zarizeni"></a>
## Sensor settings (a form)

Settings of a sensor work as a form: changing a field only **drafts** the value
(checked at once against the product: writable, within min..max; a wrong value
is refused with a reason). Nothing is sent until **1 Apply changes**.
**2 Discard changes** only drops the unsent edits; it sends nothing.
**3 Factory defaults** fills the form with the product's factory values (as
after a new pairing) and the gateway's default check-in interval; only values
that differ become drafts, you review them and press Apply. It is refused for a
product the gateway has no profile of. The web panel (*Sensors and their settings*) edits the same drafts, so HA and the panel always show the same form.

The configuration card of a TH10, in HA's order:

| Entity | Meaning |
|---|---|
| **1 Apply changes**, **2 Discard changes**, **3 Factory defaults** | the form's buttons |
| **4 Remove sensor** | removes the sensor (see above) |
| **Humidity: on change**, **Humidity: Δ** | report when humidity changes, by at least Δ (%) |
| **Temp 1: on change**, **Temp 1: Δ**, **Temp 2: …** | the same per temperature (°C) |
| **Temps: Δ (both)** | one Δ for both temperatures at once (empty when they differ); a later single-channel edit overrides it for that channel |
| **Timing: check-in** | network check-in interval (s): the sensor receives settings at its contacts |
| **Timing: max report** | longest time between reports (s) |
| **Battery: on change**, **Battery: Δ** | TH10 firmware with battery measurement (descriptor revision 2): report when the battery voltage changes by Δ (V; factory 0.1 V, off: the voltage travels in every report anyway) |

TH10 firmware with battery measurement (descriptor revision 2) adds channel 4,
**Battery voltage** (V, device class voltage, a measurement on the Sensors
card like the temperatures; a failed measurement shows as unknown). The
gateway knows both revisions: a sensor updated from revision 1 is identified
again at its next report, the battery entities appear and the existing ones
keep their IDs. In the web panel the channel is *Battery voltage*.

A channel is named by its quantity, numbered only when the sensor has more of
it (Temperature 1, Temperature 2, Humidity); internal channel IDs appear only in
topics, unique IDs and diagnostics. *Sampling interval* and *Min report
interval* are fixed by the product and shown as diagnostics. A product the
gateway has no profile of shows its measured values only. **Settings status**
(Sensors card) says where the form stands:

- *N change(s) not sent: press Apply changes*
- *Sent: waiting for the sensor (next contact in about N s)*: a sleeping
  sensor receives settings at its next contact (check-in)
- *Confirmed by the sensor*, *Rejected by the sensor: &lt;outcome&gt;*
- *No answer from the sensor: …*: device-model values are read back at the
  next contact
- *Up to date*: nothing drafted or sent

A field shows the draft, else the requested value, else the confirmed one; the
attributes of `device_config` list `not_sent` and `waiting` fields and the
`confirmed` values; `network` has `confirmed_seconds`.

Ready-to-paste dashboard card for one TH10 (replace `0123456789abcdef` with the
sensor's ID; an installation from before 0.7.0 keeps older entity IDs, see
Settings → Entities, filtered by the sensor's device):

```yaml
type: entities
title: TH10 settings
entities:
  - type: section
    label: Reporting
  - entity: number.ropixon_0123456789abcdef_timing_check_in
    name: Check-in interval
  - entity: number.ropixon_0123456789abcdef_timing_max_report
    name: Max report interval
  - type: section
    label: Change thresholds
  - entity: number.ropixon_0123456789abcdef_temp_1_d
    name: Temperature 1 Δ
  - entity: switch.ropixon_0123456789abcdef_temp_1_on_change
    name: Temperature 1 on change
  - entity: number.ropixon_0123456789abcdef_temp_2_d
    name: Temperature 2 Δ
  - entity: switch.ropixon_0123456789abcdef_temp_2_on_change
    name: Temperature 2 on change
  - entity: number.ropixon_0123456789abcdef_temps_d_both
    name: Both temperatures Δ
  - entity: number.ropixon_0123456789abcdef_humidity_d
    name: Humidity Δ
  - entity: switch.ropixon_0123456789abcdef_humidity_on_change
    name: Humidity on change
  - type: section
    label: Apply
  - entity: sensor.ropixon_0123456789abcdef_settings_status
    name: Status
  - type: buttons
    entities:
      - entity: button.ropixon_0123456789abcdef_1_apply_changes
        name: Apply
      - entity: button.ropixon_0123456789abcdef_2_discard_changes
        name: Discard
      - entity: button.ropixon_0123456789abcdef_3_factory_defaults
        name: Factory
```

<a id="signal"></a>
## Link quality

Each paired sensor has **Receive quality**, a separate numeric diagnostic
sensor (0–100 %, unique ID suffix `_signal_percent`), next to **Signal
strength** (RSSI, dBm) and **Signal SNR** (dB, unknown when the dongle firmware
does not report SNR). The word rating **Signal quality** (Good/Fair/Weak, suffix
`_signal`) was removed in app 0.9.2; the gateway deletes it from Home Assistant
at startup. So was the timestamp entity **Last seen** (suffix `_last_seen`): it
changed at every contact and filled the device's Activity (logbook). The last
contact is now the attribute `last_seen` of **Sensor connectivity**, whose
state changes only when the sensor goes offline or comes back. The panel
shows the same percentage and five bars: 0 has no filled bars; 1–20 has one,
21–40 two, 41–60 three, 61–80 four, 81–100 five. Unknown has no percentage.

For each authenticated received packet, the gateway computes:

```
RSSI score = clamp(100 * (RSSI - sensitivity(SF, BW)) / 30, 0, 100)
SNR score  = clamp(100 * (SNR - SNR_floor(SF)) / (5 - SNR_floor(SF)), 0, 100)
packet score = min(RSSI score, SNR score), or RSSI score without SNR
```

**Receive quality** averages the last eight individual packet scores and rounds
once to an integer. Each score uses that packet's actual RX settings, including
the pairing channel; historical RSSI/SNR are never rescored after a profile
change. The serial reader stamps queued packets with their RX profile. During
retuning the profile is unknown. Unknown/unsupported profiles produce unknown,
never a silent SF7 assumption. A new authenticated session, removal or gateway
restart clears history. An unknown sample breaks the scoring window.

RSSI-only samples mark the window as an **estimate** (`signal_estimated`, panel
“odhad RSSI”). Between packets the last percentage stays unchanged, including
during sleep. `signal_age_s` and `signal_stale` are separate diagnostics; stale
starts after max(30 s, 3 × confirmed check-in + 10 s), or the explicit MQTT
`stale_seconds` override. Connectivity remains a separate HA entity. Passage
of time neither decreases nor hides the percentage.

The current AT interface does not identify gateway receiver silicon or RX gain.
The default is explicitly **sx126x_boosted_approximation**, not measured dongle
performance. Central constants in `ropixon/limits.py` permit later calibration:
SX1261/2 boosted RX sensitivity anchors for SF7/SF12 are −124/−137 dBm at
BW125, −121/−134 at BW250, −117/−129 at BW500; SF8–11 use linear interpolation.
SNR floors for SF7–12 are −7.5, −10, −12.5, −15, −17.5, −20 dB. SF5/6 are not
supported by the current RopixNet PHY and remain unknown. The model is exposed
as `signal_model`, with the last packet profile in `signal_rx_profile`.

Sources: [Semtech SX1261/2 datasheet rev 1.2, table 3-8 and 6-1](https://cdn.sparkfun.com/assets/6/b/5/1/4/SX1262_datasheet.pdf)
(the table numbering differs in other revisions), and [Semtech FAQ](https://www.semtech.com/design-support/faq)
for +5 dB SNR saturation. The 30 dB RSSI span is our approved UI scale. This is
indicative quality of reception at the gateway, not delivery percentage or
bidirectional link quality. No physical RX sensitivity was measured here.

The dongle reports a received packet as `+RX:<length>,<hex>,RSSI:<dBm>` and, with
SNR-capable firmware, `+RX:<length>,<hex>,RSSI:<dBm>,SNR:<dB>` (for example
`+RX:5,48656C6C6F,RSSI:-45,SNR:9`; SNR an integer, a decimal part such as
`-7.25` is accepted, range −40..40). Both forms are understood; any other line
is rejected and counted as a parser error.

<a id="zmena-intervalu-checkinu"></a>
## Changing the checkin interval

**Timing: check-in** (command topic
`ropixon/<gateway>/sensor/<sensor>/command/checkin`) is a field of the form
above: press **1 Apply changes** to queue it. The value is in seconds, range
1..86400. The gateway passes it at the next REPORT/POLL contact; even a sensor
not reporting temperature receives it through POLL. `confirmed_seconds` changes
only after durable storage is acknowledged, not merely after queueing. Details
are in main-repository `docs/configuration.md`.

<a id="dostupnost-vypadky-a-neznamy-senzor"></a>
## Availability, outages, and unknown sensors

- Stable device identifiers come from the persistent database, not the COM port number.
- Discovery and state are refreshed on broker connection and HA birth
  `homeassistant/status=online`. The default discovery prefix is `homeassistant`.
- Gateway uses retained Last Will `offline`. After a crash, unavailability
  appears when the broker detects the outage, not necessarily immediately
  (30 s keepalive).
- REPORT and POLL confirm sensor availability; after gateway restart, a fresh
  radio contact is required. Default timeout is
  `max(30 s, 3 × confirmed checkin + 10 s)`. Measurement age is evaluated
  separately using `max(30 s, 3 × reported period + 10 s)` and POLL does not
  refresh it. `sensor_stale_seconds` overrides both; 0 means automatic.
  With a long local `--interval-ms` on the reference sensor, set timeout
  manually: local override does not change the protocol's stored period.
- **Channel load** (web panel; `channel_load_percent` in the gateway state) is the share of the last hour the
  radio channel was busy with received and transmitted packets. Above 10 %
  the gateway logs a warning (`channel_overloaded` in the gateway state):
  roughly 1 of 6 packets then collides. Sensors should report less often, or
  the sensors should be split between two gateways.
- Battery voltage is published in volts; percentage is not estimated without
  knowing battery type.
- Unknown profiles/schemas are still authenticated and acknowledged at the
  radio layer. HA shows decoder diagnostics and hex payload, not fake temperatures.
- MQTT outages do not block ACK. Only latest state is retained in memory,
  not full history. Radio ACK therefore does not confirm a write to HA
  history. Durable store-and-forward requires another milestone.
- USB disconnect ends the service with an error and MQTT offline; restart
  after reconnecting. HA OS app Watchdog can handle restart if enabled.

<a id="verejne-sdileni-a-aktualizace"></a>
## Public sharing and updates

`home-assistant/ha_repository` is prepared as the root of a separate app
repository. After verification on actual HA OS it can be published and users
can add its repository URL to the HA store. Official-repository approval is
not required. Neither is HACS. Updates change the app, not the entire HA
installation; the state in `/config` is preserved (also across a reinstall, see above).

Before public release choose a license and issue-reporting contact, and verify
Raspberry Pi build/operation. Prebuilt aarch64/amd64 images may later simplify
installation. This version publishes no GitHub repository or image. After
source changes run
`python gateway/Ropixon_LoRa_prototyp_v0.6.0/tools/build_ha_repository.py` to
refresh source copies in the Docker context. Do not publish `mqtt.json`,
key databases, or private logs. A native HA Core integration would be a
different reviewed approach; this MQTT solution does not need it.

<a id="co-bylo-overeno"></a>
## What was verified

Automated tests verify the protocol, MQTT Discovery/commands/availability,
and transfer through a local actual MQTT broker and emulated UART. This
environment has no HA OS, Docker daemon, or your physical dongles. Building
and installing the app on Raspberry Pi and viewing entities in live HA await
the first joint test.

Official references:

- https://www.home-assistant.io/integrations/mqtt/
- https://www.home-assistant.io/integrations/sensor.mqtt/
- https://www.home-assistant.io/integrations/text.mqtt/
- https://www.home-assistant.io/integrations/select.mqtt/
- https://www.home-assistant.io/integrations/button.mqtt/
- https://www.home-assistant.io/dashboards/entities/
- https://developers.home-assistant.io/docs/apps/configuration/
- https://developers.home-assistant.io/docs/apps/testing/
- https://developers.home-assistant.io/docs/apps/repository/
