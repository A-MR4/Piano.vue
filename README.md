# Piano.vue

a visual, polyphonic piano keyboard that runs in the browser.

![Image of the piano in browser](.\piano-browser\imgs\CapPiano.png)

## Summary
- Renders white and black keys in a responsive interface.
- Produces sound with **sawtooth** oscillators and allows control of attack and release.
- Supports input via **mouse**, **touch**, and **physical keyboard**.
- Octave modifiers: **comma ( , )** lowers one octave; **Shift** raises one octave.

## Audio behavior (technical summary)
- **Audio context**: an `AudioContext` is created on demand with `ensureAudioContext()`.
- **Voice**: each note creates an `OscillatorNode` of type `sawtooth` and a `GainNode`.
  ![Sawtooth wave](.\piano-browser\imgs\STW.webp)
- **Simplified ADSR**: the code applies a short attack and a release ramp to avoid clicks. Parameters in the code are:
  - `ATTACK = 0.005`
  - `SUSTAIN = 0.95`
  - `RELEASE = 0.12`
  - `MIN_GAIN = 0.0001`
- **Polyphony**: `activeVoices` is a `Map` that stores lists of voices by `keyId` (a key that includes octave modifiers).
- **Safe retrigger**: before creating a new voice for the same base key, the code releases (release + stop) any previous voice associated with that base (`forceReleaseAndRemoveByBase`), avoiding unwanted overlaps.

## Modifiers and octave mapping
- **Comma ( , )**: holding the `,` key sets the `commaDown` flag; notes are played at half the frequency (one octave down).
- **Shift**: when `e.shiftKey` is active, notes are played at double the frequency (one octave up).
- The function `makeKeyId(keyChar, comma, shift)` distinguishes the same key under different modifier states to manage separate voices.

## Interaction and visual state
- **Interaction events**: `mousedown`, `mouseup`, `mouseleave`, `touchstart`, `touchend` on each key call `startKey` / `stopKey`.
- **Visual state**: `pressedVisual` (a reactive object) marks which keys show the `pressed` style. The functions `setPressedVisual` and `isPressedVisual` manage that state.
- **Retrigger prevention**: `pressedKeyIds` (a `Set`) prevents starting the same `keyId` multiple times without releasing it first.

## Notes, frequencies and oscillators
- **Note**: musical name (e.g., `A4`, `C4`) corresponds to a frequency in Hz. In the code each note has a fixed frequency (e.g., `A4 = 440.00 Hz`).
- **Frequency**: doubling the frequency raises one octave; halving it lowers one octave.
- **Oscillator**: generates the basic waveform. The component uses `sawtooth`, which produces a timbre rich in harmonics, suitable for synthetic demos.
- **GainNode and envelope**: the `GainNode` controls amplitude; attack and release ramps on the gain node prevent clicks and make the sound smoother.
