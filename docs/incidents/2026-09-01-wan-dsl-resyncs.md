# Analyse: Wiederholte VDSL2-DSL-Resyncs nach Modemwechsel

**Datum:** 2026-09-01
**Status:** Offen (Leitung instabil) — Telekom-Störungsmeldung vorbereitet
**Symptom:** Langsamer/abgebrochener externer Foto-Upload + wiederholte kurze Internet-Ausfälle

## Zusammenfassung
Am 31.08.2026 wurde das DSL-Modem getauscht (FRITZ!Box → **DrayTek Vigor 166** als Bridge;
Router/PPPoE = **UniFi UCG Max**). Seitdem **resynchronisiert die VDSL2-Leitung wiederholt**
(je ~1–2 Min Ausfall). Das erklärt den zähen Upload und das „gerade rausgeflogen". Die Leitung
ist im synchronen Betrieb **elektrisch sauber** (keine ES/SES/UAS/LOS), Ursache ist die
**Leitungs-/DLM-Profilierung nach dem Modemwechsel** bei grenzwertig niedriger SNR-Reserve.
**Nicht** beteiligt: WLAN, DNS/AdGuard, Synology, UniFi-Konfiguration, Hausverkabelung.

## Setup / Topologie (aktuell)
- **DrayTek Vigor 166**: reines Bridge-Modem, VDSL2 Profil 35b, Firmware 4.2.7 (aktuell).
  Erreichbar unter `192.168.178.2`. Die Zeile „IPv4 Internet Access WAN1 … Disconnected" ist
  im Bridge-Modus **normal** (das Modem wählt sich nicht selbst ein).
- **UCG Max**: macht die **PPPoE-Einwahl** (`ppp0`, öffentliche IP, Peer = Telekom-BNG).
  Kein Doppel-NAT mehr (das gab es nur in der alten FRITZ!Box-Zeit).
- Gebuchte Rate ~250/40 Mbit/s.

## Belegte Zeitachse der Resyncs (01.09.2026)
Quelle: UCG `/var/log/messages`, `wan-failover-group-base … is down/up (ppp0)`, bestätigt durch
Rücksprünge der **DSL Up Time** im DrayTek.

| Ausfall (down) | wieder up | Dauer |
|---|---|---|
| 06:34:38 | 06:36:28 | ~1m50s |
| 06:38:32 | 06:40:27 | ~1m55s |
| 07:12:46 | 07:14:44 | ~2m (echter DSL-Resync, DrayTek Up-Time bestätigt) |
| 13:01:39 | 13:03:42 | ~2m |
| 13:04:40 | 13:06:29 | ~2m |
| 13:33:12 | — | — |
| 13:39:25 | — | — |
| 13:57:32 | — | — |
| 14:13:07 | — | — |

**9 Ausfälle**, nachmittags **zunehmend** (6 in ~70 Min zwischen 13:01 und 14:13) → beruhigt
sich nicht von selbst.

## Leitungswerte (DrayTek → Diagnostics → DSL Status)
- Actual Rate: **251.298 / 43.992 Kbps** (Down/Up); Attainable **287.948 / 47.761 Kbps**
- **SNR-Reserve: ~6 dB Upstream / 7 dB Downstream** (grenzwertig niedrig)
- **ES = 0, SES = 0, UAS = 0, LOS/LOF/LPR/NCD/LCD-Failure = 0**; CRC niedrig (61/29)
- FEC-Korrekturen (FECS) vorhanden, aber harmlos (werden korrigiert)
- Modem-Firmware **4.2.7** (aktuell)

## Root Cause
**Leitungs-/DLM-Profilierung nach dem Modemwechsel** bei knapper SNR-Reserve:
- Ein Modemwechsel setzt bei der Telekom das **DLM-Leitungsprofil** zurück; die vorher über
  Monate eingespielte FRITZ!Box-Profilierung ist weg, die Leitung trainiert neu.
- Bei nur ~6 dB SNR-Reserve fehlt Störabstand → wiederholte Resyncs.
- **Verkabelung ausgeschlossen:** Die FRITZ!Box lief an identischer Hausverkabelung stabil.
- Da ES/SES/UAS = 0 sind, ist die Leitung im Betrieb sauber — die Resyncs sind das Problem,
  nicht die Übertragungsqualität.

## Bezug zum langsamen Upload (ehrliche Einordnung)
Der eigentliche Upload lief laut Nutzer erst ab ~08:xx. Im UCG-Log gab es in diesem Fenster
**keine** WAN-Drops; jene konkrete Verzögerung war daher eher **Bufferbloat/transient**, nicht
ein Resync. Generell gilt aber: eine im Minutentakt resyncende Leitung zerlegt jeden längeren
Upload — daher ist die Leitungsstabilität die vorrangige Baustelle.

## Wo man den Status sieht
- **DSL-Leitung:** DrayTek → *Online Status → Physical Connection* → **State (`SHOWTIME`)** und
  **Up Time**. Up Time springt auf 0 = Resync.
- **Internet (PPPoE):** UCG/UniFi-Dashboard (ISP-Status, Abbruch-Warnung) bzw. UCG-Log
  (`wan-failover … is down` — mit echter Uhrzeit; der DrayTek hat keine NTP-Zeit).

## Maßnahmen
1. **Telekom-Störung melden** (Meldungstext siehe Anhang) — Resyncs nehmen zu, nicht abwarten.
2. **Beobachten:** DSL Up Time / Ausfallliste weiterführen als Beleg.
3. **SQM/Smart Queues** auf dem UCG (Upload ~38–40 Mbit deckeln) gegen Bufferbloat — separat,
   behebt die Resyncs nicht.
4. **Große Uploads** bis zur Stabilisierung über **5G**.
5. Firmware ist bereits aktuell (4.2.7); Kabel wurde neu gesteckt.

## SQM / Bufferbloat — behoben (Update 01.09. 15:0x)
Am UCG Max wurden **Smart Queues** aktiviert (WAN → Telekom):
- **Download 220 / Upload 40 Mbit/s** (knapp unter Sync 251/44).
- Bufferbloat-Test (waveform.com): **Note A+** — Ruhe 9 ms, unter Download-Last **+5 ms**,
  unter Upload-Last **+0 ms**.
- Verifikation: externer Foto-Upload **340 MB / 115 Fotos, 15:03–15:11** lief **ohne Abbruch**
  durch (Durchsatz schwankte durch Klein-Datei-Overhead, aber stetiger Fortschritt, kein Resync
  im Fenster).

**Klarstellung:** SQM behebt **Bufferbloat** (Latenz-/Durchsatzeinbruch bei Sättigung), **nicht**
die DSL-Resyncs. Config-Gegencheck ergab zudem: Bridge-Modus ok, **VLAN 7 genau einmal getaggt**
(im DrayTek, VDSL2/G.fast Customer-Tag = 7; UCG-WAN „VLAN-ID" bewusst aus), PPPoE + MTU 1492
korrekt. Die Resyncs sind also **kein Konfig-Fehler** → Telekom-Meldung bleibt der Weg.

---

## Anhang: Telekom-Störungsmeldung (Platzhalter ausfüllen)

> **Betreff:** Störungsmeldung VDSL2 – wiederholte DSL-Resyncs seit Modemwechsel
>
> Sehr geehrte Damen und Herren,
>
> seit einem Modemwechsel am **31.08.2026** kommt es an meinem VDSL2-Anschluss zu **wiederholten
> DSL-Neusynchronisierungen (Resyncs)** mit jeweils ca. 1–2 Minuten Internet-Ausfall.
>
> **Anschluss / Kunde:**
> - Anschlussinhaber: [Name]
> - Adresse / Anschluss: [Straße, PLZ Ort]
> - Kundennummer / Anschlusskennung: [eintragen]
> - Rückruf: [Telefon]
>
> **Technik:**
> - VDSL2 (Super-Vectoring, Profil 35b), gebucht ca. 250/40 Mbit/s
> - Modem neu: DrayTek Vigor 166 (Bridge, Firmware 4.2.7); Router dahinter macht die PPPoE-Einwahl
> - Vorher: FRITZ!Box an identischer Hausverkabelung – lief stabil
>
> **Leitungswerte (am Modem ausgelesen):**
> - Sync 251/44 Mbit/s (Attainable 288/48)
> - SNR-Reserve nur ~6 dB Upstream / 7 dB Downstream
> - Fehlerzähler ES/SES/UAS/LOS/LOF = 0 (Leitung im Betrieb sauber, nur FEC-Korrekturen)
>
> **Ausfälle am 01.09.2026 (~1–2 Min):** 06:34, 06:38, 07:12, 13:01, 13:04, 13:33, 13:39,
> 13:57, 14:13 – nachmittags zunehmend.
>
> **Bitte:** Da die Leitung im Betrieb fehlerfrei ist (ES/SES = 0), meldet ein automatischer
> Leitungstest evtl. „in Ordnung“ – die Resyncs sind jedoch real (siehe Zeitstempel). Ich bitte
> um Prüfung der DLM-Profilierung für das neue Modem bzw. Einstellung eines stabileren Profils
> mit höherer SNR-Reserve sowie Prüfung der Leitung auf die Resync-Ursache.
>
> Vielen Dank und freundliche Grüße
> [Name]

> Hinweis: Dieses Dokument enthält bewusst keine Zugangsdaten/Secrets (keine Kundennummer,
> kein WLAN-/PPPoE-Passwort) — nur Platzhalter.
