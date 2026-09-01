# Incident-Protokoll: Ausfall WLAN „Fichtelrouter" (IoT-Geräte)

**Datum:** 2026-09-01
**Status:** Behoben
**Betroffen:** Alle WLAN-IoT-Geräte (Shelly, Meross, Anker) am SSID „Fichtelrouter"
**Nicht betroffen:** Home Assistant (LAN), Clients auf „FichtenUnifi"/„FichtelGast"

## Zusammenfassung
Am Morgen des 2026-09-01 verloren schlagartig alle IoT-WLAN-Geräte die Verbindung. Der
ursprüngliche Verdacht fiel auf AdGuard (`192.168.178.5`). Die Auswertung von UniFi-Syslog,
zwei AP-Support-Dateien (U7 Long-Range, U7 Outdoor) und den Controller-Logs des Gateways
(UCG Max) ergab eindeutig: Das 2,4-GHz-only-WLAN „Fichtelrouter" wurde um **08:52:40** über
einen versehentlich angetippten **„Play/Pause"-Shortcut in der UniFi-iPhone-App** deaktiviert
(`enabled=false`). Der AP entfernte daraufhin die zugehörige Funkschnittstelle, und die Geräte
fielen weg. Nach dem Wieder-Aktivieren des WLANs kamen alle Geräte automatisch zurück.

## Zeitachse (belegt)
| Zeit (lokal, CEST) | Ereignis | Quelle |
|---|---|---|
| 2026-08-31 ~22:01 | IoT-Geräte mit „Fichtelrouter" verbunden (Basis für „10h 51m verbunden") | Disconnect-Event |
| 2026-09-01 08:52:40.737 | `PUT /proxy/network/api/s/default/rest/wlanconf/6a57b573efbb365653ff243a` von `ip=192.168.178.234`, user „UniFi User" (iPhone) | `nginx-access.log` |
| 2026-09-01 08:52:42 | U7 Outdoor wendet Config an: 2,4-GHz-Virtual-AP „Fichtelrouter" (`wifi0ap1`) vollständig entfernt | AP fastapply-Diff `20260901085242` |
| 2026-09-01 08:52:47 | Erster Client getrennt: „WiFi Client Disconnected" (Carport-IT/Shelly), RSSI −51 dBm, Kanal 1, Band ng | UniFi-Syslog (CEF) |
| 2026-09-01 08:57:11 | Weiterer Client getrennt (Gartenhaus/Shelly), RSSI −62 dBm | UniFi-Syslog (CEF) |
| 2026-09-01 ~10:4x | WLAN „Fichtelrouter" wieder aktiviert → Geräte kehren zurück | Behebung |

## Ursache
Versehentlich angetippter **WLAN-„Play/Pause"-Shortcut** auf dem Dashboard der UniFi-App.
Der Shortcut war einige Tage zuvor beim Erkunden über „+ Shortcut hinzufügen" angelegt worden.
Mehrfaches Antippen schaltete das WLAN um; zuletzt auf **aus**. Ein solcher Shortcut setzt
direkt `wlanconf.enabled=false` — es war **keine** verstellte Einstellung (z. B. Min-RSSI,
Datenrate, Multicast) und **kein** Ausfall.

## Ausgeschlossene Ursachen (mit Begründung)
- **AdGuard / DNS:** kann keine Clients aus dem WLAN werfen; das war ein reines Funk-/Config-Ereignis.
- **Home Assistant (UniFi-Integration, `192.168.178.164`):** im gesamten Zeitfenster nur
  `GET .../trafficrules` und `.../trafficroutes` — nie ein `PUT` auf `wlanconf`.
- **Min-RSSI / Datenrate / Multicast-Tuning:** im AP-Diff wurde die komplette Schnittstelle
  **entfernt**, kein Einzelwert geändert; zudem war das Signal exzellent (−51/−62 dBm), sodass
  kein RSSI-Schwellwert die Geräte hätte trennen können.
- **U7 Long-Range:** anderer AP, andere SSID, anderer 2,4-GHz-Kanal (6 statt 1); der Reboot
  dieses AP erfolgte manuell zu Testzwecken und lag nach dem Vorfall.
- **Foto-Upload / ISP:** die zeitgleich hohe Airtime/Interference war nur Begleitrauschen,
  nicht die Ursache.

## Behebung
WLAN „Fichtelrouter" im UniFi-Controller wieder aktiviert — alle IoT-Geräte verbanden sich
automatisch neu (SSID + Passwort unverändert gespeichert).

## Vorbeugung
- „Play/Pause"-Shortcut für „Fichtelrouter" **entfernt**.
- Keine solchen Ein/Aus-Shortcuts für kritische (IoT-)WLANs anlegen.
- Optional: Home-Assistant-Alarm, wenn mehrere IoT-Entitäten gleichzeitig `unavailable` werden.
- Optional: zweiter DNS als Fallback im DHCP, falls AdGuard einmal ausfällt.

> Hinweis: Bewusst ohne Zugangsdaten/Secrets dokumentiert (WLAN-Passwort nicht enthalten).
