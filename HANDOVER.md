# PV-Simulator Balkonkraftwerk — Übergabe-Kontext

## Projekt
Interaktiver Web-Simulator für deutsche Balkonkraftwerke (BKW). Single-Page-App als `index.html`, deployed auf GitHub Pages: https://orvillewilbur.github.io/pv/

## Repo-Struktur
- `index.html` (~1300 Zeilen) — die gesamte App (CSS + HTML + JS)
- `TASK.md` — aktueller Implementierungsauftrag (Abschnitte A–F, Details unten)
- `HANDOVER.md` — dieses Dokument
- `SPEZIFIKATION_PV_SIMULATOR.md` — ursprüngliche Spezifikation
- `pv_ferienhaus_simulation.py`, `pv_schmiedefeld_simulation.py` — Python-Referenzsimulationen

## Technologie
- Vanilla JS, kein Framework
- Chart.js 4.4.0 (CDN)
- CSS-Variablen für Theming
- Responsive ab 920px Breakpoint

## Architektur `index.html`
- **CSS:** Styles inkl. Media Query ≤920px
- **HTML:** Header, Input-Panel mit 4 Cards (Standort, Anlage, Verbrauch, Wirtschaftlichkeit), Results mit KPIs, 2 Charts (monatlich + täglich), Finanztabelle, Optimierungstabelle, Methodik-Abschnitt
- **Daten:** LOCATIONS (24 Städte mit PVGIS-Einstrahlung), SUNRISE_SUNSET, DAYS_PER_MONTH, MONTH_NAMES, SNOW_DIST
- **`computeHourlyLoadW(p)`** — 24h-Lastprofil aus Grundlast + Tagesprofil
- **`buildWeights(sunrise, sunset)`** — sinusoidale PV-Gewichtung
- **`simulateYear(p)`** — Hauptsimulation (8760h), Energiefluss PV→Last→Batterie→Netz→Kappung, **inkl. Spitzenlast-Korrektur**
- **`simulateYearTotals(p)`** — schnelle Optimizer-Version (gleiche Logik inkl. Spitzenlast)
- **`calcFinance(r, p)`** — Wirtschaftlichkeit inkl. Batteriekosten
- **`initCharts()`** — zwei Chart.js-Instanzen
- **`updateResults()`** — Simulation → KPIs → Charts
- **`updateMethodology()`** — dynamischer Methodik-Abschnitt (7 Kapitel), aktualisiert sich bei Parameteränderung
- **Optimierung:** 96 Konfigurationen, nutzt `simulateYearTotals()`

## Chart-Datasets (8 pro Chart, identische Struktur)
```
0: PV → Last        (bar, green, stack:'e')
1: Batterie → Last  (bar, blue, stack:'e')
2: Netz → Last      (bar, red, stack:'e')
3: PV → Batterie    (bar, yellow, stack:'e')
4: PV → Einspeisung (bar, purple, stack:'e')
5: Kappung          (bar, gray, stack:'e')
6: PV-Produktion    (line, orange, stack:'pv')
7: Verbrauch        (line, black dashed, stack:'cons')
```

**Kritisch:** Jedes Line-Dataset MUSS eine eigene `stack`-Property haben (`'pv'` / `'cons'`). Ohne das stapelt Chart.js 4.4.0 bei `scales.y.stacked:true` alle Datasets — die Lines landen visuell auf den Bars.

## Bereits implementiert (nicht erneut anfassen!)

### Spitzenlast-Modell (Abschnitt 0 der ursprünglichen TASK.md)
- Konstanten `SPIKE_FRACTION = 0.50`, `SPIKE_POWER_W = 2000`
- 50% des variablen Lastanteils als Großverbraucher (2000W) modelliert
- PV-Deckung begrenzt auf `min(1, acMaxW/SPIKE_POWER_W)` = 40% bei 800W WR
- `spikeGridKWh` direkt zu `grid_to_load` addiert
- Identische Logik in `simulateYear()` und `simulateYearTotals()`
- Methodik Kapitel 3 ergänzt mit Spitzenlast-Absatz
- Fußnote ⁸ (Spitzenlast) eingefügt — sowohl Marker im Text als auch Fußnotentext
- Disclaimer-Bullet für Spitzenlast aktualisiert

### Verbrauchslinie tagesgewichtet
`DAYS_PER_MONTH.slice(1).map(days => ...)` statt `Array(12).fill(yearlyConsumption/12)`

### Chart.js Line-Stacking-Fix
Eigene `stack`-Property pro Line-Dataset (s.o.)

## Abschnitte A–F (TASK.md) — alle implementiert

| Abschnitt | Was | Status |
|-----------|-----|--------|
| A | PDF-Export-Button (`window.print()` + `@media print`, `#printBtnWrap`) | ✅ erledigt |
| B | Fußnoten ¹–⁷ in `updateMethodology()` (Marker + Tooltips + Fußnotenliste) | ✅ erledigt |
| C | Disclaimer-Abschnitt (`#disclaimer`) mit vollständigem Text | ✅ erledigt |
| D | Impressum (`#impressum`, §5 DDG, Spectabilis) | ✅ erledigt |
| E | Reihenfolge: `</main>` → `#methodology` → `#printBtnWrap` → `#disclaimer` → `#impressum` | ✅ erledigt |
| F | Responsive: `#methodology,#disclaimer,#impressum,#printBtnWrap { padding:0 16px; }` bei ≤920px | ✅ erledigt |

TASK.md enthält die ursprünglichen Spezifikationen als Referenz.

## Bekannte gelöste Bugs (nicht wieder einführen!)
1. **Chart.js Line-Stacking:** Lines brauchen eigene `stack`-Property
2. **Verbrauchslinie tagesgewichtet:** `DAYS_PER_MONTH`-basiert, nicht gleichverteilt
3. **GitHub Pages Cache:** Nach Push ggf. `?v=N` Cache-Buster nötig

## Workflow
- Design/Spezifikation in Cowork → TASK.md schreiben
- Implementierung durch Claude Code: „Lies /PV/TASK.md und implementiere alle Abschnitte in /PV/index.html"
- Nach Push: Verifikation auf https://orvillewilbur.github.io/pv/

## Qualitätsanspruch
- Automatisierte Tests (JS-Injection) für Energiebilanzen, alle 24 Standorte, Finanzen, Optimierung
- Visuelle Verifikation der Charts (automatisierte Tests allein reichen NICHT)
- Deliverables vollständig validieren vor Präsentation
- Fehler proaktiv benennen, nicht warten bis der User sie findet
