# ADR-0007: Config Persistence — LittleFS + NVS Backup

Status: Accepted

## Context

Configuration (WiFi credentials, MQTT settings, RF remote registrations, logging/HA discovery settings) must survive reboots and OTA updates. Evidence: `main/config/Config.h` declares JSON-file-based `load*Config`/`save*Config` functions for `SystemConfig`, `WifiConfig`, `LogConfig`, `HADiscConfig`; `IthoRemote` config additionally has a `nvs_backup_key` parameter (`Config.h:39`, `loadFileRemotes(..., nvs_backup_key, ...)`) providing an NVS (ESP-IDF non-volatile key/value storage) fallback specifically for remote registrations, which are higher-value to protect against filesystem corruption than general settings.

## Decision

Use LittleFS JSON files (via ArduinoJson `JsonDocument`) as the primary config store for all config structs, with an additional NVS-backed redundancy path specifically for RF remote registrations.

## Consequences

**Positive**: JSON files are human-readable/debuggable (visible via the web UI file browser, `handlers/FileHandlers.cpp`), easy to extend with new fields; NVS backup protects the highest-cost-to-lose data (remote pairings, which require physical re-binding to recover) against filesystem-level issues.

**Negative**: two persistence mechanisms means remote-registration code paths must keep both in sync; general settings have no such backup, so filesystem corruption there requires falling back to defaults/reconfiguration through the web UI.

## Alternatives Considered

- **NVS for all config**: rejected — NVS key/value storage doesn't suit large structured JSON documents as well as a filesystem-JSON approach; not evidenced as attempted for non-remote config.
- **LittleFS-only, no NVS backup anywhere**: rejected specifically for remotes given the `nvs_backup_key` mechanism exists — the extra redundancy was deliberately added for that one high-value data set.

## Related Components

`main/config/Config.h`, `main/config/SystemConfig.h`, `main/config/WifiConfig.h`, `main/config/IthoRemote.h`, `main/handlers/FileHandlers.cpp`

## Related Issues

None tracked.

## AI Notes

New config fields go on the existing structs + JSON load/save functions, following the established `load*Config`/`save*Config` naming (see [.ai/coding-standards.md](../../.ai/coding-standards.md)). Only add NVS backup for a new config category if it has the same "expensive to recover, cheap to duplicate" profile as remote registrations — don't add NVS backup by default for every config addition.
