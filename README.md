# LSTack OS – Proof-of-Concept (PoC)

## Überblick
**LSTack OS** ist ein modulares, vollständig mit Open-Source-Komponenten aufgebautes Leitstellen-System.  
Ziel ist es, die wesentlichen Funktionen einer Einsatzleitstelle – wie **Einsatzverwaltung, Ressourcen-Status, Kartenanzeige und Schnittstellen zu Funk/Telefon** – mit einem klar strukturierten, sicheren und skalierbaren Architekturansatz nachzubilden.  

Das gesamte System wird mit **Ansible** installiert und konfiguriert.  
Die Architektur ist so aufgebaut, dass alle Komponenten klein und unabhängig in **Rollen** gekapselt sind. Dadurch kann man:
- Im **PoC** alle Rollen auf einer einzigen VM laufen lassen (Playbook `90_all_in_one.yml`)
- Später jede Rolle auf **eigene Server/VMs** verteilen, ohne dass Code geändert werden muss
- **Gateways** je nach Funktion modular zu- oder abschalten (z.B. Telefon, TETRA, Wetter, Geocoding)

---

## Prinzipien
- **Trennung von Software, Daten, Logs** (FHS-konform):  
  - `/opt/lstack/<rolle>` → Software  
  - `/var/lib/lstack/<rolle>` → Daten, Cache  
  - `/var/log/lstack/<rolle>` → Logs  
- **System-Account**:  
  - Benutzer: `svc_lst`  
  - Gruppe: `grp_lst`  
  - UID/GID < 1000 (kompatibel für Elastic Agent-Auswertung)  
- **Secrets im Vault**:  
  - DB-Passwörter, API-Keys, TLS, LDAP-Bind-Konto  
- **RBAC & LDAP/LDAPS**:  
  - Anmeldung der Disponenten am Client gegen AD/LDAP  
  - AD-Gruppen werden auf **App-Rollen** gemappt (Rettungsdienst, Feuerwehr, Lagedienst, Datenpflege, Admin)  
  - Offline-Cache möglich, falls LDAP kurzzeitig nicht erreichbar  
- **DB-Cluster mit VIP**:  
  - Client/Anwendungen sprechen nur die VIP an  
  - Failover-Logik liegt außerhalb (Postgres-Cluster)  
- **Backup & Monitoring Pflicht**:  
  - Automatisierte Backups über eigene Rolle  
  - Statistik-Webserver (nginx + Flask) liefert Health-/Statistikinfos  

---

## Ziel des PoC
1. **Minimal lauffähiges System**:  
   - `database` (Postgres/PostGIS, Schema)  
   - `auth_ldap_client` (LDAP-Anbindung für Login & RBAC)  
   - `client` (Qt-App, DB- & LDAP-Integration)  
   - `gateway_weather` & `gateway_geocode` (Basisdatenquellen)  
   - `webserver_stats` (Health/Statistiken)  
   - `backup` (Dumps & Filesicherungen)  
   - optional: `tile_server` (Offline-Karten)

2. **Bonusmodule vorbereitet, modular zuschaltbar**:  
   - `gateway_phone` (Telefonanlage, Headset)  
   - `gateway_tetra_internal` (Wachfunk)  
   - `gateway_tetra_external` (BOS-Netz)  
   - `gateway_alarmprinter` (PDF-Ausgabe, Messenger)

---

## Struktur
```
ansible/
├── inventories/
│   └── hosts.yml
├── group_vars/
│   └── all.yml                # minimal: UID/GID, env
├── vault.yml                  # alle Secrets
├── playbooks/
│   ├── 10_database.yml
│   ├── 15_auth_ldap_client.yml
│   ├── 20_client.yml
│   ├── 30_gateway_weather.yml
│   ├── 31_gateway_geocode.yml
│   ├── 32_gateway_phone.yml
│   ├── 33_gateway_tetra_internal.yml
│   ├── 34_gateway_tetra_external.yml
│   ├── 35_gateway_alarmprinter.yml
│   ├── 40_tile_server.yml
│   ├── 50_webserver_stats.yml
│   ├── 60_backup.yml
│   └── 90_all_in_one.yml
└── roles/
    ├── database/               # PostgreSQL/PostGIS
    ├── auth_ldap_client/       # LDAP/LDAPS-Integration
    ├── client/                 # Leitstellen-Client
    ├── gateway_weather/
    ├── gateway_geocode/
    ├── gateway_phone/
    ├── gateway_tetra_internal/
    ├── gateway_tetra_external/
    ├── gateway_alarmprinter/
    ├── tile_server/
    ├── webserver_stats/
    └── backup/
```

## Rollen – Übersicht & Variablen

### database
- **Zweck**: DB + Schema (Einsätze, Ressourcen, Rollen)
- **Pfade**: `/opt/lstack/database`, `/var/lib/lstack/database`, `/var/log/lstack/database`
- **Variablen**: `database_vip`, `database_name`, `database_user_client`, `database_password_client`, `database_enable_postgis`, `database_paths`

### auth_ldap_client
- **Zweck**: LDAP-/LDAPS-Login & RBAC (Mapping AD-Gruppen → App-Rollen)
- **Variablen**: `auth_ldap_uri`, `auth_ldap_base_dn`, `auth_ldap_bind_dn`, `auth_ldap_bind_pw`, `auth_ldap_role_map`, `auth_offline_cache_ttl`

### client
- **Zweck**: Qt-Client, GUI für Einsätze/Status/Karte
- **Pfade**: `/opt/lstack/client`, `/var/log/lstack/client`
- **Variablen**: `client_package_url`, `client_install_dir`, `client_config.db_*`, `client_config.ldap_*`, `client_runtime_user/group`

### gateway_weather
- **Zweck**: Wetterdaten via API → DB
- **Variablen**: `gateway_weather_api_url`, `gateway_weather_api_key`, `gateway_weather_interval`, `gateway_weather_target_table`

### gateway_geocode
- **Zweck**: Geocoding (Adresse ↔ Koordinaten) via API → DB
- **Variablen**: `gateway_geocode_api_url`, `gateway_geocode_interval`, `gateway_geocode_target_table`

### gateway_phone (Bonus)
- **Zweck**: Telefonanlage (z.B. Asterisk) für Call-Anzeige + Headset-Annahme
- **Variablen**: `gateway_phone_engine`, `gateway_phone_sip_user`, `gateway_phone_sip_password`, `gateway_phone_cti_table`

### gateway_tetra_internal (Bonus)
- **Zweck**: Anbindung interner Wachfunk (serielles Gerät)
- **Variablen**: `gateway_tetra_internal_device`, `gateway_tetra_internal_protocol`, `gateway_tetra_internal_table`

### gateway_tetra_external (Bonus)
- **Zweck**: BOS-TETRA Netz via API
- **Variablen**: `gateway_tetra_external_api_url`, `gateway_tetra_external_api_key`, `gateway_tetra_external_table`

### gateway_alarmprinter (Bonus)
- **Zweck**: Alarmdrucker (PDF-Erstellung) + Messenger (Matrix/Threema)
- **Variablen**: `gateway_alarmprinter_output_dir`, `gateway_alarmprinter_messenger`, `gateway_alarmprinter_matrix_*`, `gateway_alarmprinter_jobs_table`

### tile_server (optional)
- **Zweck**: OSM-Kacheln offline bereitstellen
- **Variablen**: `tile_data_dir`, `tile_region`, `tile_db_user`, `tile_db_password`

### webserver_stats
- **Zweck**: Statistik-/Health-Webserver (nginx + Flask)
- **Variablen**: `webserver_stats_port`, `webserver_stats_auth_user`, `webserver_stats_auth_password`, `webserver_stats_dashboards`

### backup
- **Zweck**: Backup & Restore (DB + Files)
- **Variablen**: `backup_target_dir`, `backup_retention_days`, `backup_cron_time`, `backup_include_paths`, `backup_db.*`, `restore_source_dir`

---

## PoC vs Bonus
- **PoC (Minimal)**: database, auth_ldap_client, client, gateway_weather, gateway_geocode, webserver_stats, backup (+ optional tile_server)
- **Bonus (modular)**: gateway_phone, gateway_tetra_internal, gateway_tetra_external, gateway_alarmprinter

---

## Rolle-READMEs
Jede Rolle enthält ein eigenes README.md mit:
- Kurzbeschreibung
- Variablenliste (mit Defaults)
- Hinweis auf FHS-Trennung & Vault-Nutzung
- Beispiel-Playbook

**Links (Platzhalter)**:
- [roles/database/README.md](roles/database/README.md)
- [roles/auth_ldap_client/README.md](roles/auth_ldap_client/README.md)
- [roles/client/README.md](roles/client/README.md)
- [roles/gateway_weather/README.md](roles/gateway_weather/README.md)
- [roles/gateway_geocode/README.md](roles/gateway_geocode/README.md)
- [roles/gateway_phone/README.md](roles/gateway_phone/README.md)
- [roles/gateway_tetra_internal/README.md](roles/gateway_tetra_internal/README.md)
- [roles/gateway_tetra_external/README.md](roles/gateway_tetra_external/README.md)
- [roles/gateway_alarmprinter/README.md](roles/gateway_alarmprinter/README.md)
- [roles/tile_server/README.md](roles/tile_server/README.md)
- [roles/webserver_stats/README.md](roles/webserver_stats/README.md)
- [roles/backup/README.md](roles/backup/README.md)
