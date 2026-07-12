# kami-ongaku-notation

**Portable notation/score data model — the missing "what does a written
piece of music look like" layer for kotoba-lang's `ongaku` (music
production) domain.** An L3-authoring
[kotoba-lang](https://github.com/kotoba-lang) capability library per
[ADR-2607121400](https://github.com/com-junkawasaki/root/blob/main/90-docs/adr/2607121400-kami-ongaku-eizo-commercial-grade-cljs-stack.md),
analogous to what Sibelius/Dorico/MuseScore's internal score model does:
part → measure → note/chord/rest, with pitch, exact-rational duration
(no floats), time/key signature, tempo, dynamics and articulation — plus
MusicXML-subset import/export so the IR can round-trip through the
notation-software ecosystem.

Portable `.cljc` across JVM / ClojureScript, zero external dependencies
(including its own minimal XML reader/writer — no platform XML library).

**Why a hand-rolled rational type** (`kami.ongaku.notation.rational`):
ClojureScript's number tower is IEEE-754 doubles only — `(/ 1 4)` is exact
`1/4` on the JVM but `0.25` under cljs. A "no floats" duration model that
relies on native `/`/ratio literals is therefore only exact on one of the
two target platforms. Every duration/time-signature computation here goes
through `kami.ongaku.notation.rational`'s `{:rational/num :rational/den}`
records instead, so exactness holds on both.

## Scope (v0)

This is a data model + structural validator + MusicXML I/O, not a
notation *application*:

- **In scope**: pitch (MIDI ⇄ scientific pitch notation ⇄ step/alter/
  octave), exact-rational duration (base type + dots, ties), dynamics
  (ppp–fff with a velocity mapping), articulation marks, chords, multiple
  voices, time/key signature, tempo, per-measure/per-voice duration-sum
  validation, MusicXML 4.0 partwise import/export for the subset above.
- **Not in scope**: visual layout/engraving (beaming, stem direction, page
  layout), playback/audio rendering (that's `kotoba-lang/audio`'s L2
  concern per the ADR), arbitrary third-party MusicXML conformance beyond
  the subset this library itself emits, triplets/irregular tuplets,
  cross-staff notation, guitar tab, percussion clefs.

## Contract

```clojure
(require '[kami.ongaku.notation :as notation]
         '[kami.ongaku.notation.pitch :as pitch]
         '[kami.ongaku.notation.validate :as validate]
         '[kami.ongaku.notation.musicxml :as musicxml])

;; pitch: MIDI <-> scientific pitch notation <-> step/alter/octave
(pitch/pitch {:step :C :alter 1 :octave 4})        ;=> {:pitch/step :C :pitch/alter 1 :pitch/octave 4}
(pitch/pitch->midi *1)                             ;=> 61
(pitch/scientific->pitch "C#4")                    ;=> same record
(pitch/pitch->scientific *1)                       ;=> "C#4"

;; note: pitches is a vector of 1+ pitches (a chord), or empty/nil for a rest
(notation/note {:pitches [{:step :C :octave 4}] :type :quarter
                :dynamic :mf :articulations #{:staccato}})
(notation/note {:type :eighth})                    ;=> a rest

;; measure -> part -> score, each returning nil on structurally invalid input
(def m (notation/measure {:number 1 :time-sig {:beats 4 :beat-type 4}
                           :key-sig {:fifths 0} :notes [...]}))
(def score (notation/score {:parts [(notation/part {:id "P1" :name "Piano" :measures [m]})]}))

;; validate: does each voice in each measure actually sum to the time
;; signature's capacity? (carries the signature forward across measures
;; that don't restate one, per MusicXML semantics)
(validate/validate-part (first (:score/parts score)))

;; MusicXML 4.0 partwise I/O
(musicxml/score->musicxml score)                   ;=> XML string
(musicxml/musicxml->score xml-string)               ;=> score IR
```

## Naming note

`kami-ongaku-*` mirrors the `kami-engine-*`/`kami-mangaka-*` family naming
convention for domain-authority repos (ADR-2607121400 §2.1); `ongaku`
(音楽) is the music-production domain, parallel to `eizo` (映像) for video.
This library models the score itself — it is not `kotoba-lang/composer`
(the outer AI-composition request/track contract mirroring
`ai.gftd.ongakuka.*`) or `kotoba-lang/ongaku` (BGM catalog + license
gating). No network, no I/O beyond pure string parsing/emission, no model
call.
