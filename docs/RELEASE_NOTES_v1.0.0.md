## Zabbix_unifi_os_ap_api_7.0 — v1.0.0

First release / Первый релиз.

**EN.** Zabbix 7.0 template for UniFi Network: automatic discovery of all sites and access points, AP health,
per-radio metrics (channel utilization, Tx retry, dropped, EIRP), uplink, clients per AP and UniFi anomalies.
API key authentication with HashiCorp Vault support. Thresholds tuned for industrial sites.

**RU.** Шаблон Zabbix 7.0 для UniFi Network: автоматическое обнаружение всех сайтов и точек доступа, состояние точек,
метрики каждого радиомодуля (утилизация канала, Tx Retry, потери, EIRP), uplink, клиенты на точке и аномалии UniFi.
Подключение по API-ключу с поддержкой HashiCorp Vault. Пороги подобраны для промышленных объектов.

### Install / Установка
1. Download `zbx_unifi_network_api_7.0.yaml` / Скачайте `zbx_unifi_network_api_7.0.yaml`.
2. Create global macros `{$UNIFI_NETWORK_API_ADDRESS}` and `{$UNIFI_NETWORK_API_KEY}` (Vault secret) /
   Создайте глобальные макросы `{$UNIFI_NETWORK_API_ADDRESS}` и `{$UNIFI_NETWORK_API_KEY}` (секрет Vault).
3. Import the template, create a host, link **UniFi Network API** /
   Импортируйте шаблон, создайте хост, привяжите **UniFi Network API**.

Full guide / Полная инструкция: `README.md` (EN) · `README.ru.md` (RU) in the repository / в репозитории.

### Requirements / Требования
Zabbix 7.0 · UniFi Network 9.x+ on UniFi OS (Integration API) · network access from Zabbix server/proxy to the console.

### Credits / Благодарности
Based on / Основан на [zbx_unifi_network_api](https://github.com/MassimilianoPasquini97/zbx_unifi_network_api) by Massimiliano Pasquini (MIT).
