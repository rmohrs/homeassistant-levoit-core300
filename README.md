# Luftreiniger Wohnzimmer – Levoit Core 300S in Home Assistant

Diese Doku beschreibt, wie der Luftreiniger **Levoit Core 300S** im Wohnzimmer automatisch gesteuert wird.
Sie ist für Home-Assistant-Einsteiger geschrieben: Jeder Begriff wird kurz erklärt, und am Ende steht, wie man alles Schritt für Schritt nachbaut.

---

## Inhalt

1. [Was macht das Ganze?](#1-was-macht-das-ganze)
2. [Kurze Begriffserklärung](#2-kurze-begriffserklärung)
3. [Verwendete Geräte](#3-verwendete-geräte)
4. [Verwendete Integrationen](#4-verwendete-integrationen)
5. [Helfer](#5-helfer)
6. [Automation 1: Steuerung](#6-automation-1-luftreiniger-wohnzimmer-steuerung)
7. [Automation 2: Filterwechsel-Meldung](#7-automation-2-luftreiniger-wohnzimmer-filterwechsel-meldung)
8. [Dashboard](#8-dashboard)
9. [Bedienung im Alltag](#9-bedienung-im-alltag)
10. [Einstellungen anpassen](#10-einstellungen-anpassen)
11. [Nachbauen Schritt für Schritt](#11-nachbauen-schritt-für-schritt)
12. [Fehlersuche](#12-fehlersuche)
13. [Dateien in diesem Repository](#13-dateien-in-diesem-repository)

---

## 1. Was macht das Ganze?

Der Luftreiniger misst selbst den **Feinstaub (PM2,5)** in der Luft. Home Assistant liest diesen Wert und entscheidet:

| Situation | Was der Reiniger macht |
|---|---|
| Luft ist sauber | **Aus** |
| Luft ist belastet, jemand ist im Wohnzimmer | **Stufe 1** (leise) |
| Luft ist belastet, niemand im Wohnzimmer | **Stufe 2** |
| Luft ist stark belastet, niemand im Wohnzimmer | **Stufe 3** (volle Leistung) |
| Nachts 0–6 Uhr, Luft belastet | **Nachtmodus** (Sleep, sehr leise) |
| Nachts 0–6 Uhr, Luft belastet **und** Bewegung im Raum | **Stufe 1 für 20 Minuten**, danach wieder Nachtmodus |
| Filter hat nur noch 5 % Restlebensdauer | **WhatsApp-Nachricht** |

**Wann gilt die Luft als „belastet“?**

- Der Reiniger **startet**, wenn PM2,5 über **25 µg/m³** steigt.
- Er **schaltet erst wieder ab**, wenn PM2,5 unter **12 µg/m³** fällt.

Der Abstand zwischen den beiden Werten ist Absicht. Läge beides bei 25, würde der Reiniger bei Werten um 25 ständig an- und ausgehen. Dieses Prinzip heißt **Hysterese** (wie beim Thermostat einer Heizung).

**Stufe 3** gibt es nur, wenn niemand im Raum ist und PM2,5 über **55 µg/m³** liegt. Ist jemand da, bleibt es aus Lärmgründen immer bei Stufe 1.

Alle drei Werte (25, 12, 55) kannst du jederzeit ändern, siehe [Abschnitt 10](#10-einstellungen-anpassen).

---

## 2. Kurze Begriffserklärung

| Begriff | Bedeutung |
|---|---|
| **Integration** | Ein „Treiber“, der ein Gerät oder einen Dienst mit Home Assistant verbindet, z. B. VeSync für Levoit-Geräte. |
| **Gerät** | Ein physisches Gerät, z. B. der Luftreiniger. Ein Gerät hat mehrere Entitäten. |
| **Entität** | Ein einzelner Wert oder Schalter eines Geräts, z. B. „PM2,5“ oder „Lüfter“. Jede Entität hat eine **Entitäts-ID** wie `sensor.airfresh_pm25`. |
| **Helfer** | Eine selbst angelegte Entität ohne echtes Gerät dahinter, z. B. ein Schalter oder eine Zahl, die man im Dashboard einstellen kann. |
| **Automation** | Eine Regel nach dem Muster **Auslöser → Bedingung → Aktion**: „Wenn X passiert und Y gilt, dann tue Z.“ |
| **Auslöser** | Das Ereignis, das die Automation startet, z. B. „PM2,5-Wert hat sich geändert“. |
| **Bedingung** | Eine Prüfung, die erfüllt sein muss, damit es weitergeht, z. B. „Automatik ist an“. |
| **Aktion** | Was dann passiert, z. B. „Reiniger auf Stufe 1 schalten“. |
| **Dashboard** | Die Bedienoberfläche mit Kacheln (Karten). |
| **PM2,5** | Feinstaubpartikel kleiner als 2,5 Mikrometer, gemessen in µg/m³ (Mikrogramm pro Kubikmeter Luft). Je niedriger, desto besser. |

---

## 3. Verwendete Geräte

### Levoit Core 300S (Luftreiniger)

- **Hersteller / Modell:** Levoit (VeSync) Core300S
- **Name in Home Assistant:** „Airfresh“
- **Bereich:** Wohnzimmer
- **Anbindung:** WLAN, über die VeSync-Cloud

| Entität | Was sie ist | Wofür sie hier gebraucht wird |
|---|---|---|
| `fan.airfresh` | Der Reiniger selbst (Lüfter). Kennt 3 Stufen (33 %, 66 %, 100 %) und die Modi `sleep` und `auto`. | Wird von der Automation geschaltet. |
| `sensor.airfresh_pm25` | Feinstaub PM2,5 in µg/m³ | **Hauptwert**, nach dem gesteuert wird |
| `sensor.airfresh_luftqualitat` | Luftqualität als Wort (z. B. „poor“ = schlecht) | Anzeige im Dashboard |
| `sensor.airfresh_restlebensdauer_des_filters` | Filter-Restlebensdauer in % | Für die Filterwechsel-Meldung |
| `switch.airfresh_display` | Display am Gerät an/aus | wird nicht verwendet |
| `switch.airfresh_kindersicherung` | Tastensperre am Gerät | wird nicht verwendet |

> **Stufen und Prozente:** Home Assistant steuert Lüfter in Prozent. Beim Core 300S entspricht 33 % = Stufe 1, 66 % = Stufe 2, 100 % = Stufe 3.

### Aqara Presence Sensor FP300 (Präsenzmelder)

- **Hersteller / Modell:** Aqara Presence sensor FP300
- **Name in Home Assistant:** „Raum Wohnzimmer“
- **Bereich:** Wohnzimmer
- **Anbindung:** Zigbee, über Zigbee2MQTT

| Entität | Was sie ist | Wofür sie hier gebraucht wird |
|---|---|---|
| `binary_sensor.raum_wohnzimmer_presence` | **Belegung** (Radar). Erkennt auch Personen, die still sitzen. | Tagsüber: Raum belegt → Stufe 1, Raum frei → Stufe 2/3 |
| `binary_sensor.raum_wohnzimmer_pir_detection` | **Bewegung** (PIR = Infrarot-Bewegungsmelder) | Nachts: Bewegung → 20 Minuten Stufe 1 |

Der Präsenzmelder hat noch viele weitere Entitäten (Temperatur, Helligkeit usw.), die hier keine Rolle spielen.

---

## 4. Verwendete Integrationen

| Integration | Wofür | Wo zu finden |
|---|---|---|
| **VeSync** | Verbindet den Levoit-Luftreiniger. Läuft über die Cloud von Levoit, braucht also Internet und ein VeSync-Konto. | Einstellungen → Geräte & Dienste → VeSync |
| **MQTT** + **Zigbee2MQTT** | Verbindet den Aqara-Präsenzmelder. Zigbee2MQTT läuft als eigene App und spricht mit dem Zigbee-Stick, MQTT reicht die Daten an Home Assistant weiter. | Einstellungen → Geräte & Dienste → MQTT |
| **Template** | Für den Helfer „Luftreiniger Wohnzimmer Status“ (ein berechneter Text). | Einstellungen → Geräte & Dienste → Helfer |
| **Input Boolean, Input Number, Timer** | Eingebaute Helfer-Typen von Home Assistant. Keine Einrichtung nötig. | Einstellungen → Geräte & Dienste → Helfer |
| **WhatsApp-Benachrichtigung** (`notify.whatsapp`) | Bereits vorhandener Benachrichtigungsdienst, der auch von den Heizungs-Automationen genutzt wird. Seine Einrichtung ist nicht Teil dieses Projekts. | Einstellungen → Entwicklerwerkzeuge/Tools → Aktionen → `notify.whatsapp` |

---

## 5. Helfer

Helfer sind selbst angelegte „Schalter“ und „Zahlen“. Mit ihnen kann man das Verhalten ändern, **ohne die Automation zu bearbeiten**. Alle liegen im Bereich Wohnzimmer.

| Helfer | Entitäts-ID | Typ | Wert | Wozu |
|---|---|---|---|---|
| Luftreiniger Wohnzimmer Automatik | `input_boolean.luftreiniger_wohnzimmer_automatik` | Schalter | An | Hauptschalter. **Aus** = Automation macht nichts, du bedienst den Reiniger von Hand. |
| Luftreiniger Wohnzimmer Einschaltschwelle | `input_number.luftreiniger_wohnzimmer_einschaltschwelle` | Zahl | 25 µg/m³ | Ab diesem PM2,5-Wert wird gereinigt. |
| Luftreiniger Wohnzimmer Ausschaltschwelle | `input_number.luftreiniger_wohnzimmer_ausschaltschwelle` | Zahl | 12 µg/m³ | Unter diesem Wert geht der Reiniger wieder aus. |
| Luftreiniger Wohnzimmer Schwelle Stufe 3 | `input_number.luftreiniger_wohnzimmer_schwelle_stufe_3` | Zahl | 55 µg/m³ | Ab hier Stufe 3, wenn niemand im Raum ist. |
| Luftreiniger Wohnzimmer Nacht-Boost | `timer.luftreiniger_wohnzimmer_nacht_boost` | Timer | 20 Minuten | Zählt nachts nach Bewegung die 20 Minuten Stufe 1 herunter. |
| Luftreiniger Wohnzimmer Status | `sensor.luftreiniger_wohnzimmer_status` | Template-Sensor | Text | Zeigt in Worten, was der Reiniger gerade tut: „Stufe 1“, „Nachtmodus“, „Aus – Luft sauber“, „Manuell – läuft“, „Nicht erreichbar“. |

Alle Einstellfelder stehen in [`helpers/helfer.yaml`](helpers/helfer.yaml).

---

## 6. Automation 1: „Luftreiniger Wohnzimmer Steuerung“

Diese Automation ist das Herzstück. Die komplette Konfiguration steht in
[`automations/luftreiniger_wohnzimmer_steuerung.yaml`](automations/luftreiniger_wohnzimmer_steuerung.yaml).

### Grundidee

Die Automation läuft bei vielen Anlässen los, prüft dann **jedes Mal die komplette Lage neu** und stellt den Reiniger passend ein. So ist es egal, *was* sie ausgelöst hat: Das Ergebnis ist immer der richtige Zustand.

### Auslöser – wann sie losläuft

| Auslöser | Erklärung |
|---|---|
| PM2,5-Wert ändert sich | Neuer Messwert vom Reiniger |
| Belegung erkannt | Jemand kommt ins Wohnzimmer |
| Belegung seit 5 Minuten weg | Raum ist leer. Die 5 Minuten verhindern, dass der Reiniger hochdreht, nur weil du kurz rausgehst. |
| 00:00 Uhr | Beginn des Nachtmodus |
| 06:00 Uhr | Ende des Nachtmodus |
| Automatik wird eingeschaltet | Damit der Reiniger sofort richtig steht |
| Eine der drei Schwellen wird geändert | Neue Werte wirken sofort |
| Home Assistant startet | Nach einem Neustart wird alles neu eingestellt |
| Bewegung erkannt (PIR) | Für den Nacht-Boost |
| Nacht-Boost-Timer abgelaufen | Zurück in den Nachtmodus |
| Jede Minute | Prüft nachts, ob der PIR noch Bewegung meldet (siehe unten) |

### Bedingungen – wann sie *nicht* weitermacht

Die Automation bricht ab, wenn

- der Schalter **Automatik aus** ist,
- der Reiniger **nicht erreichbar** ist (z. B. Cloud-Störung), oder
- der **PM2,5-Wert fehlt**.

### Aktionen – was sie tut

Vereinfacht als Entscheidungsbaum:

```
Muss gereinigt werden?
(PM2,5 über 25  ODER  Reiniger läuft schon und PM2,5 noch über 12)
│
├── NEIN → Reiniger ausschalten (falls er an ist)
│
└── JA
    ├── Ist es zwischen 0 und 6 Uhr?
    │   ├── Meldet der PIR Bewegung? → 20-Minuten-Timer (neu) starten
    │   ├── Läuft der Timer?   → Stufe 1
    │   └── Sonst              → Nachtmodus (Sleep)
    │
    ├── Ist jemand im Raum?    → Stufe 1
    ├── PM2,5 über 55?         → Stufe 3
    └── Sonst                  → Stufe 2
```

### Wichtige Details

- **Nur bei Bedarf schalten:** Vor jedem Befehl wird geprüft, ob der Reiniger schon richtig steht. Nur wenn nicht, geht ein Befehl raus. Das ist wichtig, weil jeder Befehl über die VeSync-Cloud läuft. Zu viele Befehle könnten dort zu Sperren führen.
- **Warum „jede Minute“?** Der Bewegungsmelder kann längere Zeit durchgehend „Bewegung“ melden. Ein Auslöser „Bewegung erkannt“ reagiert aber nur auf den *Wechsel* von „keine Bewegung“ auf „Bewegung“. Deshalb schaut die Automation zusätzlich jede Minute nach. Solange Bewegung gemeldet wird, beginnen die 20 Minuten von vorn. Der Reiniger bleibt also auf Stufe 1, solange sich jemand bewegt, und geht 20 Minuten nach der letzten Bewegung zurück in den Nachtmodus.
- **Modus „Neustart“ (restart):** Kommt während eines Durchlaufs ein neuer Auslöser, wird der alte abgebrochen und neu begonnen. So gilt immer die aktuellste Lage.
- **Nacht und Bewegung bei sauberer Luft:** Dann passiert nichts. Der Reiniger bleibt aus.

---

## 7. Automation 2: „Luftreiniger Wohnzimmer Filterwechsel Meldung“

Konfiguration: [`automations/luftreiniger_wohnzimmer_filterwechsel_meldung.yaml`](automations/luftreiniger_wohnzimmer_filterwechsel_meldung.yaml)

| Teil | Inhalt |
|---|---|
| **Auslöser** | Filter-Restlebensdauer fällt unter 6 %, also auf **5 % oder weniger** |
| **Bedingung** | Der vorherige Wert war eine echte Zahl. Das verhindert eine Fehlmeldung, wenn der Reiniger kurz „nicht verfügbar“ war oder Home Assistant neu gestartet wurde. |
| **Aktion** | WhatsApp-Nachricht über `notify.whatsapp` mit Titel „Luftreiniger: Filter wechseln“ und dem aktuellen Prozentwert |

Die Nachricht kommt **einmal** beim Unterschreiten, nicht täglich. Nach dem Filterwechsel setzt du den Filterzähler am Gerät oder in der VeSync-App zurück. Dann ist die Meldung automatisch für den nächsten Filter bereit.

> **Testen:** Einstellungen → Automatisierungen → die Automation öffnen → Drei-Punkte-Menü → „Ausführen“. Die Bedingung wird dabei übersprungen, die Nachricht kommt sofort (mit dem aktuellen Prozentwert).

---

## 8. Dashboard

Im Dashboard **„Wohnbereich“** (Seitenleiste) gibt es den Abschnitt **„Luftreiniger“** mit zwei Kacheln:

| Kachel | Zeigt / kann |
|---|---|
| **Luftqualität** | Luftqualität als Wort |
| **Levoit Core 300S** | An/aus und eine Stufenwahl direkt in der Kachel. Ein Tipp auf die Kachel öffnet die Details mit Modus (Sleep/Auto). |

Die Konfiguration steht in [`dashboard/luftreiniger_abschnitt.yaml`](dashboard/luftreiniger_abschnitt.yaml).

> Wer mehr sehen möchte, kann jederzeit Kacheln ergänzen, z. B. `sensor.luftreiniger_wohnzimmer_status`, `sensor.airfresh_pm25` oder den Automatik-Schalter.

---

## 9. Bedienung im Alltag

- **Nichts tun:** Normalerweise läuft alles von allein.
- **Reiniger von Hand bedienen:** Erst den Helfer **„Luftreiniger Wohnzimmer Automatik“ ausschalten**. Sonst stellt die Automation den Reiniger beim nächsten Messwert wieder zurück.
  Zu finden unter Einstellungen → Geräte & Dienste → Helfer.
- **Zurück auf Automatik:** Den Helfer wieder einschalten. Der Reiniger wird sofort passend eingestellt.
- **Filter gewechselt:** Filterzähler am Gerät bzw. in der VeSync-App zurücksetzen.

---

## 10. Einstellungen anpassen

Alles ohne die Automation anzufassen:

| Was ändern? | Wo? |
|---|---|
| Ab wann gereinigt wird | Helfer „Einschaltschwelle“ |
| Wann wieder abgeschaltet wird | Helfer „Ausschaltschwelle“ (immer **kleiner** als die Einschaltschwelle lassen!) |
| Ab wann Stufe 3 läuft | Helfer „Schwelle Stufe 3“ |
| Dauer des Nacht-Boosts | Helfer „Nacht-Boost“ → Dauer |

Weg dorthin: **Einstellungen → Geräte & Dienste → Reiter „Helfer“** → Helfer anklicken → Wert ändern.

Diese Dinge sind fest in der Automation eingetragen und werden dort geändert (Einstellungen → Automatisierungen → „Luftreiniger Wohnzimmer Steuerung“):

- Uhrzeiten des Nachtmodus (00:00 und 06:00) – stehen **dreimal** drin: zwei Auslöser und eine Bedingung
- Die 5 Minuten Wartezeit, bevor der Raum als „frei“ gilt

**Orientierung für PM2,5:** Die WHO empfiehlt im Tagesmittel höchstens 15 µg/m³. Werte über 35 µg/m³ gelten als deutlich belastet.

---

## 11. Nachbauen Schritt für Schritt

Falls du alles neu einrichten musst (z. B. auf einem neuen Home-Assistant-System):

1. **VeSync einrichten:** Einstellungen → Geräte & Dienste → „Integration hinzufügen“ → VeSync → mit VeSync-Konto anmelden. Der Reiniger erscheint als Gerät. Bereich auf „Wohnzimmer“ stellen.
2. **Präsenzmelder:** Muss in Zigbee2MQTT angelernt sein und in Home Assistant als „Raum Wohnzimmer“ erscheinen.
3. **Entitäts-IDs prüfen:** Unter Einstellungen → Entitäten nachsehen, ob die IDs aus [Abschnitt 3](#3-verwendete-geräte) genauso heißen. Wenn nicht: entweder die Entität umbenennen oder die IDs in den YAML-Dateien anpassen.
4. **Helfer anlegen:** Einstellungen → Geräte & Dienste → Helfer → „Helfer erstellen“. Alle Helfer aus [`helpers/helfer.yaml`](helpers/helfer.yaml) anlegen. **Wichtig:** Die Namen exakt so schreiben, dann entstehen automatisch die richtigen Entitäts-IDs. Danach die Startwerte eintragen und die Automatik einschalten.
5. **Automationen anlegen:** Einstellungen → Automatisierungen → „Automatisierung erstellen“ → „Neu erstellen“ → Drei-Punkte-Menü oben rechts → **„In YAML bearbeiten“** → den Inhalt der jeweiligen Datei aus `automations/` einfügen → Speichern.
6. **WhatsApp-Dienst:** Prüfen, ob `notify.whatsapp` existiert (Entwicklerwerkzeuge/Tools → Aktionen). Falls er anders heißt, in der Filter-Automation anpassen.
7. **Dashboard:** Dashboard öffnen → Stift (bearbeiten) → Abschnitt hinzufügen → Drei-Punkte-Menü des Abschnitts → „In YAML bearbeiten“ → Inhalt aus [`dashboard/luftreiniger_abschnitt.yaml`](dashboard/luftreiniger_abschnitt.yaml) einfügen.

> Die Automationen nutzen die neuen **zweckbezogenen Auslöser** (`occupancy.detected`, `motion.detected`, `timer.finished`). Diese gibt es ab **Home Assistant 2026.7**.

---

## 12. Fehlersuche

| Problem | Was prüfen |
|---|---|
| Reiniger reagiert gar nicht | Ist der Helfer **Automatik** an? Ist die Automation selbst eingeschaltet? |
| Reiniger zeigt „nicht verfügbar“ | VeSync-Cloud oder WLAN gestört. Einstellungen → Geräte & Dienste → VeSync → ggf. „Neu laden“. Die Automation macht in der Zeit absichtlich nichts. |
| Reiniger springt ständig um | Liegen Ein- und Ausschaltschwelle zu nah beieinander? Mindestens 10 µg/m³ Abstand lassen. |
| Nachts kein Boost bei Bewegung | Ist die Luft überhaupt belastet? Bei sauberer Luft gibt es keinen Boost. Meldet `binary_sensor.raum_wohnzimmer_pir_detection` Bewegung? |
| Keine WhatsApp beim Filter | Automation manuell ausführen (siehe [Abschnitt 7](#7-automation-2-luftreiniger-wohnzimmer-filterwechsel-meldung)). Kommt nichts an, liegt es am WhatsApp-Dienst. |
| Was hat die Automation zuletzt gemacht? | Automation öffnen → Drei-Punkte-Menü → **„Ablaufverfolgung“ (Traces)**. Dort sieht man für jeden Durchlauf, welcher Auslöser es war und welcher Zweig genommen wurde. Jeder Zweig hat einen sprechenden Namen, z. B. „Raum belegt – Stufe 1“. |

---

## 13. Dateien in diesem Repository

```
.
├── README.md                                            ← diese Doku
├── automations/
│   ├── luftreiniger_wohnzimmer_steuerung.yaml            ← Automation 1 (Steuerung)
│   └── luftreiniger_wohnzimmer_filterwechsel_meldung.yaml ← Automation 2 (WhatsApp)
├── helpers/
│   └── helfer.yaml                                       ← alle Helfer mit Einstellungen
└── dashboard/
    └── luftreiniger_abschnitt.yaml                       ← Dashboard-Abschnitt
```

Stand: 18.10.2026
