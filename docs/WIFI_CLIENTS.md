# Wi-Fi clients by filter — add-on template

**English** | [Русский](WIFI_CLIENTS.ru.md) · [Main README](../README.md)

An add-on for the UniFi Network API templates. It creates a **separate Zabbix host for each UniFi client that matches your filter**: a Wi-Fi network name, a client name, a MAC vendor, or a combination of them. Each client host gets signal, rates, retries, CCQ, Experience, roaming history, ICMP ping and UniFi anomalies.

Typical use: barcode scanners, payment terminals, IoT controllers, PTP bridges, cameras — any **stationary devices with fixed MAC addresses** that you want to watch one by one.

- **File:** `template/zbx_unifi_wifi_clients_7.0.yaml`
- **Zabbix:** 7.0
- **Works with or without** the main template `zbx_unifi_network_api_7.0.yaml`: it needs only the same two global macros.
- **License:** MIT · **Author:** AspidovSS

---

## Contents

- [How it works](#how-it-works)
- [Installation](#installation)
- [Filter macros](#filter-macros)
- [Examples](#examples)
- [Hosts, groups and tags](#hosts-groups-and-tags)
- [Client lifecycle](#client-lifecycle)
- [What is monitored](#what-is-monitored)
- [Triggers](#triggers)
- [Macros](#macros)
- [Random MAC addresses](#random-mac-addresses)
- [Load on the controller](#load-on-the-controller)
- [Limitations](#limitations)
- [Troubleshooting](#troubleshooting)

---

## How it works

```
Host "UniFi Clients: Scanners"          you create it, template "… - Wi-Fi Clients Filter"
  {$FILTER.SSID} = ^scanners$           your filter
  │
  └─ Discovery Filtered Clients          (all sites, every 30 min)
       └─ UniFi [<site>] - <category>:<name>     template "… - Wi-Fi Client"
```

- **One filter = one host.** For several device categories, create several filter hosts, each with its own macros.
- **The filter host reads all sites** of the controller every 30 minutes (`stat/sta`, one request per site) and keeps the clients that match.
- **All filters you fill must match** (AND). Empty filters are not used. At least one of `{$FILTER.SSID}`, `{$FILTER.NAME}`, `{$FILTER.OUI}` must be set.
- **Safe discovery.** If any site does not answer, the run is skipped, so clients of that site are not marked as lost.
- **Safety limit.** If more than `{$FILTER.MAX_CLIENTS}` (200) clients match, discovery is not run and the filter host shows an error. This protects Zabbix from a too broad filter, such as a guest network with hundreds of phones.

## Installation

1. **Global macros.** Create `{$UNIFI_NETWORK_API_ADDRESS}` and `{$UNIFI_NETWORK_API_KEY}` as described in the [main README](../README.md#installation). If the main template already works, they are there.
   Optional: create the host group **`UniFi Network`**, so that `UniFi Network/Wi-Fi Clients/...` is shown as its subgroups.
2. **Import** `template/zbx_unifi_wifi_clients_7.0.yaml` (*Data collection → Templates → Import*).
3. **Create a filter host.** Any name, for example `UniFi Clients: Scanners`. It needs no interface. Link the template **UniFi Network API - Wi-Fi Clients Filter**.
4. **Set the filter** in the *Macros* tab of that host:
   - at least one of `{$FILTER.SSID}`, `{$FILTER.NAME}`, `{$FILTER.OUI}`;
   - `{$FILTER.CATEGORY}`: the name of the device category, for example `Scanner` (unique per filter host);
   - `{$FILTER.PREFIX}`: a short unique prefix of Latin letters, digits, `.`, `_`, `-`, for example `scanner`.
5. **Check the filter.** On the item **Filtered Clients JSON (all sites)** click **Test** → *Get value and test*. You see the list of matching clients.
6. **Start discovery:** **Execute now** on **Filtered Clients JSON (all sites)**. Client hosts appear within a minute.

## Filter macros

| Macro | What it matches | Example |
|---|---|---|
| `{$FILTER.SSID}` | Wi-Fi network name | `^scanners$`, `^(iot\|sensors)$` |
| `{$FILTER.NAME}` | client name in UniFi **or** its hostname | `^wirenboard`, `^TSD-` |
| `{$FILTER.OUI}` | MAC vendor shown by UniFi | `ubiquiti`, `zebra\|honeywell` |
| `{$FILTER.SITE}` | site name (empty = all sites) | `^Tyumen$` |
| `{$FILTER.EXCLUDE}` | skip clients whose name, hostname or MAC matches | `test\|spare` |
| `{$FILTER.WIRED}` | `1` = also wired clients (only without an SSID filter) | `1` |
| `{$FILTER.RANDOM_MAC}` | `1` = also random (private) MAC addresses | `0` |

All regular expressions are case-insensitive. In a Zabbix macro, write the regex as is, without slashes.

## Examples

| Goal | Filter host macros |
|---|---|
| All clients of the network `scanners` | `{$FILTER.SSID}` = `^scanners$`, `{$FILTER.CATEGORY}` = `Scanner`, `{$FILTER.PREFIX}` = `scanner` |
| Only Zebra devices in that network | add `{$FILTER.OUI}` = `zebra` |
| Wiren Board controllers, wired and wireless | `{$FILTER.NAME}` = `^wirenboard`, `{$FILTER.WIRED}` = `1`, `{$FILTER.CATEGORY}` = `Wirenboard`, `{$FILTER.PREFIX}` = `wb` |
| Ubiquiti PTP bridges (e.g. airMAX Loco M2) | `{$FILTER.OUI}` = `ubiquiti`, `{$FILTER.CATEGORY}` = `PTP`, `{$FILTER.PREFIX}` = `ptp` |
| Terminals of one site, without test devices | `{$FILTER.SSID}` = `^pos$`, `{$FILTER.SITE}` = `^Tyumen$`, `{$FILTER.EXCLUDE}` = `test` |

## Hosts, groups and tags

| | Value |
|---|---|
| Host name | `UniFi [<site>] - <category>:<name>` |
| Technical host name | `<prefix>-<mac without colons>` |
| Host groups | `UniFi Network/Wi-Fi Clients` and `UniFi Network/Wi-Fi Clients/<category>` |
| Host tags | `UniFi` = `<category>`, `UniFi Site` = site name |
| Interface | the client IP from UniFi (used by ICMP ping) |

- **`<name>`** is the client name in UniFi, or its hostname, or its MAC. If two clients of one site have the same name, the MAC is added: `Scanner (24:5a:4c:11:22:33)`. Give devices clear names in UniFi to get clear host names in Zabbix.
- **`{$FILTER.PREFIX}` and `{$FILTER.CATEGORY}` must be unique per filter host.** Zabbix needs unique technical and visible host names. If one device matches two filters, it gets two independent hosts only when both the prefix and the category differ.

## Client lifecycle

| Time not connected (or not matching the filter) | What happens |
|---|---|
| any | data items stop updating; the host keeps working |
| 48 h | event *Client not connected for more than 48h* (Information) |
| 30 days | host is **disabled** |
| 31 days | host is **deleted** together with its history |

- **Connected again within 30 days:** discovery finds the client again, and monitoring continues on the same host.
- **Moved to another network** that the filter does not cover: for this filter the client is gone, and the same rules apply.
- After import, check the discovery rule **Discovery Filtered Clients**: *Disable lost resources: after 30d*, *Delete lost resources: after 31d*.

## What is monitored

### Filter host

| Item | Meaning |
|---|---|
| Filtered Clients Connected | connected clients that match the filter (all sites) |
| Random MAC Clients Skipped | matching clients skipped because of a random MAC |

### Client host (`UniFi Network API - Wi-Fi Client`)

| Group | Items |
|---|---|
| Connection | connected (1/0), access point (stored **only on change**: roaming history, 30 days), AP MAC, SSID, network, IP, connected time, roam count, first seen, last association, hostname, MAC vendor, connection type (Wired / Wireless) |
| Signal | signal, noise, SNR, Tx/Rx rate, Tx retries, CCQ, MCS, Tx dropped %, Experience |
| Radio | band, channel, channel width, Wi-Fi standard, MIMO |
| Traffic | download / upload (bps), Tx packets/s, total downloaded / uploaded |
| Ping | ICMP ping, loss, response time |
| Anomalies | UniFi anomaly count, active anomalies, duration, minutes in the last hour, intervals in 24 h |

- **Wired clients.** Radio items stay empty instead of showing zeros, so they never cause false alerts. The calculated item *Tx Dropped* shows *Not supported* for them, which is expected.
- **Values UniFi does not report** (for example CCQ or Experience on some clients) are not stored as 0: the item keeps its last value, so there are no false low-value alerts.

## Triggers

### Filter host

| Trigger | Severity | Default condition |
|---|---|---|
| Client filter: no data for {$TRIGGER_FILTER_NODATA} | Warning | no data for 2 h: API error, no filter, invalid regex or limit exceeded |

### Client host

| Trigger | Severity | Default condition |
|---|---|---|
| Client not connected for more than … | Info | 48 h |
| Client roamed to AP "…" (event: `Client roamed: <from> -> <to>`) | Info | AP changed; closes after 1 min |
| Very weak signal … | Average | < −80 dBm for 10 min |
| Weak signal … | Warning | < −75 dBm for 10 min |
| Tx / Rx Rate above … Mbps | Info | **off**; > `{$TRIGGER_CLIENT_RATE_HIGH}` Mbps for 10 min |
| Tx / Rx Rate below … Mbps | Warning | **off**; < `{$TRIGGER_CLIENT_RATE_LOW}` Mbps for 10 min |
| Tx Retries > …% | Warning | > 30% for 30 min, only if > 5 packets/s |
| Low CCQ … | Warning | < 50% for 30 min, only if > 5 packets/s |
| Dropped frames > …% | Warning | > 5% for 30 min, only if > 5 packets/s |
| Low Experience: average below …% | Warning | 7-day average < 70% |
| UniFi anomaly for … | Warning | active anomaly ≥ 30 min |
| Client is connected in UniFi but does not respond to ping … | Average | no ping for 5 min while connected; switch off with `{$TRIGGER_CLIENT_PING}` = `0` |

Trigger names are shown as in Zabbix; `…` stands for the threshold or value.

- **Rate alerts are off by default.** An idle device often sends at the minimum rate, so a low-rate alert makes sense only for devices with constant traffic, such as PTP bridges.
- **Where to change thresholds.** In the macros of the template **UniFi Network API - Wi-Fi Client**: they apply to client hosts of all filters. Macros of the filter host are **not** passed to the client hosts (Zabbix discovered hosts do not inherit them).

## Macros

<details>
<summary><b>UniFi Network API - Wi-Fi Clients Filter</b> (filter host)</summary>

| Macro | Default | Description |
|---|---|---|
| `{$FILTER.CATEGORY}` | `Wi-Fi Client` | Category: host name "UniFi [<site>] - <category>:<name>", group "UniFi Network/Wi-Fi Clients/<category>", host tag UniFi. Unique per filter host |
| `{$FILTER.EXCLUDE}` | — | Regex: skip clients whose name, hostname or MAC matches. Empty = no exclusions |
| `{$FILTER.MAX_CLIENTS}` | `200` | Safety limit (positive number): if more clients match, discovery is not run and the item shows an error |
| `{$FILTER.NAME}` | — | Regex: client name in UniFi or hostname. Empty = not used. Example: ^wirenboard |
| `{$FILTER.OUI}` | — | Regex: MAC vendor as shown by UniFi. Empty = not used. Example: ubiquiti |
| `{$FILTER.PREFIX}` | `client` | Technical host name prefix "<prefix>-<mac>": Latin letters, digits, ".", "_", "-". Unique per filter host |
| `{$FILTER.RANDOM_MAC}` | `0` | 1 = also clients with random (private) MAC addresses; 0 = skip them (recommended) |
| `{$FILTER.SITE}` | — | Regex: site name. Empty = all sites |
| `{$FILTER.SSID}` | — | Regex: Wi-Fi network name (SSID). Empty = not used. Example: ^scanners$ |
| `{$FILTER.WIRED}` | `0` | 1 = also wired clients (only without {$FILTER.SSID}); 0 = wireless only |
| `{$ITEM_FILTER.HISTORY}` | `30d` | History of the counters on this host |
| `{$ITEM_FILTER.UPDATE}` | `30m` | How often the clients of all sites are checked against the filter |
| `{$LLD_FILTER.HEARTBEAT}` | `1d` | Discovery runs when the list changes or at least this often |
| `{$TRIGGER_FILTER_NODATA}` | `2h` | Alert if the filter script returns no data for this period |

</details>

<details>
<summary><b>UniFi Network API - Wi-Fi Client</b> (client host)</summary>

| Macro | Default | Description |
|---|---|---|
| `{$ANOMALY_ACTIVE_MIN}` | `15` | Anomaly is active if seen within this many minutes |
| `{$ANOMALY_IGNORE}` | — | Anomaly types to ignore, comma separated (e.g. USER_DNS_TIMEOUT) |
| `{$CLIENT.MAC}` | — | Client MAC. Filled by discovery, DO NOT CHANGE OR DELETE. |
| `{$ITEM_CLIENT.ANOMALY.TEXT_HISTORY}` | `7d` | Anomaly text History |
| `{$ITEM_CLIENT.ANOMALY_UPDATE}` | `5m` | Item 'client.anomalies.json' Update Interval (UniFi uses 5-minute intervals) |
| `{$ITEM_CLIENT.BYTES_HISTORY}` | `7d` | Total traffic History |
| `{$ITEM_CLIENT.INFO_HISTORY}` | `30d` | Client text items History |
| `{$ITEM_CLIENT.JSON_UPDATE}` | `2m` | Item 'client.json' Update Interval |
| `{$ITEM_CLIENT.PING_UPDATE}` | `1m` | ICMP ping interval |
| `{$ITEM_CLIENT.RAW_HISTORY}` | `1d` | Raw counter rates History |
| `{$ITEM_CLIENT.ROAM_HISTORY}` | `30d` | Roaming history (Access Point changes) |
| `{$ITEM_CLIENT.TIMEOUT}` | `15s` | Timeout of the client data and anomalies requests (each reads the list of its site) |
| `{$ITEM_CLIENT_HISTORY}` | `30d` | Client numeric items History |
| `{$SITE.REF}` | — | Site short name. Filled by discovery, DO NOT CHANGE OR DELETE. |
| `{$TRIGGER_CLIENT_ANOMALY_TIME}` | `30` | Anomaly alert after this many minutes without interruption |
| `{$TRIGGER_CLIENT_CCQ_MIN}` | `50` | Low CCQ threshold, % |
| `{$TRIGGER_CLIENT_CCQ_TIME}` | `30m` | CCQ must stay below threshold this long |
| `{$TRIGGER_CLIENT_EXPERIENCE_MIN}` | `70` | Low Experience threshold, % (average) |
| `{$TRIGGER_CLIENT_EXPERIENCE_TIME}` | `7d` | Experience averaging period |
| `{$TRIGGER_CLIENT_OFFLINE_TIME}` | `48h` | Offline alert after the client is not connected this long |
| `{$TRIGGER_CLIENT_PING_TIME}` | `5m` | Connected but no ping: alert after this period |
| `{$TRIGGER_CLIENT_PING}` | `1` | 1 = alert when the client is connected in UniFi but does not answer ping; 0 = off |
| `{$TRIGGER_CLIENT_RATE_HIGH}` | `0` | Rate above this (Mbps) for {$TRIGGER_CLIENT_RATE_TIME}: Info event. 0 = off |
| `{$TRIGGER_CLIENT_RATE_LOW}` | `0` | Rate below this (Mbps) for {$TRIGGER_CLIENT_RATE_TIME}: Warning. 0 = off |
| `{$TRIGGER_CLIENT_RATE_TIME}` | `10m` | Tx/Rx Rate must stay above/below threshold this long |
| `{$TRIGGER_CLIENT_ROAM_SHOW}` | `1m` | Roaming event closes automatically after this time |
| `{$TRIGGER_CLIENT_SIGNAL_CRIT}` | `-80` | Very weak signal threshold, dBm |
| `{$TRIGGER_CLIENT_SIGNAL_MIN}` | `-75` | Weak signal threshold, dBm |
| `{$TRIGGER_CLIENT_SIGNAL_TIME}` | `10m` | Signal must stay below threshold this long |
| `{$TRIGGER_CLIENT_TXDROPPED_TIME}` | `30m` | Dropped frames must stay above threshold this long |
| `{$TRIGGER_CLIENT_TXDROPPED}` | `5` | Dropped frames threshold, % |
| `{$TRIGGER_CLIENT_TXRETRIES_MIN_PPS}` | `5` | Tx Retries, Low CCQ and Dropped frames triggers fire only if the AP sends more than this many packets/s to the client |
| `{$TRIGGER_CLIENT_TXRETRIES_TIME}` | `30m` | Tx Retries must stay above threshold this long |
| `{$TRIGGER_CLIENT_TXRETRIES}` | `30` | Tx Retries threshold, % |

</details>

## Random MAC addresses

Phones, tablets and laptops usually connect with a **random (private) MAC address**. It is different for each network and can change over time. Every new MAC would become a new Zabbix host.

- **Skipped by default.** A random MAC is recognised by the address itself (the "locally administered" bit). Such clients are counted in **Random MAC Clients Skipped** but get no host.
- **To include them,** set `{$FILTER.RANDOM_MAC}` = `1`. Do this only if the devices keep their MAC, for example when private addresses are switched off on them.

## Load on the controller

| Source | Requests |
|---|---|
| each filter host | (number of sites + 1) every 30 min |
| each client host | ~0.7 per minute (client data every 2 min + anomalies every 5 min) |

**Example:** 100 client hosts ≈ 70 requests per minute. Each client request reads the client list of its site, so on sites with many clients increase `{$ITEM_CLIENT.JSON_UPDATE}` in the template **UniFi Network API - Wi-Fi Client**.

ICMP ping goes from Zabbix directly to the client and does not load the controller.

## Limitations

- **Zabbix 7.0.** Uses the undocumented internal UniFi API (`stat/sta`, `stat/anomalies`), which may change in future UniFi versions.
- **Client thresholds are common for all filter hosts** (template macros). Separate thresholds per category are not supported in this version.
- **Devices behind a bridge** (for example behind a Loco M2 in station mode, MAC NAT) are not visible to UniFi and so cannot be discovered.
- **ICMP ping** needs a route from the Zabbix server or proxy to the client networks and `fping`. Without it, switch off the ping alert: `{$TRIGGER_CLIENT_PING}` = `0`.
- **The roaming event name** uses an expression macro to show the previous AP. If it shows `*UNKNOWN*`, the event still works.
- **The lost-resource settings** (30 / 31 days) use the Zabbix 7.0 export fields `enabled_lifetime_type` / `lifetime_type`. Check them after import.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `No filter set: …` | fill at least one of `{$FILTER.SSID}`, `{$FILTER.NAME}`, `{$FILTER.OUI}` on the filter host |
| `Invalid regular expression in the … filter` | fix the regex in that macro |
| `Filter matches N clients, the limit is 200` | narrow the filter, or raise `{$FILTER.MAX_CLIENTS}` if this is intended |
| `stat/sta failed for N of M sites` | the API did not answer for those sites; discovery skips this run on purpose |
| Client is missing | **Test** the filter item; check the client name / SSID / vendor in UniFi; check *Random MAC Clients Skipped* |
| `Host with the same name … already exists` / `… same visible name …` | the device is already a host of another filter or template: use a different `{$FILTER.PREFIX}` and `{$FILTER.CATEGORY}` |
| `The prefix filter macro may contain only …` | use only Latin letters, digits, `.`, `_`, `-` in `{$FILTER.PREFIX}` |
| ICMP Ping always 0 | no route from Zabbix to the client network, or `fping` missing |
