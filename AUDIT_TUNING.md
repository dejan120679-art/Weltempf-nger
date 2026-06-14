# AUDIT_TUNING — Tiefen-Audit der Tuning-Funktion (v109, HEAD 562a23c)

Status: **DOKU ONLY**, keine Code-Änderung. Datei-Referenz: `index.html`.
Methodik (Karpathy): Annahmen explizit, jeden Befund mit Zeilennummer, nichts beschönigen.

---

## 0. Annahmen für die Analyse

- Beispiel-Band für Zahlenrechnungen ist SW2 (5200–10000 kHz, 10 Stationen). Daraus: `spacing = 4800/10 = 480 kHz`, `cap = 480 · 0.75 = 360 kHz`, `core = cap · 0.12 = 43 kHz`.
- "Realität" = ein analoger Mehrfach-Konversions-KW-Empfänger mit AGC und 6-kHz-AM-Filter. Selektivität ~±5–10 kHz, Übergang zwischen Stationen wenige kHz, nicht hunderte.
- Frames pro Sekunde des audioTick: ~30 (33 ms `setInterval`, Z. 1051 nicht hier referenziert — gesetzt in v95-Block).
- "Hin" = `tuneKhz` aufwärts, "Zurück" = abwärts.
- "Capture" / "Fang" = der Moment, in dem `loadPreset(bestNew)` aus dem audioTick gerufen wird.

---

## 1. Glocken-/Empfangskurve (computeRx, captureFor)

### Was der Code TUT

`captureFor` ([index.html:216–223](index.html:216)):
```js
function captureFor(stationIdx){
  const bw=BANDS[b].max-BANDS[b].min;
  const n=bandStationCount[b]||1;
  const spacing=bw/n;
  return Math.max(40, spacing*0.75);
}
```
→ `cap` ist eine reine Band-Charakteristik, **unabhängig von der konkreten Station**. Für SW2: 360 kHz. Für SW1 (3 Stationen): 550 kHz. Für SW4 (1 Station): 2625 kHz.

`computeRx` ([index.html:224–234](index.html:224)):
```js
function computeRx(off,cap,seed){
  const seedV=(seed||0)*0.025;
  const core=cap*0.12;
  const flankR=cap*(1+seedV), flankL=cap*(1-seedV);
  const a=Math.abs(off);
  if(a<core) return 1;
  const flank=off>=0?flankR:flankL;
  if(a>=flank) return 0;
  const t=(a-core)/Math.max(1e-6,flank-core);
  return 0.5*(1+Math.cos(Math.PI*t));
}
```

Form für SW2 (cap=360, core=43, seedV≈0.04 für seed≈1.5 → flankR≈374, flankL≈346):

| `|off|` [kHz] | t | rx |
|---|---|---|
| 0 | – | **1.000** (Kern) |
| 30 | – | **1.000** (Kern, flach) |
| 43 | 0.000 | 1.000 (Kern-Rand) |
| 80 | 0.112 | 0.971 |
| 150 | 0.323 | 0.751 |
| 200 | 0.474 | 0.539 |
| 250 | 0.625 | 0.327 |
| 300 | 0.776 | 0.124 |
| 350 | 0.927 | 0.013 |
| 360 | 0.957 | **0.005** |
| ≥374 (flankR) | 1.0+ | **0** (Schnitt) |

### Was real PASSIERT

Reale KW-Trichterkurve (6-kHz-AM-Filter): SNR fällt fast linear von Kern bis ca. ±10 kHz; ab ±10 kHz nur noch Rauschen mit Geisterresten. Höhen verschwinden zuerst (Lo-Pass-Verhalten), Verständlichkeit bleibt länger als Klangqualität. AGC zieht den Restpegel nach oben ("pumpt"), das Rauschen wird WÄHREND des Drehens hörbar lauter und das Signal stetig leiser. Übergang Leere→Station ist **stetig und sanft**, kein Plateau und kein scharfer Schnitt.

### Abweichung

- **Plateau im Kern**: `|off|<core` ⇒ `rx=1` ist eine **flache Kuppel ±43 kHz breit**. Innerhalb dieser ±43 kHz tut sich NICHTS — keine Pegel-Änderung, kein Klangwandel, kein wahrgenommenes "näher dran sein". **Schweregrad: BLOCKER**. Real ist die Selektivitäts-Kuppel an der Spitze rund, nicht flach. Im Kern müsste der Hauch von "im Sweet-Spot stehen" hörbar sein (gar nichts ändert sich → fühlt sich identisch an unter ±43 kHz).
- **Kantenschnitt bei `|off|≥flank`**: rx geht hart auf 0 (statt asymptotisch). Im Übergang von t=0.95 (rx≈0.013) zu t=1.0 (rx=0 hart) bricht ein winziger Schwanz ab. In der Praxis kaum hörbar, aber mathematisch kein C¹-stetiger Übergang. **Schweregrad: KOSMETIK**.
- **cap ist viel zu groß**: 360 kHz Halbradius bedeutet, eine Station hat ihre Glocke bis ±360 kHz, also fast bis zur Nachbar-Station hin. Real wäre die Selektivität ±5–10 kHz. Das v101-Design (Variante B, "überlappende Glocken") ist akustisch genau gegenteilig zur Realität: zu sanft und zu breit. **Schweregrad: STÖRT** (Designsentscheidung, aber realitätsfern).
- **Höhen-Verlauf fehlt in `computeRx`**: das ist über `lpcBase` separat modelliert (Z. 957), nicht hier — aber der KORRELIERTE Verfall (rx und Höhen-Bandbreite müssen zusammen schrumpfen) wird nicht über `rx` als Master gesteuert. Die LP-Frequenz ergibt sich aus `cur.lp * (P.lp/3785)`, nicht `rx`-gekoppelt. Dadurch: man kann eine Station mit voller Höhe hören, obwohl rx=0.5 — unphysikalisch. **Schweregrad: STÖRT**.

---

## 2. Fang-/Hysterese-Logik

### Was der Code TUT

audioTick ([index.html:835–858](index.html:835)):
```js
if(bandOf(cur.freq)===bandSel){
  off = tuneKhz - cur.freq;
  rxMain = computeRx(off, captureFor(idx), cur.seed);
}
// bestNew-Loop über alle Stationen im Band außer idx
const scanActive = !!scanTimer;
if(bestNew>=0 && (scanActive ? bestNewRx > 0.9 : (bestNewRx > rxMain + 0.15 && bestNewRx > 0.5))){
  loadPreset(bestNew);
  return;
}
```

Hysterese-Bedingung **ohne** scanTimer: `bestNewRx > rxMain + 0.15 && bestNewRx > 0.5`.

loadPreset ([index.html:648](index.html:648)):
```js
function loadPreset(i){ idx=...; cur=PRESETS[idx]; curR=ctlRange(cur);
  stopForeign(); renderReadout(); if(A.ctx) startForeign();
  applyPreset(); updateRescueUI();
  if(frozen){ frozen=false; physOff=0; freezeT=0; }
  updateFreezeUI(); if(extractMode){ extractMode=false; } updateRecUI(); }
```

### Was im Moment des Wechsels passiert (Inventar)

| Quelle | Effekt im Capture-Moment |
|---|---|
| `idx, cur, curR` | hart neu zugewiesen ([Z.648](index.html:648)) |
| `stopForeign()` | `A.fgSrc.stop()` + `A.fg.disconnect()` HARTE Trennung ([Z.646](index.html:646)) |
| `startForeign()` | neuer BufferSource mit `Math.random()*duration` Start, neuer Gain ([Z.620–644](index.html:620)) — Lied/Stimme der **vorher gespielten Fremdquelle wird unterbrochen, neue beginnt an zufälliger Stelle** |
| `applyPreset()` → `Object.assign(P,…)` ([Z.1259](index.html:1259)) | P.noise/fgLevel/breathe/lp/crush/het/nr/reverb/delay/wow/grain/warm — **alle hart auf neue cur.ctl-Werte gesetzt** ohne Ramp |
| `applyPreset()` → noiseFilt.type-Sandwich ([Z.1267–1278](index.html:1267)) | noiseGain rampt in 20 ms auf 0, 25 ms später Filter-Typ ersetzt, dann hochrampt → **kurze Rauschlücke ~40 ms** beim Wechsel |
| `applyControls()` ([Z.588–613](index.html:588)) | rHP/rComp/rMakeup/rTame/revWet/delWet/del.delayTime/delFb/fbLP/delLFOdepth/wowDepth/warmShelf/out.gain — alle via setTargetAtTime (~50 ms ramps) → **Ziele springen, Ist-Werte rampen** |
| `applyControls()` → `crushShaper.curve = distCurve(P.crush/100)` ([Z.606](index.html:606)) | **WaveShaper-Kurve hart ersetzt** — Spektrum schaltet sofort um |
| `frozen=false; physOff=0; freezeT=0` ([Z.648](index.html:648)) | wenn vorher Freeze aktiv, virtuelle Zeit T springt zurück auf `now` |
| `extractMode=false` ([Z.648](index.html:648)) | **Erzwungener Reset des Extrakt-Modus** beim Capture — Nutzer-Werkzeug wird stillschweigend abgeschaltet |
| audioTick `return` ([Z.857](index.html:857)) | dieser Frame wird übersprungen — alle bisher in diesem Tick geplanten Targets sind nicht gesetzt; **nächster Tick rechnet mit neuem cur** und springt entsprechende rx, q, songLvl, noiseBase, hetBase, lpcBase, k, og, etc. komplett um |
| `lastK` ([Z.964](index.html:964)) | wird im NÄCHSTEN Tick neu gesetzt wegen Sprung in cur.art — Shaper-Kurve wird erneut ersetzt |
| `rxMain` (im nächsten Tick) | von ~0.15 (vorher cur=A bei off=280) auf ~0.51 (jetzt cur=B bei off=200) — **Faktor 3.4 Sprung** in Empfangsstärke ⇒ songLvl springt entsprechend, q springt, noiseBase springt invers |

### Warum richtungsabhängig (Herleitung)

Beispiel SW2, A.freq=6000, B.freq=6480 (∆=480, typisches Spacing). cur=A initial.

**Vorwärts** (`tuneKhz` steigt):
- bei tuneKhz=6280 (off_A=+280, off_B=−200): rxA=0.151, rxB=0.508. Bedingung `0.508 > 0.151+0.15 = 0.301` ✓ und `0.508 > 0.5` ✓ → **FANG bei 6280 kHz**.

**Rückwärts nach Fang** (cur=B, `tuneKhz` fällt vom Fang-Punkt zurück):
- bei tuneKhz=6280 (off_B=−200): rxB=0.508. Kein Fang (rxA als bestNew nur 0.151, viel zu klein).
- bei tuneKhz=6200 (off_B=−280, off_A=+200): rxB=0.151, rxA=0.508. `0.508 > 0.301` ✓ und `>0.5` ✓ → **FANG bei 6200 kHz**.

⇒ Fang-Punkt **80 kHz Differenz** zwischen Vorwärts (6280) und Rückwärts (6200). Bei spatialer Position 6240 kHz (genau dazwischen):
- Vorwärts: cur=A, rxA=0.302 → Lied A
- Rückwärts: cur=B, rxB=0.302 → Lied B
**Selbe Frequenz, anderes Lied, andere Filter, anderer Rauschtyp.** Das ist Standard-Hysterese — **bewusst eingebaut gegen Flimmern**, aber genau dieselbe Hysterese erzeugt das vom User berichtete Symptom.

### Warum "hin-zurück-hin" anders ist als "hin"

- **Hin (erste Tour)**: cur=A → bei 6280 fire → cur=B mit allen oben gelisteten harten/sanften Sprüngen gleichzeitig → für ~50–200 ms hörbarer "Ruck" (Fremdquelle stoppt+startet hart, Shaper-Curve wechselt, noiseFilt-Sandwich-Rauschloch, Pegel-Targets neu).
- **Zurück, ohne A wieder zu fangen**: cur=B bleibt — du wanderst durch B's Glocke (rxB sinkt/steigt stetig), kein Fang-Event, kein Sprung, vollkommen stetig.
- **Hin wieder**: cur=B bleibt — du wanderst wieder durch B's Glocke aufwärts, rxB nähert sich 1, **weicher Übergang weil KEIN Capture-Event ausgelöst wird** (bestNewRx betrifft Nachbarn C, die noch nicht in Reichweite sind). User-wahrnehmung: "kommt natürlicher rein" — korrekt, weil das einzige weiche Wechseln (rxB von Flanke zu Kern) übrig bleibt.

⇒ Direktes Korrelat: **das "Ruckartig" entsteht im Capture-Event, nicht im Tunen selbst**. **Schweregrad: BLOCKER**.

### Sekundärbefunde

- **Hysterese-Schwelle 0.5 statisch**: Zwischen rxMain=0 (Leere) und rxMain=0.5 könnte der bestNewRx schon nahe 1 sein — das wartet aber trotzdem aufs Capture. Beispiel: cur=A weit weg, rxA=0.0, kommt aus Leere an B heran → bestNewRx steigt, fängt erst bei >0.5 (≈±200 kHz von B's Mitte). Davor ist B "unsichtbar" obwohl rxB schon hörbar wäre. **Schweregrad: STÖRT**.
- **`return` im audioTick nach loadPreset** ([Z.857](index.html:857)): unterbricht den Tick mitten in der Schreib-Sequenz. fg/fg2/songGain/noiseGain etc. für DIESEN Tick werden nicht aktualisiert. Erst nächster Tick. Das ist eine 33-ms-Lücke in der Audio-Steuerung mitten im Wechsel. **Schweregrad: STÖRT**.
- **`extractMode = false`** ([Z.648](index.html:648)): erzwungener Modus-Reset beim Tunen-Fang. User-Eingabe wird ohne Hinweis verworfen. **Schweregrad: STÖRT**.

---

## 3. Bleed (Übergang zwischen Stationen)

### Was der Code TUT

audioTick ([index.html:860–872](index.html:860)):
```js
if(rxMain<0.1){
  const near=nearestStationInBand(tuneKhz, bandSel);
  if(near){
    const np=PRESETS[near.idx];
    const off_n=tuneKhz-np.freq;
    const cap_n=captureFor(near.idx);
    if(Math.abs(off_n)<1.5*cap_n){
      const bell=computeRx(off_n/1.5, cap_n, np.seed);
      const rxBleed=0.12*bell;
      if(rxBleed>rxMain){ rxMain=rxBleed; bleedActive=true; }
    }
  }
}
```

### Was real PASSIERT

Im Übergang zwischen zwei nahen Stationen hört man real **beide gleichzeitig** mit unterschiedlichem Pegel, deren Heterodyn-Schwebung, und das Bandrauschen dazwischen. Lautstärke der einen geht stetig hoch während die andere stetig runtergeht — keine Lücke.

### Abweichung

- **Schwelle `rxMain<0.1` als Gate**: Solange rxMain für die GEFANGENE Station ≥0.1 ist, gibt es keinen Bleed. Aber rxMain hat einen flachen Verlauf — bei SW2 fällt rxMain unter 0.1 erst bei `|off|≥310 kHz`. Davor (von |off|=200 bis 310, also 110 kHz!) trägt **nichts vom Nachbarn** zum Sound bei. Hier müsste eigentlich schon der "Schemen" der nächsten Station mitlaufen. **Schweregrad: STÖRT**.
- **Bleed nutzt cur's Audio, nicht das des Nachbarn**: Sobald bleedActive=true, wird `rxMain = rxBleed` und damit `q = ... × rxBleed`. Das Lied/Foreign/Noise/Het hört der User als **CUR-Audio mit reduziertem Pegel** — nicht als Glimmer der Nachbarstation. Der Bleed-Effekt ist also "ferner Schemen der GEFANGENEN Station", obwohl es konzeptionell "ferner Schemen des NEBEN-Senders" sein sollte. **Schweregrad: BLOCKER** für Realismus.
- **Bleed-Glocke `computeRx(off_n/1.5, cap_n, ...)`**: das Argument `off_n/1.5` reduziert den effektiven Offset und erlaubt einen breiteren Wirkbereich. Aber im Übergangs-Bereich ist `bell` typischerweise 0.8–0.9 → `rxBleed = 0.096–0.108` — fast deckungsgleich mit der 0.1-Schwelle. Bei einem Geringfügig anderen rxMain (z.B. 0.105) bekommt man KEIN Bleed, sondern weiterhin "Schemen von cur". **Schweregrad: STÖRT** (Übergang ist nicht stetig).
- **`!bleedActive`-Guard im Het-Block** ([Z.972](index.html:972)): Het wird im Bleed-Bereich ZURÜCKGESETZT — gerade dort, wo die Schwebung zweier Träger real am stärksten ist. **Schweregrad: BLOCKER**.
- **Kein "doppelte-Lieder"-Pfad**: Das System hat genau einen `songGain` für genau ein Lied. Mehrere Stationen gleichzeitig zu hören würde mehrere Lied-Quellen verlangen — strukturell aktuell nicht möglich. **Schweregrad: STÖRT** (Designgrenze).

---

## 4. Heterodyn beim Tunen

### Was der Code TUT

audioTick ([index.html:970–978](index.html:970)):
```js
let hetFqOut=500+cur.q0*1400;
let hetLevelOut=0;
if(rxMain>0.01 && !bleedActive){
  hetFqOut=Math.max(40, Math.abs(off)*180);
  const bellFlank=1-rxMain;
  hetLevelOut=hetBase*bellFlank*2.0;
}
```

### Was real PASSIERT

Het = Schwebungsnote zweier Träger, einer ist der eigene Empfänger-BFO/Mischer, der andere die Station. Reale Hörbarkeit: konstant, solange ein Träger im Durchlassbereich liegt — wandert linear mit Verstimmung (1 kHz Detuning = 1 kHz Schwebungston). Verschwindet bei Zero-Beat (Mitte der Station) und steigt nach außen, IST AUCH ZWISCHEN STATIONEN HÖRBAR (zwei Schwebungstöne, einer pro Station).

### Abweichung

- **Faktor 180 in `|off|*180`**: Real wären es 1000 (1 kHz Detuning = 1000 Hz Pfeifton). Mit Faktor 180:
  - off=10 kHz → 1800 Hz Het — passt.
  - off=30 kHz → 5400 Hz — passt für hohe Töne.
  - off=50 kHz → 9000 Hz — Grenzbereich hörbar.
  - off=100 kHz → 18000 Hz — schon kaum hörbar.
  - off=200 kHz → 36000 Hz — unhörbar (Nyquist-Limit).
  
  Da die Glocke bis 360 kHz reicht, ist der Het über die meiste Strecke der Glocke **unhörbar hoch**. Spürbar nur in den ersten ±50 kHz vom Kern. **Schweregrad: STÖRT** (Faktor zu klein gewählt für die viel zu große cap, Inkonsistenz zwischen cap-Geometrie und het-Skalierung).
- **`!bleedActive`-Guard**: Het = 0 im Bleed-Bereich — siehe oben, **BLOCKER**. Im realen Übergang sollte Het besonders stark sein.
- **Nur eine Het-Quelle** (`A.hetOsc`): es kann nur EIN Schwebungston gleichzeitig wandern. Real: zwei nahe Stationen erzeugen zwei Schwebungstöne. Mit einer Quelle nicht modellierbar. **Schweregrad: STÖRT** (Designgrenze).
- **`hetFqOut` Default `500+cur.q0*1400`** außerhalb der Capture-Zone wird gesetzt aber Pegel ist 0 → unsichtbar. Aber: der Default wandert beim cur-Wechsel (q0 jumpt) — der OSC-Sweep ist sanft (setTargetAtTime .1) aber wenn Hetgain im nächsten Capture wieder hochkommt, hat hetOsc bereits eine andere Frequenz angefahren. **Schweregrad: KOSMETIK**.

---

## 5. Parameter-Ramping: hart vs. weich beim Tunen/Fangen

### Hart gesetzt (= Knack/Ruck)

| Wert | Zeile | Effekt |
|---|---|---|
| `idx, cur, curR` | [Z.648](index.html:648) | sofort |
| `A.fgSrc.stop()` + `A.fg.disconnect()` | [Z.646](index.html:646) | sofortiger Audio-Abbruch der Fremdquelle |
| Object.assign(P, {…}) | [Z.1259](index.html:1259) | P.noise, fgLevel, breathe, lp, crush, het, nr, reverb, delay, wow, grain, warm — **alle hart**, audioTick liest neue Werte ab nächstem Tick |
| `A.noiseFilt.type = np.filt` | [Z.1273](index.html:1273) | BiquadFilter-Typ nicht rampbar, Sandwich-Trick: 20 ms Mute → Typ-Wechsel → ~70 ms Hochkommen |
| `crushShaper.curve = distCurve(...)` | [Z.606](index.html:606) | WaveShaper-Kurve sofort ersetzt |
| `A.shaper.curve = tanhCurve(k)` | [Z.964](index.html:964) | WaveShaper-Kurve sofort ersetzt sobald `|k-lastK|>0.02` |
| `frozen, physOff, freezeT = false/0/0` | [Z.648](index.html:648) | hart wenn frozen war, T springt |
| `extractMode = false` | [Z.648](index.html:648) | Modus hart abgeschaltet |
| `renderReadout()` Texte | [Z.799–808](index.html:799) | sofortiger DOM-Update |
| `setTuneKhz` rundet auf `Math.round(v)` | [Z.698](index.html:698) | tuneKhz quantisiert auf 1 kHz |

### Weich (setTargetAtTime)

| Wert | Zeile | τ (s) |
|---|---|---|
| `A.songGain.gain` | [Z.955](index.html:955) | .008 (.015 bei Extrakt) |
| `A.noiseGain.gain` | [Z.956](index.html:956) | .06 |
| `A.fg.gain, A.fg2.gain` | [Z.953–954](index.html:953) | .08 / .1 |
| `A.lp.frequency` | [Z.960](index.html:960) | .04 |
| `A.noiseFilt.frequency` | [Z.901](index.html:901) | .1 |
| `A.crushWet/Dry.gain` | [Z.966–967](index.html:966) | .08 |
| `A.hetGain.gain` | [Z.977](index.html:977) | .05 |
| `A.hetOsc.frequency` | [Z.978](index.html:978) | .1 |
| `A.sweepGain.gain` | [Z.980–981](index.html:980) | .04 / .1 |
| `A.chainGate.gain` | [Z.993](index.html:993) | .08 |
| `A.origGate.gain` | [Z.991](index.html:991) | .08 |
| `A.origLP.frequency` | [Z.990](index.html:990) | .05 |
| `A.revIn*, A.delIn*` | [Z.947–950](index.html:947) | .06 |
| `A.grainDryOut.gain` | [Z.952](index.html:952) | .06 |
| applyControls-Block (rHP, rComp, rMakeup, rTame, revWet, delWet, del.delayTime, delFb, fbLP, delLFOdepth, wowDepth, warmShelf, out.gain) | [Z.591–612](index.html:591) | .05–.06 |
| Pool-Voices, Wuseln, Geister | [Z.954, 920, 967–979](index.html:920) | .02–.12 |
| `deEss.gain` Sidechain | [Z.1041–1046](index.html:1041) | .03 / .15 |

### Sub-Befund: Sprung in Ziel-Werten, nicht in Ist-Werten

Selbst wo `setTargetAtTime` läuft: das **Target springt** beim cur-Wechsel um Faktor 2–5 in vielen Parametern (cur.q0, cur.gain, cur.noise, cur.art, cur.lp, cur.fadeDepth, cur.seed sind alle gleichzeitig neu). Das Ist-Wert-Ramp glättet zwar, **aber die Summe der gleichzeitig wandernden Ziele liefert eine erkennbare "Phase der Neuordnung" über ~150–300 ms** nach dem Capture. Das ist genau die Wahrnehmung "Ruck". **Schweregrad: BLOCKER**.

---

## 6. Richtungs- und Zustands-Abhängigkeit

### Stellen mit Zustand

| Variable | Wo gelesen | Wo geschrieben | Wirkung |
|---|---|---|---|
| `cur, idx` | überall im audioTick | loadPreset | bestimmt alle stationsspezifischen Parameter |
| `tuneKhz` | audioTick (Z.836, 844, 864), setTuneKhz | setTuneKhz, startScan, setBand, setMode | wandert mit User-Eingabe |
| `bandSel` | audioTick (Z.842) | setBand, setMode | begrenzt Capture-Suche aufs aktive Band |
| `lastDisplayIdx/Rx` | uiTick | audioTick Z.849–850 | NUR Anzeige, kein Audio |
| `breath.t, breath.v` | audioTick Z.908, 887 | audioTick Z.887 (Random!) | persistiert über loadPreset, Random-Komponente |
| `T = now - physOff` | audioTick | physOff via loadPreset Z.648 (nur wenn frozen war) | sonst läuft T monoton — fade/Wob sind also zeit-, nicht zustandsabhängig |
| `lastK` | audioTick Z.964 | audioTick Z.964 | Cache der Shaper-Kurve, persistiert |
| AudioParam-Ist-Werte | Web Audio intern | setTargetAtTime-Targets | "schleppen" sich an Ziel-Werte an, Zustand zwischen Ticks erhalten |
| `frozen, physOff, freezeT` | audioTick | loadPreset Z.648 | Freeze geht beim Fang verloren |
| `extractMode` | audioTick Z.894, 926 | loadPreset Z.648 | Extrakt geht beim Fang verloren |
| `bandGhosts, bandNightSignals, bandWuseln, bandStationCount` | audioTick, ensureBand*, captureFor | lazy initialisiert | rein deterministisch pro Band, kein Tuning-State |

### Direkte Erklärung "hin-zurück-hin"

Pfad "hin" (cur=A):
1. tuneKhz steigt von A.freq Richtung B.freq
2. rxMain (= rxA) fällt von 1 → 0.15
3. bestNewRx (= rxB) steigt von 0 → 0.5
4. Bei tuneKhz ≈ A.freq+280: Hysterese-Bedingung greift → loadPreset(B)
5. Im Capture-Moment: harte Sprünge aller stationsspezifischer Werte (siehe §5). Wahrnehmung: **RUCK**.

Pfad "zurück" (jetzt cur=B):
1. tuneKhz fällt von Fang-Punkt zurück Richtung A.freq
2. rxMain (= rxB) fällt von 0.5 → wieder kleiner
3. bestNewRx (= rxA) steigt langsam
4. Wenn User VOR `tuneKhz=6200` umkehrt: **kein Fang**, cur=B bleibt. Keine harten Sprünge. Stetiges Wandern in B's Glocke.
5. Sonst (durchgezogen): Fang fires bei `tuneKhz=6200` → loadPreset(A) mit erneutem Ruck.

Pfad "hin wieder" (Annahme User hat NICHT bis 6200 zurückgedreht, cur=B):
1. tuneKhz steigt wieder vom umgekehrten Punkt
2. cur=B bleibt, rxB wandert in seinen Kern (Richtung 1)
3. **Kein Fang-Event nötig** → stetiges Approach an B's Kern → klingt "natürlich".

⇒ Das vom User berichtete Phänomen ist eine **direkte Konsequenz der Hysterese + harter Capture-Aktionen**. Ohne Capture-Event ist das Tuning **bereits stetig**; mit Capture ist es **ruckartig**.

### Sekundäre Zustands-Effekte (kleiner)

- `breath.v` Random alle ~700 ms ([Z.887](index.html:887)) → Pegel-Faktor ±5%, persistent → identische Replay nicht möglich (aber nicht direktional, nur stochastisch).
- `T` läuft frei → fade01-Hüllkurve unterscheidet sich zwischen zwei verschiedenen Zeitpunkten, auch bei identischer cur. Hin-Zurück-Hin im Abstand von Sekunden hat verschiedene fade-Phasen.
- `A.shaper.curve` (Cache via `lastK`): wird beim Wechsel neu gesetzt, hörbarer Sprung in Sättigung. Zwischen den Capture-Events stabil.
- ghosts / wuseln / night-signals: deterministisch pro Band-Position, **nicht** zustandsabhängig nach Capture — diese Schicht ist richtungsneutral. ✓

---

## 7. Soll-Bild — ideales realistisches KW-Tuning (als Zielvorgabe)

1. **Kein flaches Kern-Plateau**: rx-Kurve eine glatte Glocke ohne Sweet-Spot-Plateau. Auch im Kern fühlt sich die exakte Mitte HÖRBAR anders an als ±10 kHz daneben (volle Höhen, Rauschen minimal, AGC ruhig).
2. **Schmalere `cap`**: ±5–10 kHz typischer Selektivitäts-Halbradius. Stationen sind klar voneinander getrennt, nicht überlappend bis ±360 kHz.
3. **rx-gekoppelte Höhen**: `lpc` muss multiplikativ mit rx schrumpfen — bei rx=0.5 nur halbe Bandbreite. Höhen verschwinden zuerst beim Wegdrehen, kommen zuletzt beim Herandrehen.
4. **AGC-Pumpen**: Beim Übergang von voller Empfangsstärke zu Flanken-Bereich kompensiert die AGC kurz → Restpegel bleibt fast konstant, dann reißt es ab. Heute fehlt das (`songLvl` folgt q linear).
5. **Kein Capture-Event als Sprung**: Stationswechsel ist ein STETIGER CROSSFADE zwischen zwei Stationen. Beide Lieder/Fremdquellen sind kurzzeitig hörbar (mit jeweiligem Pegel × bell), filter/noise/het wandern als Cross-Mix. Keine harten Re-Assignments.
6. **Stetiger Bleed**: Pegel der Nachbar-Station folgt seiner eigenen Glocke ohne Schwelle. Im Übergang sind beide Stationen anteilig hörbar.
7. **Heterodyn-Schwebung durchgehend**: Auch außerhalb der Capture-Zone (zwischen Stationen) hörbar als Pfeifton, der mit `tuneKhz` wandert. Sinkt nur ganz weit weg auf 0. Faktor näher an 1000 Hz/kHz (real), nicht 180.
8. **Richtungsunabhängig**: spatiale Position → eindeutiger Klang. Identische tuneKhz → identische Stations-Mischung, egal ob aus Vorwärts- oder Rückwärtsrichtung erreicht.
9. **User-Modi werden NICHT erzwungen zurückgesetzt** (extractMode, frozen) beim Tunen-Fang. Tunen ist Tunen, Modus ist Modus.
10. **Foreign-Quellen wechseln per Crossfade**, nicht per `stop()+start()`. Mehrere persistente Slots, gain-modulated nach welcher Station gerade dominiert.

---

## 8. Befundliste (sortiert)

### BLOCKER (bricht Realismus / verursacht Ruck)

| # | Symptom | Ursache | Zeile | SOLL |
|---|---|---|---|---|
| B1 | "Ruckartig" beim Fang — hörbarer Sprung von ~150–300 ms | Capture-Event setzt `cur, P.*, foreign-Sources, shaper.curve, noiseFilt.type` gleichzeitig | [648](index.html:648), [1259](index.html:1259), [646](index.html:646), [606](index.html:606), [964](index.html:964), [1273](index.html:1273) | Crossfade zwischen alter und neuer Station über ~200–500 ms; keine harten Re-Assignments |
| B2 | "Vorwärts ≠ rückwärts" — selber Punkt anders gefangen | Hysterese-Schwelle 0.5 erzeugt 80 kHz Differenz zwischen Vor- und Rückwärts-Fang-Punkt | [855](index.html:855) | Symmetrischer Cross-Mix: rx zweier Stationen blenden über, beider Pegel = bell(off_i), Summe normalisiert |
| B3 | Sweet-Spot-Plateau ±43 kHz: Mitte fühlt sich gleich an wie ±43 kHz | `computeRx` returns `1` im Kern flach | [229](index.html:229) | Glatte Kuppel: rx geht von 1 in der Mitte stetig (z.B. cos² oder gauß-artig) runter, kein Plateau |
| B4 | Bleed-Lücke 200–310 kHz (Schwelle `rxMain<0.1`) — Sender entgleitet bevor Nachbar einsetzt | Bleed-Gate hart bei 0.1 | [860](index.html:860) | Beide Stationen jederzeit anteilig hörbar gemäß ihrer eigenen Glocke; keine Gate-Schwelle |
| B5 | Het verschwindet im Bleed-Bereich — gerade dort wo Schwebung real am stärksten ist | `if(rxMain>0.01 && !bleedActive)` | [972](index.html:972) | Het immer aktiv solange irgendeine Station in Reichweite, Schwebung folgt minimalem `|off|` aller Nachbar-Stationen |
| B6 | Bleed zeigt CUR-Audio statt Nachbar-Audio — "Schemen" sind das gefangene Lied, nicht das fremde | Bleed setzt nur `rxMain = rxBleed`, ändert NICHT die Audio-Quelle | [869](index.html:869) | Mehrere persistente Foreign-Slots, gain-modulated pro Station; im Bleed klingt der Nachbar |

### STÖRT (realitätsfern, aber kein Ruck)

| # | Symptom | Ursache | Zeile | SOLL |
|---|---|---|---|---|
| S1 | `cap=360 kHz` für SW2 — viel zu breit, Stationen überlappen massiv | `captureFor` mit Faktor 0.75 auf Spacing | [222](index.html:222) | `cap = 5–15 kHz` typisch, ±5 kHz Kern, ±10 kHz Flanke |
| S2 | Höhen nicht rx-gekoppelt — man kann Lied mit voller Bandbreite hören bei rx=0.5 | `lpcBase = cur.lp*(P.lp/3785)`, dann via `baseLpcQ = 900+q*(lpc-900)` indirekt q-gekoppelt, aber `lpc` selbst freq-fest | [957–959](index.html:957) | `lpc` direkt mit `rx` skalieren, nicht nur via q |
| S3 | Fang erfolgt erst bei `bestNewRx > 0.5`, davor ist Nachbar "unsichtbar" | Hysterese-Bedingung statisch | [855](index.html:855) | Bei Crossfade-Modell entfällt Schwelle ganz |
| S4 | `extractMode` und `frozen` werden beim Fang erzwungen zurückgesetzt | `loadPreset` Z.648 hart | [648](index.html:648) | Modi bleiben erhalten, Tunen wirkt orthogonal |
| S5 | `stopForeign() + startForeign()` reißt Fremdquelle hart ab, neue startet an Random-Position | Implementierung als BufferSource ohne Crossfade | [646, 620](index.html:620) | Persistent N Foreign-Slots, gain-modulated |
| S6 | Het-Faktor 180 Hz/kHz → mostly inaudible über das meiste der Glocke | `Math.abs(off)*180` | [973](index.html:973) | Faktor näher an 1000 Hz/kHz wenn cap auf realistische Größe schrumpft |
| S7 | audioTick `return` nach loadPreset überspringt diesen Tick → 33-ms-Schreib-Lücke | Z.857 | [857](index.html:857) | Kein early-return, nächster Tick rechnet sauber weiter |
| S8 | `Object.assign(P, …)` hart, keine Ramp auf User-relevante P-Werte | `applyPreset` Z.1259 | [1259](index.html:1259) | P-Werte sind Sollwerte für stationsspezifische Defaults; hart OK, ABER bei Crossfade-Modell ohnehin gemittelt |
| S9 | Shaper-Curve hart ersetzt bei k-Sprung → spektrale Diskontinuität im Sättigungs-Charakter | [964](index.html:964), [606](index.html:606) | Curve-Crossfade über 2 WaveShaper parallel mit gegenläufigen Gains |
| S10 | noiseFilt.type via setTimeout(25) → vom Browser-Timer abhängig, glitchy bei Hintergrund-Tab | [1271–1278](index.html:1271) | Zwei parallele BiquadFilter mit gegenläufigem Gain-Crossfade statt Typ-Switch |
| S11 | `frozen=false; physOff=0; freezeT=0` erzwingt T-Reset auf now, fade-Hüllkurve springt phasen-mäßig | [648](index.html:648) | Freeze und Capture entkoppeln |
| S12 | hetFqOut-Default `500+cur.q0*1400` wandert beim cur-Wechsel, OSC-Phase phantom-driftet | [970, 978](index.html:970) | hetFq=0 oder fester Park-Wert wenn ungenutzt |

### KOSMETIK

| # | Symptom | Ursache | Zeile | SOLL |
|---|---|---|---|---|
| K1 | `tuneKhz` rundet auf 1 kHz, kein sub-kHz-Detail | `Math.round(v)` | [698](index.html:698) | Float-Frequenz für glatten Heterodyn-Pitch-Sweep |
| K2 | computeRx-Kantenschnitt bei flank: C¹-unstetig (cos-Verlauf endet abrupt bei rx=0) | [231](index.html:231) | cos² oder einfach asymptotischer Tail über cap hinaus |
| K3 | `captureFor` min 40 kHz arbiträr | [222](index.html:222) | Minimum eher 8–10 kHz |
| K4 | SCAN-Step 8 kHz → grobe Quantisierung beim animierten Scan, hörbar diskret | [779](index.html:779) | 1–2 kHz Steps mit höherer Tick-Rate, oder Ramp auf scanTimer-Basis |
| K5 | `breath.v = 0.92+Math.random()*0.10` → minimale Pegel-Stochastik, identische Replay unmöglich | [887](index.html:887) | OK, ist Designentscheidung; bei Crossfade-Modell weiter so |
| K6 | `lastDisplayIdx` zwischen idx und bestNew springt → Anzeige-Ruckeln am Übergang | [849–850](index.html:849) | Hysterese auch in der Display-Logik oder kontinuierlicher Mix-Indikator |

---

## Kurzfassung für die Diagnose

Die "Ruckartigkeit" und Richtungsabhängigkeit sind **kein Bug in einer einzelnen Zeile**, sondern **strukturelle Folgen** dreier Entscheidungen:

1. **Diskrete Stationen mit `cur` als globaler Single-Station-State** statt eines kontinuierlichen Mix-Modells.
2. **Hysterese um Capture-Flimmern zu verhindern** ⇒ kommt automatisch Richtungsabhängigkeit.
3. **Capture-Event reaßiniiert mehrere Pfade gleichzeitig hart** (Foreign-Source, noiseFilt.type, Shaper-Curve, P.*) ⇒ wahrnehmbare ~200 ms "Phase der Neuordnung".

Ein realistischer Fix führt am ehesten über **Crossfade-Mix mehrerer Stationen ohne diskreten cur-Wechsel**: jede Station hat permanent ihre eigene Foreign-Audio-Quelle (oder Pool davon), Pegel jeweils per `bell(off_i)`, mit einem "fokussierten" Index für UI/Steuerung der Slider, aber ohne dass der Audiopfad daran fest hängt. Dann sind §1, §2, §3, §5 und §6 strukturell aufgelöst.

**Empfehlung Reihenfolge**: B3 (Plateau weg) → B2/B4 (Crossfade statt Hysterese) → B1 (Fang als Crossfade-Anker statt Re-Init) → S2/S6 (Höhen-Kopplung, Het-Faktor) → Rest.
