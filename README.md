# Zabbix Template Monitoring AP over api 7.0

**UniFi Network API — Zabbix 7.0 template for UniFi OS access points**

**English** | [Русский](README.ru.md)

Zabbix 7.0 template for monitoring **UniFi Network**. It automatically finds every site and every access point on the controller and monitors them. It connects with an **API key**, and the key can be stored in **HashiCorp Vault**.

- **Version:** 1.0.0
- **Zabbix:** 7.0
- **License:** MIT
- **Author:** AspidovSS

This is a fork of [MassimilianoPasquini97/zbx_unifi_network_api](https://github.com/MassimilianoPasquini97/zbx_unifi_network_api) (Zabbix 7.4), rewritten for Zabbix 7.0 and extended. See [Credits](#credits).

---

## Contents

- [Features](#features)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Updating](#updating)
- [Hosts, groups and tags](#hosts-groups-and-tags)
- [What is monitored](#what-is-monitored)
- [Triggers](#triggers)
- [Macros](#macros)
- [Load on the controller](#load-on-the-controller)
- [Limitations](#limitations)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Credits](#credits)
- [License](#license)

---

## Features

- **Automatic discovery.** One Zabbix host per site and one per access point, across all sites of the controller. Switches and gateways are skipped: a device is discovered only if UniFi marks it with the `accessPoint` feature.
- **Access point health.** State, CPU, memory, load average, uptime, firmware.
- **Per-radio metrics** (2.4 / 5 / 6 GHz):
  - channel, width, transmit power, EIRP, antenna gain;
  - channel utilization (Busy / Rx / Tx);
  - Tx retry, Tx/Rx dropped;
  - packets and bytes per second;
  - clients, Wi-Fi Experience.
- **Uplink.**
  - Parent device (UniFi name, LLDP name or MAC), port, type.
  - Speed, with an alert when the speed drops.
  - Down/Up traffic.
- **Clients.**
  - Clients per AP and per radio, guests, average connection time, and the list of connected devices.
  - Wireless clients per site.
- **UniFi anomalies.**
  - Anomalies of the AP itself: high drop rate, slow association.
  - Which clients of the AP have anomalies right now. This shows whether a problem is in the client or in the Wi-Fi zone.
- **Thresholds for industrial sites.** Separate thresholds for 2.4 and 5 GHz (context macros). Tx Retry alerts are suppressed when there is little traffic.
- **Safe discovery.** If the API fails for any site, discovery skips the run. Access points of that site are not marked as lost and are not deleted.
- **Secrets in Vault.** The API key is a Vault secret macro. Raw API responses are not stored, and device keys returned by the internal API are stripped in preprocessing.

## How it works

```
Host "UniFi Network API"  (you create it, template "UniFi Network API")
 │
 ├─ Discovery Sites  (once a day)
 │    └─ UniFi [<site>] - Site                 template "UniFi Network API - Site"
 │         wireless clients, APs total / online / offline
 │
 └─ Discovery Access Points  (list every 30 min, discovery on change or once a day)
      └─ UniFi [<site>] - AP:<name>            template "UniFi Network API - Access Point"
           state, CPU, memory, radios, uplink, clients, anomalies
```

Zabbix 7.0 cannot run host prototypes on discovered hosts. That feature appeared in 7.4. This is why access points are discovered by the main host, not by the site hosts.

The template uses two UniFi APIs, both with the same API key (header `X-API-KEY`):

| API | Path | Used for |
|---|---|---|
| Official Integration API v1 | `/proxy/network/integration/v1/...` | sites, devices, clients, device statistics |
| Internal Network API | `/proxy/network/api/s/<site>/stat/device/<mac>`, `/stat/anomalies` | radio details, uplink, anomalies |

The internal API is not officially documented, so it may change in future UniFi releases (see [Limitations](#limitations)).

## Requirements

- **Zabbix 7.0** server or proxy. The template uses HTTP agent, Script and Dependent items.
- **UniFi Network** with the Integration API (UniFi Network 9.x or newer). It must run on **UniFi OS**: UDM / UCG / UDR / Cloud Key Gen2+ / UniFi OS Server. The template uses the `/proxy/network` path prefix.
- **Network access.** The Zabbix server or proxy must reach the console over HTTPS. The port may be non-standard, for example `192.168.1.10:11443`.
- **A UniFi API key.**
- **(Recommended) HashiCorp Vault** configured in Zabbix (KV v2). Without Vault, store the key in a *Secret text* macro instead.

## Installation

### 1. Create an API key in UniFi

UniFi Network → **Settings → Control Plane → Integrations → Create API Key**. Copy the key: it is shown only once.

Check the key from the Zabbix server. Do not type the key on the command line directly:

```bash
read -s KEY        # paste the key, it is not shown
curl -sk -H "X-API-KEY: $KEY" "https://<console>/proxy/network/integration/v1/sites"
```

The response must be JSON with a `data` list of your sites.

### 2. Store the key in Vault

```bash
vault kv put secret/zabbix/unifi api-key="<key>"
```

In Zabbix the Vault macro value is written as `path:key`. With a KV v2 engine mounted at `secret`, that is `secret/zabbix/unifi:api-key`. If your mount point is different, the first part of the path is the mount point.

### 3. Create global macros

**Administration → Macros:**

| Macro | Type | Example |
|---|---|---|
| `{$UNIFI_NETWORK_API_ADDRESS}` | Text | `192.168.1.10` or `unifi.example.local:11443`. No `https://`, no spaces. |
| `{$UNIFI_NETWORK_API_KEY}` | Vault secret | `secret/zabbix/unifi:api-key` |

Both macros **must be global**. Site and access point hosts are created by discovery and do not inherit macros from the main host. Do not define these two macros on templates either: a template macro overrides the global one.

### 4. (Optional) Create the parent host group

Discovered hosts go to the groups `UniFi Network/Sites` and `UniFi Network/Access Points`. Zabbix shows them as subgroups only if a group named exactly **`UniFi Network`** exists.

### 5. Import the template

**Data collection → Templates → Import** → `template/zbx_unifi_network_api_7.0.yaml`.

### 6. Create the main host

Create a host with any name, for example `UniFi Controller`. It needs no interface. Link the template **UniFi Network API**.

### 7. Start discovery

Discovery runs once a day by default. To start it now, open the main host and click **Execute now** on:

1. **Sites JSON**, then **Discovery Sites**;
2. **Access Points JSON (all sites)**, then **Discovery Access Points**.

After 1–2 minutes the hosts appear in `UniFi Network/Sites` and `UniFi Network/Access Points`. Radio items appear after the first poll of each access point, within about 2 minutes.

## Updating

Import the new version of the file with these import rules:

- **Update existing** for host groups, templates, items, triggers and discovery rules;
- **Delete missing** for items, triggers, discovery rules and prototypes.

Changes to host names and groups reach existing hosts on the next discovery run. To apply them at once, use **Execute now** as in step 7.

## Hosts, groups and tags

| Host | Name | Group |
|---|---|---|
| Site | `UniFi [<site>] - Site` | `UniFi Network/Sites` |
| Access point | `UniFi [<site>] - AP:<AP name>` | `UniFi Network/Access Points` |

**Host tags:**

- `UniFi Type` = `Site` / `Access Point`
- `UniFi Site` = site name
- `UniFi Model` = AP model

**Item tags:**

- `UniFi Component` = `System`, `CPU`, `Memory`, `Clients`, `Radio`, `Uplink`, `Anomalies`, `Access Points`, `Raw`
- `UniFi Radio` = `2.4GHz` / `5GHz` / `6GHz`

All tags start with `UniFi` and always have a value.

## What is monitored

### Main host (`UniFi Network API`)

- UniFi Network application version.
- Number of access points and offline access points across all sites.

### Site (`UniFi Network API - Site`)

- Wireless clients on the site. All clients are collected with pagination, 200 per request.
- Access points: total, online, offline.

### Access point (`UniFi Network API - Access Point`)

| Group | Items |
|---|---|
| System | state, model, name, IP, MAC, firmware, uptime, load average |
| CPU / memory | utilization, % |
| Clients | total, guests, average connection time, list of clients (`name \| IP \| MAC`), Wi-Fi Experience |
| Uplink | parent device, parent MAC / port / LLDP description, type (wire / wireless), speed, down/up rate, packets, byte counters |
| Radio (per band) | state, channel, width, Tx power, EIRP, antenna gain, clients, guests, Wi-Fi Experience, channel utilization Busy/Rx/Tx, Tx retry %, Tx/Rx packets/s and bytes/s, Tx/Rx dropped % |
| Anomalies | active AP anomalies, high drop rate duration, slow associations per hour, clients with anomalies (count, %, list) |

**Write frequency for slowly changing values:**

- EIRP, antenna gain, uplink type: stored on change and at least once a day.
- Channel width, uplink speed: stored on change and at least once an hour.

These values still come from the regular 2-minute request. No extra requests are made for them.

## Triggers

### Main host

| Trigger | Severity | Default condition |
|---|---|---|
| Application Version Changed | Info | version changed |
| No Data for 30m | High | no response from the API for `{$TRIGGER_NODATA}` |

### Access point

| Trigger | Severity | Default condition |
|---|---|---|
| State is "OFFLINE / CONNECTION_INTERRUPTED / ISOLATED" | High | current state |
| State is "PENDING_ADOPTION / DELETING" | Warning | current state |
| State is "UPDATING / GETTING_READY" | Info | current state |
| CPU at … | Warning / Average | average of last 3 values > 80% / 90% |
| Memory at … | Warning / Average | average of last 3 values > 80% / 90% |
| Device Rebooted | Info | uptime decreased |
| Too many clients on AP | Warning | > 40 clients for 15 min |
| Uplink parent device changed | Info | the AP was connected to another device |
| Uplink speed dropped | Warning | wired uplink is slower than its maximum over 7 days, e.g. 1 Gbps → 100 Mbps |
| Radio … state is "…" for 10m | Average | the radio was not in `RUN` for 10 minutes |
| Radio … channel utilization | Warning / Average | 5 GHz: > 50% / 70%; 2.4 GHz: > 60% / 80%; for 15 min |
| Radio … Tx Retry | Warning / Average | 5 GHz: avg > 15% / min > 25%; 2.4 GHz: avg > 25% / min > 40%; over 1 h, only if the radio sends > 20 packets/s |
| AP high drop rate (UniFi anomaly) | Warning | anomaly lasts ≥ 15 min |
| Clients connect slowly to AP (UniFi anomaly) | Warning | ≥ 3 five-minute intervals per hour |
| Wi-Fi zone problem | Warning | ≥ 30% and ≥ 3 clients of the AP have anomalies for 15 min |

The thresholds are tuned for **industrial sites** with a lot of metal and moving machinery. For offices, lower them in the macros. Override them on a host or template level if different sites need different values.

## Macros

### Global macros (you create them)

| Macro | Description |
|---|---|
| `{$UNIFI_NETWORK_API_ADDRESS}` | IP or DNS name of the UniFi OS console, with `:port` if it is not 443 |
| `{$UNIFI_NETWORK_API_KEY}` | API key: Vault secret (`path:key`) or Secret text |

### Template macros

Context macros like `{$TRIGGER_CU_HIGH:"2.4GHz"}` override the base macro for one band only.

<details>
<summary><b>UniFi Network API</b> (main host)</summary>

| Macro | Default | Description |
|---|---|---|
| `{$ITEM_APPLICATION.VERSION_HISTORY}` | `7d` | Item 'application.version' History |
| `{$ITEM_APPLICATION.VERSION_UPDATE}` | `10m` | Item 'application.version' Update Interval |
| `{$ITEM_APS.JSON_UPDATE}` | `30m` | Item 'aps.json' Update Interval (AP counters; discovery itself runs on change or once a day) |
| `{$ITEM_APS_HISTORY}` | `7d` | Items 'aps.*' History |
| `{$ITEM_SITES.JSON_UPDATE}` | `1d` | Item 'sites.json' Update Interval |
| `{$LLD_APS_HEARTBEAT}` | `1d` | AP discovery runs when the AP list changes (added/removed/renamed) or at least once per this period |
| `{$TRIGGER_NODATA}` | `30m` | Trigger 'No Data' after 30 minutes of missing data |

</details>

<details>
<summary><b>UniFi Network API - Site</b></summary>

| Macro | Default | Description |
|---|---|---|
| `{$ITEM_APS_HISTORY}` | `7d` | Items 'aps.*' History |
| `{$ITEM_CLIENTS.JSON_UPDATE}` | `5m` | Item 'clients.json' Update Interval |
| `{$ITEM_CLIENTS.WIRELESS_HISTORY}` | `7d` | Item 'clients.wireless' History |
| `{$ITEM_DEVICES.JSON_UPDATE}` | `10m` | Item 'devices.json' Update Interval |
| `{$SITE.ID}` | — | Site ID. Used by Template, DO NOT CHANGE OR DELETE. |
| `{$SITE.NAME}` | — | Site NAME. Used by Template, DO NOT CHANGE OR DELETE. |

</details>

<details>
<summary><b>UniFi Network API - Access Point</b></summary>

| Macro | Default | Description |
|---|---|---|
| `{$ANOMALY_ACTIVE_MIN}` | `15` | Anomaly is active if seen within this many minutes |
| `{$ANOMALY_IGNORE}` | — | Anomaly types to ignore, comma separated (e.g. USER_DNS_TIMEOUT) |
| `{$AP.MAC}` | — | AP MAC. Filled by discovery, DO NOT CHANGE OR DELETE. |
| `{$ITEM_AP.ANOMALY.TEXT_HISTORY}` | `7d` | Anomaly text items History |
| `{$ITEM_AP.ANOMALY_HISTORY}` | `30d` | Anomaly numeric items History |
| `{$ITEM_AP.CLIENTS.LIST_HISTORY}` | `1d` | Item 'ap.clients.list' History |
| `{$ITEM_AP.CLIENTS_HISTORY}` | `30d` | Items 'ap.clients.total/avg_duration' History |
| `{$ITEM_CLIENTS.JSON_UPDATE}` | `5m` | Item 'clients.json' Update Interval |
| `{$ITEM_CPU.UTILIZATION_HISTORY}` | `7d` | Item 'cpu.utilization' History |
| `{$ITEM_DETAILS.JSON_UPDATE}` | `10m` | Item 'details.json' Update Interval |
| `{$ITEM_FIRMWARE.VERSION_HISTORY}` | `1h` | Item 'firmware.version' History |
| `{$ITEM_IP_HISTORY}` | `1h` | Item 'ip' History |
| `{$ITEM_LOADAVG_HISTORY}` | `7d` | Item 'loadAverage1Min' History |
| `{$ITEM_MACADDRESS_HISTORY}` | `1h` | Item 'macaddress' History |
| `{$ITEM_MEMORY.UTILIZATION_HISTORY}` | `7d` | Item 'memory.utilization' History |
| `{$ITEM_MODEL_HISTORY}` | `1h` | Item 'model' History |
| `{$ITEM_NAME_HISTORY}` | `1h` | Item 'name' History |
| `{$ITEM_RADIO.RAW_HISTORY}` | `1d` | Raw counter rates used for Dropped % History |
| `{$ITEM_RADIO_HISTORY}` | `30d` | Radio items History |
| `{$ITEM_STAT.JSON_UPDATE}` | `2m` | Item 'stat.json' Update Interval (radio details) |
| `{$ITEM_STATE_HISTORY}` | `1h` | Item 'state' History |
| `{$ITEM_STATISTICS.JSON_UPDATE}` | `5m` | Item 'statistics.json' Update Interval |
| `{$ITEM_UPLINK.INFO_HISTORY}` | `30d` | Uplink text items History |
| `{$ITEM_UPLINK_HISTORY}` | `7d` | Item 'uplink' History |
| `{$ITEM_UPTIME_HISTORY}` | `7d` | Item 'uptime' History |
| `{$SITE.ID}` | — | Site ID. Filled by discovery, DO NOT CHANGE OR DELETE. |
| `{$SITE.NAME}` | — | Site NAME. Filled by discovery, DO NOT CHANGE OR DELETE. |
| `{$SITE.REF}` | — | Site short name (internalReference). Filled by discovery, DO NOT CHANGE OR DELETE. |
| `{$TRIGGER_AP_ANOMALY_DROP_TIME}` | `15` | AP_HIGH_DROP_RATE alert after this many minutes |
| `{$TRIGGER_AP_ANOMALY_LONGASSOC_COUNT}` | `3` | Slow association alert: intervals per hour |
| `{$TRIGGER_AP_ANOMALY_ZONE_MIN}` | `3` | Zone problem: minimum number of clients with anomalies |
| `{$TRIGGER_AP_ANOMALY_ZONE_PCT}` | `30` | Zone problem: % of AP clients with anomalies |
| `{$TRIGGER_AP_ANOMALY_ZONE_TIME}` | `15m` | Zone problem must last this long |
| `{$TRIGGER_AP_CLIENTS_HIGH}` | `40` | Trigger: max wireless clients per AP |
| `{$TRIGGER_AP_CLIENTS_TIME}` | `15m` | Trigger: clients above threshold for this period |
| `{$TRIGGER_CPUUTIL_AVERAGE}` | `90` | Trigger 'CPU Utilization' % trigger |
| `{$TRIGGER_CPUUTIL_WARNING}` | `80` | Trigger 'CPU Utilization' % trigger |
| `{$TRIGGER_CU_HIGH:"2.4GHz"}` | `80` | Channel utilization Average %, 2.4GHz |
| `{$TRIGGER_CU_HIGH}` | `70` | Channel utilization Average %, default (5GHz/6GHz) |
| `{$TRIGGER_CU_TIME}` | `15m` | Channel utilization period |
| `{$TRIGGER_CU_WARNING:"2.4GHz"}` | `60` | Channel utilization Warning %, 2.4GHz |
| `{$TRIGGER_CU_WARNING}` | `50` | Channel utilization Warning %, default (5GHz/6GHz) |
| `{$TRIGGER_MEMUTIL_AVERAGE}` | `90` | Trigger 'Memory Utilization' % trigger |
| `{$TRIGGER_MEMUTIL_WARNING}` | `80` | Trigger 'Memory Utilization' % trigger |
| `{$TRIGGER_RADIO_STATE_TIME}` | `10m` | Radio state trigger fires only if the radio is not RUN for this whole period |
| `{$TRIGGER_TXRETRIES_AVERAGE:"2.4GHz"}` | `40` | Tx Retry Average %, 2.4GHz |
| `{$TRIGGER_TXRETRIES_AVERAGE}` | `25` | Tx Retry Average %, default (5GHz/6GHz) |
| `{$TRIGGER_TXRETRIES_MIN_PPS}` | `20` | Tx Retry triggers fire only if radio avg Tx > this many packets/s |
| `{$TRIGGER_TXRETRIES_TIME}` | `1h` | Tx Retry evaluation period |
| `{$TRIGGER_TXRETRIES_WARNING:"2.4GHz"}` | `25` | Tx Retry Warning %, 2.4GHz |
| `{$TRIGGER_TXRETRIES_WARNING}` | `15` | Tx Retry Warning %, default (5GHz/6GHz) |
| `{$TRIGGER_UPLINK_SPEED_PERIOD}` | `7d` | Uplink speed is compared with its maximum over this period |

</details>

## Load on the controller

With the default intervals, each access point makes about **1.4 requests per minute**. Each site adds about 0.3 requests per minute, and the main host adds about one request per site every 30 minutes.

| Access points | Requests / min (approx.) |
|---|---|
| 100 | ~150 |
| 500 | ~700 |

To reduce the load, increase `{$ITEM_STAT.JSON_UPDATE}` (radio details, 2m by default) and `{$ITEM_CLIENTS.JSON_UPDATE}` (clients and anomalies, 5m).

## Limitations

- **Zabbix 7.0 only** in this release. The original 7.4 template is linked in [Credits](#credits).
- **Internal API.** Radio details, uplink and anomalies come from an undocumented UniFi API. It may change in future UniFi versions.
- **Not available:** channel Auto/Manual mode, Tx power mode and Rx retry. The API gives no reliable data for them.
- **Uplink byte totals** count since the interface came up and do not match the UniFi UI. Rates (bps, pps) are accurate.
- **Self-hosted UniFi Network Application** without UniFi OS was not tested. It may not use the `/proxy/network` prefix.
- **Template names and UUIDs** are the same as in the original `UniFi Network API` template. Importing this file updates that template if it is already installed.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `URL rejected: Bad hostname` | `{$UNIFI_NETWORK_API_ADDRESS}` is empty, misspelled, contains spaces or `https://`. Use **Test** on the item to see the resolved macro. |
| HTTP 401 | Wrong key, or the Vault secret is not resolved. Check `zabbix_server.log`, then run `zabbix_server -R secrets_reload` (and `zabbix_proxy -R secrets_reload` on proxies). |
| Site hosts exist, access point hosts do not | Run **Test** on *Access Points JSON (all sites)* on the main host, then check the **Info** column of *Discovery Access Points*. |
| `devices request failed for N of M sites` | The API did not answer for the listed sites. Discovery skips this run on purpose, so that APs are not marked as lost. |
| Groups are not nested | Create or rename the parent group to exactly `UniFi Network`. |
| No radio items on an AP | Check *Device Stat JSON*: the macros `{$SITE.REF}` and `{$AP.MAC}` are filled by discovery. Re-run *Discovery Access Points*. |

## Security

- **Do not type the API key on the command line, in logs, chats or tickets.** Use `read -s KEY` and `$KEY`. If the key leaks, delete it in UniFi, create a new one and update Vault.
- **Use a separate API key for Zabbix.**
- **Raw API responses are not stored** (history 0). The internal device API also returns device keys (`x_authkey` and others). The template drops them in preprocessing before anything is stored.

## Credits

- Original template: [zbx_unifi_network_api](https://github.com/MassimilianoPasquini97/zbx_unifi_network_api) by Massimiliano Pasquini, MIT License.
- Zabbix 7.0 port and extensions: **AspidovSS**.

## License

[MIT](LICENSE). The license file keeps the copyright notice of the original project, as the MIT License requires.
