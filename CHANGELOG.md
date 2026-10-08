# Changelog

All notable changes to this project are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/), versioning: [SemVer](https://semver.org/).

## [1.0.0] - 2026-10-09

First release. Zabbix 7.0 port of
[zbx_unifi_network_api](https://github.com/MassimilianoPasquini97/zbx_unifi_network_api) (Zabbix 7.4) with extensions.

### Added
- Discovery of access points of all sites from the main host (Zabbix 7.0 does not run host prototypes on discovered hosts).
- Access point radio metrics from the UniFi internal API: channel, width, Tx power, EIRP, antenna gain,
  channel utilization (Busy/Rx/Tx), Tx retry, Tx/Rx dropped %, packets and bytes per second, clients, guests, Wi-Fi Experience.
- Uplink: parent device (UniFi name / LLDP / MAC), port, LLDP description, type, speed, Down/Up packets and byte counters.
- Clients per access point: total, guests, average connection time, list of connected devices.
- UniFi anomalies for access points: high drop rate, slow association, clients of the AP with anomalies (count, %, list).
- Triggers: radio state, channel utilization and Tx retry with separate 2.4/5 GHz thresholds (context macros) for industrial sites,
  Tx retry suppressed at low traffic, uplink speed drop, uplink parent change, too many clients, AP anomalies, Wi-Fi zone problem.
- Fail-safe discovery: a partial API failure never returns an incomplete AP list, so access points are not lost.
- API key via HashiCorp Vault secret macro; device secrets from the internal API are stripped in preprocessing.
- Item tags `UniFi Component` / `UniFi Radio`, host tags `UniFi Type` / `UniFi Site` / `UniFi Model`.
- Documentation in English and Russian.

### Changed
- Export format Zabbix 7.0.
- Monitoring scope reduced to sites and access points (switches, gateways, ports, VPN and per-client hosts removed).
- Host names `UniFi [<site>] - Site`, `UniFi [<site>] - AP:<name>`; groups `UniFi Network/Sites`, `UniFi Network/Access Points`.
- Site discovery once a day; AP discovery on change of the AP list or once a day.
- Paginated requests for clients and devices (200 per page).

### Fixed
- `{$SITE.ID}` was not passed to discovered hosts in the original template.
- Swapped memory Warning/Average thresholds.

---

# Журнал изменений

## [1.0.0] - 2026-10-09

Первый релиз. Перенос шаблона
[zbx_unifi_network_api](https://github.com/MassimilianoPasquini97/zbx_unifi_network_api) (Zabbix 7.4) на Zabbix 7.0 с доработками.

### Добавлено
- Обнаружение точек доступа всех сайтов с основного хоста (Zabbix 7.0 не запускает хост-прототипы на обнаруженных хостах).
- Радио-метрики точек из внутреннего API UniFi: канал, ширина, мощность, EIRP, усиление антенны, утилизация канала (Busy/Rx/Tx),
  Tx Retry, Tx/Rx Dropped %, пакеты и байты в секунду, клиенты, гости, Wi-Fi Experience.
- Uplink: вышестоящее устройство (имя UniFi / LLDP / MAC), порт, описание LLDP, тип, скорость, пакеты и счётчики Down/Up.
- Клиенты на точке: всего, гости, среднее время подключения, список подключённых устройств.
- Аномалии UniFi для точек: потеря кадров, медленное подключение, клиенты точки с аномалиями (число, %, список).
- Триггеры: состояние радио, утилизация канала и Tx Retry с отдельными порогами 2.4/5 ГГц (контекстные макросы) для промышленных
  объектов, Tx Retry не срабатывает при малом трафике, падение скорости uplink, смена вышестоящего устройства, перегрузка клиентами,
  аномалии точки, проблема зоны Wi-Fi.
- Безопасное обнаружение: при частичном сбое API неполный список точек не отдаётся, точки не теряются.
- API-ключ через макрос-секрет HashiCorp Vault; ключи устройств из внутреннего API вырезаются при предобработке.
- Теги элементов `UniFi Component` / `UniFi Radio`, теги хостов `UniFi Type` / `UniFi Site` / `UniFi Model`.
- Документация на английском и русском.

### Изменено
- Формат экспорта Zabbix 7.0.
- Мониторинг только сайтов и точек доступа (коммутаторы, шлюзы, порты, VPN и отдельные хосты клиентов убраны).
- Имена хостов `UniFi [<сайт>] - Site`, `UniFi [<сайт>] - AP:<имя>`; группы `UniFi Network/Sites`, `UniFi Network/Access Points`.
- Обнаружение сайтов раз в сутки; обнаружение точек при изменении списка или раз в сутки.
- Постраничные запросы клиентов и устройств (по 200).

### Исправлено
- `{$SITE.ID}` не передавался на обнаруженные хосты в исходном шаблоне.
- Перепутанные пороги памяти Warning/Average.
