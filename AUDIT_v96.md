# Weltempfänger · Werkbank SW — Komplett-Audit v96

**Datei-Stand:** `index.html` auf Commit `6c83bec` (v96, 663 Zeilen, 49 KB).
**Audit-Methode:** Read-only. Keine Code-Änderung. Karpathy-Guidelines:
Annahmen offen, Unklarheiten benannt, kein Beschönigen.

> **Hinweis zur Vorgabe:** Der Auftrag verlangte ursprünglich Audit auf
> v95 (HEAD `518e9d6`). Tatsächlicher HEAD war beim Aufruf aber `6c83bec`
> (v96 — FREEZE + Dry/Wet als zusammenatmende Öffnung; gepusht in der
> vorherigen Session). Auf Bitte des Users wurde auf v96 umgeschwenkt.
> Dieser Bericht beschreibt v96.

---

## 0. Modul-Topologie

Single-File HTML/CSS/JS. Tailwind aus CDN. Inline-Script
([index.html:92-661](index.html:92)). Keine externen Module außer den
audio/-Snippets unter `audio/` und MP3-Stream-URLs.

**Top-Level-Lebenszyklus:**
- Skript-Load: PRESETS, POOL, Helfer definiert; UI-Render läuft (Sliders, Buttons); `applyPreset()` setzt Initial-Werte; Endlos-Loops `crackle()`, `grains()`, `uiTick()` und `setInterval(audioTick, 33)` werden gestartet ([index.html:440, 446, 462, 518](index.html:440)).
- `ensure()` baut den Audio-Graph einmalig auf — getriggert beim ersten `togglePlay`, `loadStream`, `setExtract`, `setFreeze` ([index.html:213-308](index.html:213)).
- Audio-Modulation läuft per `setInterval(audioTick, 33)` (~30 Hz; im Hintergrund-Tab Browser-Drosselung auf ~1 Hz akzeptiert per Spec v95).

---

## 1. Signalfluss-Karte

### 1.1 Quellen

| Quelle | Anlage in `ensure()` | Type | Direkte Ziele |
|---|---|---|---|
| `songSrc` (BufferSource für Lied-File) | [index.html:520](index.html:520) `startSong` | mono über `songGain` | `songGain`, `origRaw` |
| `streamSrc` (MediaElementSource für Stream) | [index.html:525](index.html:525) `loadStream` | wie geladen | `songGain`, `origRaw` |
| `noiseSrc` (BufferSource, 2 s Weißrauschen, loop) | [index.html:287-290](index.html:287) | mono | `noiseFilt` |
| `hetOsc` (Sinus) | [index.html:301-302](index.html:301) | mono | `hetGain` |
| `sweepOsc` (Sawtooth) | [index.html:303-304](index.html:303) | mono | `sweepGain` |
| `fgSrc` (BufferSource für Fremdsignal #1) | [index.html:334](index.html:334) `startForeign` | wie Buffer | `fg` |
| `fgSrc2` (BufferSource für Fremdsignal #2) | [index.html:343](index.html:343) `startForeign` | wie Buffer | `fg2` |
| `wowLFO` (Sinus 5.2 Hz) | [index.html:219-220](index.html:219) | n/a (steuert `wow.delayTime`) | `wowDepth → wow.delayTime` |
| Crackle-BufferSource (Ad-hoc je Trigger) | [index.html:454](index.html:454) | mono | `bp → g → bus` |
| Grain-BufferSource (Ad-hoc je Korn) | [index.html:494](index.html:494) | mono | `g → grainOut` |

### 1.2 Empfänger-Kette (Bus → Mix)

```
bus
 → wow (Delay 12 ms, LFO-moduliert)                       [index.html:263]
 → hp (Highpass 320 Hz)                                   [index.html:263]
 → lp (Lowpass, dynamisch in audioTick)                   [index.html:263, 422]
 → pkf (Peaking 1700 Hz, Q 1.1, +4.5 dB)                  [index.html:263]
 → shaper (WaveShaper, tanh(k); k dynamisch via audioTick) [index.html:263, 425]
 → ┬─→ crushShaper (WaveShaper, distCurve) ─→ crushWet (=1) ─┐  [index.html:264, 322]
   └─→                                       crushDry (=0) ──┤  [index.html:265]
                                                             ▼
 → agc (DynamicsCompressor: thr -28, ratio 4, atk 0.05, rel 0.4)  [index.html:266]
 → rHP (Highpass dynamisch via Rettung: 120 + nr*300 Hz)   [index.html:266, 313]
 → rComp (Rettung-Kompressor; dynamisch)                   [index.html:266, 314-316]
 → rMakeup (Gain dynamisch: 1 + nr*3.3)                    [index.html:266, 317]
 → rTame (Peaking 3.2 kHz, gain = -nr*3 dB)                [index.html:266, 318]
 → lim (Brickwall: thr -3, ratio 20, atk 3 ms, rel 0.12)   [index.html:266]
```

### 1.3 Raum-Sends (parallel zu `mix`)

```
lim → mix                                                 [index.html:267]
lim → conv (Convolver, makeIR(2.6s, decay 2.8)) → revWet → mix   [index.html:268]
lim → del (Delay 0.28 s, feedback 0.35 via delFb) → delWet → mix [index.html:269]
```

### 1.4 Tail: Master + Safety

```
mix → safeBell (Peaking 3500 Hz, gain -2 dB)              [index.html:271]
safeBell → mHP (Highpass 90 Hz)
        → mLow (Lowshelf 180 Hz +3.5 dB)
        → mMid (Peaking 1500 Hz -3.5 dB)
        → mAir (Highshelf 4200 Hz +2 dB)
        → mComp (Bus-Glue: thr -24, ratio 2, knee 6)
        → mMakeup (gain 1.7)
        → mOn (gain = masterOn ? 1 : 0)               [index.html:272]
        → softSafe (WaveShaper tanh(1.5), 4× oversample)
safeBell → mOff (gain = masterOn ? 0 : 1) → softSafe    [index.html:273]
softSafe → safetyLim (Brickwall: thr -1.5, ratio 20, atk 2 ms, rel 0.05) → chainGate → out  [index.html:274]
```

### 1.5 Original-Pfad (parallel zur ganzen Kette)

```
songSrc/streamSrc → origRaw → dryDelay (12 ms)            [index.html:520, 525, 256]
                          → origLP (lowpass, mono erzwungen, dynamisch in audioTick)  [index.html:258-260, 433]
                          → origGate (dynamisch in audioTick) → out                   [index.html:275, 436]
```

> **Beobachtung:** `origGate → out` umgeht safeBell/mHP/mLow/mMid/mAir/mComp/mMakeup/mOn/mOff/softSafe/safetyLim. Das Original ist **am Limiter und an der Safety vorbei** geroutet. Siehe §4 Physik-Check.

### 1.6 Ausgänge

```
out → ctx.destination                                     [index.html:276]
out → recDest (MediaStreamDestination für MITSCHNITT)     [index.html:277]
```

### 1.7 Grain-Abgriffe (Quellen-Tap, v95)

```
songGain → musicTap (ScriptProcessor, schreibt in A.musicRing) [index.html:300]
musicTap → grainSink (gain=0) → ctx.destination               [index.html:281]

noiseGain → stoerSum (gain=1)                                  [index.html:296]
hetGain   → stoerSum                                           [index.html:302]
sweepGain → stoerSum                                           [index.html:304]
fg        → stoerSum (in startForeign, falls A.stoerSum exists) [index.html:333]
fg2       → stoerSum                                           [index.html:342]
stoerSum → stoerTap (ScriptProcessor, schreibt in A.stoerRing) [index.html:288]
stoerTap → grainSink (gain=0) → ctx.destination                [index.html:281]

grainOut (gain 0.9) → bus                                      [index.html:289]
Grain-Korn-Source → g → grainOut → bus                         [index.html:511]
```

> **Beobachtung 1:** `grainSink.gain=0` → die Tap-Audio geht nicht hörbar zur Destination; der Sink dient nur dazu, die ScriptProcessoren am Leben zu halten. ✓ Konzept ok.

> **Beobachtung 2:** Körner gehen via `grainOut → bus` durch die GANZE Empfänger-Kette zurück. Konzeptuell sauber. **Rekursionsfrage:** musicTap und stoerTap kriegen Eingang von songGain bzw. den Stör-Gains. Grain-Ausgang fließt zurück in bus, dann durch wow→…→mix → … → out → ctx.destination. Greift NICHT auf songGain oder noiseGain/hetGain/sweepGain zu. ✓ Keine Rekursion.

### 1.8 Crackle

```
Trigger alle 240-940 ms, Wahrscheinlichkeit 0.05 + cur.noise*0.10:   [index.html:448-461]
  AudioBuffer (Rauschen mit pow(1-i/len, 2)-Hüllkurve)
  → BufferSource (s)
  → bp (Bandpass 900-3100 Hz, Q 0.7)
  → g (gain = (P.noise/100)*cur.noise*(0.5+rand)*0.5*curDuck)
  → bus
```

Crackle landet im **bus** → durch die Kette. ✓

### 1.9 Komplett-Diagramm (verbal)

Drei parallele Pfade enden in **`out`**:
- Empfänger-Kette: `bus → wow → ... → lim → mix → safeBell → (master|direct) → softSafe → safetyLim → chainGate → out`
- Original-Pfad: `(songSrc|streamSrc) → origRaw → dryDelay → origLP → origGate → out`
- Tap-Sinks: `(musicTap|stoerTap) → grainSink → ctx.destination` (nicht in `out`, dient nur dem ScriptProcessor)

`out` ist verzweigt zu `ctx.destination` und `recDest` (MITSCHNITT).

---

## 2. Regler-Inventar

Notation: **Tooltip-Versprechen** vs. **Code-Wirkung** vs. **Wechselwirkungen** vs. **Urteil**. Default-Wert in `P`-Objekt auf Zeile 200, Per-Preset-Override via `cur.ctl` und Clamping via `ctlRange(cur)`.

### 2.1 `noise` — "Rauschen" (Slider, 0-100, default 30, per-Preset gesetzt)

- **Versprechen** ([index.html:562](index.html:562)): "Stärke von Grundrauschen und Knistern."
- **Code-Wirkung**:
  - `noiseBase = (P.noise/100)*cur.noise*(0.12+(1-q)*0.85)*(1+(1-fade)*1.2)*(1.5-0.5*min(1,prop))*duckF*0.5` ([index.html:395](index.html:395))
  - Phase A der Dry/Wet-Öffnung: `noiseBase *= (1 - o*0.8)` ([index.html:402](index.html:402))
  - Schließlich `A.noiseGain.gain = noiseBase*xm` ([index.html:419](index.html:419))
  - Auch Crackle-Pegel: `(P.noise/100)*cur.noise*…*curDuck` ([index.html:456](index.html:456))
  - Auch Crackle-Häufigkeit: `Math.random() < 0.05+cur.noise*0.10` ([index.html:450](index.html:450))
  - Auch in `drive`-Mischung: `drive = min(1, cr*0.8 + (P.noise/100)*0.3 + (P.fgLevel/100)*0.15)` → Lied-Flatter ([index.html:390](index.html:390))
- **Wechselwirkungen**:
  - **Rettung** `nr`: `clean = 1 - nr*0.6` → wirkt indirekt auf `duckF`. Bei Voll-Rettung sinkt `noiseBase` auf ~40 %.
  - **Crush** `crush`: `crushDuck = pow(1-cr*0.7, 1.2)` → bei Voll-Crush sinkt `noiseBase` ebenfalls (reziproke Schiebung).
  - **Extrakt** `xm=0`: setzt `A.noiseGain.gain` direkt auf 0 → unhörbar. Aber `noiseBase` (vor `*xm`) bleibt für die Masker-Berechnung.
  - **Crackle** zusätzlich ausgeblendet im Extrakt-Modus durch eigenen Guard `!extractMode` ([index.html:449](index.html:449)).
  - **Dry/Wet** Phase A: bei `w < 1.0` sinkt noiseBase (bis −80 % bei `w ≤ 0.66`).
  - **ctlRange-Untergrenze**: `round((1-p.q0)*40)` → bei sehr unsicheren Empfängen (kleines q0) ist 0 nicht erreichbar.
- **Urteil**: Deckungsgleich mit Versprechen. Erweitert um Kopplungen, die Spec-konform sind (Rettung, Crush-Reziprok, Extrakt-Stumm, Dry-Öffnung). ✓

### 2.2 `fgLevel` — "Fremdsignal" (Slider, 0-100, default 55, per-Preset)

- **Versprechen** ([index.html:563](index.html:563)): "Lautstärke fremder Sender/Störstationen, die sich übers Lied legen."
- **Code-Wirkung**:
  - `fgBase = cur.foreign ? gate*(P.fgLevel/100)*0.9*duckF : 0` ([index.html:386](index.html:386))
  - `fg2Base = cur.foreign2 ? gate2*(P.fgLevel/100)*0.6*duckF : 0` ([index.html:387](index.html:387))
  - Phase A: `fgBase *= (1 - o*0.7)`, `fg2Base *= (1 - o*0.7)` ([index.html:404-405](index.html:404))
  - Sweep-Störer skaliert auch mit `P.fgLevel`: `0.014*(0.4+P.fgLevel/100)*duckF*xm` ([index.html:430](index.html:430))
  - In `drive`-Mischung mit Anteil 0.15 ([index.html:390](index.html:390))
- **Wechselwirkungen**: wie `noise` (Rettung, Crush, Extrakt, Dry-Öffnung). Außerdem Voraussetzung `cur.foreign` (Preset definiert es); ohne `cur.foreign` ist `fgLevel` wirkungslos (Pegel = 0 für fg-Quelle, Sweep aber teilweise unabhängig).
- **ctlRange-Untergrenze**: `p.foreign ? 15 : 0` → bei Stationen mit Fremdsignal ist 15 % Minimum, "ganz aus" nicht möglich. ✓ Sinnvoll, denn ohne fg wäre der "Charakter" abwesend.
- **Urteil**: Deckungsgleich. Eine **Inkonsistenz**: `cur.foreign2` aktiviert `fg2` (zweite Fremdquelle), aber im Tooltip steht "Fremdsender" Plural — der User sieht/hört nicht direkt, dass es 2 Quellen gibt. Kein Bruch, nur Doku-Lücke.

### 2.3 `breathe` — "Schwund" (Slider, 0-100, default 50, per-Preset)

- **Versprechen** ([index.html:564](index.html:564)): "Wie stark das Signal an- und abschwillt (QSB-Fading). 0 = steht still."
- **Code-Wirkung**:
  - `effDepth = clamp(cur.fadeDepth*(P.breathe/50), 0, 0.95)` ([index.html:380](index.html:380))
  - `fade = (1-effDepth) + effDepth*fadeFn(T,cur.seed)` ([index.html:382](index.html:382))
  - Wirkt auf `q` (Signalqualität) und damit alle q-abhängigen Größen.
- **Wechselwirkungen**:
  - **FREEZE**: friert `T` ein → `fade` bleibt stehen. ✓ Versprechen "Atmen einfrieren" eingehalten.
  - **`breathe`=0**: `effDepth=0` → `fade=1`. Aber im Code wird `fade` weiterhin in `noiseBase`-Faktor `(1+(1-fade)*1.2)` und in `shaper k` und `het*(0.5+(1-fade))` verwendet — bei `fade=1` werden diese Terme zu (1+0)*1.2 = 1.2 bzw. neutralisieren sich. ✓ "steht still" stimmt.
  - **ctlRange-Untergrenze**: `round(p.fadeDepth*35)` → bei tiefen Atmungs-Presets ist Minimum > 0. Damit ist "0 = steht still" am Slider nicht ganz erreichbar, wenn der Preset starkes Atmen vorgibt. Versprechen + ctlRange leicht widersprüchlich.
- **Urteil**: Inhaltlich deckungsgleich. **Kleine Inkonsistenz**: Tooltip "0 = steht still" gilt nur, wenn `ctlRange.breathe[0]` zufällig 0 ergibt (bei Presets mit `fadeDepth < 0.015`). Für die meisten Presets ist Slider 0 = Untergrenze > 0 → restliches Atmen bleibt. Nicht beschönigen: Spec-Detail "0 = steht still" ist für die meisten Stationen NICHT exakt erreichbar; das stützt aber den Spirit "physikalische Untergrenze".

### 2.4 `lp` — "Bandbreite" (Slider, 1200-6000 Hz Slider-Range, default 3785, per-Preset)

- **Versprechen** ([index.html:566](index.html:566)): "Empfangsbreite: schmal = dumpf/fern, breit = heller/klarer. Obergrenze = was der Sender physikalisch liefert."
- **Code-Wirkung**:
  - `lpcBase = max(700, min(6000, cur.lp*(P.lp/3785)))` ([index.html:420](index.html:420))
  - Phase A: `lpc = lpcBase*(1-o) + 6000*o` ([index.html:421](index.html:421))
  - `A.lp.frequency = 900 + q*(lpc-900)` ([index.html:422](index.html:422))
  - Tieferer `q` (schwächeres Signal) drückt die effektive Cutoff Richtung 900 Hz.
- **Wechselwirkungen**:
  - **ctlRange-Obergrenze**: `[1200, p.lp]` → der User kann nie über die preset-spezifische Obergrenze. ✓ Versprechen erfüllt.
  - **Dry/Wet Phase A**: `lpc` öffnet sich Richtung 6000 Hz bei sinkendem `w`. Das ist Spec-konform "die Kette öffnet sich".
- **Urteil**: Deckungsgleich mit Versprechen und Zielsetzung. ✓

### 2.5 `crush` — "Verzerrung" (Slider, 0-100, default 0, per-Preset)

- **Versprechen** ([index.html:567](index.html:567)): "Übersteuerung & Demodulator-Verzerrung des Empfangs — wie ein überfahrener AM-Detektor bei schwachem Träger."
- **Code-Wirkung**:
  - `A.crushShaper.curve = distCurve(P.crush/100)` ([index.html:322](index.html:322), in `applyControls`, NICHT in audioTick)
  - `cr = P.crush/100` wird in `drive`-Mischung (Lied-Flattern) verwendet ([index.html:390](index.html:390))
  - `crushDuck = pow(1-cr*0.7, 1.2)` → reziproker Absenk-Faktor auf `duckF` für Stör-Quellen ([index.html:374-375](index.html:374))
  - Im Limiter-Vor-Verstärker: indirekt über `shaper.curve` mit `k` (das ist der TANH-Shaper, NICHT der distCurve) ([index.html:423-425](index.html:423))
- **Wechselwirkungen**:
  - **Rettung**: `clean = 1 - nr*0.6` → wirkt auf `k` des tanh-Shapers (`k *= clean` über den Multiplikator). **Aber NICHT auf distCurve(crush)** — die Crush-Verzerrung wird von Rettung **nicht weggemildert**. ✓ Spec-konform ("Deformation bleibt").
  - **Dry/Wet Phase A**: `k = k*(1-o) + 1.1*o` → tanh-Shaper wird in der Dry-Öffnung milder. Aber `distCurve(crush)` bleibt aktiv. **Inkonsistenz**: Phase A reduziert nur EINEN der beiden Verzerrer (tanh). Die distCurve-Verzerrung bleibt voll auch im weichesten Dry-Modus, solange `crush > 0`. Spec sagt "Phase A: weniger Rauschen/Verzerrung" — das ist nur teilweise erfüllt.
  - `crushWet.gain=1`, `crushDry.gain=0` sind in `ensure()` gesetzt und **werden nie wieder moduliert**. Der Dry/Wet-Mix der Crush-Stufe ist toter Code. Bei `distCurve(0)` (linear identity) ist es egal. **Toter Code, harmlos.**
- **Urteil**: Versprechen "AM-Detektor-Übersteuerung" ist durch die asymmetrische distCurve gut umgesetzt. **Abweichung**: in Dry-Phase A bleibt die distCurve-Verzerrung voll wirksam, obwohl die Spec eine generelle Reduktion der Verzerrung verspricht.

### 2.6 `het` — "Pfeifen" (Slider, 0-100, default 60, per-Preset)

- **Versprechen** ([index.html:568](index.html:568)): "Heterodyn-Pfeifton benachbarter Träger (Zero-Beat)."
- **Code-Wirkung**:
  - `hetBase = (P.het/1000)*cur.het*(0.5+(1-fade))*0.5*clean` ([index.html:396](index.html:396))
  - Phase A: `hetBase *= (1 - o*0.8)` ([index.html:403](index.html:403))
  - `A.hetGain.gain = hetBase*xm` ([index.html:426](index.html:426))
  - `hetF` (Frequenz): driftend bei `cur.drift` (Sinus-Drift auf T), sonst fest 500+q0*1400 Hz ([index.html:427](index.html:427))
- **Wechselwirkungen**:
  - **Rettung**: Multipliziert mit `clean` → bei Voll-Rettung sinkt `hetBase` auf 40 %.
  - **Extrakt**: `xm=0` → stumm.
  - **Dry/Wet Phase A**: −80 % bei `w ≤ 0.66`.
  - **FREEZE**: `hetF`-Sinus speist sich aus `T` → driftendes Pfeifen friert ein. Statisches Pfeifen ist hetF-unabhängig.
- **ctlRange-Untergrenze**: `round(p.het*12)` → bei `cur.het=0` ist Untergrenze 0 (Slider kann auf null). Bei `cur.het > 0` (stationen-spezifisch) ist Untergrenze > 0.
- **Urteil**: Deckungsgleich. ✓

### 2.7 `nr` — "Rettung" (Slider, 0-100, default 0, NICHT per-Preset überschrieben)

- **Versprechen** ([index.html:570](index.html:570)): "Grundreinigung: hebt das fragile Signal nach vorn — nur lauter, nichts Verlorenes zurückgeholt. Störendes weicht bis auf einen Rest, die Deformation des Signals bleibt. Nur bei fernen Stationen."
- **Code-Wirkung**:
  - `nr = isRescue(cur) ? P.nr/100 : 0` ([index.html:370](index.html:370)). `isRescue` = `dist ∈ {fern, sehr fern}`.
  - Rettungs-Stage (`rHP`, `rComp`, `rMakeup`, `rTame`):
    - `rHP.frequency = 120 + nr*300` → räumt Bässe weg ([index.html:313](index.html:313))
    - `rComp.threshold = -10 - nr*44` ([index.html:314](index.html:314))
    - `rComp.ratio = 1 + nr*17` (1 bei nr=0 → transparent) ([index.html:315](index.html:315))
    - `rComp.knee = 30*(1-nr)` ([index.html:316](index.html:316))
    - `rMakeup.gain = 1 + nr*3.3` ([index.html:317](index.html:317))
    - `rTame.gain = -nr*3` (Schmerz-Cut bei 3.2 kHz) ([index.html:318](index.html:318))
  - `clean = 1 - nr*0.6` → Stör-Quellen werden auf ≥40 % reduziert ([index.html:373](index.html:373))
    - Wirkt auf `duckF` (mult. mit Rauschen/Sweep), auf `hetBase` direkt, auf `shaper-k` (tanh-Shaper milder).
- **Wechselwirkungen**:
  - **Nur ferne Stationen** ([index.html:552-558](index.html:552)): Slider wird disabled gesetzt für nahe/mittlere Stationen, Label wechselt auf "Rettung — nur fern".
  - **Crush**: NICHT abgeschwächt durch `clean` → "Deformation bleibt". ✓ Spec-konform.
  - **Reset bei Preset-Wechsel**: `applyPreset` setzt `P.nr = 0` ([index.html:547](index.html:547)). ✓
- **Urteil**: Deckungsgleich. ✓ Sehr sauber umgesetzt, inkl. expliziter Disable-Logik für nicht-rettungsfähige Stationen.

### 2.8 `reverb` — "Hall" (Slider, 0-100, default 0, reset auf 0 per Preset)

- **Versprechen** ([index.html:572](index.html:572)): "Räumlicher Nachhall — verklebt das verletzte Signal, erzeugt Distanz."
- **Code-Wirkung**: `A.revWet.gain = (P.reverb/100)*0.9` ([index.html:319](index.html:319), in applyControls)
- **Wechselwirkungen**: Send-Pfad lim → conv → revWet → mix. Wirkt nur auf den Empfänger-Pfad, NICHT auf Original.
- **Urteil**: Deckungsgleich. ✓

### 2.9 `delay` — "Echo" (Slider, 0-100, default 0, reset auf 0 per Preset)

- **Versprechen** ([index.html:573](index.html:573)): "Verzögerte Wiederholungen mit Rückkopplung."
- **Code-Wirkung**: `A.delWet.gain = (P.delay/100)*0.6` ([index.html:320](index.html:320))
- **Wechselwirkungen**: Send mit eigenem Feedback `delFb.gain=0.35` → klassisches Tape-Echo.
- **Urteil**: Deckungsgleich. ✓

### 2.10 `wow` — "Wow" (Slider, 0-100, default 0, reset auf 0 per Preset)

- **Versprechen** ([index.html:574](index.html:574)): "Tonhöhen-Schwanken wie bei Tonband/altem Empfänger."
- **Code-Wirkung**: `A.wowDepth.gain = (P.wow/100)*0.0035` ([index.html:321](index.html:321)). LFO bei 5.2 Hz, Tiefe maximal 3.5 ms (Modulation von wow.delayTime).
- **Wechselwirkungen**:
  - Wow ist FIX VOR HP in der Kette → wirkt auf ALLES im Bus, also auch auf Rauschen, fg, het, Sweep, Crackle, Grain.
  - **Spec-Frage:** Soll Wow wirklich auf alle Stör-Quellen wirken oder nur auf die Musik? Aktuell: ja, alles wird gewowt. Realitätsnah (alle empfangenen Signale durch dieselbe Empfänger-Mechanik), aber u. U. nicht intuitiv.
  - Wow läuft auch in Dry-Phase-A weiter → Spec sagt zwar "die Kette öffnet sich", aber wow ist kein "Schaden" wie Rauschen. Bleibt aktiv. ✓
- **Urteil**: Deckungsgleich. ✓

### 2.11 `grain` — "Grain" (Slider, 0-100, default 0, reset auf 0 per Preset)

- **Versprechen** ([index.html:575](index.html:575)): "Macro-Granulator: unten subtiles Schimmern, Mitte ambiente Wolke, oben fast unkenntlich. Quelle wählbar: MUSIK / STÖR / ALLES. Körner laufen durch den Empfänger."
- **Code-Wirkung**: Spawner-IIFE ([index.html:465-518](index.html:465)). Skaliert mit `amt = P.grain/100`:
  - Korngröße (piecewise lerp 30-80 / 80-250 / 250-800 ms)
  - Spawn-Intervall (180 ms → 35 ms)
  - Pitch-Spread (±2 → ±12 HT)
  - Oktavsprung 15 % ab amt>0.7
  - Stretch 30 % ab amt>0.6
  - Reverse 25-40 % ab amt>0.3
  - Freeze (Wiederverwendung der Lesepos.) — **siehe §5 Code-Gesundheit, Bug!**
  - Ausgang: `g → A.grainOut → bus` → durch die ganze Empfänger-Kette.
- **Wechselwirkungen**:
  - **Quellenwahl-Buttons** MUSIK/STÖR/ALLES ([index.html:589-596](index.html:589)): Default "ALLES", bleibt über Preset-Wechsel stehen.
  - **EXTRAKT**: Spawner hat KEINEN `!extractMode`-Guard. Im Extrakt-Modus läuft Grain weiter. MUSIK-Körner enthalten die durchs Maskierungs-Gate gedimmte Musik (aud-modified). STOER-Körner enthalten Stille (alle stör-Quellen sind `xm=0`-stumm geschaltet). ALLES → 50 % stille STOER + 50 % aud-Musik.
  - **FREEZE**: Spawner verwendet `ctx.currentTime` und `Math.random()` — unabhängig von `T`. Körner werden weiter gespawnt im FREEZE. Konzeptuell ok ("auf das Skelett gelegt"), aber FREEZE-Versprechen "Atmen bleibt stehen" gilt nicht für Grain.
  - **Crackle**: läuft ebenfalls weiter im FREEZE, gleicher Mechanismus.
- **Urteil**: **Mostly deckungsgleich**, mit zwei Abweichungen vom Versprechen:
  1. Im Extrakt-Modus produziert STOER stille Körner (technisch korrekt, aber unbedacht: User mit `STÖR` im Extrakt-Modus hört nichts vom Grain).
  2. Macro-Granulator-Freeze (interner Mechanismus) zündet praktisch nie wegen Bug — siehe §5.

### 2.12 `vol` — "Lautstärke" (Slider, 0-100, default 85, reset auf 50 per Preset)

- **Versprechen** ([index.html:577](index.html:577)): "Ausgangspegel."
- **Code-Wirkung**: `A.out.gain = P.vol/100` ([index.html:324](index.html:324))
- **Wechselwirkungen**: wirkt auf `out`, also auf BEIDE Pfade (Empfänger + Original) gleichzeitig. ✓
- **Urteil**: Deckungsgleich. ✓

### 2.13 `drywet` — "Dry / Wet" (Slider, 0-100, default 100, NICHT per-Preset)

- **Versprechen** ([index.html:578](index.html:578)): "Öffnet den Empfang: erst weicht die Kette (weniger Rauschen/Verzerrung), dann kommt das Original — gefärbt und mit dem Fading atmend, bis es ganz am Anschlag frei ist. Nie zwei Schichten."
- **Code-Wirkung** (in audioTick, NICHT in applyControls):
  - `w = P.drywet/100`
  - `o = clamp((1-w)/0.34, 0, 1)` → Phase A ab w<1, voll bei w≤0.66
  - `b = clamp((0.66-w)/0.66, 0, 1)` → Phase B ab w<0.66, voll bei w=0
  - Phase A: noiseBase, hetBase × (1-o*0.8); fgBase, fg2Base × (1-o*0.7); `lpc → lpcBase*(1-o)+6000*o`; `k → k*(1-o)+1.1*o`
  - Phase B: `origLP.frequency = 3000 + b²*15000`; `coupleDepth = (1-b)*0.9`; `origLvl = sin(b*π/2) * ((1-cd)+cd*fade)`; `chainGate = cos(b*π/2)`
- **Wechselwirkungen**:
  - **EXTRAKT**: masker-Berechnung verwendet die Phase-A-modifizierten noiseBase/fgBase/fg2Base/hetBase. Bei Dry-Öffnung sinkt der masker → mehr Lied wird audible. **Sinnvoll, aber komplex.**
  - **FREEZE**: `fade` ist eingefroren, damit ist auch der Atmungs-Kopplungs-Anteil von origLvl eingefroren. ✓
  - **Crush**: distCurve-Verzerrung bleibt voll, auch wenn der tanh-Shaper milder wird. Siehe §2.5.
- **Versprechen "Nie zwei Schichten"**: Strikt gelesen, gibt es bei Phase B (b>0) sowohl `chainGate*cos(b*π/2)` als auch `origGate*sin(b*π/2)*…` → beides klingt parallel, ist also wohl-temperierter Equal-Power-Crossfade. Die Spec-Aussage "Nie zwei Schichten" zielt eher darauf, dass das Original NICHT ÜBER dem gefilterten Empfang liegt — und das stimmt: in Phase B nimmt chainGate ab, während origGate aufmacht. Mathematisch sind beide aktiv, aber nicht als unabhängige Schichten gleicher Lautstärke.
- **Urteil**: Deckungsgleich mit Versprechen. **Aber:** "weniger Verzerrung" ist nur teilweise erfüllt (siehe §2.5). Außerdem siehe §4 Physik-Check: Dry-Anschlag = Original ungefiltert/ungesichert.

### 2.14 `btnMaster` — "MASTER AN/AUS" (Toggle, default an)

- **Versprechen** ([index.html:32](index.html:32)): "Einheitliches Mastering: Aufräumen unten, milde Bus-Glue, gleichlaut, Schutz."
- **Code-Wirkung** ([index.html:645](index.html:645)): `mOn.gain` und `mOff.gain` werden cross-fadet. Bei AUS umgeht das Signal die Master-EQ + Bus-Glue + Loudness-Makeup.
- **Wechselwirkungen**: `softSafe` und `safetyLim` (Schutz!) sind in BEIDEN Wegen aktiv. ✓ "Schutz" bleibt auch bei Master AUS.
- **Urteil**: Deckungsgleich. ✓

### 2.15 `btnRecMix` — "● MITSCHNITT" (Push, Toggle Aufnahme)

- **Versprechen** ([index.html:41](index.html:41)): "Nimmt auf, was du gerade hörst — im Extrakt-Modus das Skelett, sonst den vollen Empfang."
- **Code-Wirkung** ([index.html:605-617, 641](index.html:605)): MediaRecorder an `A.recDest.stream`, Audio/WebM bevorzugt, Fallback m4a; Auto-Download nach Stop.
- **Wechselwirkungen**:
  - Dateiname: `cur.id + "_" + kind + "_" + Date + ".webm|m4a"`. `kind = extractMode ? "extrakt" : "mitschnitt"`.
  - **Recording wird nicht abgebrochen, wenn User EXTRAKT mitten in der Aufnahme toggelt.** Recording-Suffix bleibt auf dem Wert zum Start.
  - **Recording wird auch nicht abgebrochen bei Preset-Wechsel.** Dateiname bleibt auf dem Preset zum Start; FREEZE wird im Preset-Wechsel zurückgesetzt → Aufnahme-Sound ändert sich abrupt.
- **Urteil**: Deckungsgleich. Hinweise oben sind keine Brüche, eher Verhaltens-Nuancen.

### 2.16 `btnExt` — "◉ EXTRAKT / ◉ EXTRAKT AN" (Toggle)

- **Versprechen** ([index.html:42](index.html:42)): "Hörmodus Wahrnehmungs-Extrakt: nur das, was durch den Rauschwald wirklich hörbar ist — Maskiertes bleibt Stille. Jederzeit an/aus; speichern über MITSCHNITT."
- **Code-Wirkung**:
  - `extractMode = true` → `xm = 0` ([index.html:384](index.html:384)) macht fg/fg2/noise/het/sweep stumm
  - Crackle eigener Guard ([index.html:449](index.html:449))
  - Maskierungs-Gate auf `songGain.gain` ([index.html:408-414, 418](index.html:408))
- **Wechselwirkungen**:
  - **FREEZE**: das `fade` (eingefroren) speist sich in `q` → in `noiseBase` → in den Masker. Das Versprechen "Wirkt auch auf das Extrakt-Gate" ist erfüllt (Tooltip des FREEZE-Buttons).
  - **Grain**: läuft weiter (siehe §2.11). STOER-Körner sind stille.
  - **Dry/Wet Phase A**: reduziert die Masker-Quellen → mehr aud → mehr Musik hörbar.
- **Urteil**: Deckungsgleich mit Versprechen. ✓

### 2.17 `btnFreeze` — "❄ FREEZE / ❄ FREEZE AN" (Toggle, v96 NEU)

- **Versprechen** ([index.html:43](index.html:43)): "Friert den Empfangs-Zustand ein — Fading, Flattern, Störungs-Atmung bleiben auf diesem Moment stehen. Das Lied läuft weiter. Wirkt auch auf das Extrakt-Gate."
- **Code-Wirkung** ([index.html:620-626](index.html:620)):
  - Einfrieren: `freezeT = now - physOff`, `frozen = true`
  - Auftauen: `physOff = now - freezeT`, `frozen = false`
  - In audioTick: `T = frozen ? freezeT : now - physOff` ([index.html:369](index.html:369))
  - `breath.v`-Würfeln geguardet mit `!frozen` ([index.html:377](index.html:377))
- **Wechselwirkungen**:
  - Friert ein: `fade`, `f1`/`f2`-Flatter, `hetF`-Drift, `sweep ph`, fg/fg2-Gates, breath. ✓
  - Friert NICHT ein:
    - **Crackle** (`Math.random()`-Trigger, unabhängig) — Spec sagt "Störungs-Atmung" — Crackle ist Mikro-Störung, die strenger gelesen mit eingefroren werden sollte. **Inkonsistenz/Versprechen leicht gebrochen.**
    - **Grain-Spawner** (`Math.random()`, `ctx.currentTime`) — konzeptuell ok ("läuft darüber").
    - **Audio-Modulation per Slider**: `noise`/`fgLevel`/`het`/`crush`/`wow`/`reverb`/`delay`/`grain`/`vol`/`drywet` reagieren weiter live auf User-Eingaben. ✓ (Konzeptuell richtig: FREEZE friert die *Physik*, nicht die Regler.)
  - **Lied läuft weiter**: `A.songSrc` oder `A.streamEl` werden nicht angehalten. ✓
  - **Reset bei Preset-Wechsel**: `loadPreset` setzt `frozen=false; physOff=0; freezeT=0` ([index.html:351](index.html:351)). ✓ Versprechen erfüllt.
  - **Kein SCAN-Button im UI vorhanden** — Spec im FREEZE-Auftrag erwähnte "Preset-Wechsel und SCAN heben FREEZE auf". SCAN existiert nicht. Nur Preset-Wechsel hebt auf. Nicht falsch, nur Spec-Detail unerfüllbar.
- **Urteil**: Mostly deckungsgleich. **Inkonsistenz**: Crackle läuft weiter im FREEZE, obwohl Tooltip "Störungs-Atmung bleibt auf diesem Moment stehen" sagt.

### 2.18 `prev` / `next` — Preset-Navigation

- **Versprechen** ([index.html:85-86](index.html:85)): "◄ ► wechselt die Station (Lied läuft nahtlos weiter, loopt am Ende)."
- **Code-Wirkung** ([index.html:351](index.html:351), `loadPreset(idx±1)`):
  - Index circulär
  - `stopForeign()` + `startForeign()` (fg/fg2 neu)
  - `applyPreset()` (P-Werte aus cur.ctl, ctlRange neu, Slider min/max anpassen)
  - `updateRescueUI()` (nr-Slider enabled/disabled)
  - `FREEZE → off` (mit UI-Update)
- **Nahtlos**: Das Lied (songSrc/streamSrc) wird NICHT angehalten oder neu gestartet. ✓
- **Urteil**: Deckungsgleich. ✓

### 2.19 `daynight` — "☀ TAG / ☾ NACHT" (Toggle)

- **Versprechen**: Kein Tooltip vorhanden.
- **Code-Wirkung** ([index.html:653](index.html:653)): `night = !night`. Verwendet in `prop = night?(1+nf*0.45):(1-nf*0.45)` ([index.html:379](index.html:379)). `nf` skaliert mit Band-Wellenlänge.
  - Nacht: niedrige Bänder (90 m, 75 m, 60 m, 49 m) bekommen mehr `prop` (besser empfangbar) → höhere `q` → klareres Signal, weniger Rauschen.
  - Nacht: hohe Bänder (16 m, 13 m) bekommen weniger `prop` → weniger Signal, mehr Rauschen.
  - Tag: umgekehrt.
- **Wechselwirkungen**: realistische Ionosphären-Physik. ✓
- **Urteil**: Inhaltlich solide, **Lücke**: kein Tooltip → User weiß nicht, dass das auf die Empfangsqualität pro Band wirkt.

### 2.20 `play` — "▶ PLAY / ⏸ PAUSE"

- **Code-Wirkung** ([index.html:531](index.html:531)): `togglePlay()` → AudioContext resume/suspend, songSrc start/stop oder streamEl play/pause, startForeign.
- **Urteil**: Funktional. Kein Tooltip.

### 2.21 `btnStream` + `streamUrl` — Webradio-Stream

- **Code-Wirkung** ([index.html:522-530](index.html:522)): `streamMode=true`, MediaElementSource auf URL, ICY-Metadata parsen.
- **Wechselwirkungen**: `A.streamSrc.connect(A.songGain)` + `connect(A.origRaw)` → identische Routings wie File-Quelle. Mono über songGain, Original-Pfad parallel.
- **Urteil**: Funktional.

### 2.22 `file` / `drop` — Lied laden

- **Code-Wirkung** ([index.html:520-521, 654-658](index.html:520)): file → arrayBuffer → ID3 → decodeAudioData → A.songBuf. Bei Drag-Drop dieselbe Route.
- **Wechselwirkungen**: setzt `streamMode=false`, stoppt ggf. Stream.
- **Urteil**: Funktional.

---

## 3. Modi-Matrix

| Modus A × Modus B | Beobachtung |
|---|---|
| **EXTRAKT × Grain (MUSIK)** | Körner aus aud-modulierter Musik → enthalten nur, was am Masker vorbeikommt. Konsistent. |
| **EXTRAKT × Grain (STÖR)** | Körner sind **stille** (xm=0 mutet alle Stör-Quellen). Bei `STÖR`-Wahl ist Grain im Extrakt-Modus de-facto aus. Versprechen "Körner laufen durch" gilt formal noch — aber unhörbar. |
| **EXTRAKT × Grain (ALLES)** | 50 % Musik, 50 % Stille. Halbiert wahrgenommene Korn-Dichte. |
| **EXTRAKT × Rettung** | nr wirkt auf `clean` → wirkt auf `hetBase`, `shaper-k`. Masker enthält noiseBase und fgBase (nicht clean-skaliert?) — doch, `duckF = clean*crushDuck`, und `noiseBase` enthält `duckF`. Also: Rettung senkt Masker UND senkt songLvl-Modulation? Nein, songLvl ist nicht von clean abhängig. Konsequenz: bei Voll-Rettung + Extrakt sinkt der Masker auf ~40 % → mehr aud → mehr Musik hörbar. ✓ Klingt richtig: Rettung räumt auf, im Extrakt wird das hörbarer. |
| **EXTRAKT × FREEZE** | FREEZE friert `fade` → Masker bleibt stabil → aud bleibt stabil. ✓ Gut für reproduzierbares Hören-Speichern. |
| **EXTRAKT × Dry/Wet Phase A** | o>0 reduziert Masker-Quellen → mehr Lied im Extrakt. Logisch konsistent. |
| **EXTRAKT × Dry/Wet Phase B** | songGain wird trotzdem aud-moduliert. Phase B routet zusätzlich Original via origGate. Original ist NICHT aud-moduliert. Bei b>0 kommt also auch im EXTRAKT-Modus das (ungefilterte!) Original durch. **Sprachliche Unschärfe**: "Extrakt = nur was hörbar" — bei Phase B ist auch Original hörbar. Konsistenz nur, wenn man EXTRAKT als Modus für die *Empfänger-Kette* liest, nicht für *Original*. |
| **FREEZE × Grain** | Grain läuft weiter. Spec-Lesart: konzeptuell ok ("läuft darüber"). |
| **FREEZE × Crackle** | Crackle läuft weiter — Tooltip-Konflikt (siehe §2.17). |
| **Master AUS × Safety** | Safety bleibt aktiv. ✓ |
| **Master AUS × Aufnahme** | MITSCHNITT erfasst out, das durch chainGate/origGate kommt. Master-AUS-Zustand wird mit aufgenommen. ✓ |
| **MITSCHNITT × FREEZE** | Aufnahme erfasst den eingefrorenen Sound. Wenn der User mitten in der Aufnahme FREEZE drückt, wird der Übergang aufgenommen. ✓ |
| **MITSCHNITT × Preset-Wechsel** | Aufnahme läuft weiter, Dateiname behält initiales `cur.id`. Sound ändert sich abrupt. **UX-Frage**, kein Bruch. |
| **MITSCHNITT × Extrakt-Toggle mitten drin** | Dateiname behält initiales `kind`. Inhaltlich wechselt der Sound. ✓ Konsistent mit "nimmt auf, was du hörst". |
| **Stream × FREEZE** | Stream läuft live weiter (kann nicht eingefroren werden — extern). Empfänger-Physik friert ein. Versprechen erfüllt. |
| **Stream × Aufnahme** | Erfasst die aufbereitete Stream-Ausgabe inkl. Stör-Schicht. ✓ |
| **EXTRAKT × Stream** | Maskierungs-Gate auf songGain (= Stream-Quelle). Konsistent. |
| **Grain × Dry/Wet Phase B (Original kommt)** | Grain-Eingang (musicTap) sitzt auf songGain, nicht auf origGate. Original-Audio gelangt NICHT in den musicRing. Bei Dry-Anschlag spielen also Körner aus *gespeicherter alter Musik* (Ring-Puffer 2 s), aber die *aktuell hörbare* Quelle ist das ungefilterte Original. **Inkonsistenz** — Grain entkoppelt sich vom Hörstrom. |

---

## 4. Physik-Check

### 4.1 Wege, auf denen Signal an der Empfänger-Kette vorbeikommt

1. **Original-Pfad bei Dry-Anschlag** (drywet=0):
   - Path: `songSrc → origRaw → dryDelay → origLP(18000 Hz) → origGate(1.0) → out → destination`
   - **Bypassed**: wow, hp, lp, pkf, shaper, crushShaper, crushWet, agc, rHP, rComp, rMakeup, rTame, lim, mix, conv, revWet, del, delWet, safeBell, mHP/mLow/mMid/mAir/mComp/mMakeup (oder mOff), softSafe, **safetyLim**
   - **origLP bei b=1**: `frequency = 3000 + 1*15000 = 18000 Hz` → nahezu identisch mit unbearbeitet.
   - **Resultat**: Das Lied klingt bei Dry-Anschlag ZIEMLICH GENAU wie das Original. Mono ist die einzige sicher erzwungene Färbung (über origLP.channelCount=1).
   - **Bricht**: Zielsetzung "Lied klingt NIE wie das Original".
2. **Tap-Sinks** sind gemutet (`grainSink.gain=0`) → keine Leak.
3. **`stoerSum.gain=1`** ist nur Tap-Sammel-Knoten, ohne Output zum Hörweg. ✓ Keine Leak.

### 4.2 Original zu sauber — bestätigt

Selbst in Phase B (0 < b < 1) ist die einzige Färbung durch origLP (3000-18000 Hz Lowpass) und die fade-gekoppelte Amplitudenmodulation. **Keine Empfangs-Färbung** (HP 320 Hz, peakings, distortion, AGC, limiter, master EQ, safetyLim) wirkt auf das Original.

### 4.3 Rauschen ganz auf 0 möglich?

- **Slider `noise` mit ctlRange-Untergrenze** `round((1-p.q0)*40)` → bei `q0=1.0` ist Untergrenze 0. Bei `q0<1.0` ist sie >0. STN-01 hat q0=0.95 → Untergrenze 2. STN-08 (q0=0.34) → Untergrenze 26. ✓ Physikalische Untergrenze gewahrt.
- **`noiseBase`** hat den Faktor `(1-o*0.8)` → reduziert sich bei Dry-Öffnung um bis zu 80 %. Bei Voll-Dry (o=1) ist noiseBase = noiseBase_original × 0.2.
- **Crackle** ist ebenfalls von `P.noise/100 * cur.noise` skaliert und hat `curDuck` als Faktor. Bei P.noise=Untergrenze + low cur.noise + Voll-Crush oder Voll-Rettung wird Crackle sehr leise, aber nicht ganz aus (außer in Extrakt, wo `!extractMode`-Guard greift).
- **EXTRAKT**: xm=0 → noiseGain.gain = 0 → Rauschen *völlig* aus. ABER: gewollt — Extrakt-Versprechen.
- **Urteil**: Außerhalb von Extrakt bleibt immer ein hörbarer Rauschanteil; Stille nur durch Extrakt. ✓ Spec-konform.

### 4.4 Sicherheits-Limiter umgangen

`safetyLim` ist Teil des chainGate-Pfads ([index.html:274](index.html:274)). Original geht direkt zu `out` ohne Safety. Bei lautem Lied + voll Dry kann Output clippen oder lauter als gewollt sein.

### 4.5 Mono-Erzwingung lückenhaft

`songGain` ist mono. `origLP` ist mono. Aber **fg/fg2 sind Gain-Nodes mit default channelCountMode="max"** → wenn das geladene Foreign-Sample stereo ist, propagiert das Stereo durch `bus → wow → hp → ... → mix → ... → chainGate → out`. Damit kann der Empfänger-Pfad Stereo werden, sobald ein Foreign-Sample stereo ist.

Status der Stör-Quellen:
- noiseSrc (BufferSource): mono Buffer (`createBuffer(1, len, sampleRate)`) → mono
- hetOsc/sweepOsc: Oscillator → mono
- crackle: mono Buffer
- grain: mono Buffer
- fg/fg2: könnten stereo sein (abhängig vom MP3)

**Risiko**: Bei Stereo-Foreign-Sample wird die Empfänger-Kette stereo, wodurch das Bus → Limiter-Tail stereo läuft. Subtile Spec-Verletzung "Mono".

### 4.6 Grain-Output rekursiver Feedback?

grainOut → bus → durch die Empfänger-Kette → mix → ... → out → destination. Aber `bus` ist *nicht* mit musicTap oder stoerSum verbunden. Diese Taps hängen direkt auf songGain bzw. den Stör-Gains. ✓ Keine Rekursion.

---

## 5. Code-Gesundheit

### 5.1 Toter Code / verwaiste Variablen

- **`crushWet.gain=1`** und **`crushDry.gain=0`** sind statisch ([index.html:226](index.html:226)). Wet/Dry-Crossfade der Crush-Stufe wird nicht moduliert. Toter Code seit v93 (Bitcrush wurde durch distCurve ersetzt, alter Wet/Dry-Mix nie aufgeräumt).
- **`mLow`, `mMid`, `mAir`** sind in `ensure()` definiert ([index.html:246-248](index.html:246)), in den Master-Signalweg verkabelt ([index.html:272](index.html:272)), aber **nicht in `Object.assign(A, ...)`** ([index.html:305](index.html:305)). Sie sind harmlos aktive Filter im Audio-Graph, aber nirgends von außen erreichbar (keine `A.mLow.gain.value=...`). Inkonsistenz zur Spec-Idee "über A zugänglich".
- **`crushShaper`** ist in A. `crushDry`/`crushWet` auch. ✓ Aber siehe oben — werden nicht moduliert.
- Im `setExtract(on)` ([index.html:619](index.html:619)) wird `ensure()` aufgerufen + `A.ctx.resume()` (auch beim Abschalten — `if(on) A.ctx.resume()` schützt davor; OK).
- `setFreeze(on)` ruft analog `ensure()` + `A.ctx.resume()` für Einfrieren (nicht für Auftauen) ([index.html:621](index.html:621)). ✓
- `intfActive` wird in audioTick beschrieben ([index.html:417](index.html:417)), aber an **keiner anderen Stelle gelesen**. **Toter Code.**
- `curNr` wird in audioTick gesetzt ([index.html:371](index.html:371)), aber an **keiner anderen Stelle gelesen**. **Toter Code.**
- `recDest` und Recording-Mechanik: ok genutzt.
- `streamSrc` als Property von `A` ([index.html:525](index.html:525)) — wird gesetzt aber nie wieder gelesen. Lebenszyklus ok (Garbage-Collection-Schutz). Borderline-toter Code.

### 5.2 Deprecated APIs

- **`ScriptProcessorNode`** (`musicTap`, `stoerTap`) — W3C-deprecated. Nachfolger ist AudioWorklet (eigene JS-Datei nötig). Funktioniert weiter in allen großen Browsern. Bekannte Schuld.

### 5.3 Performance-Risiken

- **`audioTick` per `setInterval(33ms)`** und **`uiTick` per rAF**: Beide laufen unabhängig. UiTick liest cached Werte, leichte Last. AudioTick ist schwer (~50 setTargetAtTime-Calls pro Tick + Curve-Generation für shaper jeden Tick). 30 Hz mal ~50 Param-Sets = 1500 ops/s. **Akzeptabel**, aber `A.shaper.curve = tanhCurve(k)` baut JEDEN Tick einen neuen Float32Array(1024). ~30/s × 1024 = 30 720 Allocations/s. **GC-Druck.** Sollte mit `setValueAtTime` auf `gain` umstellbar sein oder Curve cachen.
- **`crackle` Endlosschleife**: setTimeout 240-940 ms. ~1-4/s neue BufferSource + Filter + Gain. Stop nach Buffer-Ende. OK.
- **`grains` Endlosschleife**: bei amt=1 alle 35-75 ms ein Korn. ~13-28/s neue BufferSource + Gain. Stop nach Korn-Ende. OK, aber bei voll amt + großen Körnern + ALLES + lebhaftem Preset entstehen viele parallele Korn-Quellen. **GC-Druck**.
- **`grainSink` und `stoerSum`**: stehen permanent als Sinks/Verteiler. OK.

### 5.4 Bugs

- **Grain-Freeze zündet praktisch nie** (vom User explizit zur Verifikation angefragt):
  - Code ([index.html:482](index.html:482)): `if(amt>0.8 && Math.random()<0.25 && A.grainFreezeStart!=null && A.grainFreezeLen===len) useFreeze=true;`
  - `len` ist `Math.max(1, (dur*ctx.sampleRate)|0)` mit `dur = durMin + Math.random()*(durMax-durMin)` ([index.html:477-478](index.html:477))
  - `A.grainFreezeLen` wird im vorigen Korn auf `len` gesetzt ([index.html:486](index.html:486))
  - Damit `grainFreezeLen===len` true wird, muss `len` (= `(dur*sampleRate)|0`) für zwei aufeinanderfolgende Körner gleich sein. `dur` ist Random aus einem ~50-550 ms-Fenster (bei amt=1). Wahrscheinlichkeit, dass zwei Random-Ziehungen auf demselben Sample-Wert landen, ist quasi 0.
  - **Bestätigt: Freeze-Bedingung zündet praktisch nie.** Toter Mechanismus innerhalb des Grain-Spawners.
- **Inkonsistenz "Freeze friert alles": Crackle läuft im FREEZE weiter** (siehe §2.17).
- **Tooltip-Text "0 = steht still"** bei `breathe` ist nur erreichbar, wenn ctlRange-Untergrenze 0 ergibt (siehe §2.3).
- **Drywet-Slider triggered `applyControls()`**, aber applyControls macht für drywet nichts mehr (nur Kommentar Zeile 323). Slider-Bewegung VOR Play: chainGate/origGate bleiben auf Init-Werten. Beim ersten Audiotick nach Play wird's korrigiert. **Subtile Verzögerung** der Hör-Wirkung, vermutlich unhörbar.

### 5.5 Race-Condition / Initialisierungs-Reihenfolge

- `updateMeters` ist auf Zeile 541 definiert, wird in uiTick ([index.html:443](index.html:443)) aufgerufen. uiTick startet auf Zeile 441 synchron beim Skript-Load, aber der rAF-Callback wird erst beim nächsten Frame ausgeführt — bis dahin ist updateMeters längst definiert. ✓ Kein Bug.
- `applyPreset()` wird beim Skript-Load aufgerufen (Zeile 600), aber `ensure()` ist noch nicht gelaufen (`A.ctx` nicht definiert). `applyPreset` → `applyControls` → guard `if(!A.ctx) return;` ([index.html:311](index.html:311)). ✓ Sicher.
- `applyPreset` setzt `SLD[k].inp.min/max/value` — nur, wenn `SLD[k]` existiert. SLD wird vorher in `sl.forEach` befüllt. ✓ OK.

---

## 6. Unklarheiten (offene Fragen)

1. **Wow im Dry-Modus**: Soll Wow auch in Phase A reduziert werden ("die Kette weicht")? Oder ist Wow eine Eigenschaft des Empfänger-Geräts und bleibt aktiv? Aktuell: bleibt aktiv. Spec gibt keine eindeutige Antwort.
2. **Crackle im FREEZE**: Soll Crackle als "Störungs-Atmung" mit eingefroren werden, oder ist Crackle-Stille im FREEZE unerwünscht (zu still)? Aktuell: läuft weiter.
3. **Grain im Extrakt-Modus, Quelle STÖR**: Wenn alle Stör-Quellen stumm sind, gibt STÖR nur Stille. Soll der STÖR-Button in EXTRAKT disabled werden oder gibt's einen Spec-Wunsch dazu?
4. **Original-Pfad-Safety**: Soll origGate ebenfalls durch softSafe / safetyLim laufen? Es würde das Konzept "Lied klingt NIE wie das Original" verletzen, aber den User vor Clipping schützen.
5. **Wow auf Original-Pfad**: Sollte Wow auch auf das Original wirken? Aktuell nicht (Original umgeht die ganze Kette).
6. **Mono-Erzwingung global**: Soll der gesamte Empfänger-Pfad mono erzwungen sein (z. B. mono am bus), oder reicht die Quellen-Mono-Erzwingung? Aktuell unvollständig (fg-Stereo leakt durch).
7. **`crushDry`/`crushWet`-Crossfade**: Toter Code aus v89-Ära. Bewusste Entscheidung der bisherigen Iterationen oder vergessen?
8. **Dry/Wet ohne Lied**: Wenn kein Lied geladen ist und User dreht drywet, hat das im Code keine direkte Auswirkung (songGain.gain wird über `(A.songBuf||streamMode)?songLvl*aud:0` geguardet). Aber die Stör-Quellen werden trotzdem von Phase-A-Multiplikatoren beeinflusst. Konzeptuell: User dreht drywet → Empfänger-Sound (nur Stör) "weicht" zurück, kein Original kommt. Akzeptabel?
9. **`A.shaper.curve = tanhCurve(k)` jeden Tick** allokiert. Welche Performance-Schwelle ist tolerierbar?
10. **`recDest`** erfasst auch die `chainGate=0`-Phase im Dry-Anschlag → die Aufnahme ist dann das ungefilterte Original. Beabsichtigt? (Konsistent mit Tooltip "nimmt auf, was du hörst" — aber im Dry-Modus ist das eben das Original.)
11. **SCAN-Button**: Spec erwähnt im FREEZE-Auftrag SCAN. Existiert nicht. Geplant für später, oder hat sich das Konzept verändert?

---

## 7. Zusammenfassung — Abweichungen Ziel ↔ Code

Sortiert nach **Schwere** (bricht Konzept → stört → kosmetisch). Keine Lösungsvorschläge.

### 7.1 Bricht das Konzept

| # | Abweichung | Ort | Zielsetzung verletzt |
|---|---|---|---|
| K1 | **Bei Dry-Anschlag spielt das Original ungefiltert (bzw. nur mono + 18 kHz Lowpass) durch.** chainGate=0 schaltet den Empfänger ab; origGate=1 lässt das Original via origLP→origGate→out direkt zur destination. Keine Empfangs-Färbung, kein Limiter, keine Safety. | [index.html:269, 275-276, 433-437](index.html:269) | "Lied klingt NIE wie das Original" |
| K2 | **Crush/distCurve-Verzerrung wird in Phase A NICHT reduziert** (nur der tanh-Shaper wird milder). Bei `crush>0` bleibt die Demodulator-Verzerrung auch im "öffnenden" Dry-Modus voll. | [index.html:322, 400-405](index.html:322) | Dry/Wet-Versprechen "weniger Rauschen/Verzerrung" |
| K3 | **safetyLim umgeht den Original-Pfad.** Das Original landet ungeschützt am Ausgang. Bei lauter Quelle clippt es. | [index.html:274-275](index.html:274) | "Eine unsichtbare Sicherung bügelt Spitzen/Schärfe am Ausgang" |
| K4 | **Mono nicht durchgängig erzwungen.** Foreign-Samples können stereo sein → Empfänger-Pfad wird stereo. | [index.html:333, 342](index.html:333) | "Mono" als Empfänger-Konzept |
| K5 | **Grain-Eingang entkoppelt von gehörter Quelle bei Dry-Anschlag.** musicTap sitzt auf songGain (Empfänger-Pfad), aber bei Dry-Anschlag spielt das Original. Körner kommen aus altem songGain-Stream (Empfänger), während User Original hört. | [index.html:300, 466-472](index.html:300) | Grain-Idee "läuft durch den Empfänger" — bei Dry passt nicht mehr |

### 7.2 Stört (Versprechen leicht gebrochen / Inkonsistenz)

| # | Abweichung | Ort | Anmerkung |
|---|---|---|---|
| S1 | **Grain-Spawner-Freeze (`useFreeze`) zündet praktisch nie** (Längen-Match-Bedingung mit Random-Längen). | [index.html:482, 486](index.html:482) | Toter Mechanismus; nicht erkennbar für den User |
| S2 | **Crackle läuft im FREEZE weiter** (Tooltip "Störungs-Atmung bleibt stehen"). | [index.html:448-461](index.html:448) | Subtil; aber Spec-Wortlaut bricht |
| S3 | **Grain im EXTRAKT × STÖR**: stille Körner → User mit STÖR im Extrakt hört nichts (Grain). | [index.html:469-472](index.html:469) | Funktional korrekt, aber UX-Verwirrung |
| S4 | **`breathe`-Slider 0 ≠ "steht still"** für Presets mit starkem Atmen-fadeDepth (ctlRange-Untergrenze > 0). | [index.html:135](index.html:135) | Wording-Konflikt |
| S5 | **FREEZE × Stör-Quellen-Sliders**: User kann während FREEZE noch `noise`/`fgLevel`/`het` ändern. Modus-Versprechen "Empfang eingefroren" wird durch Slider-Wirkung partiell umgangen. | n/a | Spec-Frage: sollen Slider in FREEZE auch frozen sein? |
| S6 | **`drywet`-Slider triggert applyControls()`, das aber für drywet nichts mehr macht.** Verzögerung der Hör-Wirkung bis erster audioTick nach Play. | [index.html:323, 586](index.html:323) | Subtil |

### 7.3 Kosmetisch / Code-Hygiene

| # | Befund | Ort |
|---|---|---|
| C1 | `crushWet.gain=1`, `crushDry.gain=0` permanent → Wet/Dry-Crossfade der Crush-Stufe ist toter Code (seit v89). | [index.html:226](index.html:226) |
| C2 | `mLow`, `mMid`, `mAir` aktive Filter im Master-Pfad, aber nicht in `A`. | [index.html:246-248, 305](index.html:246) |
| C3 | `intfActive` und `curNr` werden in audioTick gesetzt, aber nirgends gelesen. | [index.html:371, 417](index.html:371) |
| C4 | `daynight`-Button hat keinen Tooltip (im Gegensatz zu allen anderen). | [index.html:53](index.html:53) |
| C5 | `A.shaper.curve = tanhCurve(k)` jeden Tick → 30 720 Float32Array(1024)-Allokationen/s. GC-Druck. | [index.html:425](index.html:425) |
| C6 | `ScriptProcessorNode` deprecated. Migration zu AudioWorklet offen. | [index.html:276, 284](index.html:276) |
| C7 | Kein SCAN-Button, obwohl Spec im FREEZE-Auftrag erwähnt. | n/a |
| C8 | Tooltip "Fremdsignal" sagt nicht, dass es bis zu 2 Quellen pro Preset gibt. | [index.html:563](index.html:563) |
| C9 | Spec-Doku "PRESET-Modus (v96)" + "v95: Audio-Modulation per setInterval" Kommentar — Versionshinweise vermischt. | [index.html:18, 362](index.html:18) |

---

**Ende Audit v96. Keine Code-Änderung an `index.html`.**
