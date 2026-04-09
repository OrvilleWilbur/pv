# Aktueller Auftrag: Spitzenlast-Modell + PDF-Export + Fußnoten + Disclaimer + Impressum

Datei: `index.html`

Zwei Aufträge in einem Commit. Zuerst das Simulationsmodell erweitern, dann die Darstellungs-Features.

---

## 0. Spitzenlast-Modell (Simulationsänderung)

### Problem
Das Modell hat stündliche Auflösung. Großverbraucher (Wasserkocher 2 kW, Herd 2,5 kW, Durchlauferhitzer 18 kW) werden über die Stunde gemittelt. Da der WR nur 800 W AC liefert, können diese Geräte in der Realität nie vollständig durch PV gedeckt werden. Das Modell überschätzt den Deckungsgrad.

### Lösung
50% des variablen Lastanteils (alles außer Grundlast) als „Spitzenlast" mit 2000 W mittlerer Leistung modellieren. Für diese Hälfte kann PV nur 800/2000 = 40% der Energie decken. Die restlichen 60% gehen immer ans Netz.

### Keine neuen UI-Eingabefelder
Die Werte (50% Anteil, 2000 W Spitzenlast) werden als Konstanten im Code definiert:

```javascript
const SPIKE_FRACTION = 0.50;   // 50% des variablen Verbrauchs sind Spitzenlast
const SPIKE_POWER_W  = 2000;   // Mittlere Leistung der Großverbraucher
```

### Änderung in `computeHourlyLoadW(p)`

Die Funktion gibt aktuell pro Stunde einen einzigen Lastwert zurück. Änderung: Stattdessen ein Objekt mit zwei Komponenten zurückgeben:

```javascript
// VORHER: return baseLoadW + varLoad;
// NACHHER:
return {
  normalW: baseLoadW + varLoad * (1 - SPIKE_FRACTION),
  spikeW:  varLoad * SPIKE_FRACTION
};
```

Alternativ (einfacher, weniger Code-Umbau): Die Funktion gibt weiterhin einen einzelnen Wert zurück, aber wir berechnen den Spike-Effekt in der Simulation.

### Bevorzugte Umsetzung (minimaler Code-Umbau)

In `simulateYear()` und `simulateYearTotals()`, bei der Berechnung von `pvForLoadW`:

```javascript
// Bestehender Code:
const loadW = hourlyLoadW[h];
const loadKWh = loadW / 1000;

// NEU: Variable Last aufteilen
const baseLoadW_h = (p.yearlyConsumption * p.baseFraction / 100) / 8760 * 1000;
const varLoadW_h = Math.max(0, loadW - baseLoadW_h);

// Normaler Lastanteil (Grundlast + 50% des variablen Anteils): deckbar bis AC-Limit
const normalLoadW = baseLoadW_h + varLoadW_h * (1 - SPIKE_FRACTION);
// Spitzenlastanteil (50% des variablen Anteils): nur anteilig deckbar
const spikeLoadW = varLoadW_h * SPIKE_FRACTION;
const spikePvCoverageRatio = Math.min(1, acMaxW / SPIKE_POWER_W);
// → Bei 800W AC und 2000W Spike: 40% deckbar durch PV

// PV deckt zuerst den normalen Anteil
const pvForNormalW = Math.min(pvW, normalLoadW, acMaxW);
let remPvW = pvW - pvForNormalW;

// PV deckt den deckbaren Teil des Spike-Anteils
const spikeKWhTotal = spikeLoadW / 1000;
const spikePvKWh = spikeKWhTotal * spikePvCoverageRatio;
const pvForSpikeW = Math.min(remPvW, spikePvKWh * 1000);
remPvW -= pvForSpikeW;

// Netz deckt den nicht-deckbaren Spike-Anteil IMMER
const spikeGridKWh = spikeKWhTotal * (1 - spikePvCoverageRatio);

// Gesamt PV→Last
const pvForLoadW_total = pvForNormalW + pvForSpikeW;
const pvForLoadKWh = pvForLoadW_total / 1000;

// Restlast für Batterie/Netz (ohne den Spike-Grid-Anteil, der schon zugeteilt ist)
let remLoad = loadKWh - pvForLoadKWh - spikeGridKWh;
```

Die `spikeGridKWh` werden direkt zu `grid_to_load` addiert.

### WICHTIG: Auch in `simulateYearTotals()` identisch umsetzen!

Die gleiche Logik muss auch in der Optimizer-Version `simulateYearTotals()` implementiert werden, sonst weichen Optimizer und Hauptsimulation voneinander ab.

### Auswirkung auf Methodik-Abschnitt

Im Kapitel „3. Energiefluss-Simulation" in `updateMethodology()` ergänzen:

```
Spitzenlast-Korrektur: 50% des variablen Lastanteils wird als Großverbraucher 
(mittlere Leistung 2000 W) modelliert. Da der Wechselrichter maximal {acMax} W AC 
liefert, können davon nur {acMax}/2000 = {coverage}% durch PV gedeckt werden. 
Die restlichen {100-coverage}% gehen immer ans Netz.
```

### Erwartete Auswirkung
- Deckungsgrad sinkt um ca. 3–8 Prozentpunkte (abhängig von Grundlast-Anteil)
- Bei 100% Grundlast (baseFraction=100): kein Effekt (kein variabler Anteil)
- Bei 0% Grundlast: maximaler Effekt

---

## A. PDF-Export-Button

### Position
Unterhalb des Methodik-Abschnitts (`#methodology`), vor dem Disclaimer. Volle Breite, zentriert.

### Funktion
Button „📄 Bericht als PDF exportieren" — nutzt `window.print()` mit einem `@media print`-Stylesheet.

### Print-Stylesheet (in `<style>` ergänzen)

```css
@media print {
  header { break-after: avoid; }
  #inputs { display: none; }
  .btn, #optimizeBtn { display: none; }
  #optResults { display: none; }
  .chart-select { display: none; }
  main { display: block; }
  #results { max-width: 100%; }
  .chart-wrap { break-inside: avoid; }
  #methodology { break-before: page; }
  #methodology details { open: true; }
  #printBtn { display: none; }
  #disclaimer, #impressum { break-inside: avoid; }
  body { font-size: 11px; }
  .kpi-grid { grid-template-columns: repeat(4, 1fr); }
}
```

### Button-HTML

```html
<div style="text-align:center; padding:1.5rem 50px; max-width:1400px; margin:0 auto;">
  <button class="btn btn-opt" id="printBtn" onclick="window.print()" style="max-width:400px;">
    📄 Bericht als PDF exportieren
  </button>
</div>
```

---

## B. Fußnoten im Methodik-Abschnitt

### Prinzip
Alle Datenquellen und Annahmen im Methodik-Text mit hochgestellten Fußnotennummern [¹], [²] etc. markieren. Fußnoten-Liste am Ende des Methodik-Abschnitts (innerhalb `#methodContent`), vor dem Quellenverzeichnis.

### Fußnoten-Zuordnung

Die Funktion `updateMethodology()` muss folgende Fußnoten-Marker in den Text einfügen:

| Nr. | Stelle im Text | Fußnotentext |
|-----|----------------|-------------|
| ¹ | Bei "PVGIS-SARAH2" | Satellite Application Facility on Climate Monitoring (CM SAF). Datenbank: SARAH-2, Zeitraum 2005–2020. Abgerufen über PVGIS 5.2, EU Science Hub. https://re.jrc.ec.europa.eu/pvg_tools/en/ |
| ² | Bei Sonnenauf-/untergangszeiten | Astronomische Berechnung für den 15. jedes Monats, Mitteleuropäische Zeit (CET, UTC+1), ohne Sommerzeit. Berechnung nach NOAA Solar Calculator. |
| ³ | Bei Schneetage-Default-Tabelle | Vereinfachtes Modell basierend auf DWD-Klimadaten (Referenzperiode 1991–2020). Tatsächliche Schneedeckentage variieren regional erheblich. Tieflagen (<200 m): 0–40 Tage je nach Region; Mittelgebirge (400–600 m): 20–60 Tage; Hochlagen (>600 m): 60–120+ Tage. Die im Simulator verwendeten Defaults (0/5/15/34 Tage) sind konservative Mittelwerte für die PV-Ertragsminderung, nicht meteorologische Schneedeckentage. Quelle: DWD, „Schneedecke", https://www.dwd.de/DE/service/lexikon/begriffe/S/Schneedecke_pdf.pdf |
| ⁴ | Bei "Schneetage-Verteilung" (30/25/10/10/25%) | Empirische Verteilung der Schneetage auf die Wintermonate, abgeleitet aus DWD-Monatsmittelwerten für Mittelgebirgsstationen. |
| ⁵ | Bei "800 W AC-Obergrenze" | §12a EEG 2023 i.V.m. Solarpaket I (BGBl. 2024 I Nr. 151), in Kraft seit 16.05.2024. Erhöhung der Bagatellgrenze von 600 W auf 800 W Wechselrichterleistung für steckerfertige PV-Anlagen. |
| ⁶ | Bei "2,11 kWh LiFePO₄" | Herstellerangabe Solakon GmbH, Produkt „Solakon ONE", Stand 2024. Nutzbare Kapazität nach Entladetiefe (DoD). |
| ⁷ | Bei "92% Round-Trip" | Typischer Round-Trip-Wirkungsgrad für LiFePO₄-Zellen (Lade- × Entladewirkungsgrad). Literaturbereich: 90–95%. Quelle: Dubarry et al., „Battery Energy Storage System modeling", Journal of Power Sources, 2019. |
| ⁸ | Bei Spitzenlast-Korrektur (NEU) | Pauschale Modellierung: 50% des variablen Lastanteils entfällt auf Großverbraucher (Herd, Wasserkocher, Boiler) mit mittlerer Leistung 2000 W. Da der WR nur 800 W AC liefert, können davon max. 40% durch PV gedeckt werden. Konservative Annahme zur Kompensation der stündlichen Zeitauflösung. |

### Fußnoten-Rendering

Am Ende von `#methodContent`, VOR dem Quellenverzeichnis:

```html
<hr style="margin:1.5rem 0;">
<div style="font-size:0.75rem; color:var(--muted); line-height:1.6;">
  <p><sup>1</sup> CM SAF SARAH-2 … </p>
  <p><sup>2</sup> Astronomische Berechnung … </p>
  <!-- etc. -->
</div>
```

Fußnoten-Marker im Fließtext als `<sup style="color:var(--accent);cursor:help;" title="...">¹</sup>` — Tooltip zeigt den Kurztext.

---

## C. Disclaimer

### Position
Eigener Abschnitt nach dem PDF-Button, vor dem Impressum. Gleiche Breite wie `#methodology`.

### HTML-Struktur

```html
<section id="disclaimer" style="max-width:1400px; margin:1rem auto; padding:0 50px;">
  <div style="background:var(--panel); border:1px solid var(--border); border-radius:8px; padding:1.5rem; font-size:0.82rem; color:var(--muted); line-height:1.7;">
    <h3 style="font-size:0.9rem; color:var(--text); margin-bottom:0.75rem;">⚠️ Haftungsausschluss</h3>
    <p><!-- Text --></p>
  </div>
</section>
```

### Text (komplett übernehmen)

```
Dieser Simulator dient ausschließlich der informativen Abschätzung und stellt keine Energieberatung, Anlageberatung oder verbindliche Ertragsprognose dar. Alle Berechnungen basieren auf vereinfachten Modellen und historischen Durchschnittswerten — tatsächliche Erträge können aufgrund von Wetter, Verschattung, Modulalterung, Wechselrichterverlusten und weiteren Faktoren erheblich abweichen.

Wesentliche Modellvereinfachungen:
• Keine Berücksichtigung von Moduldegradation (typisch 0,3–0,5 %/Jahr)
• Keine Inflation der Strompreise oder Einspeisevergütung
• Keine Kapitalkosten (Opportunitätskosten, Finanzierung)
• Keine Temperaturabhängigkeit der Modulleistung
• Keine Verschattungsmodellierung (nur pauschales Derating)
• Keine Wechselrichterverluste (im Derating-Faktor pauschal enthalten)
• Keine saisonale Batteriedegradation oder Kalendarische Alterung
• Einstrahlungsdaten sind langjährige Mittelwerte — Einzeljahre können ±15 % abweichen
• Spitzenlast-Korrektur: 50 % des variablen Verbrauchs wird pauschal als Großverbraucher (2000 W) modelliert. Die tatsächliche Spitzenlastverteilung hängt vom individuellen Nutzungsverhalten ab.

Die angegebenen Einstrahlungsdaten stammen aus der PVGIS-SARAH2-Datenbank (Referenzperiode 2005–2020) und beziehen sich auf die horizontale Globalstrahlung. Reale PV-Erträge hängen zusätzlich von Ausrichtung, Neigung und lokaler Verschattung ab, die hier nur über den Derating-Faktor pauschal berücksichtigt werden.

Für verbindliche Ertragsprognosen und Wirtschaftlichkeitsberechnungen wird die Beratung durch einen qualifizierten Energieberater empfohlen.
```

---

## D. Impressum

### Position
Letzter Abschnitt der Seite, nach Disclaimer.

### HTML-Struktur

```html
<footer id="impressum" style="max-width:1400px; margin:1rem auto 2rem; padding:0 50px;">
  <div style="background:var(--panel); border:1px solid var(--border); border-radius:8px; padding:1.5rem; font-size:0.82rem; color:var(--muted); line-height:1.7;">
    <h3 style="font-size:0.9rem; color:var(--text); margin-bottom:0.75rem;">Impressum</h3>
    <p>
      Angaben gemäß § 5 DDG:<br>
      Spectabilis<br>
      E-Mail: 2025@bowo.de
    </p>
    <p style="margin-top:0.75rem;">
      Inhaltlich verantwortlich gemäß § 18 Abs. 2 MStV:<br>
      Spectabilis
    </p>
    <p style="margin-top:0.75rem; font-size:0.75rem;">
      Simulationsmodell und Implementierung: Claude (Anthropic) im Auftrag des Betreibers.<br>
      Einstrahlungsdaten: © European Commission, PVGIS-SARAH2.<br>
      Quellcode: <a href="https://github.com/orvillewilbur/pv" style="color:var(--accent);">github.com/orvillewilbur/pv</a>
    </p>
  </div>
</footer>
```

### Hinweis
§5 TMG wurde 2024 durch §5 DDG (Digitale-Dienste-Gesetz) abgelöst. Daher „gemäß § 5 DDG" statt „gemäß § 5 TMG".

---

## E. Reihenfolge am Seitenende

1. `</main>` (bestehend)
2. `#methodology` (bestehend)
3. PDF-Export-Button
4. `#disclaimer`
5. `#impressum`

---

## F. Responsive

Alle neuen Abschnitte: bei ≤920px `padding: 0 16px` statt `0 50px`. Bereits im bestehenden Media-Query ergänzen oder inline mit `max()`.

---

## Commit-Message

```
feat: Spitzenlast-Korrektur (50%/2000W), PDF-Export, Fußnoten, Disclaimer, Impressum
```
