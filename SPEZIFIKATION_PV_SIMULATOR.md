# PV-Simulator Balkonkraftwerk — Spezifikation V1

## Zweck
Interaktive Webseite zur Simulation von Balkonkraftwerken (BKW) mit Batteriespeicher.
Zeigt Eigenverbrauch, Einspeisung, Deckungsgrad und Amortisation.
Deployment: GitHub Pages (statische HTML-Datei).

## Technologie
- Eine einzige `index.html` mit eingebettetem CSS + JavaScript
- Chart.js via CDN für Diagramme
- Kein Framework, kein Build-Schritt
- Responsive Layout (Desktop + Tablet)

---

## Eingabeparameter

### Anlage
| Parameter | Typ | Default | Bereich | Infobox |
|---|---|---|---|---|
| Anzahl Module | Zahl | 4 | 1–16 | — |
| Wp pro Modul | Zahl | 500 | 100–600 | Nennleistung unter Standardtestbedingungen (STC) |
| Anzahl Wechselrichter | Zahl | 1 | 1–3 | Jeder WR hat ein eigenes AC-Limit |
| AC-Limit pro WR | Zahl (W) | 800 | 600–2000 | Gesetzliche Einspeisegrenze BKW: 800W seit 2024 |
| Speicherkapazität | Zahl (kWh) | 2.0 | 0–20 | Nutzbare Kapazität nach DoD |
| Batteriewirkungsgrad | Slider (%) | 92 | 80–98 | Verluste beim Laden + Entladen (Round-Trip) |

### Standort
| Parameter | Typ | Default |
|---|---|---|
| Standort | Dropdown | Erfurt |
| Derating | Slider (%) | 20 | 
| Schneetage/Jahr | Zahl | standortabhängig (s.u.) |

**Schneetage-Defaults pro Standort** (automatisch vorbelegt bei Standortwechsel, überschreibbar):

| Höhenlage | Schneetage | Standorte |
|---|---|---|
| <200m | 0 | Kiel, Hamburg, Schwerin, Bremen, Hannover, Berlin, Potsdam, Magdeburg, Düsseldorf, Münster, Mainz, Wiesbaden, Göttingen, Kassel, Dresden |
| 200–400m | 5 | Stuttgart, Freiburg, Saarbrücken, Siegen, Passau, Erfurt, Würzburg |
| 400–600m | 15 | München |
| 600–800m | 34 | Schmiedefeld |

**Derating-Infobox:**
- 0%: Optimale Südausrichtung, keine Verschattung, Schrägdach
- 10–20%: Leichte Abweichung von Süd oder leichte Teilschatten
- 30–40%: Ost/West-Ausrichtung oder deutliche Teilschatten
- 50%+: Flachdach mit Verschattung, ungünstige Ausrichtung
- 70%+: Starke Verschattung (z.B. Nordseite, Bäume)

**Schneetage-Logik:** An Schneetagen ist die PV-Produktion = 0. Verteilung auf Monate:
Jan 30%, Feb 25%, Mär 10%, Nov 10%, Dez 25%.
Ab Standort-Höhe >500m wird ein Vorschlag eingeblendet (z.B. Schmiedefeld: 34 Tage).

### Verbrauch
| Parameter | Typ | Default | Infobox |
|---|---|---|---|
| Jahresverbrauch | Zahl (kWh) | 4000 | Gesamter Stromverbrauch pro Jahr |
| Grundlast-Anteil | Slider (%) | 65 | Anteil des Verbrauchs, der 24/7 konstant anfällt (Kühlschränke, Router, Standby, Heizungspumpe). Bei 4000 kWh und 65% = 297W Dauerverbrauch. |
| Verteilung Restverbauch: | | | Aufteilung der restlichen 35% auf Tageszeiten |
| — Vormittag (6–14 Uhr) | Slider (%) | 50 | Kochen, Waschen, Geschirrspüler |
| — Nachmittag (14–22 Uhr) | Slider (%) | 50 | Kochen, TV, Licht |
| — Nacht (22–6 Uhr) | ergibt sich | 0 | Neben der Grundlast gibt es nachts selten zusätzlichen Verbrauch |

**Verbrauchsverteilung-Logik:**
- Grundlast in W = (Jahresverbrauch × Grundlast%) / 8760
- Restverbrauch pro Jahr = Jahresverbrauch × (1 - Grundlast%)
- Slider-Kopplung: Vormittag + Nachmittag + Nacht = 100%. Zwei frei wählbar, dritter ergibt sich.

**Konzentrationsfaktor pro Zeitfenster:**
Jedes Zeitfenster hat einen Slider „Konzentration" (50%–100%, Default 50%).
Der Wert bestimmt, in welchem Anteil des Zeitfensters der variable Verbrauch anfällt.
- 50% = Verbrauch fällt nur in die erste Hälfte des Zeitraums, zweite Hälfte = nur Grundlast
- 75% = Verbrauch auf 75% der Stunden verteilt
- 100% = Gleichverteilung über den gesamten Zeitraum

Berechnung:
- aktive_stunden = zeitfenster_stunden × konzentration
- Für Stunden innerhalb der aktiven Phase: variable_last = (Restverbrauch × Zeitfenster%) / (365 × aktive_stunden)
- Für Stunden außerhalb der aktiven Phase: variable_last = 0

Beispiel Vormittag (6–14 Uhr), Konzentration 50%:
- 6–10 Uhr (aktiv): Grundlast + (Restverbrauch × Vormittag%) / (365 × 4h)
- 10–14 Uhr (passiv): nur Grundlast
Effekt: Der Verbrauch liegt vor der PV-Spitze → weniger Eigenverbrauch, mehr Einspeisung/Kappung.

Infobox Konzentration: „Wie viele der 8 Stunden wird tatsächlich Strom über die Grundlast hinaus verbraucht? 50% = z.B. Berufstätige, die morgens 4h aktiv sind und dann das Haus verlassen."

**Stündliche Gesamtlast** = Grundlast_W + variable_last_W (wenn Stunde in aktiver Phase, sonst 0)

### Finanzen
| Parameter | Typ | Default | Infobox |
|---|---|---|---|
| Strompreis Netzbezug | Zahl (ct/kWh) | 35 | Aktueller Arbeitspreis |
| Einspeisevergütung | Zahl (ct/kWh) | 8.0 | EEG-Vergütung für Einspeisung |
| Kosten pro Modul | Zahl (€) | 70 | 500Wp-Modul ab Händler |
| Kosten pro WR | Zahl (€) | 669 | z.B. Solakon ONE mit 2.11 kWh LiFePO4 |
| Sonstige Kosten | Zahl (€) | 240 | Shelly 3EM (90€), Kabel, Aufständerung |

---

## Simulationslogik (Port aus Python)

### Datengrundlage
- 24 Standorte mit PVGIS-SARAH2-Monatswerten (2005–2020 Mittel), kWh/m²/Tag
- Individuelle Sonnenauf-/untergangszeiten pro Standort, pro Monat (CET, keine Sommerzeit)

### Stundenweise Simulation (8760h pro Jahr)

Für jeden Monat, jeden Tag, jede Stunde:

1. **PV-Produktion berechnen:**
   - Sinusförmiges Tagesprofil zwischen Sonnenaufgang und -untergang
   - Gewicht pro Stunde h: `w(h) = sin((h - sunrise) / dayLength × π)` wenn sunrise ≤ h ≤ sunset, sonst 0
   - Normiert auf Tagesertrag: `pv_w = (w(h) / Σw) × dailyProd_kWh × 1000`
   - `dailyProd_kWh = irradiance[monat] × totalKWp × (1 - derating)`
   - An Schneetagen: pv_w = 0

2. **Energiefluss-Priorität (wenn PV > 0):**
   1. PV → Last (min(pv_w, last_w, ac_max_w))
   2. Restlast aus Batterie (wenn SOC > 0)
   3. Restlast aus Netz
   4. Überschuss-PV → Batterie laden (wenn SOC < Kapazität)
   5. Überschuss-PV → Netz einspeisen (max ac_max_w - bereits_eingespeist)
   6. Rest → Kappung (Curtailment)

3. **Energiefluss (wenn PV = 0, nachts):**
   1. Last aus Batterie (wenn SOC > 0)
   2. Rest aus Netz

4. **Batterie-SOC** wird stundenweise über das ganze Jahr fortgeschrieben.

### Tracked Variables pro Monat
- pv_produced (Brutto-PV-Erzeugung)
- pv_to_load (PV direkt verbraucht)
- bat_to_load (Batterie → Verbrauch)
- pv_to_battery (PV → Batterie)
- pv_to_grid (Einspeisung ins Netz)
- pv_curtailed (Kappung über AC-Limit)
- grid_to_load (Netzbezug)

---

## Ausgabe / Diagramme

### 1. Monatsbalken (gestapeltes Balkendiagramm)
- X-Achse: Jan–Dez
- Gestapelte Balken: PV→Last (grün), Batterie→Last (blau), Netzbezug (rot)
- Linie überlagert: PV-Gesamtproduktion (orange)
- Zweite Balkenreihe (halbtransparent): Einspeisung (lila), Kappung (grau)

### 1b. Typischer Tag (Stundendiagramm)
- Dropdown: Monat wählen (Jan–Dez) oder „Jahresmittel"
- X-Achse: 0–23 Uhr
- Farbige Hintergrund-Bänder für Tageszeiten: Nacht (22–6, grau), Vormittag (6–14, gelb), Nachmittag (14–22, orange)
- Gestapelte Balken pro Stunde: PV→Last (grün), Batterie→Last (blau), Netzbezug (rot)
- Linie 1: PV-Produktion (orange, durchgezogen)
- Linie 2: Verbrauch/Last (schwarz, gestrichelt)
- Zeigt die Schere zwischen Produktion und Verbrauch: Wann deckt PV den Bedarf, wann springt Batterie/Netz ein
- Datenquelle: Simulationsergebnisse gemittelt über alle Tage des gewählten Monats (pro Stunde Mittelwert)

### 2. Jahresübersicht (Kennzahlen-Karten)
- PV-Produktion gesamt (kWh)
- Eigenverbrauch (PV→Last + Bat→Last) (kWh + %)
- Deckungsgrad (Eigenverbrauch / Gesamtverbrauch × 100)
- Netzbezug (kWh)
- Einspeisung (kWh)
- Kappung (kWh)

### 3. Finanztabelle
- Jährliche Ersparnis = Eigenverbrauch × Strompreis
- Jährlicher Einspeiseerlös = Einspeisung × Einspeisevergütung
- Gesamtertrag pro Jahr
- Gesamtinvestition = Module × Modulpreis + WR × WR-Preis + Sonstiges
- Amortisationszeit = Investition / Gesamtertrag
- 10-Jahres-Gewinn = Gesamtertrag × 10 - Investition

### 4. Amortisationskurve (Liniendiagramm)
- X-Achse: Jahre 0–15
- Y-Achse: Kumulierter Ertrag minus Investition (€)
- Horizontale Linie bei 0€ = Break-Even
- Markierung am Schnittpunkt = Amortisationszeitpunkt

### 5. Energiefluss (Kreisdiagramm / Pie)
- Wohin geht der PV-Strom?
- Segmente: Eigenverbrauch direkt, Batterieladung, Einspeisung, Kappung
- Prozentwerte im Label

---

## Standortdaten

### Strahlungswerte (PVGIS-SARAH2, kWh/m²/Tag, Mittel 2005–2020)

```javascript
const LOCATIONS = {
  "Kiel (SH)":           { irradiance: [0.53, 1.11, 2.30, 4.19, 5.14, 5.49, 5.19, 4.23, 3.05, 1.64, 0.72, 0.40], elevation: 5, annual: 977 },
  "Hamburg (HH)":         { irradiance: [0.58, 1.23, 2.39, 4.20, 4.97, 5.39, 5.23, 4.23, 3.13, 1.71, 0.77, 0.46], elevation: 6, annual: 1011 },
  "Schwerin (MV)":        { irradiance: [0.56, 1.22, 2.40, 4.23, 5.05, 5.51, 5.17, 4.28, 3.15, 1.70, 0.75, 0.44], elevation: 38, annual: 999 },
  "Bremen (HB)":          { irradiance: [0.60, 1.28, 2.46, 4.22, 4.97, 5.36, 5.19, 4.28, 3.15, 1.76, 0.82, 0.49], elevation: 5, annual: 1016 },
  "Hannover (NI)":        { irradiance: [0.68, 1.35, 2.53, 4.27, 4.98, 5.46, 5.22, 4.32, 3.22, 1.79, 0.87, 0.56], elevation: 55, annual: 1046 },
  "Göttingen (NI-Süd)":   { irradiance: [0.70, 1.36, 2.61, 4.30, 4.97, 5.45, 5.30, 4.43, 3.30, 1.86, 0.91, 0.58], elevation: 150, annual: 1065 },
  "Münster (NW)":         { irradiance: [0.70, 1.37, 2.62, 4.28, 4.97, 5.40, 5.23, 4.40, 3.22, 1.85, 0.92, 0.56], elevation: 60, annual: 1060 },
  "Düsseldorf (NW)":      { irradiance: [0.78, 1.44, 2.66, 4.27, 4.98, 5.42, 5.24, 4.35, 3.26, 1.89, 0.98, 0.61], elevation: 38, annual: 1073 },
  "Siegen (NW-Süd)":      { irradiance: [0.71, 1.36, 2.59, 4.21, 4.73, 5.29, 5.19, 4.37, 3.21, 1.80, 0.88, 0.58], elevation: 260, annual: 1032 },
  "Kassel (HE-Nord)":     { irradiance: [0.71, 1.36, 2.60, 4.26, 4.88, 5.36, 5.22, 4.37, 3.24, 1.81, 0.88, 0.56], elevation: 166, annual: 1048 },
  "Wiesbaden (HE)":       { irradiance: [0.85, 1.56, 2.89, 4.60, 5.17, 5.77, 5.67, 4.74, 3.55, 1.99, 0.99, 0.66], elevation: 115, annual: 1143 },
  "Mainz (RP)":           { irradiance: [0.86, 1.59, 2.95, 4.68, 5.27, 5.88, 5.71, 4.78, 3.60, 2.03, 1.01, 0.67], elevation: 86, annual: 1163 },
  "Saarbrücken (SL)":     { irradiance: [0.87, 1.58, 2.87, 4.53, 5.11, 5.82, 5.74, 4.73, 3.62, 2.05, 1.05, 0.70], elevation: 230, annual: 1143 },
  "Stuttgart (BW)":       { irradiance: [1.01, 1.73, 3.02, 4.49, 5.02, 5.71, 5.67, 4.76, 3.64, 2.22, 1.26, 0.87], elevation: 245, annual: 1190 },
  "Freiburg (BW-Süd)":    { irradiance: [0.99, 1.72, 2.85, 4.18, 4.59, 5.34, 5.38, 4.65, 3.55, 2.19, 1.23, 0.85], elevation: 278, annual: 1143 },
  "Würzburg (BY-Nord)":   { irradiance: [0.85, 1.58, 2.83, 4.62, 5.28, 5.90, 5.74, 4.80, 3.63, 2.08, 1.07, 0.71], elevation: 177, annual: 1168 },
  "München (BY)":         { irradiance: [1.06, 1.84, 3.00, 4.49, 4.87, 5.49, 5.58, 4.80, 3.54, 2.26, 1.30, 0.95], elevation: 519, annual: 1207 },
  "Passau (BY-Ost)":      { irradiance: [0.68, 1.70, 2.92, 4.58, 5.03, 5.69, 5.67, 4.85, 3.53, 2.14, 0.96, 0.52], elevation: 312, annual: 1120 },
  "Berlin (BE)":          { irradiance: [0.69, 1.41, 2.55, 4.40, 5.12, 5.71, 5.39, 4.62, 3.35, 1.83, 0.88, 0.56], elevation: 34, annual: 1083 },
  "Potsdam (BB)":         { irradiance: [0.69, 1.42, 2.52, 4.39, 5.10, 5.68, 5.40, 4.62, 3.36, 1.84, 0.89, 0.56], elevation: 32, annual: 1082 },
  "Magdeburg (ST)":       { irradiance: [0.72, 1.42, 2.61, 4.44, 5.19, 5.80, 5.47, 4.59, 3.37, 1.91, 0.93, 0.60], elevation: 56, annual: 1101 },
  "Dresden (SN)":         { irradiance: [0.78, 1.49, 2.63, 4.34, 5.02, 5.43, 5.44, 4.54, 3.33, 1.99, 1.05, 0.70], elevation: 113, annual: 1096 },
  "Erfurt (TH)":          { irradiance: [0.78, 1.43, 2.64, 4.31, 4.96, 5.49, 5.39, 4.48, 3.31, 1.91, 1.00, 0.67], elevation: 195, annual: 1079 },
  "Schmiedefeld (TH, 720m)": { irradiance: [0.66, 1.30, 2.48, 4.12, 4.67, 5.13, 5.15, 4.25, 3.15, 1.71, 0.83, 0.51], elevation: 720, annual: 1000 }
};
```

### Sonnenauf-/untergangszeiten (CET durchgehend, Dezimalstunden, 15. des Monats)

```javascript
// Format: {1: [sunrise, sunset], 2: [sunrise, sunset], ..., 12: [sunrise, sunset]}
const SUNRISE_SUNSET = {
  "Kiel (SH)":           {1:[7.33,14.96], 2:[6.49,15.99], 3:[5.39,16.93], 4:[4.11,17.90], 5:[3.06,18.81], 6:[2.55,19.45], 7:[2.86,19.33], 8:[3.69,18.47], 9:[4.61,17.24], 10:[5.54,15.98], 11:[6.58,14.92], 12:[7.36,14.47]},
  "Hamburg (HH)":         {1:[7.26,15.03], 2:[6.45,16.03], 3:[5.38,16.94], 4:[4.13,17.88], 5:[3.12,18.75], 6:[2.63,19.37], 7:[2.93,19.26], 8:[3.73,18.43], 9:[4.62,17.23], 10:[5.51,16.01], 11:[6.52,14.97], 12:[7.28,14.55]},
  "Schwerin (MV)":        {1:[7.27,15.02], 2:[6.45,16.02], 3:[5.38,16.94], 4:[4.13,17.88], 5:[3.11,18.76], 6:[2.62,19.38], 7:[2.92,19.27], 8:[3.73,18.43], 9:[4.62,17.23], 10:[5.52,16.00], 11:[6.53,14.97], 12:[7.29,14.55]},
  "Bremen (HB)":          {1:[7.22,15.07], 2:[6.43,16.05], 3:[5.38,16.94], 4:[4.15,17.86], 5:[3.15,18.72], 6:[2.67,19.33], 7:[2.97,19.22], 8:[3.76,18.41], 9:[4.62,17.22], 10:[5.50,16.02], 11:[6.49,15.00], 12:[7.24,14.60]},
  "Hannover (NI)":        {1:[7.16,15.12], 2:[6.40,16.08], 3:[5.37,16.95], 4:[4.17,17.84], 5:[3.20,18.67], 6:[2.74,19.26], 7:[3.03,19.16], 8:[3.79,18.37], 9:[4.63,17.21], 10:[5.48,16.04], 11:[6.44,15.05], 12:[7.17,14.67]},
  "Göttingen (NI-Süd)":   {1:[7.10,15.19], 2:[6.36,16.11], 3:[5.37,16.95], 4:[4.19,17.81], 5:[3.25,18.62], 6:[2.82,19.18], 7:[3.10,19.09], 8:[3.83,18.33], 9:[4.64,17.21], 10:[5.46,16.06], 11:[6.39,15.10], 12:[7.09,14.74]},
  "Münster (NW)":         {1:[7.13,15.16], 2:[6.38,16.10], 3:[5.37,16.95], 4:[4.18,17.83], 5:[3.23,18.64], 6:[2.78,19.22], 7:[3.07,19.13], 8:[3.81,18.35], 9:[4.63,17.21], 10:[5.47,16.05], 11:[6.42,15.08], 12:[7.13,14.70]},
  "Düsseldorf (NW)":      {1:[7.07,15.21], 2:[6.35,16.13], 3:[5.37,16.96], 4:[4.20,17.81], 5:[3.27,18.60], 6:[2.84,19.16], 7:[3.12,19.07], 8:[3.85,18.32], 9:[4.64,17.20], 10:[5.45,16.07], 11:[6.37,15.12], 12:[7.07,14.77]},
  "Siegen (NW-Süd)":      {1:[7.05,15.24], 2:[6.33,16.14], 3:[5.36,16.96], 4:[4.21,17.80], 5:[3.30,18.57], 6:[2.87,19.13], 7:[3.15,19.04], 8:[3.86,18.30], 9:[4.65,17.20], 10:[5.44,16.08], 11:[6.35,15.15], 12:[7.04,14.80]},
  "Kassel (HE-Nord)":     {1:[7.08,15.21], 2:[6.35,16.12], 3:[5.37,16.96], 4:[4.20,17.81], 5:[3.27,18.60], 6:[2.83,19.17], 7:[3.12,19.08], 8:[3.84,18.32], 9:[4.64,17.20], 10:[5.45,16.07], 11:[6.38,15.12], 12:[7.08,14.76]},
  "Wiesbaden (HE)":       {1:[6.99,15.30], 2:[6.30,16.17], 3:[5.36,16.96], 4:[4.23,17.77], 5:[3.34,18.53], 6:[2.94,19.06], 7:[3.21,18.98], 8:[3.90,18.27], 9:[4.65,17.19], 10:[5.42,16.10], 11:[6.30,15.19], 12:[6.97,14.86]},
  "Mainz (RP)":           {1:[6.98,15.30], 2:[6.30,16.17], 3:[5.36,16.96], 4:[4.24,17.77], 5:[3.35,18.52], 6:[2.95,19.06], 7:[3.22,18.98], 8:[3.90,18.26], 9:[4.65,17.19], 10:[5.42,16.10], 11:[6.30,15.20], 12:[6.97,14.87]},
  "Saarbrücken (SL)":     {1:[6.93,15.36], 2:[6.27,16.20], 3:[5.35,16.97], 4:[4.26,17.75], 5:[3.39,18.48], 6:[3.01,18.99], 7:[3.27,18.92], 8:[3.93,18.23], 9:[4.66,17.18], 10:[5.40,16.12], 11:[6.25,15.24], 12:[6.90,14.93]},
  "Stuttgart (BW)":       {1:[6.90,15.39], 2:[6.25,16.22], 3:[5.35,16.97], 4:[4.27,17.74], 5:[3.42,18.45], 6:[3.04,18.96], 7:[3.30,18.89], 8:[3.95,18.21], 9:[4.67,17.18], 10:[5.39,16.13], 11:[6.23,15.27], 12:[6.87,14.97]},
  "Freiburg (BW-Süd)":    {1:[6.85,15.44], 2:[6.23,16.25], 3:[5.34,16.98], 4:[4.29,17.72], 5:[3.46,18.41], 6:[3.10,18.90], 7:[3.35,18.84], 8:[3.98,18.18], 9:[4.67,17.17], 10:[5.38,16.14], 11:[6.19,15.31], 12:[6.81,15.02]},
  "Würzburg (BY-Nord)":   {1:[6.97,15.32], 2:[6.29,16.18], 3:[5.36,16.97], 4:[4.24,17.77], 5:[3.36,18.51], 6:[2.96,19.04], 7:[3.23,18.96], 8:[3.91,18.25], 9:[4.66,17.19], 10:[5.42,16.10], 11:[6.29,15.21], 12:[6.95,14.89]},
  "München (BY)":         {1:[6.86,15.43], 2:[6.23,16.24], 3:[5.34,16.98], 4:[4.29,17.72], 5:[3.45,18.41], 6:[3.09,18.91], 7:[3.34,18.85], 8:[3.98,18.18], 9:[4.67,17.17], 10:[5.38,16.14], 11:[6.19,15.30], 12:[6.82,15.01]},
  "Passau (BY-Ost)":      {1:[6.89,15.40], 2:[6.25,16.23], 3:[5.35,16.97], 4:[4.28,17.73], 5:[3.43,18.44], 6:[3.06,18.94], 7:[3.32,18.88], 8:[3.96,18.20], 9:[4.67,17.18], 10:[5.39,16.13], 11:[6.22,15.28], 12:[6.85,14.98]},
  "Berlin (BE)":          {1:[7.18,15.11], 2:[6.40,16.07], 3:[5.38,16.95], 4:[4.16,17.84], 5:[3.19,18.68], 6:[2.73,19.27], 7:[3.02,19.17], 8:[3.78,18.38], 9:[4.63,17.22], 10:[5.49,16.03], 11:[6.45,15.04], 12:[7.18,14.65]},
  "Potsdam (BB)":         {1:[7.17,15.12], 2:[6.40,16.08], 3:[5.37,16.95], 4:[4.17,17.84], 5:[3.20,18.67], 6:[2.74,19.26], 7:[3.03,19.16], 8:[3.79,18.37], 9:[4.63,17.21], 10:[5.48,16.04], 11:[6.45,15.05], 12:[7.17,14.66]},
  "Magdeburg (ST)":       {1:[7.14,15.14], 2:[6.39,16.09], 3:[5.37,16.95], 4:[4.18,17.83], 5:[3.21,18.65], 6:[2.76,19.24], 7:[3.05,19.14], 8:[3.80,18.36], 9:[4.63,17.21], 10:[5.48,16.04], 11:[6.43,15.07], 12:[7.15,14.69]},
  "Dresden (SN)":         {1:[7.06,15.23], 2:[6.34,16.13], 3:[5.36,16.96], 4:[4.21,17.80], 5:[3.28,18.58], 6:[2.86,19.14], 7:[3.14,19.06], 8:[3.85,18.31], 9:[4.64,17.20], 10:[5.45,16.07], 11:[6.36,15.13], 12:[7.05,14.78]},
  "Erfurt (TH)":          {1:[7.06,15.23], 2:[6.34,16.14], 3:[5.36,16.96], 4:[4.21,17.80], 5:[3.29,18.58], 6:[2.86,19.14], 7:[3.14,19.05], 8:[3.86,18.31], 9:[4.64,17.20], 10:[5.45,16.07], 11:[6.36,15.14], 12:[7.05,14.79]},
  "Schmiedefeld (TH, 720m)": {1:[7.03,15.26], 2:[6.32,16.15], 3:[5.36,16.96], 4:[4.22,17.79], 5:[3.31,18.56], 6:[2.89,19.11], 7:[3.17,19.02], 8:[3.87,18.29], 9:[4.65,17.20], 10:[5.44,16.08], 11:[6.34,15.16], 12:[7.02,14.82]}
};
```

---

## Optimierungsfunktion

### Button „Optimale Konfiguration finden"

Durchrechnet alle sinnvollen Kombinationen per Brute Force und zeigt die Top-3 nach Amortisationszeit:

**Suchraum:**
- Module: 1–16 (Schritt 1)
- Wechselrichter: 1–3
- Speicher: 0, 2, 4, 6, 8, 10 kWh (Schritt 2)
- = max. 16 × 3 × 6 = 288 Kombinationen

**Nebenbedingungen:**
- Module pro WR ≤ 4 (MPPT-Limit Solakon ONE) → n_module ≤ n_wr × 4
- Speicher ≤ n_wr × max_bat_per_wr (Standard 6 kWh)
- AC-Limit wird pro WR angewendet

**Zielfunktion:**
- Primär: kürzeste Amortisationszeit (Investition / Jahresertrag)
- Sekundär bei Gleichstand: höchster 10-Jahres-Gewinn

**Ausgabe:**
Tabelle mit Rang 1–3:
| Rang | Module | WR | Speicher | Investition | Ertrag/a | Amortisation | 10J-Gewinn |

Alle fixen Parameter (Standort, Derating, Verbrauch, Preise) werden aus den aktuellen Slider-Werten übernommen.

**Performance:** 288 × 8760h = ~2.5 Mio Berechnungen. In JavaScript auf modernem Browser <500ms.

---

## Bekannte Einschränkungen

1. **Horizontale Strahlung**: PVGIS-Werte sind für horizontale Fläche. Schrägdach-Bonus oder Fassaden-Malus sind im Derating-Slider pauschal abgebildet, nicht physikalisch modelliert.
2. **Keine stochastische Bewölkung**: Jeder Tag im Monat hat identisches Produktionsprofil. Reale Schwankungen (Wolken, Regen) werden durch Monatsmittel geglättet.
3. **Batterie vereinfacht**: Kein Leistungslimit beim Laden/Entladen modelliert, nur Kapazität und Wirkungsgrad.
4. **Keine Degradation**: PV-Module verlieren ~0.5%/a Leistung. Nicht modelliert.
5. **Kein Temperaturfaktor**: Hohe Temperaturen senken den Modulwirkungsgrad (~0.35%/°C über 25°C). Nicht modelliert.

---

## Changelog
- V1 (2026-04-08): Initiale Spezifikation mit 24 PVGIS-Standorten und vollständiger Simulationslogik
- V1.1 (2026-04-08): Optimierungsfunktion ergänzt (Brute-Force über Suchraum, Top-3 nach Amortisation)
- V1.2 (2026-04-08): Konzentrationsfaktor pro Zeitfenster, standortspezifische Schneetage-Defaults, Tagesdiagramm (24h Stundenansicht)
