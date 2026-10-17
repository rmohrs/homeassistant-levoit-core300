# Luftreiniger Wohnzimmer – Levoit Core 300S in Home Assistant

Automatische Steuerung des Levoit Core 300S nach Feinstaub (PM2,5), Raumbelegung und Uhrzeit, plus WhatsApp-Meldung beim Filterwechsel.

Die Anleitung setzt voraus, dass du dich in Home Assistant schon etwas auskennst: Du weißt, was Entitäten, Helfer und Automationen sind, und hast schon einmal eine Automation im Editor angelegt.

---

## Inhalt

1. [Verhalten auf einen Blick](#1-verhalten-auf-einen-blick)
2. [Voraussetzungen](#2-voraussetzungen)
3. [Geräte und Entitäten](#3-geräte-und-entitäten)
4. [Integrationen](#4-integrationen)
5. [Helfer](#5-helfer)
6. [Automation: Steuerung](#6-automation-luftreiniger-wohnzimmer-steuerung)
7. [Automation: Filterwechsel-Meldung](#7-automation-luftreiniger-wohnzimmer-filterwechsel-meldung)
8. [Dashboard](#8-dashboard)
9. [Bedienung und Anpassung](#9-bedienung-und-anpassung)
10. [Einrichtung](#10-einrichtung)
11. [Fehlersuche](#11-fehlersuche)
12. [Dateien](#12-dateien)

---

## 1. Verhalten auf einen Blick

| Situation | Reiniger |
|---|---|
| PM2,5 unter Ausschaltschwelle (12 µg/m³) | aus |
| Belastet, Raum belegt | Stufe 1 |
| Belastet, Raum seit 5 min frei | Stufe 2 |
| Über 55 µg/m³, Raum frei | Stufe 3 |
| 00:00–06:00, belastet | Sleep |
| 00:00–06:00, belastet, PIR meldet Bewegung | Stufe 1 für 20 min (verlängert sich bei weiterer Bewegung), danach Sleep |
| Filter ≤ 5 % | WhatsApp-Nachricht |

**Hysterese:** Eingeschaltet wird über **25 µg/m³**, ausgeschaltet erst unter **12 µg/m³**. Dadurch pendelt der Reiniger nicht um einen einzelnen Grenzwert.

Alle Schwellen und die Boost-Dauer sind Helfer und lassen sich ohne Eingriff in die Automation ändern.

---

## 2. Voraussetzungen

- Home Assistant **2026.7 oder neuer**. Die Automation nutzt die zweckbezogenen Auslöser `occupancy.detected`, `occupancy.cleared`, `motion.detected` und `timer.finished`.
- VeSync-Integration mit dem Core 300S
- Präsenzmelder mit getrennter Belegungs- und Bewegungserkennung (hier Aqara FP300 über Zigbee2MQTT)
- Ein Benachrichtigungsdienst `notify.whatsapp` (nur für die Filtermeldung)

---

## 3. Geräte und Entitäten

### Levoit Core 300S („Airfresh“, Bereich Wohnzimmer)

Anbindung über WLAN und die VeSync-Cloud.

| Entität | Verwendung |
|---|---|
| `fan.airfresh` | Wird geschaltet. 3 Stufen als Prozent: 33 / 66 / 100. Presets: `sleep`, `auto`. |
| `sensor.airfresh_pm25` | Führungsgröße der Steuerung (µg/m³) |
| `sensor.airfresh_luftqualitat` | Nur Anzeige im Dashboard |
| `sensor.airfresh_restlebensdauer_des_filters` | Auslöser der Filtermeldung (%) |

### Aqara Presence Sensor FP300 („Raum Wohnzimmer“, Bereich Wohnzimmer)

Anbindung über Zigbee2MQTT.

| Entität | Verwendung |
|---|---|
| `binary_sensor.raum_wohnzimmer_presence` | Belegung (mmWave-Radar, erkennt auch Stillsitzen). Tagsteuerung Stufe 1 vs. 2/3. |
| `binary_sensor.raum_wohnzimmer_pir_detection` | Bewegung (PIR). Löst nachts den Boost aus. |

**Warum zwei Sensoren?** Tagsüber zählt, *ob jemand da ist* – dafür ist das Radar zuverlässiger. Nachts soll der Reiniger nur hochdrehen, wenn sich jemand *bewegt*, nicht weil jemand auf dem Sofa schläft.

### Alle verwendeten Sensoren auf einen Blick

| Sensor | Gerät / Herkunft | Geräteklasse | Einheit / Werte | Verwendet in |
|---|---|---|---|---|
| `sensor.airfresh_pm25` | Levoit Core 300S (VeSync) | `pm25` | µg/m³ | Steuerung: Ein-/Ausschalten, Stufe 3 |
| `sensor.airfresh_luftqualitat` | Levoit Core 300S (VeSync) | – | Text, z. B. `good`, `poor` | Dashboard |
| `sensor.airfresh_restlebensdauer_des_filters` | Levoit Core 300S (VeSync) | – | % | Filterwechsel-Meldung |
| `binary_sensor.raum_wohnzimmer_presence` | Aqara FP300, Radar (Zigbee2MQTT) | `occupancy` | `on` = belegt, `off` = frei | Steuerung: Stufe 1 vs. 2/3 |
| `binary_sensor.raum_wohnzimmer_pir_detection` | Aqara FP300, PIR (Zigbee2MQTT) | `motion` | `on` = Bewegung, `off` = keine | Steuerung: Nacht-Boost |
| `sensor.luftreiniger_wohnzimmer_status` | Template-Helfer | – | Text, z. B. „Stufe 1“, „Nachtmodus“ | Anzeige, frei verwendbar |

Gesteuert wird nur ein Aktor: `fan.airfresh`. Die übrigen Sensoren der beiden Geräte (z. B. Temperatur, Luftfeuchte und Helligkeit des FP300) werden hier nicht verwendet.

---

## 4. Integrationen

| Integration | Rolle |
|---|---|
| **VeSync** | Luftreiniger. Cloud-basiert, daher kurze Ausfälle möglich und Befehle sparsam einsetzen. |
| **MQTT / Zigbee2MQTT** | Präsenzmelder |
| **Template** | Status-Sensor (UI-Helfer, kein YAML) |
| **Input Boolean / Input Number / Timer** | Helfer (eingebaut) |
| **notify.whatsapp** | Bestehender Dienst, auch von den Heizungs-Automationen genutzt. Seine Einrichtung gehört nicht zu diesem Projekt. |

---

## 5. Helfer

Alle im Bereich Wohnzimmer, angelegt über die Oberfläche. Details zum Nachbauen: [`helpers/helfer.yaml`](helpers/helfer.yaml).

| Entität | Typ | Wert | Zweck |
|---|---|---|---|
| `input_boolean.luftreiniger_wohnzimmer_automatik` | Schalter | an | Hauptschalter. Aus = Automation greift nicht ein. |
| `input_number.luftreiniger_wohnzimmer_einschaltschwelle` | Zahl (Box) | 25 µg/m³ | Reinigung starten über … |
| `input_number.luftreiniger_wohnzimmer_ausschaltschwelle` | Zahl (Box) | 12 µg/m³ | Reinigung beenden unter … |
| `input_number.luftreiniger_wohnzimmer_schwelle_stufe_3` | Zahl (Box) | 55 µg/m³ | Stufe 3 über … (nur Raum frei) |
| `timer.luftreiniger_wohnzimmer_nacht_boost` | Timer | 20 min, `restore: true` | Dauer des Nacht-Boosts |
| `sensor.luftreiniger_wohnzimmer_status` | Template-Sensor | Text | Klartext-Zustand: „Stufe 1“, „Nachtmodus“, „Aus – Luft sauber“, „Manuell – läuft/aus“, „Nicht erreichbar“ |

---

## 6. Automation „Luftreiniger Wohnzimmer Steuerung“

Datei: [`automations/luftreiniger_wohnzimmer_steuerung.yaml`](automations/luftreiniger_wohnzimmer_steuerung.yaml) · Modus: `restart`

### Prinzip

Eine einzige Automation statt vieler kleiner: Jeder Auslöser führt zur selben Prüfung, die den **Soll-Zustand aus der aktuellen Lage berechnet**. Welcher Auslöser es war, spielt für das Ergebnis keine Rolle. Dadurch gibt es keine widersprüchlichen Automationen, und nach einem Neustart stimmt der Zustand sofort wieder.

### Auslöser

| ID | Auslöser |
|---|---|
| `pm` | `sensor.airfresh_pm25` ändert sich |
| `belegt` | `occupancy.detected` auf dem Präsenzmelder |
| `frei` | `occupancy.cleared`, 5 min stabil |
| `nacht_start` / `nacht_ende` | 00:00 / 06:00 |
| `automatik_an` | Automatik-Schalter wird eingeschaltet |
| `schwellen` | Einer der drei Schwellen-Helfer ändert sich |
| `start` | Home Assistant startet |
| `bewegung` | `motion.detected` auf dem PIR |
| `boost_ende` | `timer.finished` des Nacht-Boosts |
| `minutentakt` | `time_pattern` jede Minute |

### Bedingungen

- Automatik ist an
- `fan.airfresh` ist `on` oder `off` (also erreichbar)
- PM2,5 ist nicht `unavailable`/`unknown`

### Ablauf

```
Reinigen nötig?
  PM2,5 > Einschaltschwelle
  ODER (Reiniger an UND PM2,5 > Ausschaltschwelle)
│
├─ nein → fan.turn_off (nur wenn an)
│
└─ ja
   ├─ 00:00–06:00
   │    ├─ PIR = on        → timer.start (startet den Timer neu)
   │    ├─ Timer aktiv     → Stufe 1
   │    └─ sonst           → Preset sleep
   ├─ Belegung = on        → Stufe 1
   ├─ PM2,5 > Schwelle St.3 → Stufe 3
   └─ sonst                → Stufe 2
```

Jeder Zweig hat einen Alias (z. B. „Raum belegt – Stufe 1“). In der **Ablaufverfolgung** siehst du so sofort, welcher Zweig gegriffen hat.

### Designentscheidungen

- **Nur bei Abweichung schalten.** Jeder Zweig prüft vorher Zustand, `percentage` und `preset_mode`. Ein Befehl geht nur raus, wenn sich etwas ändern muss. Bei einer Cloud-Integration vermeidet das unnötige API-Aufrufe und mögliche Drosselung.
- **Hysterese ohne Hilfszustand.** „Reiniger läuft schon“ dient als Gedächtnis: Ist er an, gilt die niedrigere Ausschaltschwelle, sonst die Einschaltschwelle.
- **Minutentakt für den PIR.** `motion.detected` feuert nur beim Wechsel auf „Bewegung“. Bleibt der PIR dauerhaft `on`, käme kein weiterer Auslöser. Darum prüft die Automation nachts jede Minute den Zustand und startet den Timer bei `on` neu. Effekt: Stufe 1 bis 20 min nach der letzten gemeldeten Bewegung.
- **5 min Verzögerung bei „frei“.** Kurz den Raum verlassen soll den Reiniger nicht auf Stufe 2/3 hochdrehen lassen.
- **Modus `restart`.** Ein neuer Auslöser bricht einen laufenden Durchlauf ab. Es gilt immer die neueste Lage.
- **Nachtfenster.** Das Fenster 00:00–06:00 steht dreimal in der Automation: in zwei Zeit-Auslösern und einer `time`-Bedingung. Bei einer Änderung alle drei anpassen.

---

## 7. Automation „Luftreiniger Wohnzimmer Filterwechsel Meldung“

Datei: [`automations/luftreiniger_wohnzimmer_filterwechsel_meldung.yaml`](automations/luftreiniger_wohnzimmer_filterwechsel_meldung.yaml) · Modus: `single`

| Teil | Inhalt |
|---|---|
| Auslöser | `numeric_state` Filter-Restlebensdauer `below: 6` (also ≤ 5 %) |
| Bedingung | Vorheriger Zustand war eine Zahl (Template-Guard) |
| Aktion | `notify.whatsapp` mit Titel und aktuellem Prozentwert, `continue_on_error: true` |

**Warum der Guard?** Ein `numeric_state`-Auslöser wird durch `unavailable` neu „scharf“ gestellt. Nach einem Neustart oder einem Cloud-Aussetzer würde er bei einem Wert unter 6 % sonst erneut feuern und eine doppelte Nachricht schicken. Der Guard lässt nur echte Wechsel von Zahl zu Zahl durch.

Nach dem Filterwechsel den Zähler am Gerät oder in der VeSync-App zurücksetzen. Steigt der Wert wieder über 5 %, ist die Meldung für den nächsten Filter bereit.

**Test:** Automation öffnen → ⋮ → **Ausführen**. Das überspringt Auslöser und Bedingung und schickt sofort eine Nachricht mit dem aktuellen Wert.

---

## 8. Dashboard

Dashboard **„Wohnbereich“** (`dashboard-wohnzimmer`), Ansichtstyp Abschnitte, Abschnitt **„Luftreiniger“**:

- Kachel `sensor.airfresh_luftqualitat`
- Kachel `fan.airfresh` mit Feature `fan-speed` (Stufenwahl direkt auf der Kachel)

Bewusst schlank gehalten. Gute Ergänzungen bei Bedarf: `sensor.luftreiniger_wohnzimmer_status`, `sensor.airfresh_pm25`, der Automatik-Schalter.

Konfiguration: [`dashboard/luftreiniger_abschnitt.yaml`](dashboard/luftreiniger_abschnitt.yaml)

---

## 9. Bedienung und Anpassung

**Manuell bedienen:** Erst `input_boolean.luftreiniger_wohnzimmer_automatik` ausschalten. Sonst überschreibt die Automation deine Einstellung beim nächsten PM2,5-Wert. Beim Wiedereinschalten wird sofort der passende Zustand gesetzt.

**Über Helfer anpassen** (Einstellungen → Geräte & Dienste → Helfer):

| Ziel | Helfer |
|---|---|
| Früher/später reinigen | Einschaltschwelle |
| Länger/kürzer nachlaufen | Ausschaltschwelle (mind. ~10 µg/m³ unter der Einschaltschwelle lassen) |
| Stufe 3 früher/später | Schwelle Stufe 3 |
| Nacht-Boost länger/kürzer | Timer „Nacht-Boost“ → Dauer |

Änderungen an den Schwellen wirken sofort, weil sie selbst Auslöser sind.

**In der Automation anpassen:** Nachtfenster (siehe oben), Wartezeit bis „frei“ (`occupancy.cleared` → `for`).

**Orientierung PM2,5:** WHO-Richtwert im Tagesmittel 15 µg/m³. Ab etwa 35 µg/m³ gilt die Luft als deutlich belastet.

---

## 10. Einrichtung

1. **VeSync** hinzufügen, Gerät dem Bereich Wohnzimmer zuordnen.
2. **Entitäts-IDs abgleichen.** Weichen sie bei dir ab, entweder umbenennen oder per Suchen/Ersetzen in den YAML-Dateien anpassen.
3. **Helfer anlegen** laut [`helpers/helfer.yaml`](helpers/helfer.yaml). Namen exakt übernehmen, dann stimmen die IDs automatisch. Startwerte setzen, Automatik einschalten.
   Beim Template-Sensor die Vorlage **in eine Zeile** einfügen.
4. **Automationen:** Neue Automation → ⋮ → **In YAML bearbeiten** → Datei aus `automations/` einfügen. Die Zeile `id:` kannst du weglassen, Home Assistant vergibt dann eine neue.
5. **WhatsApp-Dienst prüfen** (Entwicklerwerkzeuge → Aktionen → `notify.whatsapp`). Heißt er anders, in der Filter-Automation anpassen.
6. **Dashboard:** Abschnitt hinzufügen → ⋮ → In YAML bearbeiten → Inhalt aus `dashboard/` einfügen.
7. **Testen:** Steuerungs-Automation einmal über ⋮ → Ausführen starten und die Ablaufverfolgung ansehen.

---

## 11. Fehlersuche

| Symptom | Ansatz |
|---|---|
| Keine Reaktion | Automatik-Helfer an? Automation aktiviert? Ablaufverfolgung prüfen: Bricht sie bei einer Bedingung ab? |
| `fan.airfresh` nicht verfügbar | VeSync-Cloud/WLAN. Integration neu laden. Die Automation pausiert in der Zeit absichtlich. |
| Reiniger wechselt ständig die Stufe | Ein-/Ausschaltschwelle zu nah beieinander, oder die Belegung flackert (Präsenzmelder-Empfindlichkeit prüfen). |
| Kein Nacht-Boost | Ist die Luft überhaupt über der Schwelle? Steht der PIR auf `on`? Läuft der Timer? |
| Keine WhatsApp | Filter-Automation manuell ausführen. Kommt nichts an, liegt es am `notify.whatsapp`-Dienst. |
| Stufe stimmt nicht | Hat jemand am Gerät oder in der VeSync-App geschaltet? Beim nächsten Auslöser (spätestens nach einer Minute) korrigiert die Automation das. |

Die **Ablaufverfolgung** (Automation → ⋮ → Ablaufverfolgung) ist das wichtigste Werkzeug: Sie zeigt Auslöser-ID, Bedingungsergebnisse und den gewählten Zweig.

---

## 12. Dateien

```
.
├── README.md
├── automations/
│   ├── luftreiniger_wohnzimmer_steuerung.yaml
│   └── luftreiniger_wohnzimmer_filterwechsel_meldung.yaml
├── helpers/
│   └── helfer.yaml
└── dashboard/
    └── luftreiniger_abschnitt.yaml
```

Stand: 18.10.2026 · exportiert aus der laufenden Home-Assistant-Instanz
