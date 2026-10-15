## Zabbix_unifi_os_ap_api_7.0 — v1.1.0

Add-on: Wi-Fi clients by filter / Дополнение: Wi-Fi клиенты по фильтру.

**EN.** New optional template `template/zbx_unifi_wifi_clients_7.0.yaml`. It creates a Zabbix host for each UniFi client
that matches your filter: Wi-Fi network name (SSID), client name or hostname, MAC vendor, site — in any combination.
Each client host gets signal, rates, retries, CCQ, MCS, Experience, roaming history (30 days), ICMP ping and UniFi anomalies.
Made for stationary devices with fixed MAC addresses (scanners, terminals, IoT controllers, PTP bridges):
random (private) MAC addresses are skipped by default, and a safety limit (200 clients per filter) protects Zabbix
from a too broad filter. The main template is not changed.

**RU.** Новый необязательный шаблон `template/zbx_unifi_wifi_clients_7.0.yaml`. Он создаёт хост Zabbix для каждого клиента UniFi,
подходящего под ваш фильтр: имя Wi-Fi сети (SSID), имя клиента или hostname, производитель MAC, сайт — в любом сочетании.
На каждом хосте клиента — сигнал, скорости, повторы, CCQ, MCS, Experience, журнал роуминга (30 дней), ICMP ping и аномалии UniFi.
Для стационарных устройств с постоянным MAC (сканеры, терминалы, IoT-контроллеры, PTP-мосты): случайные (приватные) MAC
по умолчанию пропускаются, а предохранитель (200 клиентов на фильтр) защищает Zabbix от слишком широкого фильтра.
Основной шаблон не изменён.

### Install / Установка
1. Global macros `{$UNIFI_NETWORK_API_ADDRESS}` and `{$UNIFI_NETWORK_API_KEY}` (same as for the main template) /
   Глобальные макросы `{$UNIFI_NETWORK_API_ADDRESS}` и `{$UNIFI_NETWORK_API_KEY}` (те же, что для основного шаблона).
2. Import `zbx_unifi_wifi_clients_7.0.yaml` / Импортируйте `zbx_unifi_wifi_clients_7.0.yaml`.
3. Create a host, link **UniFi Network API - Wi-Fi Clients Filter**, set `{$FILTER.SSID}` / `{$FILTER.NAME}` / `{$FILTER.OUI}`,
   `{$FILTER.CATEGORY}` and `{$FILTER.PREFIX}` /
   Создайте хост, привяжите **UniFi Network API - Wi-Fi Clients Filter**, задайте `{$FILTER.SSID}` / `{$FILTER.NAME}` / `{$FILTER.OUI}`,
   `{$FILTER.CATEGORY}` и `{$FILTER.PREFIX}`.

Full guide / Полная инструкция: `docs/WIFI_CLIENTS.md` (EN) · `docs/WIFI_CLIENTS.ru.md` (RU).

### Check after import / Проверьте после импорта
- *Discovery Filtered Clients*: disable lost resources after 30d, delete after 31d /
  отключать потерянные ресурсы через 30d, удалять через 31d.
- ICMP ping needs a route from Zabbix to the client networks and `fping`; otherwise set `{$TRIGGER_CLIENT_PING}` = `0` /
  ICMP ping требует маршрута от Zabbix в сети клиентов и `fping`; иначе задайте `{$TRIGGER_CLIENT_PING}` = `0`.

### Requirements / Требования
Zabbix 7.0 · UniFi Network 9.x+ on UniFi OS · network access from Zabbix server/proxy to the console.
