# Morse Code Decoder — DJ0CU / G4ADF

A browser-based CW decoder. It listens on the sound card, shows a waterfall for
tuning, and prints decoded text. One self-contained `index.html` file, no
install, no libraries.

The timing rules follow N. Kyriazis, "Morse decoding — a machine-code program
for decoding Morse transmissions on a home computer", *Wireless World*,
February 1981, pp. 44–46, with the speed tracking rebuilt (see below). The
issue is scanned at
[worldradiohistory.com](https://worldradiohistory.com/UK/Wireless-World/80s/Wireless-World-1981-02.pdf).

## Running it

Open `index.html` and press **Start**, then allow microphone access when
prompted.
## Controls

| Control | What it does |
| --- | --- |
| **Start / Stop** | Opens and closes the microphone. |
| **Auto tune** | Jumps to the strongest tone in 300–1200 Hz. Press it while the signal is keying, not during a gap. |
| **Reset speed** | Returns to the Start speed and clears the timing history. |
| **Tone** | Detector centre frequency. Clicking the waterfall does the same. |
| **Sensitivity** | Where the decision level sits between key-up and key-down, and how far the tone must stand above the noise to count as keyed at all. Higher is more sensitive; lower rejects more noise. Default 60%. |
| **Start speed** | Where the decoder begins before it has locked. Default 21 WPM. |
| **Follow the tone** | Tracks drift, up to ±70 Hz from the slider setting. |
| **Fast speed lock** | Uses the template fit below. Off reverts to the 1981 adaptation rule. |
| **Hold output until locked** | Suppresses characters while the speed is unknown — before the first lock, and again if lock is lost. |
| **Auto sensitivity** | Adjusts the sensitivity by itself: a slow sweep while unlocked, then hill climbing on the fit error once locked. Moving the slider by hand hands control back. |
| **Buffered output** | Holds text ten characters behind, so a lost-sync verdict can withdraw the characters that caused it. Caught up after two seconds of silence, or the moment the box is unticked. Off prints each character as it is decoded. |

The readouts show the current speed and unit length, the tone-to-neighbour
ratio in dB, and the fitted speed with its **jitter**.

Jitter is how far the measured marks and gaps stray from the ideal 1 : 3 : 7
pattern, averaged over the recent elements and given as a percentage of each
element's own length. It measures the *consistency* of the timings, not
whether the decoded letters are correct — there is no way to know what was
actually sent. Under 25% is clean (shown green), 25–45% is workable (amber),
above 45% the decoder stops trusting the fit and holds output (red).

It is the most useful figure for diagnosis. Ragged text with low jitter means
the speed tracking is fine and the trouble is in the detector — tuning or
sensitivity. Rising jitter means the element timings themselves are being
mangled, which points at the signal.

## How it works

### Tone detection

A Goertzel filter at the tuned frequency, over a 512-sample window (`ENV_N`),
gives the envelope. The same filter is run ±280 Hz either side to give a
reference level; the tone is only considered present when it stands well clear
of that reference. This is the software equivalent of the NE567 tone decoder
the 1981 article assumed ahead of the computer.

The envelope is smoothed with a time constant that scales with the sending
speed, then compared against a decision level sitting part way between the
running key-up and key-down levels. Five samples at 4 ms are majority-voted
before a state change is accepted, as in the original.

### Element timing

Straight from the article's flow chart:

- a mark over two units is a dah, under two units a dit;
- a space under 1.5 units is an inter-element space, under 4 a character
  space, 4 or more a word space;
- a mark or space shorter than half a unit is interference. Its length is
  **added to the other period** rather than discarded, so a click in the
  middle of a dah does not split it into two dits;
- a mark longer than eight units abandons the character and resets the speed,
  which is what recovers the decoder from a carrier or a calibrator.

### Speed tracking

This is the one part that departs from the article. The original adapts UNIT
only from inter-element spaces, which is one-sided: if the sender is slower
than UNIT, every gap measures more than 1.5 units, is read as a character
space, and never feeds the adaptation. The decoder can speed up but cannot
slow down.

Instead, the last 36 marks and 36 gaps are fitted against the whole 1 : 3 : 7
pattern, scoring candidate speeds from 8 to 45 WPM. The best-scoring speed is
taken, so correction works in both directions and does not depend on the
current UNIT being right.

A decaying tone — room acoustics, a slow AGC, a soft keyed transmitter —
releases the detector late, stretching every mark and clipping every gap by
about the same amount. That offset is fitted as a second parameter and removed
from the decision thresholds.

Lock is a live judgement, not a one-off. It is dropped when the jitter
rises above 45%, when the fit lands on either end of the speed range, after
five consecutive single-symbol characters (E T I M S O H 5 0 or `*`), or when
two lone `TT` words appear inside the last 24 characters. That last test came
from on-air use: `TT` inside a word is ordinary — LETTER, BETTER, ATTACK all
decode that way — but `TT` standing alone as a word, twice, means long
elements are being chopped into pairs of T.

Because output is held back, those verdicts trim the text as well as stopping
it: the held characters are cut back to the first offending one, so the
garbage never reaches the screen.

## Tested behaviour

Driven end to end with synthetic tone envelopes (10 ms decay tail, 8% noise
floor), starting from 21 WPM:

    sent 10 wpm -> "M CQ DE DJ0CU TEST"
    sent 12 wpm -> "A CQ DE DJ0CU TEST"
    sent 18 wpm -> "Q CQ DE DJ0CU TEST"
    sent 21 wpm -> "Q CQ DE DJ0CU TEST"
    sent 27 wpm -> "Q CQ DE DJ0CU TEST"
    sent 35 wpm -> "V CQ DE DJ0CU TEST"
    sent 40 wpm -> "Q DE DJ0CU TEST"

One character is lost while the fit commits. With 8% timing jitter and a click
splitting one mark in twelve, 22 to 35 WPM still decode cleanly. Tested on air
against hand-sent and machine-sent Morse from 10 to 45 WPM.

## Known limits

- **Decay tails.** Usable up to about a 20 ms tail. Beyond 30 ms the gaps are
  smeared away before the detector sees them, and no threshold setting
  recovers them. Move the source closer and turn its volume down.
- **Speed changes mid-transmission.** The fit buffer holds a mixture of old
  and new elements until the new ones outnumber them: about a dozen ragged
  characters after a large change.
- **QSB.** The envelope trackers work on absolute levels with a fixed 0.9 s
  time constant, so a fast fade drags the decision level. Tracking mark and
  space levels separately in dB, and putting the threshold at their geometric
  mean, would make this scale-invariant.
- **Crowded bands.** The reference is the mean of two offsets; if another
  signal lands on one of them, the presence gate is lifted. A median of four
  or six offsets would ignore the occupied one. The detection bandwidth is
  about 94 Hz at `ENV_N` = 512, wider than CW needs — choosing the window
  length from the locked speed would sharpen adjacent-signal rejection.
- The analysis window and the vote length are fractions of a dit rather than
  fixed times, so they narrow at speed. Tested to 45 WPM, the top of the fit
  range; above that the 4 ms poll starts to dominate and sample-accurate edge
  timing would need an AudioWorklet.

## Tuning constants

Near the top of the script:

| Constant | Default | Meaning |
| --- | --- | --- |
| `ENV_N` | 512 | Envelope buffer, samples. The window actually used is `WIN_FRAC` of a dit, up to this. |
| `WIN_FRAC` | 0.20 | Analysis window as a fraction of a dit. |
| `POLL_MS` | 4 | Detector sampling interval. |
| `VOTE_FRAC` / `VOTES_MAX` | 0.30 / 5 | Vote length as a fraction of a dit, and its cap. |
| `FIT_LEN` | 36 | Elements held for the speed fit. |
| `MIN_UNIT` / `MAX_UNIT` | 45 / 8 WPM | Limits of the speed fit. |
