---
title: "Music theory for seismologists"
date: 2026-05-24
summary: "Pitch, octaves, notes, scales, keys, chords and harmonics — recast in the language of frequency, ratios, and spectra."
tags: []
---

A compact, technically grounded introduction to music fundamentals, framed using analogies that should be intuitive to a seismologist. Everything here lives on the frequency axis.

## Sound, frequency, and pitch

Sound is a pressure wave, characterised primarily by frequency (Hz), amplitude, and waveform. **Pitch** is the perceptual correlate of frequency: higher frequency, higher pitch.

Reference values:

- 110 Hz ≈ note **A2**
- 220 Hz ≈ **A3**
- 440 Hz ≈ **A4** (standard tuning reference)

Doubling the frequency raises pitch by one **octave**: $f \to 2f$. This is a scale invariance — the system "looks the same" one octave up. Notes an octave apart sound so similar that they are given the same name.

## Notes and the octave

Within one octave, Western music divides pitch space into **12 equal steps**. Each step (a **semitone**) multiplies frequency by

$$2^{1/12} \approx 1.0595.$$

This is logarithmic binning of the frequency axis, with 12 bins per doubling.

The 12 notes are:

$$\text{C}, \ \text{C}\sharp/\text{D}\flat, \ \text{D}, \ \text{D}\sharp/\text{E}\flat, \ \text{E}, \ \text{F}, \ \text{F}\sharp/\text{G}\flat, \ \text{G}, \ \text{G}\sharp/\text{A}\flat, \ \text{A}, \ \text{A}\sharp/\text{B}\flat, \ \text{B}.$$

After B, the cycle repeats one octave up (next C). The sharp ($\sharp$, "raised a semitone") and flat ($\flat$, "lowered a semitone") symbols mean the *same pitch* approached from different directions: C♯ and D♭ are the same key on a piano. Which name is used depends on musical context.

### How to pronounce them

The letters are spoken with their ordinary English alphabet sound, and the accidentals as English words:

| symbol | pronunciation |
|---|---|
| C | "see" |
| C♯ / D♭ | "see sharp" / "dee flat" |
| D | "dee" |
| D♯ / E♭ | "dee sharp" / "ee flat" |
| E | "ee" |
| F | "eff" |
| F♯ / G♭ | "eff sharp" / "gee flat" |
| G | "gee" (hard *g*, as in *geese*) |
| G♯ / A♭ | "gee sharp" / "ay flat" |
| A | "ay" |
| A♯ / B♭ | "ay sharp" / "bee flat" |
| B | "bee" |

In some European traditions you may hear solfège syllables instead: **Do, Re, Mi, Fa, Sol, La, Si** for C, D, E, F, G, A, B.

## Pitch vs note

- **Pitch**: a continuous physical quantity (frequency in Hz).
- **Note**: a discrete label assigned to a frequency band, the bin being one semitone wide.

Two signals with slightly different frequencies are labelled the same note if they fall within the same bin.

## Harmonics and timbre

When you play a note, you do *not* get a single frequency. You get a **fundamental** at $f$ plus **overtones** at integer multiples $2f, 3f, 4f, \dots$ — directly analogous to a fundamental mode plus higher modes in elastic wave propagation. The relative amplitudes of these overtones determine **timbre** (tone colour), not pitch. A violin and a flute playing the same note differ mainly in their harmonic amplitude spectra.

## Intervals

An **interval** is the ratio between two frequencies. The named ones with simple integer ratios are the consonant ones:

| interval | ratio |
|---|---|
| octave | 2:1 |
| perfect fifth | 3:2 |
| perfect fourth | 4:3 |
| major third | 5:4 |
| minor third | 6:5 |

Consonance roughly correlates with simple ratios because the harmonics of the two notes align rather than beat against each other.

## Scale, key, and tonic — the three concepts disentangled

These three are often confused. The cleanest way to see them:

- A **scale** is a *pattern* of intervals within one octave. It is a template, expressed in semitone steps. The **major scale** is the pattern $2, 2, 1, 2, 2, 2, 1$. The **natural minor scale** is $2, 1, 2, 2, 1, 2, 2$. The pattern does not fix any actual frequency.
- A **tonic** is a specific starting pitch — a *reference note* in absolute frequency space.
- A **key** is the combination of a scale (the pattern) anchored to a tonic (the starting pitch). "C major" means the major-scale pattern starting on C; "G major" is the same pattern starting on G.

Seismological analogy:

- Scale = the *shape* of a band-pass filter expressed in log-frequency units.
- Tonic = the centre frequency where you place that filter.
- Key = the filter instantiated at a specific centre frequency.

Concretely, the C major scale gives the notes C D E F G A B C — all white keys on a piano, no sharps or flats. The G major scale gives G A B C D E F♯ G — same interval pattern, shifted up. The "F♯" appears not because G major is fancier, but because the major pattern from G lands on F♯, not F.

A piece "in G major" uses these seven notes as its primary palette and treats G as the gravitational centre (see below).

## Chords

A **chord** is multiple notes played simultaneously — the superposition of discrete spectral peaks with stable phase relationships. The basic three-note chords (triads) are:

- **Major triad**: root, major third, perfect fifth (ratios ≈ $1 : 5/4 : 3/2$).
- **Minor triad**: root, minor third, perfect fifth (ratios ≈ $1 : 6/5 : 3/2$).

The difference between major and minor is one semitone in the middle note. Perceptually it is the difference between bright and sombre.

## Tonality and resolution

Western tonal music is hierarchical: one pitch (the tonic) acts as a gravitational centre, and other notes create tension that tends to resolve back to it. This is perceptual, not physical, but it is strongly reinforced by harmonic relationships and a lifetime of exposure. Think of it as a potential well in pitch space.

## Summary in seismological terms

| music | seismology |
|---|---|
| pitch | frequency |
| note | frequency bin / label |
| octave | scale invariance (×2 frequency) |
| interval | frequency ratio |
| timbre | spectral shape |
| chord | superposition of modes |
| scale | filter shape in log-frequency |
| tonic | filter centre frequency |
| key | filter instantiated at a centre |
| tonality | a potential well in pitch space |
