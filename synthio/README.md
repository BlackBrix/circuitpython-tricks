

Synthio Tricks
===============

<!--ts-->
   * [What is synthio?](#what-is-synthio)
   * [Getting started](#getting-started)
      * [Get Audio out working](#get-audio-out-working)
      * [Play a note every second](#play-a-note-every-second)
      * [Play a chord](#play-a-chord)
      * [USB MIDI Input](#usb-midi-input)
      * [Serial MIDI Input](#serial-midi-input)
      * [Using AudioMixer for adjustable volume &amp; fewer glitches](#using-audiomixer-for-adjustable-volume--fewer-glitches)
   * [Basic Synth Techniques](#basic-synth-techniques)
      * [Amplitude envelopes](#amplitude-envelopes)
         * [Envelope for entire synth](#envelope-for-entire-synth)
         * [Per-note velocity envelopes with synthio.Note](#per-note-velocity-envelopes-with-synthionote)
      * [LFOs: vibrato, tremolo, and more](#lfos-vibrato-tremolo-and-more)
         * [Vibrato: pitch bend with LFO](#vibrato-pitch-bend-with-lfo)
         * [Tremolo: volume change with LFO](#tremolo-volume-change-with-lfo)
   * [Advanced Techniques](#advanced-techniques)
      * [Keeping track of pressed notes](#keeping-track-of-pressed-notes)
      * [Detuning oscillators for fatter sound](#detuning-oscillators-for-fatter-sound)
      * [Using LFO values in your own code](#using-lfo-values-in-your-own-code)
      * [Using synthio.Math with synthio.LFO](#using-synthiomath-with-synthiolfo)
      * [Wavetable mixing](#wavetable-mixing)
      * [Loading WAV files into synthio](#loading-wav-files-into-synthio)

<!-- Added by: tod, at: Mon May 29 18:01:47 PDT 2023 -->

<!--te-->

## What is `synthio`?

- CircuitPython core library available since 8.2.0-beta0 and still in development!
- Features:
  - Polyphonic (11 to 24? oscillator) & stereo, 16-bit, with adjustable sample rate
  - Oscillators are wavetable-based, wtih real-time adjustable wavetables
  - ADSR amplitude envelope per oscillator
  - Oscillator ring modulation w/ customizable ring oscillator wavetable
  - Extensive LFO system
    - multiple LFOs per oscillator (amplitude, panning, pitch bend, ring mod)
    - LFOs can repeat or run once (becoming a kind of envelope)
    - Each LFO can have a custom wavetable with linear interpolation
    - LFO outputs can be used by user code
    - LFOs can plug into one another
  - LFOs have customized wavetables and can be applied to your own code
  - Math blocks to adjust LFO ranges, offsets, scales
  - Utility functions to easily convert from MIDI or V/Oct modular to frequency
  - Plugs into existing `AudioMixer` system for use alongside `audiocore.WaveFile` sample playing

## Getting started

###  Get Audio out working
  - Example circuits:
    - Pico w/ RC filter and `audiopwmio.PWMAudioOut`
    (R1=1k, C1=100n, [Sparkfun TRRS](https://www.sparkfun.com/products/11570))
    <img src="./imgs/synthio_pico_pwm_bb.jpg" width=500>

    - Pico w/ [I2S PCM5102](https://amzn.to/3MGOTJH) and `audiobusio.I2SOut`
    <img src="./imgs/synthio_pico_i2s_bb.jpg" width=500>

### Play a note every second

Use this to test that you can actually hear what `synthio` is doing.
This a single mono PWM (~10-bit resolution) audio out with an RC-filter

```py
import board, time
import synthio

# for PWM audio with an RC filter
import audiopwmio
audio = audiopwmio.PWMAudioOut(board.GP10)
# for I2S audio with external I2S DAC board
#import audiobusio
#audio = audiobusio.I2SOut(bit_clock=board.GP11, word_select=board.GP12, data=board.GP10)

synth = synthio.Synthesizer(sample_rate=22050)
audio.play(synth)

while True:
    synth.press(65) # midi note 65 = F4
    time.sleep(0.5)
    synth.release(65) # release the note we pressed
    time.sleep(0.5)
```

Note we'll be assuming PWMAudioOut in the examples below, but if you're using an I2S DAC instead,
the `audio` line would look like the commented out part above. The particular choices for the three
signals depends on the chip, on RP2040-based boards like the Pico,
[many pin combos are available](https://learn.adafruit.com/adafruit-i2s-stereo-decoder-uda1334a/circuitpython-wiring-test#wheres-my-i2s-2995476))

### Play a chord

We can send a list of [MIDI note numbers]() to be "pressed" and "released" to turn a note on and off

```py
import board, time
import audiopwmio
import synthio

audio = audiopwmio.PWMAudioOut(board.GP10)
synth = synthio.Synthesizer(sample_rate=22050)
audio.play(synth)

while True:
  synth.press( (65,70,72) ) # midi notes 65,70,72  = F4, A4#, C5
  time.sleep(0.5)
  synth.release( (65,70,72) )
  time.sleep(0.5)
```


### USB MIDI Input

How about a MIDI synth in 20 lines of CircuitPython?

(To use with a USB MIDI keyboard, plug both the keyboard & CirPy device into a computer,
and on the computer run a DAW like Ardour, LMMS, Ableton Live, etc,
to forward MIDI from keyboard to CirPy)

```py
import board
import audiopwmio
import synthio
import usb_midi
import adafruit_midi
from adafruit_midi.note_on import NoteOn
from adafruit_midi.note_off import NoteOff

audio = audiopwmio.PWMAudioOut(board.GP10)
synth = synthio.Synthesizer(sample_rate=22050)
audio.play(synth)

midi = adafruit_midi.MIDI(midi_in=usb_midi.ports[0], in_channel=0 )

while True:
    msg = midi.receive()
    if isinstance(msg, NoteOn) and msg.velocity != 0:
        print("noteOn: ", msg.note, "vel:", msg.velocity)
        synth.press( msg.note )
    elif isinstance(msg,NoteOff) or isinstance(msg,NoteOn) and msg.velocity==0:
        print("noteOff:", msg.note, "vel:", msg.velocity)
        synth.release( msg.note )

```

### Serial MIDI Input

The same as above, but replace the `usb_midi` with a `busio.UART`

```py
# ... as before
import busio
uart = busio.UART(tx=board.TX, rx=board.RX, timeout=0.001)
midi = adafruit_midi.MIDI(midi_in=uart, in_channel=0 )
while True:
  msg = midi.receive()
  # ... as before
```

### Using AudioMixer for adjustable volume & fewer glitches

Stick an AudioMixer in between `audio` and `synth` and we get three benefits:
- Volume control over the entire synth
- Can plug other players (like `WaveFile`) to play samples simultaneously
- An audio buffer that helps eliminate glitches from other I/O

```py
import audiomixer
audio = audiopwmio.PWMAudioOut(board.GP10)
mixer = audiomixer.Mixer(sample_rate=22050, buffer_size=2048)
synth = synthio.Synthesizer(sample_rate=22050)
audio.play(mixer)
mixer.voice[0].play(synth)
mixer.voice[0].level = 0.25  # 25% volume might be better
```

Setting the AudioMixer `buffer_size` argument is handy for reducing giltches that happen when the chip is
doing other work like updating a display reading I2C sensors. Increase the buffer to eliminate glitches
but it does increase latency.

## Basic Synth Techniques

Assuming the setup above where we have a `synth` object and we can hear the output on

### Amplitude envelopes

The amplitude envelope describes how a sound's loudness changes over time.
In synthesizers, [ADSR envelopes](https://en.wikipedia.org/wiki/Envelope_(music))
are used to describe that change. In `synthio`, you get the standard ADSR parameters,
and a default fast attack, max sustain level, fast release envelope.

#### Envelope for entire synth

To create your own envelope with a slower attack and release time, and apply it to every note:


```py
import board, time, audiopwmio, synthio
audio = audiopwmio.PWMAudioOut(board.GP10)
synth = synthio.Synthesizer(sample_rate=22050)
audio.play(synth)

amp_env_slow = synthio.Envelope(attack_time=0.2, release_time=0.8, sustain_level=1.0)
amp_env_fast = synthio.Envelope(attack_time=0.01, release_time=0.2, sustain_level=0.5)
synth.envelope = amp_env_slow  # could also set in synth constructor

while True:
  synth.press(65) # midi note 65 = F4
  time.sleep(0.5)
  synth.release(65)
  time.sleep(1.0)
  synth.envelope = amp_env_fast
  synth.press(65)
  time.sleep(0.5)
  synth.release(65)
  time.sleep(1.0)
  synth.envelope = amp_env_slow
```

#### Per-note velocity envelopes with `synthio.Note`

To give you more control over each oscillator, `synthio.Note` lets you override
the default envelope and waveform of your `synth` with per-note versions.
For instance, you can create a new envelope based on incoming MIDI note velocity to
make a more expressive instrument. You will have to convert MIDI notes to frequency by hand,
but synthio provides a helper for that.

```py
import board, time, audiopwmio, synthio, random
audio = audiopwmio.PWMAudioOut(board.GP10)
synth = synthio.Synthesizer(sample_rate=22050)
audio.play(synth)

while True:
    midi_note = 65
    velocity = random.choice( (1, 0.1, 0.5) )  # 3 different fake velocity values 0.0-1.0
    print("vel:",velocity)
    amp_env = synthio.Envelope( attack_time=0.1 + 0.6*(1-velocity),  # high velocity, short attack
                                release_time=0.1 + 0.9*(1-velocity) ) # low velocity, long release
    note = synthio.Note( synthio.midi_to_hz(midi_note), envelope=amp_env )
    synth.press(note) # press with note object
    time.sleep(0.5)
    synth.release(note) # must release with same note object
    time.sleep(2.0)
```

The choice of how you scale velocity to attack times, sustain levels and so on,
is dependent on your application.

For an example of how to use this with MIDI velocity,
see [synthio_midi_synth.py](https://gist.github.com/todbot/96a654c5fa27625147d65c45c8bfd47b)


### LFOs: vibrato, tremolo, and more

LFOs (Low-Frequency Oscillators) were named back when it was very different
to build an audio-rate oscillator vs an oscillator that changed over a few seconds.
In synthesis, LFOs are often used to "automate" the knob twiddling one would do to perform an instrument.
`synthio.LFO` is a flexible LFO system that can perform just about any kind of
automated twiddling you can imagine.

The waveforms for `synthio.LFO` can be any waveform (even the same waveforms used for oscillators),
and the default waveform is a sine wave.

#### Vibrato: pitch bend with LFO

Here we create an LFO with a rate of 5 Hz and amplitude of 0.5% max.
For each note, we apply that LFO to the note's `bend` property to create vibrato.

If you'd like the LFO to start over on each note on, do `lfo_vibra.retrigger()`.

```py
import board, time, audiopwmio, synthio, random
audio = audiopwmio.PWMAudioOut(board.GP10)
synth = synthio.Synthesizer(sample_rate=22050)
audio.play(synth)

lfo_vibra = synthio.LFO(rate=5, scale=0.05)  # 5 Hz lfo at 0.5%

while True:
    midi_note = 65
    note = synthio.Note( synthio.midi_to_hz(midi_note), bend=lfo_vibra )
    synth.press(note)
    time.sleep(1.0)
    synth.release(note)
    time.sleep(1.0)

```

#### Tremolo: volume change with LFO

Similarly, we can create rhythmic changes in loudness with an LFO attached to `note.amplitude`.
And since each note can get their own LFO, you can make little "songs" with just a few notes

```py
lfo_trema1 = synthio.LFO(rate=3)  # 3 Hz for fastest note
lfo_trema2 = synthio.LFO(rate=2)  # 2 Hz for middle note
lfo_trema3 = synthio.LFO(rate=1)  # 1 Hz for lower note
lfo_trema4 = synthio.LFO(rate=0.75) # 0.75 Hz for lowest bass note

midi_note = 65
note1 = synthio.Note( synthio.midi_to_hz(midi_note), amplitude=lfo_trema1)
note2 = synthio.Note( synthio.midi_to_hz(midi_note-7), amplitude=lfo_trema2)
note3 = synthio.Note( synthio.midi_to_hz(midi_note-12), amplitude=lfo_trema3)
note4 = synthio.Note( synthio.midi_to_hz(midi_note-24), amplitude=lfo_trema4)
synth.press( (note1, note2, note3, note4) )

while True:
    print("hi, we're just groovin")
    time.sleep(1)
```


## Advanced Techniques

### Keeping track of pressed notes

### Detuning oscillators for fatter sound

### Using LFO values in your own code

### Using `synthio.Math` with `synthio.LFO`

### Wavetable mixing

### Loading WAV files into synthio