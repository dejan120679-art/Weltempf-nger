# Weltempfänger · Werkbank SW — Komplett-Diagnose v98

**Datei-Stand:** `index.html` auf Commit `cbe8012` (v98, 895 Zeilen, 50 KB).
**Methode:** Read-only. Karpathy-Guidelines: Annahmen offen, Unklarheiten benannt, kein Beschönigen.

> Diese Diagnose macht keine Aussage, was hörbar IST — sondern was der Code
> SOLLTE oder MÜSSTE. Die Hörprobe muss am Gerät passieren.

---

## 1 · Priorität: EXTRAKT-LECK-Analyse

**User-Symptom:** "Extrakt AN ändert NICHTS — Rauschen, Stimmen und Lied laufen unverändert weiter."

### 1.1 Quellen-zu-Destination-Tabelle (im Modus `extractMode=true`, `xm=0`)

| Quelle | Anlage | Ziel-Knoten | Pfad zu `ctx.destination` | Im Extrakt hörbar? | Warum |
|---|---|---|---|---|---|
| `noiseSrc` | [index.html:338](index.html:338) | `noiseGain → stoerCollect` ([index.html:341](index.html:341)) | stoerCollect → { stoerMute*0 → bus | revInStoer*0 → conv | delInStoer*0 → del | stoerTap → grainSink(g=0) } | **Nein** (sollte) | Alle Audio-Pfade gehen über stoerMute, revInStoer oder delInStoer — alle drei werden in audioTick auf 0 gezogen ([index.html:500, 504, 505](index.html:500)). stoerTap landet in grainSink mit gain=0 ([index.html:322-323](index.html:322)). |
| Crackle (Buffer pro Trigger) | [index.html:545-559](index.html:545) | `g.connect(A.stoerCollect\|\|A.bus)` ([index.html:554](index.html:554)) | identisch wie noise | **Nein** (sollte) | Fallback `A.bus` greift nur, wenn `A.stoerCollect` `undefined` ist. Nach `ensure()` IST `A.stoerCollect` gesetzt (siehe Object.assign [index.html:353](index.html:353)). |
| `fg` (Foreign 1) | [index.html:404](index.html:404), `g.connect(A.stoerCollect\|\|A.bus)` ([index.html:406](index.html:406)) | identisch | **Nein** (sollte) | Selber Fallback. `A.stoerCollect` ist gesetzt. |
| `fg2` (Foreign 2) | [index.html:415](index.html:415), [index.html:417](index.html:417) | identisch | **Nein** (sollte) | Selber Fallback. |
| `hetOsc` | [index.html:349-350](index.html:349) | `hetGain → stoerCollect` | identisch | **Nein** (sollte) | hetGain.gain wird in audioTick auf `hetBase` (ohne xm) gesetzt ([index.html:521](index.html:521)) — die Stummschaltung kommt erst an stoerMute. |
| `sweepOsc` | [index.html:351-352](index.html:351) | `sweepGain → stoerCollect` | identisch | **Nein** (sollte) | sweepGain.gain in audioTick OHNE xm ([index.html:525](index.html:525)). Mute übernimmt stoerMute. |
| `songSrc` / `streamSrc` | [index.html:645, 653](index.html:645) | `→ songGain` und `→ origRaw` (parallel) | **siehe 1.2 unten** | **Ja, falls songLvl*aud > 0** | songGain hängt direkt am bus und an revInMusic/delInMusic. xm wirkt NICHT auf songGain. Nur das aud-Gate gilt. |
| Empfangs-Körner | [index.html:637](index.html:637) | `g → A.grainOut → bus` ([index.html:328](index.html:328)) | bus → wow → … → out | **Ja** (Design) | Spec: "GRAIN bleibt bewusst ungefiltert". Ungemutet. Quelle der Körner: `musicRing` oder `stoerRing` — beide enthalten in Extrakt noch volle Werte (stoerTap ist vor stoerMute). |
| Original-Körner | [index.html:637](index.html:637) | `g → A.grainDryOut → softSafe` ([index.html:308](index.html:308)) | softSafe → safetyLim → out | **Ja** (Design) | Bypassed bus völlig. Geht nur durch End-Sicherung. |
| `origGate` (Original-Pfad) | [index.html:272-278](index.html:272) | `origGate → softSafe → safetyLim → out` ([index.html:306, 309](index.html:306)) | direkt zu out | **Ja, falls drywet < 100** | origGate.gain ist im Default (drywet=100, og=0) bei 0. Bei drywet < 100 öffnet sich der Pfad. Aud-Gate wirkt NICHT auf origGate. |
| `revWet` | conv-Output | `revWet → bus` ([index.html:296](index.html:296)) | bus → … → out | **Ja, falls reverb > 0** UND revInMusic-Eingang Signal hat | conv hat zwei Inputs: revInMusic (Music) und revInStoer (Stör). In Extrakt: revInStoer*0=0 → STÖR-Seite ist 0. revInMusic ungeguardet → Music-Hall geht durch. Music ist aber AUD-gegatet, also reduziert. |
| `delWet` | del-Output | `delWet → bus` ([index.html:297](index.html:297)) | identisch | **Ja, falls delay > 0** | Selbe Struktur wie revWet. |

### 1.2 Konkrete Antworten auf die Spec-Fragen

**a) Hängt stoerCollect zusätzlich direkt am bus/mix/Effekt-Eingang vorbei an stoerMute?**

`stoerCollect.connect(...)` findet sich an genau diesen Stellen:
- [index.html:283](index.html:283) `stoerCollect.connect(stoerMute)` → der gemutete Hauptpfad
- [index.html:325](index.html:325) `stoerCollect.connect(stoerTap)` → ScriptProcessor → `grainSink` (gain 0) — **inaudibel**, dient nur als Tap-Datenstrom
- [index.html:327](index.html:327) `stoerCollect.connect(revInStoer); stoerCollect.connect(delInStoer)` — beide gehen über `revInStoer`/`delInStoer`, die in audioTick auf `base*xm` gesetzt werden (in Extrakt also 0)

**Keine Direkt-Verbindung von `stoerCollect` an `bus`, `mix` oder einen ungeschützten Effekt-Eingang.** ✓ Aus Sicht des Code ist stoerCollect korrekt gekapselt.

**b) Wird `xm` wirklich auf `stoerMute.gain` angewendet — oder überschreibt jemand?**

Schreiber auf `A.stoerMute.gain`:
- Initial: [index.html:282](index.html:282) `stoerMute.gain.value=1`
- [index.html:500](index.html:500) `A.stoerMute.gain.setTargetAtTime(xm,now,.04)` — **einziger Laufzeit-Schreiber**

Kein anderer Code-Pfad in der Datei schreibt auf `A.stoerMute.gain`. Grep verifiziert: 0 weitere Treffer.

**Schreiber auf `A.revInStoer.gain`:**
- Initial: [index.html:293](index.html:293) `revInStoer.gain.value=0.75`
- [index.html:388](index.html:388) `A.revInStoer.gain.setTargetAtTime(s,t,.05)` aus `applyRevSrc()` — wird in `applyControls()` ([index.html:381](index.html:381)) und in jedem Source-Button-Click ([index.html:748](index.html:748)) aufgerufen. **Schreibt UNGEMUTET den Quellen-Basiswert (0/1/0.75).**
- [index.html:504](index.html:504) `A.revInStoer.gain.setTargetAtTime(revStoerBase*xm,now,.06)` aus audioTick — **schreibt gemutet (mit xm)**

**Hier ist eine Race-Situation:** Wenn der User in Extrakt einen Slider bewegt (egal welchen), wird applyControls → applyRevSrc aufgerufen, das `revInStoer.gain` auf den Basiswert (z. B. 1 bei STOER-Mode) setzt. audioTick überschreibt das innerhalb ≤33 ms wieder mit `base*0=0`. In dem ≤33-ms-Fenster ist die STÖR-Seite des Halls offen.

**Gleicher Race-Effekt auf `A.delInStoer.gain` über `applyDelSrc()`.**

Wenn die `reverb`/`delay`-Slider auf 0 stehen (Default), ist revWet/delWet = 0 → kein Audio-Effekt. Wenn aber Hall oder Echo aufgedreht sind UND der User Slider bewegt UND der Source-Mode auf STOER/ALLES steht: kurze, hörbare Bursts in extractMode.

**c) Wirkt das Song-Gate (`aud`) tatsächlich auf `songGain`?**

[index.html:508](index.html:508): `A.songGain.gain.setTargetAtTime((A.songBuf||streamMode)?songLvl*aud:0,now,extractMode?.015:.008);`

`aud` ist die im if(extractMode)-Block berechnete Größe ([index.html:488-498](index.html:488)). Bei `extractMode=false` bleibt `aud=1`. Bei `extractMode=true` wird `aud` aus dem masker berechnet.

**Kein Variablen-Schatten:** `aud` ist im audioTick-Scope deklariert ([index.html:488](index.html:488)) und der einzige Lese-Zugriff ist 20 Zeilen tiefer.

Aber: **die Masker-Berechnung selbst hat eine Eigenheit** ([index.html:492-494](index.html:492)):
```
const masker = noiseBase + (fgBase+fg2Base)*0.8 + hetBase*0.4
             + songLvl*(cr*0.55 + drive*0.25)  // Verzerrungs-Selbstmaskierung
             + tAbs;
```

Bei einem klaren Sender (z. B. STN-01: q0=0.95, cur.noise=0.5) mit Default-Reglern:
- `noiseBase ≈ 0.01`
- `fgBase ≈ kleiner Wert mit voice-Snippet`
- `hetBase = 0` (cur.het=0)
- `songLvl*(cr*0.55 + drive*0.25) ≈ 0.95 * 0.04 ≈ 0.04` (P.crush=0, drive≈0.17)
- `tAbs = 0.013` (default extThr=30)

→ `masker ≈ 0.01 + 0.08 + 0 + 0.04 + 0.013 ≈ 0.14`
→ `snr = songLvl/masker ≈ 0.95/0.14 ≈ 6.8`
→ `s0 = 0.2 + 0.33 = 0.53`
→ `aud = clamp((6.8 - 0.53)/1.5) = 1`

**Für klare Sender ist aud ≈ 1** → songGain bleibt ungedämpft. **Das Lied klingt in Extrakt identisch wie ohne Extrakt.** Aus User-Sicht ändert sich nichts.

**d) Ist `extractMode` beim Klick wirklich `true`?**

Click-Handler: [index.html:872](index.html:872) `$("btnExt").addEventListener("click",()=>setExtract(!extractMode));`

`setExtract`: [index.html:831](index.html:831) `function setExtract(on){ ensure(); if(on) A.ctx.resume(); extractMode=on; updateRecUI(); }`

Setzt direkt auf die Modulvariable. Kein Re-Initialisierungs-Pfad findet sich, der `extractMode` unmittelbar zurücksetzt — außer:
- `loadPreset` ([index.html:426](index.html:426)) am Ende: `if(extractMode){ extractMode=false; } updateRecUI();` — Reset bei Senderwechsel ist Spec-konform (v97-Nachzügler).

**Kein versehentlicher Reset gefunden.** Wenn der User nichts klickt außer btnExt, bleibt `extractMode=true`.

**e) Crackle-Ziel:**

[index.html:554](index.html:554): `g.connect(A.stoerCollect||A.bus);`

Wenn `A.stoerCollect` truthy: → stoerCollect. Nach `ensure()` ist A.stoerCollect der Gain-Knoten (truthy). ✓ Crackle landet in stoerCollect.

Fallback `A.bus` greift nur, wenn `ensure()` nie aufgerufen wurde. Da die Crackle-IIFE selbst guarded ist durch `if(A.ctx && playing && cur && !frozen)`, läuft sie nur, wenn `A.ctx` existiert — was nur nach `ensure()` der Fall ist. **Der Fallback ist toter Code.** Aber harmlos.

### 1.3 Fazit zur Extrakt-Leck-Analyse

**Aus reiner Code-Sicht ist die Extrakt-Stummschaltung der Stör-Quellen funktionell intakt.** Alle Pfade von noise, het, sweep, fg, fg2, crackle führen über `stoerCollect`, und ALLE drei Ausgänge von stoerCollect sind in extractMode auf 0 (stoerMute, revInStoer, delInStoer) bzw. inaudibel (stoerTap).

**Plausible Erklärungen für die User-Wahrnehmung "nichts ändert sich":**

| Erklärung | Wahrscheinlichkeit | Hinweis |
|---|---|---|
| **Klare Station, hohes aud** — songGain bleibt fast unverändert, der User hört das Lied weiter. Die Stör-Quellen waren von vornherein leise, der Wechsel fällt nicht auf. | hoch | Auf STN-01 / STN-09 / STN-03 etc. ist `cur.noise` klein und Default-Slider greifen nicht. Test auf STN-12 (sehr fern, noise=1.9, q0=0.30) zwingend nötig. |
| **Grain leakt Stör im Extrakt** — wenn `P.grain > 0` UND `grainSrcMode ∈ {STOER, ALLES}`. Der grain-Output geht via grainOut → bus an stoerMute vorbei. Spec-konform ("Grain bleibt bewusst ungefiltert"). | mittel | Nur wenn User Grain hochgedreht hat. Default `grain=0`. |
| **Race-Window bei Slider-Bewegung** — applyRevSrc/applyDelSrc setzen Stör-Eingänge des Halls/Echos kurz ungemutet. Nur bei aufgedrehtem Hall/Echo hörbar. | mittel | Nur wenn `P.reverb > 0` oder `P.delay > 0`. |
| **Browser-Cache** — User hat alte Version (vor v97 F13 oder v98). | gering, aber möglich | `cache-control` ist no-cache, aber das ist nur ein Hinweis. Hard-Refresh nötig. |
| **Hidden Code-Pfad in audioTick / extraTick** | sehr gering | Aus dem Code-Trace nicht ersichtlich. |
| **Lied selbst hat einen Anteil "noise" durch HP/LP/Crush/AGC** — wenn das Lied durch die Empfänger-Kette läuft, klingt es bandbegrenzt/gefärbt. User könnte diese Färbung mit "Rauschen" verwechseln. | mittel | Für Test-Vergleich: parallel ohne Extrakt mit drywet=0 (Original direkt) und mit Extrakt vergleichen. |

**Empfehlung für Reproduktion:** Test auf STN-12 mit Default-Reglern. Vergleich: 5 Sek normal → Extrakt drücken → 5 Sek lauschen. Wenn `cur.noise=1.9` und `masker ≈ 0.5+` und `songLvl ≈ 0.27`, sollte `snr ≈ 0.5` und `aud ≈ 0` werden. Lied + Stör SOLLEN auf Stille fallen.

---

## 2 · Signalfluss-Gesamtkarte (v98)

### 2.1 Quellen → Bus

```
                              ┌─── songGain ─────────────────────────────────► bus
songSrc / streamSrc ──────────┤   (gain = songLvl*aud, in extract: gated)         [index.html:345]
                              ├─── musicTap ─► grainSink (gain=0) ─► destination  [index.html:345]
                              ├─── revInMusic ─► conv ─► revWet ─► bus            [index.html:347, 296]
                              └─── delInMusic ─► del  ─► delWet ─► bus            [index.html:347, 297]

songSrc / streamSrc ──► origRaw ─► dryDelay ─► origLP ─► origGate ─► softSafe       [index.html:645, 653, 274, 278, 306]

noiseSrc ─► noiseFilt ─► noiseGain ─► stoerCollect                                    [index.html:341]
hetOsc   ─► hetGain   ────────────► stoerCollect                                    [index.html:350]
sweepOsc ─► sweepGain ────────────► stoerCollect                                    [index.html:352]
Crackle  ─► bp ─► g  ─────────────► stoerCollect                                    [index.html:554]
fg       ─► g  ──────────────────► stoerCollect                                     [index.html:406]
fg2      ─► g2 ──────────────────► stoerCollect                                     [index.html:417]

stoerCollect ┬─► stoerMute (×xm) ─► bus                                              [index.html:283]
             ├─► stoerTap ─► grainSink (gain=0) ─► destination                       [index.html:325, 323]
             ├─► revInStoer (×revStoerBase*xm) ─► conv ─► revWet ─► bus              [index.html:327, 296]
             └─► delInStoer (×delStoerBase*xm) ─► del ─► delWet ─► bus               [index.html:327, 297]

grainOut    ─────────────────────► bus                                               [index.html:328]
grainOut    ─► grainCapMix ─► grainRecTap ─► grainSink                              [index.html:331, 334]
grainDryOut ─► softSafe                                                              [index.html:308]
grainDryOut ─► grainCapMix                                                           [index.html:331]
```

### 2.2 Bus → Empfänger-Kette → out

```
bus ─► wow ─► hp ─► lp ─► pkf ─► shaper ─┬─► crushShaper ─► crushWet ─► agc          [index.html:285-287]
                                          └─► crushDry ────────────► agc

agc ─► rHP ─► rComp ─► rMakeup ─► rTame ─► warmShelf ─► lim                          [index.html:288]
lim ─► mix                                                                            [index.html:289]

mix ─► safeBell ─┬─► [Master AN] mHP ─► mLow ─► mMid ─► mAir ─► mComp ─► mMakeup ─► mOn ─► chainGate    [index.html:302-303]
                 └─► [Master AUS] mOff ─► chainGate                                                       [index.html:304]
chainGate (=cos(og*π/2)) ─► softSafe                                                  [index.html:305]
origGate (=sin(og*π/2)*…)  ─► softSafe                                                [index.html:306]
grainDryOut (=0.8)          ─► softSafe                                                [index.html:308]

softSafe ─► safetyLim ─► out                                                          [index.html:309]
out ─► ctx.destination                                                                [index.html:310]
out ─► recordTap ─► grainSink (Aufnahme)                                              [index.html:314, 323]
```

### 2.3 Effekt-Routing-Matrix (Hall/Echo)

```
conv: revInMusic + revInStoer ─► conv ─► revWet ─► bus  (Hall läuft durch Empfänger)
del:  delInMusic + delInStoer ─► del  ─► delWet ─► bus  (Echo läuft durch Empfänger)

del internal feedback: del ─► fbLP ─► fbSat ─► delFb ─► del   [index.html:253]
del wobbel: delLFO ─► delLFOdepth ─► del.delayTime            [index.html:256]
```

### 2.4 Recording-Taps

| Tap | Quelle | Ziel | State | Aktiv |
|---|---|---|---|---|
| `recordTap` | `out` ([index.html:314](index.html:314)) | `grainSink` ([index.html:323](index.html:323)) | `A.recActive`, `A.recChunks` | MITSCHNITT-Aufnahme |
| `grainRecTap` | `grainCapMix` (= grainOut + grainDryOut) ([index.html:331, 334](index.html:331)) | `grainSink` ([index.html:334](index.html:334)) | `A.recGrainActive`, `A.recGrainChunks` | GRAIN-Aufnahme |

**Beobachtung Recording:** `recordTap` hört auf `out` ab. Out enthält **alles**, was zu ctx.destination geht, also auch das Grain (grainOut → bus → … → out) und Grain-Dry (grainDryOut → softSafe → out). Wenn der User MITSCHNITT und Grain parallel aufnimmt, ist der Grain-Anteil **zweimal** in den Dateien.

---

## 3 · SOLL/IST je Werkzeug

| Werkzeug | Spec-Soll | Code-Ist | Deckungsgleich? | Hinweis |
|---|---|---|---|---|
| **Extrakt — Stör stumm** | xm-Mute auf stoerCollect-Ausgängen | stoerMute*xm + revInStoer*xm + delInStoer*xm | **Ja** (aus Code-Trace) | Race bei Slider-Bewegung; siehe §1.2 b. [index.html:500, 504, 505](index.html:500) |
| **Extrakt — Stör bleibt Masker** | Stoer-Quellen weiter in masker-Formel | noiseBase / fgBase / fg2Base / hetBase werden vor extractMode-Block berechnet und in masker eingesetzt | **Ja** | [index.html:467-486, 492](index.html:467) |
| **Extrakt — Verzerrungs-Selbstmaskierung** | `+ songLvl*(cr*0.55 + drive*0.25)` | identisch | **Ja** | [index.html:493](index.html:493) |
| **Extrakt — Justage-Regler wirken** | s0 aus extGate, tAbs aus extThr | `s0=0.2+(P.extGate\|\|0)/100`, `tAbs=0.004+(P.extThr\|\|0)/100*0.028` | **Ja** | [index.html:490-491](index.html:490) |
| **Extrakt — Reset bei Senderwechsel** | loadPreset setzt extractMode=false | identisch | **Ja** | [index.html:426](index.html:426) |
| **Rettung — nur fern** | UI-Disable für nicht-fern | updateSliderUI mit `isRescue(cur)` | **Ja** | [index.html:683-688](index.html:683) |
| **Rettung — monotoner Verlauf** | clean = 1-min(1, nr*1.5)*0.6; rMakeup = 1+pow(nr,1.6)*3.3 | identisch | **Ja** | [index.html:449, 365](index.html:449) |
| **Rettung — Crush bleibt** | Rettung wirkt nur über `clean` auf shaper-k (nicht auf crushShaper) | identisch | **Ja** | crushShaper-Curve hängt nur von P.crush ab ([index.html:376](index.html:376)) |
| **FREEZE — friert physische Modulation** | `T = frozen ? freezeT : (now-physOff)`, breath skip wenn frozen | identisch | **Ja** | [index.html:445, 453](index.html:445) |
| **FREEZE — friert Knistern** | crackle-IIFE guard `!frozen` | identisch | **Ja** | [index.html:546](index.html:546) |
| **FREEZE — friert NICHT Grain** | grains-IIFE hat keinen frozen-Guard | identisch | **Ja** | [index.html:562](index.html:562) |
| **Dry/Wet — durchgehender Morph** | r-basierte Pipeline, kein o/b-Splitting | identisch (r = 1-w; o entfernt; b durch og ersetzt) | **Ja** | [index.html:480-486, 528-533](index.html:480) |
| **Dry/Wet — Original durch Safety** | origGate → softSafe → safetyLim → out | identisch (KEIN out direkt) | **Ja** | [index.html:306, 309](index.html:306) |
| **Grain — Quellen-Abgriffe** | songGain → musicTap, stoerCollect → stoerTap | identisch | **Ja** | [index.html:345, 325](index.html:345) |
| **Grain — Original-Körner-Reise** | ab amt>0.4 wachsender Anteil aus A.songBuf, eigener Output grainDryOut → softSafe | identisch | **Ja** | [index.html:586-592, 637](index.html:586) |
| **Grain — STÖR gräbt Maskiertes** | stoerCollect → stoerTap VOR stoerMute | identisch | **Ja** | [index.html:325](index.html:325) |
| **Hall/Echo — Quellen-Wahl** | revInMusic/revInStoer, delInMusic/delInStoer | identisch | **Ja** | [index.html:292-295, 296-297](index.html:292) |
| **Hall/Echo — Ausgang durch Empfänger** | revWet → bus, delWet → bus | identisch | **Ja** | [index.html:296-297](index.html:296) |
| **Hall/Echo — Stör-Eingang × xm** | revInStoer.gain = base*xm, delInStoer.gain = base*xm | identisch (audioTick) **aber Race mit applyRevSrc/applyDelSrc** | **Teilweise** | [index.html:504-505, 388, 394](index.html:504) — siehe §5.1 |
| **Wärme — nur Empfänger-Pfad** | warmShelf zwischen rTame und lim; nicht in Original-Pfad | identisch | **Ja** | [index.html:288, 379](index.html:288) |
| **Wärme — ctlRange-Deckel** | warmMax = 3+cur.q0*6 → ferne Stationen weniger | gain-Multiplikator skaliert, KEIN ctlRange-Eintrag für warm-Slider min/max | **Nein** (Spec-Konflikt zur Audit-Sprache) | Slider geht 0-100 für jede Station. Effekt skaliert intern. Spec-Wortlaut "ctlRange" war hier nie umgesetzt — bewusst, weil die Multiplikator-Form genügt. Doku-Unschärfe. |
| **MITSCHNITT-Tap nimmt auf, was man hört** | recordTap an `out` | identisch | **Ja**, mit Caveat: Grain ist in der Datei mit drin (out enthält grainOut über bus + grainDryOut über softSafe) | [index.html:314](index.html:314) |
| **GRAIN-Tap nimmt nur Grain auf** | grainRecTap an grainCapMix (= grainOut+grainDryOut) | identisch | **Ja** | [index.html:331, 334](index.html:331) |
| **Recording — Float32 + WAV** | ScriptProcessor + buildWav | identisch | **Ja** | [index.html:312, 765](index.html:312) |
| **Recording — kein Auto-Download** | STOPP → recPending; SPEICHERN-Button | identisch | **Ja** | [index.html:789-803](index.html:789) |
| **EXTRAKT-Modus-Hörmodus, kein Download** | btnExt hat nur setExtract, kein Recording-Trigger | identisch | **Ja** | [index.html:872](index.html:872) |

---

## 4 · Wechselwirkungs-Matrix

### 4.1 Extrakt × …

| Mit | Wechselwirkung | Anmerkung |
|---|---|---|
| **Hall** | revInStoer × xm = 0 → STÖR-Hall stumm. revInMusic ungemutet, songGain*aud-gedämpfte Musik geht durch Hall. | Race bei Slider-Bewegung (§5.1) |
| **Echo** | analog wie Hall | Race wie Hall |
| **Grain (MUSIK)** | musicTap-Ring enthält das aud-gegatete Music-Signal → Körner sind ebenfalls leise | Konsistent |
| **Grain (STÖR)** | stoerTap-Ring enthält volles Stör-Signal (vor stoerMute) → STÖR-Körner sind LAUT auch im Extrakt — **Spec-konformes Feature** | "GRAIN bleibt bewusst ungefiltert" |
| **Grain (Original-Körner)** | grainDryOut bypasst bus → läuft durch softSafe → out. Aud-Gate wirkt NICHT auf grainDryOut. | Original-Körner spielen voll, auch wenn die Gate-Musik gedämpft ist. **Im Spec nicht explizit adressiert.** |
| **Rettung** | nr beeinflusst clean → duckF → noiseBase/hetBase → masker. Bei voll-Rettung sinkt der Masker (clean=0.4), aud steigt — Lied wird hörbarer. | Logisch konsistent |
| **Dry/Wet** | r-Reduktion senkt Stör-Basen → masker sinkt → aud steigt. Bei drywet < 100: Original spielt UNABHÄNGIG von extractMode (origGate nicht aud-gegatet). | Original-Pfad ist im Extrakt **nicht** stumm bei drywet < 100. |
| **FREEZE** | T eingefroren → fade, breath, fg-Gate, het-Drift stabil. masker ist stabil. aud stabil. | Spec-konform |
| **Wärme** | warmShelf wirkt auf Empfänger-Pfad. Im Extrakt: gedämpfte Musik wird gewärmt. Stör nicht (kommt aus stoerMute=0). | Logisch konsistent |
| **Master AUS** | umgeht Master-EQ, aber Safety ist aktiv. Extrakt-Logik unbetroffen. | OK |
| **MITSCHNITT-Aufnahme** | recordTap an `out` — nimmt das gegatete Extrakt-Ergebnis auf. Inkl. Original (drywet<100) und Grain. | OK |

### 4.2 FREEZE × …

| Mit | Wechselwirkung |
|---|---|
| **Hall/Echo Wet-Pegel** | unbeeinflusst — Werte hängen an P.reverb, P.delay (User-Slider), nicht an T |
| **Grain** | grains-IIFE läuft weiter (kein frozen-Guard) — spec-konform |
| **Dry/Wet** | r ist User-gesteuert, nicht T-abhängig — FREEZE ändert nichts an r |
| **Crackle** | mit FREEZE eingefroren ✓ |

### 4.3 Dry/Wet × …

| Mit | Wechselwirkung |
|---|---|
| **Extrakt** | r reduziert noiseBase/fgBase/hetBase im masker → mehr Musik durch das Gate |
| **Rettung** | überlappende Stör-Absenkungen: clean × (1-r*0.8) auf noiseBase. **Doppelte Dämpfung** ist gewollt (zwei semantisch verschiedene Aktionen). |
| **FREEZE** | origLvl-Atmung nutzt `fade` (aus T eingefroren) → Atmung bleibt stehen |
| **Wärme** | warmShelf wirkt voll, auch im "Dry"-Anschlag (Original geht NICHT durch warmShelf) |
| **Crush-Crossfade** | crushWet=(1-r), crushDry=r → bei r=1 nur trockenes Shaper-Signal |
| **Master/Safety** | Safety wirkt auf beide Pfade (chainGate + origGate gehen durch softSafe+safetyLim). |

---

## 5 · Redundanz & Hygiene

### 5.1 Konkurrierende Schreiber auf AudioParams

| Param | Schreiber 1 | Schreiber 2 | Verhalten |
|---|---|---|---|
| `revInMusic.gain` | applyRevSrc ([index.html:388](index.html:388)) — schreibt Basiswert (0/1/0.75) | audioTick: keiner (nur über applyRevSrc) | Kein Konflikt — applyRevSrc ist alleiniger Schreiber |
| `revInStoer.gain` | applyRevSrc ([index.html:388](index.html:388)) — **ungemutet** | audioTick ([index.html:504](index.html:504)) — `base*xm` | **Race-Konflikt im Extrakt:** applyRevSrc setzt Basiswert ohne xm; audioTick zieht auf 0 innerhalb ≤33 ms. Im Fenster sind kurzzeitig STÖR-Eingänge offen. Hörbar nur, wenn `reverb > 0`. |
| `delInMusic.gain` | applyDelSrc ([index.html:394](index.html:394)) | audioTick: keiner | Kein Konflikt |
| `delInStoer.gain` | applyDelSrc ([index.html:394](index.html:394)) — ungemutet | audioTick ([index.html:505](index.html:505)) — `base*xm` | **Race-Konflikt analog zu revInStoer.** Nur hörbar bei `delay > 0`. |
| `stoerMute.gain` | audioTick ([index.html:500](index.html:500)) — `xm` | keiner | Einziger Schreiber ✓ |
| `chainGate.gain` | audioTick ([index.html:533](index.html:533)) — `cos(og*π/2)` | keiner | Einziger Schreiber ✓ |
| `origGate.gain` | audioTick ([index.html:532](index.html:532)) — `origLvl` | keiner | ✓ |
| `origLP.frequency` | audioTick ([index.html:531](index.html:531)) | keiner | ✓ |
| `lp.frequency` | audioTick ([index.html:513](index.html:513)) | keiner | ✓ |
| `shaper.curve` | audioTick ([index.html:517](index.html:517)) mit Cache | keiner | ✓ |
| `crushShaper.curve` | applyControls ([index.html:376](index.html:376)) | keiner | ✓ |
| `crushWet.gain` / `crushDry.gain` | audioTick ([index.html:519-520](index.html:519)) | keiner | ✓ |
| `warmShelf.gain` | applyControls ([index.html:379](index.html:379)) | keiner | ✓ |
| `del.delayTime` | applyControls ([index.html:371](index.html:371)) — Slider-Setting; delLFO via delLFOdepth ([index.html:256](index.html:256)) — Wobbel (AudioParam-Eingang, nicht setTargetAtTime) | additive | OK (LFO ist gain-Modulator auf delayTime, nicht TargetAtTime-Konflikt) |
| `delFb.gain` | applyControls ([index.html:372](index.html:372)) | keiner | ✓ |
| `fbLP.frequency` | applyControls ([index.html:373](index.html:373)) | keiner | ✓ |
| `delLFOdepth.gain` | applyControls ([index.html:374](index.html:374)) | keiner | ✓ |
| `mOn.gain` / `mOff.gain` | setMaster ([index.html:877](index.html:877)) | keiner | ✓ |
| `out.gain` | applyControls ([index.html:382](index.html:382)) | keiner | ✓ |
| `noiseFilt.frequency` | audioTick ([index.html:465](index.html:465)) | keiner | ✓ |
| `noiseGain.gain` | audioTick ([index.html:509](index.html:509)) | keiner | ✓ |
| `hetGain.gain` | audioTick ([index.html:521](index.html:521)) | keiner | ✓ |
| `sweepGain.gain` | audioTick ([index.html:525-526](index.html:525)) | keiner | ✓ |
| `fg.gain` / `fg2.gain` | audioTick ([index.html:506-507](index.html:506)) | keiner | ✓ |
| `songGain.gain` | audioTick ([index.html:508](index.html:508)) | keiner | ✓ |

### 5.2 Verwaiste Knoten / Variablen aus den v95-v98-Umbauten

| Symbol | Ort | Status |
|---|---|---|
| `crushDry.gain=0` initial ([index.html:234](index.html:234)) | wird ab v98 in audioTick mit `r` moduliert | OK, aber Initial-Wert ist veraltet (überschrieben in 33ms). Kein Bug. |
| `mLow`, `mMid`, `mAir` | jetzt in A.Object.assign aufgenommen ([index.html:353](index.html:353)) | OK seit v97 H24 |
| `crushDry` Gain wird nie auf 0 zurückgesetzt | applyControls setzt nicht | audioTick übernimmt — OK |
| `recordTap` connectet zu `grainSink` ([index.html:323](index.html:323)) | grainSink hat gain=0 — recordTap ist nur "scriptproc-Anker", inaudibel | ✓ Trick zur ScriptProcessor-Liveness |
| `grainRecTap` connectet auch zu grainSink ([index.html:334](index.html:334)) | gleicher Mechanismus | ✓ |
| `A.recDest` aus v90 | Ist in v97 entfernt worden — nicht mehr im Object.assign | ✓ aufgeräumt |
| `intfActive`, `curNr` aus früheren Versionen | in v97 entfernt | ✓ |
| `wowLFO` | wird in ensure() erstellt, gestartet, aber nicht in A aufgenommen | **Toter Knoten in A** — wowLFO existiert in ensure-Closure (kein Leak, weil wowLFO → wowDepth → wow.delayTime hält ihn am Leben). Doku-Unschärfe. |
| `delLFO` | in A.Object.assign aufgenommen ([index.html:353](index.html:353)) | OK |
| `delLFOdepth` | in A | OK |
| `safetyLim`, `softSafe`, `safeBell` | **NICHT** in A.Object.assign | nicht von außen erreichbar. Kein Bug, aber inkonsistent zu z. B. `lim`. |
| `grainSink` | NICHT in A.Object.assign | analog zu safetyLim — Verbindungskette hält ihn am Leben. Inkonsistent. |
| `recPendingKind`, `recGrainPendingKind` | Modul-Variablen | korrekt benutzt |
| `lastK` | tanhCurve-Cache | korrekt benutzt |

### 5.3 ScriptProcessor-Inventar

| Knoten | Job | Quelle | Senke |
|---|---|---|---|
| `musicTap` ([index.html:318](index.html:318)) | musicRing füllen | songGain | grainSink (gain=0) → destination |
| `stoerTap` ([index.html:320](index.html:320)) | stoerRing füllen | stoerCollect | grainSink |
| `recordTap` ([index.html:312](index.html:312)) | recChunks bei recActive | out | grainSink |
| `grainRecTap` ([index.html:332](index.html:332)) | recGrainChunks bei recGrainActive | grainCapMix | grainSink |

**4 ScriptProcessors aktiv**, alle in `grainSink` mündend. **Deprecated** (W3C — AudioWorklet wäre Nachfolger). Bekannte Last.

### 5.4 Performance-Risiken

- `audioTick` läuft alle 33 ms und schreibt ~30+ AudioParam-Targets pro Tick.
- `tanhCurve(k)` mit Cache (`lastK`) — gut, kein Allokations-Sturm mehr.
- `grains()`-IIFE allokiert pro Korn einen `AudioBuffer` (bis ~800 ms Länge bei amt=1) — kann bei voll-aufgedrehtem Grain für GC-Druck sorgen.
- `crackle`-IIFE allokiert pro Trigger einen `AudioBuffer` (~20-80 ms) — wenig.

### 5.5 Tote / unbenutzte Pfade

- **Fallback `A.stoerCollect||A.bus`** in Crackle / startForeign ([index.html:406, 417, 554](index.html:406)) — der `||A.bus`-Zweig ist toter Code, weil A.stoerCollect nach `ensure()` immer gesetzt ist und die Crackle/Foreign-Pfade nur nach ensure() laufen. Kein Bug, kann irreführen.
- **`P.extGate||0` / `P.extThr||0`** in audioTick ([index.html:490-491](index.html:490)) — der `||0`-Default ist toter Code, da `P.extGate=33` und `P.extThr=30` initial gesetzt sind.

---

## 6 · Befundliste

### BLOCKER (bricht Werkzeug)

Keine eindeutigen Blocker aus reiner Code-Sicht gefunden.

### STÖRT (sichtbares oder hörbares Fehlverhalten)

| ID | Befund | Symptom | Ursache | Ort |
|---|---|---|---|---|
| **S1** | **Race auf revInStoer.gain / delInStoer.gain** | Bei aufgedrehtem Hall ODER Echo UND Slider-Bewegung im EXTRAKT: kurzzeitiger STÖR-Eingang offen (≤33 ms je Slider-Click). | applyRevSrc/applyDelSrc schreiben Basiswert ohne xm. audioTick überschreibt asynchron. | applyRevSrc [index.html:388](index.html:388), applyDelSrc [index.html:394](index.html:394), audioTick [index.html:504-505](index.html:504) |
| **S2** | **Original-Pfad nicht aud-gegatet** | Im EXTRAKT + drywet<100 spielt das Original UNGEGATETE Lautstärke (origGate.gain = sin(og*π/2)*…). aud wirkt nur auf songGain (Empfänger-Pfad). | Spec-Lücke: origGate ist nicht im aud-Multiplikator-Pfad. | audioTick [index.html:532](index.html:532) |
| **S3** | **Grain-Original-Körner nicht aud-gegatet** | Im EXTRAKT + grain>0.4: Original-Körner aus A.songBuf gehen via grainDryOut → softSafe → out, OHNE aud-Gate. | grainDryOut ist parallel zum origGate-Pfad gleich strukturiert. | grains [index.html:637](index.html:637), Verkabelung [index.html:308](index.html:308) |
| **S4** | **MITSCHNITT enthält Grain doppelt** | recordTap an `out` nimmt grainOut (über bus → Empfänger) UND grainDryOut (über softSafe) mit auf. Wenn parallel GRAIN-REC läuft, ist der Grain in beiden Dateien. | Spec-Wortlaut "Mitschnitt nimmt auf, was man hört" deckt das ab, aber Doku könnte irritieren. | [index.html:314, 331, 334](index.html:314) |
| **S5** | **Konzept "Stör-Mute via stoerMute" für den User unsichtbar bei klaren Stationen** | Auf STN-01 / STN-09 / STN-03 sind Stör-Quellen ohnehin leise; Extrakt-Klick fällt subjektiv NICHT auf. Lied bleibt durch hohes aud unverändert. Code ist korrekt, Wahrnehmung enttäuscht. | Default-Slider auf nahen Stationen geben kaum Stör. masker bleibt klein, aud≈1. | n/a — Wahrnehmungs-Effekt |
| **S6** | **Wärme-Slider ohne ctlRange-Einschränkung** | Spec sagt "ctlRange-Deckel: ferne Stationen wärmen weniger". Code lässt jeden Slider 0-100 laufen; der Multiplikator `warmMax = 3+cur.q0*6` skaliert intern. UI-Wahrnehmung: jeder Slider gleich weit. | Bewusste Auslegung in v97 (Multiplikator statt ctlRange). Spec-Doku unschärfe. | applyControls [index.html:378-379](index.html:378), ctlRange [index.html:139-144](index.html:139) |
| **S7** | **Tag/Nacht-Faktor `tnFG = night?1.35:0.7` schiebt am Tag fg um 30 % nach unten** | Bei Default-Slider sind Fremdsignale tagsüber leiser als im Tag/Nacht-Vergleich erwartet. Wahrnehmung: "Tag = leise, Nacht = laut" — passt zur Idee. | bewusst | audioTick [index.html:462, 470](index.html:462) |

### KOSMETIK / DOKUMENTATIONS-UNSCHÄRFE

| ID | Befund | Ort |
|---|---|---|
| **K1** | `safetyLim`, `softSafe`, `safeBell`, `grainSink`, `wowLFO`, `delFb` (delFb ist drin), `fbSat` sind nicht in `A.Object.assign` (manche schon — fbSat NICHT) | [index.html:353](index.html:353) |
| **K2** | Fallback `A.stoerCollect\|\|A.bus` in 3 Stellen ist totes `||A.bus` (Crackle, startForeign, startForeign-fg2) | [index.html:406, 417, 554](index.html:406) |
| **K3** | `P.extGate\|\|0` / `P.extThr\|\|0`-Default in audioTick — toter Code, da P-Defaults gesetzt | [index.html:490-491](index.html:490) |
| **K4** | Kommentar Zeile 22 noch aus v97: "Mono. Holt nichts Verlorenes zurück." — gilt weiter, aber kein Hinweis auf v98-Änderungen (Wärme, Dry/Wet-Morph, Effekt-Routing) | [index.html:22](index.html:22) |
| **K5** | Tooltip-Text vom `drywet`-Slider sagt "am Anschlag frei". Code: bei drywet=0, og=pow(0.92/0.92, 1.35)=1. cos(π/2)=0 → chainGate=0. origLvl = sin(π/2)*((1-0)+0*fade)=1 (cd=1-1*0.95=0.05; aber wait: cd = 1 - 1*0.95 = 0.05; origLvl = sin(π/2)*((1-0.05)+0.05*fade)=0.95+0.05*fade ≈ 1). Das passt. ✓ | Kein Bug, nur Verifikation. |
| **K6** | 4 ScriptProcessoren sind weiterhin deprecated | strukturell |
| **K7** | "Master AUS" als Default seit v97 — User hört initial den "Direct"-Pfad. Spec ok, aber für Erst-Erlebnis evtl. überraschend | [index.html:207](index.html:207) |

---

## 7 · Karpathy-Check zur Diagnose

- **Think Before Coding:** keine Code-Änderung — pure Code-Lese-Analyse. ✓
- **Surface Tradeoffs:** der wahrscheinlichste Grund für das User-Symptom (klare Station + aud≈1 → "nichts ändert sich") ist offen genannt, nicht überspielt.
- **Surface Unklarheiten:** mein Code-Trace fand KEINEN harten Leak. Mögliche Race-Window, Wahrnehmungs-Effekte, Browser-Cache und das "Original-Pfad ist nicht aud-gegatet"-Detail sind als Hypothesen mit Gewichtung dokumentiert.
- **Goal-Driven:** Reproduktionsempfehlung für den User-Test gegeben (STN-12, Default, vorher/nachher hören).

**Ende DIAGNOSE_v98. Keine Code-Änderung an `index.html`.**
