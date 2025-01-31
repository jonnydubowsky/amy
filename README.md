# AMY - the Additive Music synthesizer librarY

AMY is a fast, small and accurate music synthesizer library written in C with Python and Arduino bindings that deals with combinations of many oscillators very well. It can easily be embedded into almost any program, architecture or microcontroller. We've run AMY on Mac, Linux, ESP32 and ESP32S3, Teensy 3.6, Teensy 4.1, the Raspberry Pi, the Pi Pico RP2040, iOS devices, and more to come. It is highly optimized for polyphony and poly-timbral operation on even the lowest power and constrained RAM microcontroller but can scale to as many cores as you want. 

It can be used as a very good analog-type synthesizer (Juno-6 style) a FM synthesizer (DX7 style), a partial breakpoint synthesizer (Alles machine or Atari AMY), a drum machine (PCM samples included), or as a lower level toolkit to make your own combinations of oscillators, filters, LFOs and effects. 

AMY powers the multi-speaker mesh synthesizer [Alles](https://github.com/shorepine/alles), as well as the [Tulip Creative Computer](https://github.com/shorepine/tulipcc). Let us know if you use AMY for your own projects and we'll add it here!

AMY was built by [DAn Ellis](https://research.google/people/DanEllis/) and [Brian Whitman](https://notes.variogram.com), and would love your contributions.

[![shore pine sound systems discord](https://raw.githubusercontent.com/shorepine/tulipcc/main/docs/pics/shorepine100.png) **Chat about AMY on our Discord!**](https://discord.gg/TzBFkUb8pG)


It supports

 * An arbitrary number (compile-time option) of band-limited oscillators, each with adjustable frequency and amplitude:
   * pulse (+ adjustable duty cycle)
   * sine
   * saw (up and down)
   * triangle
   * noise
   * PCM, reading from a baked-in buffer of percussive and misc samples
   * karplus-strong string with adjustable feedback 
   * An operator / algorithm-based frequency modulation (FM) synth
 * Biquad low-pass, bandpass or hi-pass filters with cutoff and resonance, can be assigned to any oscillator
 * Reverb and chorus effects, set globally
 * Stereo pan or mono operation 
 * An additive partial synthesizer with an analysis front end to play back long strings of breakpoint-based sine waves
 * Oscillators can be specified by frequency in floating point or midi note 
 * Each oscillator has 2 envelope generators, which can modify any combination of amplitude, frequency, PWM duty, filter cutoff, or pan over time
 * Each oscillator can also act as an modulator to modify any combination of parameters of another oscillator, for example, a bass drum can be indicated via a half phase sine wave at 0.25Hz modulating the frequency of another sine wave. 
 * Control of overall gain and 3-band EQ
 * Built in patches for PCM, DX7, Juno and partials
 * A front end for Juno-6 patches and conversion setup commands 
 * Built-in clock for short term sequencing of events
 * Can use multi-core (including microcontrollers) for rendering if available

The FM synth provides a Python library, [`fm.py`](https://github.com/shorepine/amy/blob/main/fm.py) that can convert any DX7 patch into AMY setup commands, and also a pure-Python implementation of the AMY FM synthesizer in [`dx7_simulator.py`](https://github.com/shorepine/amy/blob/main/dx7_simulator.py).

The partial tone synthesizer provides [`partials.py`](https://github.com/shorepine/amy/blob/main/partials.py), where you can model the partials of any arbitrary audio into AMY setup commands for live partial playback of hundreds of oscillators.

The Juno-6 emulation is in [`juno.py`](https://github.com/shorepine/amy/blob/main/juno.py) and can read in Juno-6 SYSEX patches and convert them into AMY commands and generate patches.


## Using AMY in Arduino

Copy this repository to your `Arduino/libraries` folder as `Arduino/libraries/amy`, and `#include <AMY-Arduino.h>`. There are examples for the Pi Pico, ESP32 (and variants), and Teensy (works on 4.X and 3.6) Use the File->Examples->AMY Synthesizer menu to find them. 

The examples rely on the following board packages and libraries:
 * RP2040 / Pi Pico: [`arduino-pico`](https://arduino-pico.readthedocs.io/en/latest/install.html#installing-via-arduino-boards-manager)
 * Teensy: [`teensyduino`](https://www.pjrc.com/teensy/td_download.html)
 * ESP32/ESP32-S3/etc: [`arduino-esp32`](https://espressif-docs.readthedocs-hosted.com/projects/arduino-esp32/en/latest/installing.html) - use a 2.0.14+ version when installing
 * The USB MIDI example requires the [MIDI Library](https://www.arduino.cc/reference/en/libraries/midi-library/)

You can use both cores of supported chips (RP2040 or ESP32) for more oscillators and voices. We provide Arduino examples for the Arduino ESP32 in multicore, and a `pico-sdk` example for the RP2040 that renders in multicore. If you really want to push the chips to the limit, we recommend using native C code using the `pico-sdk`  or `ESP-IDF`. You can use [Alles](https://github.com/shorepine/alles) for a `ESP-IDF` example.

## Using AMY in Python on any platform

You can `import amy` in Python and have it render either out to your speakers or to a buffer of samples you can process on your own. To install the `libamy` library, run `cd src; pip install .`. You can also run `make test` to install the library and run a series of tests.

## Using AMY in any other software

To use AMY in your own software, simply copy the .c and .h files in `src` to your program and compile them. No other libraries should be required to synthesize audio in AMY. You'll want to make sure the configuration in `amy_config.h` is set up for your application / hardware. 

To run a simple C example on many platforms:

```
make
./amy-example # you should hear tones out your default speaker, use ./amy-example -h for options
```

# Using AMY

AMY can be controlled using its wire protocol or by fillng its data structures directly. It's up to what's easier for you and your application. 

In Python, rendering to a buffer of samples, using the high level API:

```python
>>> import amy
>>> m = amy.message(voices="0",load_patch=130,note=50,vel=1)
>>> print(m) # Show the wire protocol message
't76555951v0w8n50p30l1Z'
>>> amy.send_raw(m)
>>> audio = amy.render(5.0)
```

You can also start a thread playing live audio:

```python
>>> import amy
>>> amy.live() # can optinally pass in audio device ID, amy.live(2) 
>>> m = amy.send(voices="0",load_patch=130,note=50,vel=1)
>>> amy.stop()
```


In C, using the high level structures directly;
  
```c
#include "amy.h"
void bleep() {
    struct event e = amy_default_event();
    int32_t sysclock = amy_sysclock();
    e.time = sysclock;
    e.wave = SINE;
    e.freq_coefs[COEF_CONST] = 220;
    e.velocity = 1;
    amy_add_event(e);
    e.time = sysclock + 150;
    e.freq_coefs[COEF_CONST] = 440;
    amy_add_event(e);
    e.time = sysclock + 300;
    e.velocity = 0;
    e.amp = 0;
    e.freq_coefs[COEF_CONST]=0;
    amy_add_event(e);
}

void main() {
    amy_start(/* cores= */ 1, /* reverb= */ 0, /* chorus= */ 0); // initializes amy 
    amy_live_start(); // render live audio
    bleep();
}
```

Or in C, sending the wire protocol directly:

```c
#include "amy.h"

void main() {
    amy_start(/* cores= */ 1, /* reverb= */ 0, /* chorus= */ 0);
    amy_live_start();
    amy_play_message("t76555951v0w8n50p30l1Z");
}
```

If you want to receive buffers of samples, or have more control over the rendering pipeline to support multi-core, instead of using `amy_live_start()`:

```c
#include "amy.h"
...
amy_start(/* cores= */ 2, /* reverb= */ 1, /* chorus= */ 1);
...
... {
    // For each sample block:
    amy_prepare(); // prepare to render this block
    amy_render(0, OSCS/2, 0); // render oscillators 0 - OSCS/2 on core 0
    // on the other core... 
    amy_render(OSCS/2, OSCS, 1); // render oscillators OSCS/2-OSCS on core 1
    // when they are both done..
    int16_t * samples = amy_fill_buffer();
    // do what you want with samples
... }

```

On storage connstrained devices, you may want to limit the amount of PCM samples we ship with AMY. To do this, include a smaller set after including `amy.h`, like:

```c
#include "amy.h"
#include "pcm_tiny.h" 
// or, #include "pcm_small.h"
```

## Voices and patches (DX7, Juno-6, custom) support

With AMY, you can control the low level oscillators that make up a synthesizer "voice", or you can control voices directly and load in groups of oscillators by sending AMY a patch. A patch is a list of AMY commands that setup one or more oscillators. 

A voice in AMY is a collection of oscillators. You can assign patches to any voice number, or set up mulitple voices to have the same patch (for example, a polyphonic synth), and AMY will allocate the oscillators it needs under the hood. You can then play those patches (and modify them) by their voice number. For example, a multitimbral Juno/DX7 synth can be set up like:

```python
amy.send(voices="0,1,2,3", load_patch=1) # juno patch #1 on voice 0-3
amy.send(voices="4,5,6,7", load_patch=129) # DX7 patch #2 on voices 4-7
amy.send(voices="0", note=60, vel=1) # Play note 60 on voice 0
amy.send(voices="0", osc=1, filter_freq="440,0,0,0,5") # adjust the filter on the juno voice (its second oscillator)
```

Our code in `amy_headers.py` generates and bakes in these patches into AMY so they're ready for playback on any device. You can add your own patches by "recording" AMY setup commands and adding them to `patches.h`.

You can also create your own patches at runtime and use them for voices using `store_patch`=`1024,AMY_PATCH_STRING` where 1024 is a patch number from 1024-1055. This message must be the only thing in the string sent over. AMY will treat the rest of the message as a patch, not further messages.

So you can do:
```
>>> import amy; amy.live()
>>> amy.send(store_patch="1024,v0S0Zv0S1Zv1w0f0.25P0.5a0.5Zv0w0f261.63,1,0,0,0,1A0,1,500,0,0,0L1Z")
>>> amy.send(voices="0",load_patch=1024)
>>> amy.send(voices='0',vel=2,note=50)
```
We divine the number of oscs used for the patch at store_patch time. If you store a new patch over an old one, that old memory is freed and re-allocated. We rely on malloc for all of this.

Also recall you can "record" patches in amy.py, so the whole loop is:
```
>>> amy.log_patch()
>>> amy.preset(5)
>>> bass_drum = amy.retrieve_patch()
>>> bass_drum
'v0S0Zv0S1Zv1w0f0.25P0.5a0.5Zv0w0f261.63,1,0,0,0,1A0,1,500,0,0,0L1Z'
>>> amy.send(store_patch="1024,"+bass_drum)
```

**Note on patches and AMY timing**: If you're using AMY's time scheduler (see below) note that unlike all other AMY commands, allocating new voices from patches (using `load_patch`) will happen once AMY receives the message, not using any advance clock (`time`) you may have set. This default is the right decision for almost all use cases of AMY, but if you do need to be able to "schedule" voice allocations within the short term scheduling window, you can load patches by sending the patch string directly to AMY using the timer, and managing your own oscillator mapping in your code.


# Wire protocol

AMY's wire protocol is a series of numbers delimited by ascii characters that define all possible parameters of an oscillator. This is a design decision intended to make using AMY from any sort of environment as easy as possible, with no data structure or parsing overhead on the client. It's also readable and compact, far more expressive than MIDI and can be sent over network links, UARTs, or as arguments to functions or commands. We've used AMY over multicast UDP, over Javascript, in Max/MSP, in Python, C, Micropython and many more! 

AMY accepts commands in ASCII, like so:

```
v0w4f440.0l0.9
```

Here's the full list:

| Code | Python | Type-range  | Notes                                 |
| ---- | ------ | ----------  | ------------------------------------- |
| a    | amp    | float[,float...]  | Control the amplitude of a note; a set of ControlCoefficients. Default is 0,0,1,1  (i.e. the amplitude comes from the note velocity multiplied by Envelope Generator 0.) |
| A    | bp0    | string      | in commas, like 100,0.5,50,0.25,200,0 -- Envelope Generator 0's breakpoint pairs of time(ms) and level. The last pair triggers on note off (release) |
| B    | bp1    | string      | breakpoints for Envelope Generator 1. See bp0 |
| b    | feedback | float 0-1 | use for the ALGO synthesis type in FM or for karplus-strong, or to indicate PCM looping (0 off, >0, on) |
| c    | chained_osc |  uint 0 to OSCS-1 | Chained oscillator.  Note/velocity events to this oscillator will propagate to chained oscillators.  VCF is run only for first osc in chain, but applies to all oscs in chain. |
| C    | clone_osc | uint 0 to OSCS-1 | Clone oscillator.  Most parameters from the named other oscillator are copied into this one. |  
| d    | duty   |  float[,float...] | duty cycle for pulse wave, ControlCoefficients, defaults to 0.5 |
| D    | debug  |  uint, 2-4  | 2 shows queue sample, 3 shows oscillator data, 4 shows modified oscillator. will interrupt audio! |
| f    | freq   |  float[,float...]      | frequency of oscillator, set of ControlCoefficients.  Default is 0,1,0,0,0,0,1 (from `note` pitch plus `pitch_bend`) |
| F    | filter_freq | float[,float...]  | center frequency for biquad filter, set of ControlCoefficients |
| G    | filter_type | 0-4 |  0 = none (default.) 1 = lowpass, 2 = bandpass, 3 = highpass, 4 = double-order lowpass. |
| H    | reverb_liveness | float 0-1 | Reverb decay time, 1 = longest, default = 0.85. |
| h    | reverb_level | float | Level at which reverb is mixed in to final output.  Default 0, typically 1. |
| I    | ratio  | float | for ALGO types, where the base note frequency controls the modulators, or for the PARTIALS base note, where the ratio controls the speed of the playback |
| j    | reverb_damping  | float 0-1 | Reverb extra decay of high frequencies, default = 0.5. |
| J    | reverb_xover_hz | float  | Crossover frequency (in Hz) for damping decay, default = 3000. |
| k    | chorus_level | float 0-1 | Gain applied to chorus when mixing into output.  Set to 0 to turn off chorus. |
| K    | load_patch | uint 0-X | Apply a saved patch to start at the selected oscillator |
| L    | mod_source | 0 to OSCS-1 | Which oscillator is used as an modulation/LFO source for this oscillator. Source oscillator will be silent. |
| l    | vel | float 0-1+ | velocity: > 0 to trigger note on, 0 to trigger note off |
| M    | chorus_freq | float | LFO freq of chorus | 
| m    | chorus_delay | uint 1-512 | Maximum delay in chorus delay lines, in samples. Default 320 |
| N    | latency_ms | uint | sets latency in ms. default 0 (see LATENCY) | 
| n    | note | uint 0-127 | midi note, sets frequency | 
| o    | algorithm | uint 1-32 | DX7 algorithm to use for ALGO type |
| O    | algo_source | string | which oscillators to use for the algorithm. list of six (starting with op 6), use empty for not used, e.g 0,1,2 or 0,1,2,,, |
| p    | P-patch | uint | choose a preloaded PCM sample or partial patch. Not for DX7 or Juno, use load_patch for those |
| P    | phase | float 0-1 | where in the oscillator's cycle to start sampling from (also works on the PCM buffer). default 0 |
| Q    | pan   | float[,float...] | panning index ControlCoefficients (for stereo output), 0.0=left, 1.0=right. default 0.5. |
| q    | chorus_depth | float | chorus depth | 
| R    | resonance | float | q factor of biquad filter. in practice, 0.5-16.0. default 0.7 | 
| r    | voices | int[,int] | String comma separated list of voices to send message to, or load patch into | 
| S    | reset  | uint | resets given oscillator. set to > OSCS to reset all oscillators, gain and EQ |
| s    | pitch_bend | float | Sets the global pitch bend, by default modifying all note frequencies by (fractional) octaves up or down |
| t    | time | uint | ms of expected playback since some fixed start point on your host. you should always give this if you can. |
| u    | store_patch | number,string | store up to 32 patches in RAM with ID number (1024-1055) and AMY message after a comma. Must be sent alone |  
| v    | osc | uint 0 to OSCS-1 | which oscillator to control | 
| V    | volume | float 0-10 | volume knob for entire synth, default 1.0 | 
| w    | wave | uint 0-11 | waveform: [0=SINE, PULSE, SAW_DOWN, SAW_UP, TRIANGLE, NOISE, KS, PCM, ALGO, PARTIAL, PARTIALS, OFF]. default: 0/SINE |
| x    | eq_l | float | in dB, fc=800Hz amount, -15 to 15. 0 is off. default 0. |
| y    | eq_m | float |  in dB, fc=2500Hz amount, -15 to 15. 0 is off. default 0. |
| z    | eq_h | float | in dB, fc=7500Hz amount, -15 to 15. 0 is off. default 0. | 



# Synthesizer details

We'll use Python for showing examples of AMY, make sure you've installed `libamy` and are running a live AMY first by running `make test` and then

```bash
python
>>> import amy
>>> amy.live()
```

## AMY and timestamps

AMY is meant to receive messages in real time. It, on its own, is not a sequencer where you can schedule notes to play in the future. However, it does maintain a window of (configurable) 20 seconds in advance of its clock where events can be scheduled. This is very helpful in cases where you can't rely on an accurate clock from the client, or don't have one. The clock used internally by AMY is based on the audio samples being generated out the speakers, which should run at an accurate 44,100 times a second.  This lets you do things like schedule fast moving parameter changes over short windows of time. 

For example, to play two notes, one a second after the first, you could do:

```python
amy.send(osc=0, note=50, vel=1)
time.sleep(1)
amy.send(osc=0, note=52, vel=1)
```

But you'd be at the mercy of Python's internal timing, or your OS. A more precise way is to send the messages at the same time, but to indicate the intended time of the playback:

```python
start = amy.millis()  # arbitrary start timestamp
amy.send(osc=0, note=50, vel=1, time=start)
amy.send(osc=0, note=52, vel=1, time=start + 1000)
```

Both `amy.send()`s will return immediately, but you'll hear the second note play precisely a second after the first. AMY uses this internal clock to schedule step changes in breakpoints as well. 


## Examples

`amy.drums()` should play a test pattern.

Try to set the volume of the synth with `amy.volume(2)` -- that can be up to 10 or so.  The default is 1. 

`amy.reset()` resets all oscillators to default. You can also do `amy.reset(osc=5)` to do just one oscillator.

Let's set a simple sine wave first

```python
amy.send(osc=0, wave=amy.SINE, freq=220, amp=1)
```

What we're doing here should be pretty straightforward. I'm telling oscillator 0 to be a sine wave at 220Hz and amplitude 1.  You can also try `amy.PULSE`, or `amy.SAW_DOWN`, etc.

**Why can't you hear anything yet?** It's because you haven't triggered the note on for this oscillator. We accept a parameter called `vel` (velocity) that can turn a note on or off (`vel=0`.) So now that we've set up the oscillator, we just turn it on (note the oscillator remembers all its state and setup):

```python
amy.send(osc=0, vel=1)  # Note on.
```

To turn off the note, send a note off (velocity zero):

```python
amy.send(osc=0, vel=0)  # Note off.
```

You can also make oscillators louder with `amp` or `vel` over 1.  The total amplitude comes from multiplying together the oscillator amplitude (i.e., the natural level of the oscillator) and the velocity (the particular level of this note event).

You can also use `note` (MIDI note value) instead of `freq` to control the oscillator frequency for each note event:

```python
amy.reset()
amy.send(osc=0, wave=amy.SINE, note=57, vel=1)
```

This won't work without the `amy.reset()`, because once you've set the oscillator to a constant frequency with `freq=220`, it will ignore the frequency specified by `note`.  (But see **ControlCoefficients** below to see how you can use both at the same time).

Now let's make a lot of sine waves! 

```python
import time
amy.reset()
for i in range(16):
    amy.send(osc=i, wave=amy.SINE, freq=110+(i*80), vel=((16-i)/32.0))
    time.sleep(0.5) # Sleep for 0.5 seconds
```

Neat! You can see how simple / powerful it is to have control over lots of oscillators. You have up to 64 (or more, depending on your platform). Let's make it more interesting. A classic analog tone is the filtered saw wave. Let's make one.

```python
amy.send(osc=0, wave=amy.SAW_DOWN, filter_freq=3200, resonance=5, filter_type=amy.FILTER_LPF)
amy.send(osc=0, vel=1, note=40)
```

You want to be able to stop the note too by sending a note off:
```python
amy.send(osc=0, vel=0)
```

Sounds nice. But we want that filter freq to go down over time, to make that classic filter sweep tone. Let's use an Envelope Generator! An Envelope Generator (EG) creates a smooth time envelope based on a breakpoint set, which is a simple list of (time-delta, target-value) pairs - you can have up to 8 of these per EG, and 2 different EGs to control different things. They're just like ADSRs, but more powerful. You can control amplitude, oscillator frequency, filter frequency, PWM duty cycle, or pan, with an EG. It gets triggered when the note begins. So let's make an EG that turns the filter frequency down from its start at 3200 Hz to 400 Hz over 1000 milliseconds. And when the note goes off, it tapers the frequency to 50 Hz over 200 milliseconds.

```python
amy.send(osc=0, wave=amy.SAW_DOWN, resonance=5, filter_type=amy.FILTER_LPF)
amy.send(osc=0, filter_freq="50,0,0,0,1,0", bp1="0,6.0,1000,3.0,200,0")
amy.send(osc=0, vel=1, note=40)
```

There are two things to note here:  Firstly, the filter frequency is controlled by the EG using a "unit per octave" rule.  So if the envelope is zero, the filter is at its default frequency (50 Hz, the first value in the `filter_freq` list).  But the envelope starts at 6.0, which is 6 octaves higher, or 2^6 = 64x the frequency -- 3200 Hz.  It then decays to 3.0 over the first second, which is 2^3 = 8x the default frequency, giving 400 Hz.  It's only during the final release of 200 ms that it falls back to 0, giving a final filter frequency of (2^0 = 1x) 50 Hz.

Secondly, the filter frequency is controlled by a list of numbers, not just the initial 50.  `filter_freq` is an example of a set of **ControlCoefficients**, the others being `amp`, `freq`, `duty`, and `pan`.  **ControlCoefficients** are a list of up to 7 floats that are multiplied by a range of control signals, then summed up to give the final result (in this case, the filter frequency).  The control signals are:
 * A constant value of 1 - so the first number in the control coefficient list is the default value if all the others are zero or not specified.
 * The frequency corresponding to the `note` parameter to the note-on event (converted to unit-per-Hz relative to middle C).
 * The velocity from the note-on event.
 * The output of Envelope Generator 0.
 * The output of Envelope Generator 1.
 * The output of the modulating oscillator, specified by the `mod_source` parameter.
 * The current pitch bend value (from `amy.send(pitch_bend=0.5)` etc.).

The set "50,0,0,0,1" means that we have a base frequency of 50 Hz, we ignore the note frequency and velocity and EG0, but we also add the output of EG1. If you specify fewer than 7 coefficients, the remaining ones are taken as zero, so `filter_freq=5000` is equivalent to `filter_freq="5000,0,0,0,0,0,0"`.

You can use the same EG to control several things at once.  For example, we could include `freq="50,0,0,0,0.125"`, which says to modify a base note frequency of 50 Hz from the same EG1 as is controlling the filter frequency, but scaled down by 1/8th so the initial decay is over 1 octave, not 3.  Give it a go!

The note frequency is scaled relative to a zero-point of middle C (MIDI note 60, 261.63 Hz), so to make the oscillator faithfully track the `note` parameter to the note-on event, you would use something like `freq="261.63,1"` (which is its default setting before any `freq` parameter is passed).  Setting it to `freq="523.26,1"` would make the oscillator always be one octave higher than the `note` MIDI number.  Setting `freq="261.3,0.5"` would make the oscillator track the `note` parameter at half an octave per unit, so while `note=60` would still give middle C, `note=72` (C5) would make the oscillator run at F#4, and `note=84` (C6) would be required to get C5 from the oscillator.

Actually, the default set of ControlCoefficients for `freq` is "261.63,1,0,0,0,0,1", i.e. a base of middle C, tracking the MIDI note, plus pitch bend (at unit-per-octave).  Because 261.63 is such an important value, as a special case, setting the first `freq` value to zero is magically rewritten as 261.63, so `freq="0,1,0,0,0,0,1"` also yields the default behavior.  `amp` also has a set of defaults `amp="0,0,1,1,0,0,0"`, i.e. tracking note-on velocity plus modulation by EG0 (which just tracks the note-on status if it is empty).  `amp` is a little special because the individual components are *multiplied* together, instead of added together, for any control inputs with nonzero coefficients.  Finally, we add 1.0 to the coefficient-scaled LFO modulator and pitch bend inputs before multiplying them into the amplitude, to allow small variations around identity e.g. for tremolo.  These defaults are set up in [`src/amy.c:reset_osc()`](https://github.com/shorepine/amy/blob/b1ed189b01e6b908bc19f18a4e0a85761d739807/src/amy.c#L551).

We also have LFOs, which are implemented as one oscillator modulating another. You set up the lower-frequency oscillator, then have it control a parameter of another audible oscillator. Let's make the classic 8-bit duty cycle pulse wave modulation, a favorite:

```python
amy.reset()  # Clear the state.
amy.send(osc=1, wave=amy.SINE, freq=0.5, amp=1)
amy.send(osc=0, wave=amy.PULSE, duty="0.5,0,0,0,0,0.4", mod_source=1)
amy.send(osc=0, note=60, vel=0.5)
```

You see we first set up the modulation oscillator (a sine wave at 0.5Hz, with amplitude of 1).  Then we set up the oscillator to be modulated, a pulse wave with mod source of oscillator 1 and the duty **ControlCoefficients** to have a constant value of 0.5 plus 0.4 times the modulating input (i.e., the depth of the pulse width modulation, where 0.4 modulates between 0.1 and 0.9, almost the maximum depth).  The initial duty cycle will start at 0.5 and be multiplied by the state of oscillator 1 every tick, to make that classic thick saw line from the C64 et al. The modulation will re-trigger every note on. Just like with envelope generators, the modulation oscillator has a 'slot' in the ControlCoefficients - the 6th coefficient - so it can modulate PWM duty cycle, amplitude, frequency, filter frequency, or pan! And if you want to modulate more than one thing, like frequency and duty, just specify multiple ControlCoefficients:

```python
amy.send(osc=1, wave=amy.TRIANGLE, freq=5, amp=1)
amy.send(osc=0, wave=amy.PULSE, duty="0.5,0,0,0,0,0.25", freq="0,1,0,0,0,0.5", mod_source=1)
amy.send(osc=0, note=60, vel=0.5)
```

`amy.py` has some helpful presets, if you want to use them, or add to them. To make that filter bass, just do `amy.preset(1, osc=0)` and then `amy.send(osc=0, vel=1, note=40)` to hear it. Here's another one:

```python
amy.preset(0, osc=2) # will set a simple sine wave tone on oscillator 2
amy.send(osc=2, note=50, vel=1.5) # will play the note at velocity 1.5
amy.send(osc=2, vel=0) # will send a "note off" -- you'll hear the note release
amy.send(osc=2, freq=220.5, vel=1.5) # same but specifying the frequency
amy.reset()
```

## Core oscillators

We support bandlimited saw, pulse/square and triangle waves, alongside sine and noise. Use the wave parameter: 0=SINE, PULSE, SAW_DOWN, SAW_UP, TRIANGLE, NOISE. Each oscillator can have a frequency (or set by midi note), amplitude and phase (set in 0-1.). You can also set `duty` for the pulse type. We also have a karplus-strong type (KS=6). 

Oscillators will not become audible until a `velocity` over 0 is set for the oscillator. This is a "note on" and will trigger any modulators or envelope generators set for that oscillator. Setting `velocity` to 0 sets a note off, which will stop modulators and also finish the envelopes at their release pair. `velocity` also internally sets `amplitude`, but you can manually set `amplitude` after `velocity` starts a note on.

## LFOs & modulators

Any oscillator can modulate any other oscillator. For example, a LFO can be specified by setting oscillator 0 to 0.25Hz sine, with oscillator 1 being a 440Hz sine. Using the 6th parameter of **ControlCoefficient** lists, you can have oscillator 0 modulate frequency, amplitude, filter frequency, or pan of oscillator 1. You can also add targets together, for example amplitude+frequency. Set the `mod_target` and `mod_source` on the audible oscillator (in this case, oscillator 1.) The source mod oscillator will not be audible once it is referred to as a `mod_source` by another oscillator. The amplitude of the modulating oscillator indicates how strong the modulation is (aka "LFO depth.")

## Filters

We support lowpass, bandpass and hipass filters in AMY. You can set `resonance` and `filter_freq` per oscillator. 

## EQ & Volume

You can set a synth-wide volume (in practice, 0-10), or set the EQ of the entire synths's output. 

## Envelope Generators

AMY allows you to set 2 Envelope Generators (EGs) per oscillator. You can see these as ADSR / envelopes (and they can perform the same task), but they are slightly more capable. Breakpoints are defined as pairs (up to 8 per EG) of time deltas (specified in milliseconds) and target value. You can specify up to 8 pairs, but the last pair you specify will always be seen as the "release" pair, which doesn't trigger until note off. All other pairs previously have time deltas relative to the previous segment, so `100,1,100,0,0,0` goes up to 1 over 100 ms, then back down to zero over the next 100ms. The last "release" pair counts from ms from the note-off. 

An EG can control amplitude, frequency, filter frequency, duty or pan of an oscillator via the 4th (EG0) and 5th (EG1) entries in the corresponding ControlCoefficients.

For example, to define a common ADSR curve where a sound sweeps up in volume from note on over 50ms, then has a 100ms decay stage to 50% of the volume, then is held until note off at which point it takes 250ms to trail off to 0, you'd set time to be 50ms and target to be 1.0, then 100ms with target .5, then a 250ms release with ratio 0. By default, amplitude is set up to be controlled by EG0. At every synthesizer tick, the given amplitude (default of 1.0) will be multiplied by the EG0 value. In AMY wire parlance, this would look like "`v0f220w0A50,1.0,100,0.5,250,0`" to specify a sine wave at 220Hz with this envelope. 

When using `amy.py`, use the string form of the breakpoint: `bp0="50,1.0,100,0.5,250,0"`. 

Every note on (specified by setting velocity / `l` to anything > 0) will trigger this envelope, and setting `l` to 0 will trigger the note off / release section. 

You can set a completely separate envelope using the second envelope generator, for example, to change pitch and amplitude at different rates.


## FM & ALGO type

Try default DX7 patches, from 128 to 256:

```python
amy.send(voices="0", load_patch=128)
amy.send(voices="0", note=50,vel=1)
```

The `load_patch` lets you set which preset is used (0 to 127 are the Juno 106 analog synth presets, and 128 to 255 are the DX7 FM presets).  But let's make the classic FM bell tone ourselves, without a patch. We'll just be using two operators (two sine waves), one modulating the other.

![DX7 Algorithms](https://raw.githubusercontent.com/shorepine/alles/main/pics/dx7_algorithms.jpg)

When building your own algorithm sets, assign a separate oscillator as wave=`ALGO`, but the source oscillators as `SINE`. The algorithm #s are borrowed from the DX7. You don't have to use all 6 operators. Note that the `algo_source` parameter counts backwards from operator 6. When building operators, they can have their frequencies specified directly with `freq` or as a ratio of the root `ALGO` oscillator via `ratio`.


```python
amy.reset()
amy.send(osc=0, wave=amy.SINE, ratio=0.2, amp="0.1,0,0,1", bp0="0,1,1000,0,0,0")
amy.send(osc=1, wave=amy.SINE, ratio=1, amp="1")
amy.send(osc=2, wave=amy.ALGO, algorithm=1, algo_source=",,,,1,0")
```

Let's unpack that last line: we're setting up a ALGO "oscillator" that controls up to 6 other oscillators. We only need two, so we set the `algo_source` to mostly not used and have oscillator 1 modulate oscillator 0. You can have the operators work with each other in all sorts of crazy ways. For this simple example, we just use the DX7 algorithm #1. And we'll use only operators 2 and 1. Therefore our `algo_source` lists the oscillators involved, counting backwards from 6. We're saying only have operator 2 (oscillator 1) and operator 1 (oscillator 0).  From the picture, we see DX7 algorithm 1 has operator 2 feeding operator 1, so we have oscillator 1 providing the frequency-modulation input to oscillator 0.

What's going on with `ratio`? And `amp`? Ratio, for FM synthesis operators, means the ratio of the frequency for that operator to the base note. So oscillator 0 will be played at 20% of the base note frequency, and oscillator 1 will be the frequency of the base note. And for `amp`, that's something called "beta" in FM synthesis, which describes the strength of the modulation. Note we are having beta go down over 1,000 milliseconds using an envelope generator. That's key to the "bell ringing out" effect.

Ok, we've set up the oscillators. Now, let's hear it!

```python
amy.send(osc=2, note=60, vel=3)
```

You should hear a bell-like tone. Nice. (This example is also implemented using the C API in [`src/examples.c:example_fm()`](https://github.com/shorepine/amy/blob/b1ed189b01e6b908bc19f18a4e0a85761d739807/src/examples.c#L281).)

Another classic two operator tone is to instead modulate the higher tone with the lower one, to make a filter sweep. Let's do it over 5 seconds.

```python
amy.reset()
amy.send(osc=0, wave=amy.SINE, ratio=1, amp="1")  # Op 1, carrier
amy.send(osc=1, wave=amy.SINE, ratio=0.2, amp="2", bp0="0,0,5000,1,0,0")  # Op 2, modulator
amy.send(osc=2, wave=amy.ALGO, algorithm=1, algo_source=",,,,1,0")
```

Just a refresher on envelope generators; here we are saying to set the beta parameter (amplitude of the modulating tone) to 2 but have it start at 0 at time 0 (actually, this is the default), then be at 1.0x of 2 (so, 2.0) at time 5000ms. At the release of the note, set beta immediately to 0. We can play it with

```python
amy.send(osc=2, vel=2, note=50)
```

and stop it with

```python
amy.send(osc=2, vel=0)
```


## Partials

Additive synthesis is simply adding together oscillators to make more complex tones. You can modulate the breakpoints of these oscillators over time, for example, changing their pitch or time without artifacts, as the synthesis is simply playing sine waves back at certain amplitudes and frequencies (and phases). It's well suited to certain types of instruments. 

![Partials](https://raw.githubusercontent.com/shorepine/alles/main/pics/partials.png)

We have analyzed the partials of a group of instruments and stored them as presets baked into the synth. Each of these patches are comprised of multiple sine wave oscillators, changing over time. The `PARTIALS` type has the presets:

```python
amy.send(osc=0,vel=1,note=50,wave=amy.PARTIALS,patch=5) # a nice organ tone
amy.send(osc=0,vel=1,note=55,wave=amy.PARTIALS,patch=5) # change the frequency
amy.send(osc=0,vel=1,note=50,wave=amy.PARTIALS,patch=6,ratio=0.2) # ratio slows down the partial playback
```

The presets are just the start of what you can do with partials in AMY. You can analyze any piece of audio and decompose it into sine waves and play it back on the synthesizer in real time. It requires a little setup on the client end, here on macOS:

```bash
brew install python3 swig ffmpeg
python3.9 -m pip install pydub numpy --user
tar xvf loris-1.8.tar
cd loris-1.8
CPPFLAGS=`python3-config --includes` PYTHON=`which python3` ./configure --with-python
make
sudo make install
cd ..
```

And then in python:

```python
import partials
(m,s) = partials.sequence("sleepwalk.mp3") # Any audio file
109 partials and 1029 breakpoints, max oscs used at once was 8

partials.play(s, amp_ratio=2)
```

https://user-images.githubusercontent.com/76612/131150119-6fa69e3c-3244-476b-a209-1bd5760bc979.mp4


You can see, given any audio file, you can hear a sine wave decomposition version of in AMY. This particular sound emitted 109 partials, with a total of 1029 breakpoints among them to play back to the mesh. Of those 109 partials, only 8 are active at once. `partials.sequence()` performs voice stealing to ensure we use as few oscillators as necessary to play back a set. 

There's a lot of parameters you can (and should!) play with in Loris. `partials.sequence`  and `partials.play`takes the following with their defaults:

```python
def sequence(filename, # any audio filename
                max_len_s = 10, # analyze first N seconds
                amp_floor=-30, # only accept partials at this amplitude in dB, lower #s == more partials
                hop_time=0.04, # time between analysis windows, impacts distance between breakpoints
                max_oscs=amy.OSCS, # max AMY oscs to take up
                freq_res = 10, # freq resolution of analyzer, higher # -- less partials & breakpoints 
                freq_drift=20, # max difference in Hz within a single partial
                analysis_window = 100 # analysis window size 
                ) # returns (metadata, sequence)

def play(sequence, # from partials.sequence
                osc_offset=0, # start at this oscillator #
                sustain_ms = -1, # if the instrument should sustain, here's where (in ms)
                sustain_len_ms = 0, # how long to sustain for
                time_ratio = 1, # playback speed -- 0.5 , half speed
                pitch_ratio = 1, # frequency scale, 0.5 , half freq
                amp_ratio = 1, # amplitude scale
                )
```



## PCM

AMY comes with a set of 67 drum-like and instrument PCM samples to use as well, as they are normally hard to render with additive or FM synthesis. You can use the type `PCM` and patch numbers 0-66 to explore them. Their native pitch is used if you don't give a frequency or note parameter. You can update the PCM sample bank using `amy_headers.py`. 


```python
amy.send(osc=0, wave=amy.PCM, vel=1, patch=10) # cowbell
amy.send(osc=0, wave=amy.PCM, vel=1, patch=10, note=70) # higher cowbell! 
```

You can turn on sample looping, helpful for instruments, using `feedback`:

```python
amy.send(wave=amy.PCM,vel=1,patch=21,feedback=0) # clean guitar string, no looping
amy.send(wave=amy.PCM,vel=1,patch=21,feedback=1) # loops forever until note off
amy.send(vel=0) # note off
amy.send(wave=amy.PCM,vel=1,patch=35,feedback=1) # nice violin
```


## Developer zone

### Generate header files for patches and LUTs

Run `python amy_headers.py` to generate all the LUTs and patch .h files compiled into AMY.







