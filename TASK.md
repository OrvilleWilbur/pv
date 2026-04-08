# Aktueller Auftrag: Bugfixes + UI-Verbesserungen

Lies `SPEZIFIKATION_PV_SIMULATOR.md` als Referenz. Folgende Bugs und Änderungen in `index.html` umsetzen:

---

## KRITISCHE BUGS

### Bug 1: Batteriekosten fehlen in Investitionsrechnung

**Problem:** Investitionsrechnung ignoriert Batteriegröße. Zeile ~647:
```javascript
const investment = p.nModules * p.moduleCost + p.nWR * p.wrCost + p.otherCost;
```
4, 6, 8 kWh Batterie → identische Investition. Optimierung ist damit wertlos.

**Fix:**
1. Neues Eingabefeld: „Batteriekosten pro kWh" (Default: 200€, Bereich 0–500€)
2. Der WR-Preis (669€) enthält bereits 2.11 kWh Basis-Batterie pro WR. Zusätzliche kWh kosten extra.
3. Investitionsrechnung ändern:
```javascript
const baseBatKWh = p.nWR * 2.11;
const extraBatKWh = Math.max(0, p.batKWh - baseBatKWh);
const investment = p.nModules * p.moduleCost + p.nWR * p.wrCost + p.otherCost + extraBatKWh * p.batteryCostPerKwh;
```
4. Optimierungsfunktion: gleiche Formel verwenden.

### Bug 2: AC-Limit fehlt bei Nacht-Entladung

**Problem:** Batterie entlädt nachts ohne AC-Limit-Prüfung.

**Fix:** In BEIDEN Funktionen (`simulateYear()` und `simulateYearTotals()`) bei Nacht-Entladung:
```javascript
const dis = Math.min(loadKWh / batEff, batSOC, acMaxW / 1000 / batEff);
```

### Bug 3: AC-Limit bei Tages-Batterieentladung

**Problem:** Wenn tagsüber PV + Batterieentladung zusammen das AC-Limit übersteigen, wird das nicht geprüft.

**Fix:** Verbleibende AC-Kapazität nach PV→Load berechnen und Batterieentladung darauf begrenzen:
```javascript
const acRemaining = acMaxW / 1000 - pvForLoadKWh;
if (remLoad > 1e-9 && batSOC > 1e-9 && acRemaining > 0) {
  const dis = Math.min(remLoad / batEff, batSOC, acRemaining / batEff);
}
```
Gleiche Logik in `simulateYearTotals()`.

---

## UI-VERBESSERUNGEN

### 4. Batteriewirkungsgrad + Derating: Slider → Eingabefeld

**Problem:** Diese Werte stellt man einmal ein und bewegt sie nicht interaktiv. Slider sind hier unnötig.

**Fix:** Beide zu normalen Zahleneingabefeldern (input type="number") ändern. Derating: step=5, min=0, max=90. Batteriewirkungsgrad: step=1, min=80, max=98.

### 5. Infoboxen (Fragezeichen-Icons) funktionieren nicht

**Problem:** Die Infobox-Tooltips (?) neben den Parametern zeigen keinen Text an.

**Fix:** Prüfen ob die Tooltip/Infobox-Logik korrekt implementiert ist. Jeder Parameter mit Infobox-Text aus der Spezifikation muss beim Hover oder Klick den Hilfetext anzeigen. Einfachste Variante: HTML `title`-Attribut auf dem ?-Icon. Bessere Variante: Aufklapp-Box die bei Klick den Text einblendet.

Infobox-Texte aus der Spezifikation:
- Wp pro Modul: „Nennleistung unter Standardtestbedingungen (STC)"
- AC-Limit: „Gesetzliche Einspeisegrenze BKW: 800W seit 2024"
- Speicherkapazität: „Nutzbare Kapazität nach Entladetiefe (DoD)"
- Batteriewirkungsgrad: „Verluste beim Laden und Entladen (Round-Trip)"
- Derating: „0% = optimal, 20% = leichte Schatten, 50% = Flachdach verschattet, 70%+ = stark verschattet"
- Grundlast: „Anteil des Verbrauchs, der 24/7 konstant anfällt: Kühlschränke, Router, Standby, Heizungspumpe"
- Aktive Stunden: „In wie vielen der 8 Stunden wird tatsächlich Strom über die Grundlast hinaus verbraucht? z.B. 4h = Berufstätige, die morgens aktiv sind und dann das Haus verlassen"

### 6. „Konzentration" umbenennen → „Aktive Stunden"

**Problem:** „Konzentration 50%" ist unverständlich.

**Fix:** Umbenennen zu „Aktive Stunden" pro Zeitfenster. Statt Prozent-Slider (50–100%) ein Zahlenfeld:
- Eingabe: Ganzzahl 1–8 (bei 8h-Zeitfenstern)
- Default: 4 (von 8 Stunden)
- Bedeutung: Der variable Verbrauch fällt in die ERSTEN X von 8 Stunden des Zeitfensters. Die restlichen Stunden: nur Grundlast.
- Beispiel Vormittag (6–14 Uhr), Aktive Stunden = 4: Verbrauch 6–10 Uhr, danach nur Grundlast.
- Label: „Aktive Std." mit Infobox (s.o.)

### 7. Optimierung: Klarer auf Module + Speicher fokussieren

**Problem:** Optimierung zeigt WR-Anzahl als eigene Variable. Aber WR-Anzahl ergibt sich aus Modulzahl (max 4 Module pro WR, also nWR = ceil(nModule/4)).

**Fix:**
1. WR-Anzahl in der Optimierung NICHT als freie Variable. Stattdessen: `nWR = Math.ceil(nModules / 4)`
2. Optimierung variiert nur: Module (1–16) × Speicher (0, 2, 4, 6, 8, 10 kWh) = 96 Kombinationen
3. Ergebnis-Tabelle:

| Rang | Module | Speicher | (WR) | Investition | Ertrag/a | Amort. | 10J-Gewinn |

WR wird in Klammern angezeigt (abgeleitet, nicht variiert).

4. Unter der Top-3-Tabelle ein Satz: „WR-Anzahl ergibt sich aus Modulzahl (max. 4 Module pro Wechselrichter)."

---

## NACH DEN FIXES

Validiere:
- Schmiedefeld, 8 Module, 50% Derating, 4000 kWh Verbrauch, 65% Grundlast
- Erwarteter Deckungsgrad: ~60-65%
- Optimierung muss unterschiedliche Investitionen und Erträge für verschiedene Batteriegößen zeigen
- Alle Infoboxen müssen Text anzeigen
- „Aktive Stunden" statt „Konzentration" im UI
