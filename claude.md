# Plinky Code Navigation Map

**Purpose**: Quick reference for finding where functionality is implemented in the codebase. Function inventory and relationships, not detailed explanations.

---

## Project Overview

**Plinky** = 8-voice polyphonic touch synthesizer on STM32L4xx
- 8 capacitive touch strips (note input with position/pressure)
- 1 shift button strip (8 buttons)
- OLED display, 9x8 LED matrix
- Audio codec (WM8758), MIDI (USB + serial), CV I/O, accelerometer

**Main source**: `sw/Core/Src/`

---

## Entry Points & Main Loop

### `main.c` - Hardware initialization and main loop

**Key functions**:
- `main()` - Initializes all peripherals, calls `plinky_init()`, then loops calling `plinky_frame()`
- `MX_*_Init()` - Individual peripheral init functions (ADC, DAC, SAI, SPI, I2C, TSC, timers, USB)
- `SystemClock_Config()` - 80MHz clock setup
- `EncoderTick()` - Rotary encoder handler (called from interrupt)
- `update_accelerometer_raw()` - Reads accelerometer data

**Main loop**: `while(1) { plinky_frame(); }` at line 305

---

## Core Synthesis - `plinky.c` (3,400 lines)

### Audio Processing Chain

**Main audio function**:
- `DoAudio(u32 *dst, u32 *audioin)` - Called from SAI DMA callback at ~500Hz (64 samples/block)
  - Calls `RunVoice()` for each of 8 voices
  - Applies effects chain
  - Mixes to stereo output

**Voice synthesis**:
- `RunVoice(Voice *v, int fingeridx, float targetvol, u32 *outbuf)` - Renders one voice
  - Oscillator/grain synthesis
  - Filter
  - Envelope

**Envelope**:
- `UpdateEnvelope(Voice *v, int fingeridx, float targetvol)` - ADSR envelope processing

**Audio input**:
- `PreProcessAudioIn(u32* audioin)` - Processes input before mixing
- `DoRecordModeAudio(u32* dst, u32* audioin)` - Handles sample recording

### Parameter System

- `param_eval_float(u8 paramidx, int rnd, int env16, int pressure16)` - Evaluates parameter with modulations
- `param_eval_finger(u8 paramidx, int fingeridx, Finger* f)` - Per-finger parameter evaluation
- `update_params(int fingertrig, int fingerdown)` - Updates all parameters each frame

### MIDI

- `processmidimsg(u8 msg, u8 d1, u8 d2)` - Incoming MIDI message handler
- `processusbmidi()` - USB MIDI polling
- `serial_midi_update()` - Serial MIDI polling
- `midi_send_update()` - Outgoing MIDI queue processing
- `midi_panic()` - All notes off

### Grain/Sample Functions

- `sample_slice_pos8(int pos16)` - Sample position to slice mapping
- `calcloopstart(u8 sliceidx)` - Loop start for slice
- `calcloopend(u8 sliceidx)` - Loop end for slice
- `doloop(int playhead, u8 sliceidx)` - Loop boundary handling

### Utility

- `lin2db(float)`, `db2lin(float)` - Conversions
- `knobsmooth_update_knob()`, `knobsmooth_update_cv()` - Smoothing filters
- `stride(u32 scale, int stride_semitones, int fingeridx)` - Scale step calculation

---

## UI & Display - `ui.h` (1,253 lines)

### LED Animations

- `bootswish()` - **Boot radial wave animation** (line 64)
  - Calculates distance from center
  - Animates over 64 frames

- **Touch-responsive audio wave** (lines 1147-1155)
  - In play mode, creates ripple effect from audio peaks
  - Distance-based delay: `delay = 1+(((7 - y) * (7 - y) + fi * fi) >> 2)`

### Main UI Loop

- `plinky_frame()` - Called every ~2ms from main loop
  - Touch processing
  - UI updates
  - LED updates
  - Display rendering

### Display Functions

(Need to grep for draw/oled functions to populate this)

---

## Touch Processing - `touch.h` (1,338 lines)

### Data Structures

- `Finger` - Position (15-bit) + pressure (16-bit)
- `fingers_ui_time[9][8]` - UI-time finger history (9 fingers × 8 frames)
- `fingers_synth_time[8][8]` - Synth-time finger history
- `CalibResult`, `CalibProgress` - Calibration data

### Key Functions

- `touch_reset_calib()` - Reset calibration
- `touch_ui_writingframe()`, `touch_ui_frame()`, `touch_ui_prevframe()` - Frame indexing
- `touch_synth_getlatest()`, `touch_synth_getprev()` - Access finger data
- `sort8()` - Sorting network for 8 elements

### Finger Access

Inline getters for current/previous finger states at UI time vs synth time.

---

## Arpeggiator - `arp.h` (570 lines)

### Core Functions

- `arp_reset()` - Full reset
- `arp_reset_partial()` - Partial reset (keeps position/octave)
- `nextup(u8 allowedfingers)` - Next note up
- `nextdown(u8 allowedfingers)` - Next note down
- `pickrandombit(u8 mask)` - Random note selection
- `euclidstuff(euclid_state *s, int patlen, int prob, int arpmode)` - Euclidean rhythm

### State Variables

- `arpbits` - Current active fingers
- `curarpfinger` - Current finger being played
- `arpoctave` - Octave offset
- `arpmode` - Current arp mode
- `arpretrig` - Retrigger flag

---

## Sequencer

(Need to find main sequencer functions - likely in plinky.c or separate file)

---

## Parameters - `params.h` (1,155 lines)

### Parameter Definition

- `paramnames[P_LAST]` - String names for all parameters
- Parameter enums: `P_A`, `P_D`, `P_S`, `P_R`, `P_PITCH`, `P_DELAY`, etc.

### LFO

- `lfo_eval(u32 ti, float warp, unsigned int shape)` - Evaluate LFO
- `EvalTri()`, `EvalSin()`, `EvalSaw()`, `EvalSquare()`, etc. - Shape functions
- `lfofuncs[LFO_LAST]` - Function pointer array for shapes

### Modulation Sources

- Enum `EModSources`: `M_BASE`, `M_ENV`, `M_PRESSURE`, `M_A`, `M_B`, `M_X`, `M_Y`, `M_RND`

---

## Audio Codec - `codec.h` (430 lines)

- `wmcodec_write(uint8_t reg, uint16_t data)` - Write codec register via I2C
- Register definitions (PWRMGMT*, AINTFCE, CLKCTRL, etc.)

---

## LED System - `leds.h` (273 lines)

### Hardware Interface

- `led_init()` - Initialize PWM timers for LED scanning
- `led_update()` - Called regularly to multiplex LED matrix (updates one row per call)
- `led_gamma(int i)` - Gamma correction for LED brightness

### LED Buffer

- `led_ram[9][8]` - 9 rows × 8 columns brightness values (0-255)
  - Rows 0-7: Note strips
  - Row 8: Shift buttons

---

## Graphics - `gfx.h` (371 lines)

### OLED Functions

(Need to grep for specific draw functions)

---

## Scales & Enums - `enums.h` (289 lines)

### Enums

- `EPages` - UI pages (PG_SOUND1, PG_ENV1, PG_DELAY, PG_ARP, etc.)
- `EModSources` - Modulation sources
- `Scales` - 26 scales (S_MAJOR, S_MINOR, S_PENTA, S_DORIAN, etc.)
- `ELFOShape` - LFO shapes
- `EArpModes` - 15 arp modes (ARP_UP, ARP_DOWN, ARP_RANDOM, etc.)
- `ESeqModes` - 6 sequencer modes
- Edit modes, shift button states

### Name Arrays

- `pagenames[]`, `scalenames[]`, `lfonames[]`, `arpmodenames[]`, `seqmodenames[]`, `modnames[]`

---

## Flash Storage - `flash.h`

(Need to find flash read/write functions)

---

## Peripherals - Additional Headers

- `adc.h` - CV input reading
- `dac.h` - CV output
- `oled.h` - Display driver
- `spi.h` - SPI communication
- `calib.h` - Calibration system
- `edit.h` - Edit mode logic
- `webusb.h` - WebUSB interface

---

## Data Tables

- `tables.h` (2,189 lines) - Wavetables and lookup tables
- `rand.h` (1,029 lines) - Random number tables
- `sigmoid.h` - Sigmoid lookup
- `fontdata.h` - Font bitmap data
- `icons.h` - UI icons
- `wavetable.h` - Additional wavetable data

---

## Call Hierarchy (Key Paths)

### Audio Path
```
SAI DMA Interrupt
  ├─> DoAudio()
      ├─> RunVoice() [×8 voices]
      │   ├─> UpdateEnvelope()
      │   └─> Grain/oscillator synthesis
      └─> Effects processing
```

### Main Loop Path
```
main()
  └─> while(1) { plinky_frame() }
      ├─> Touch processing
      ├─> UI updates
      ├─> LED updates (led_ram[] writes)
      ├─> MIDI processing
      │   ├─> processusbmidi()
      │   ├─> serial_midi_update()
      │   └─> midi_send_update()
      └─> Parameter updates
          └─> update_params()
              └─> param_eval_float() [for each param]
```

### LED Update Path
```
Timer interrupt (high frequency)
  └─> led_update()  [multiplexes led_ram[] to hardware]

plinky_frame() (UI time)
  └─> Writes to led_ram[][]
      ├─> Touch visualization (distance-based ripple)
      ├─> Arp/seq visualization
      ├─> Mode indicators
      └─> Audio level visualization
```

---

## File Size Reference

**Largest files** (where to look for major functionality):
1. plinky.c - 3,400 lines (synthesis, MIDI, audio)
2. main.c - 1,339 lines (init, main loop)
3. touch.h - 1,338 lines (touch processing)
4. ui.h - 1,253 lines (UI, display, LEDs)
5. params.h - 1,155 lines (parameter definitions, LFO)

---

## Quick Lookup: "Where is...?"

- **Wave animation**: `ui.h` - `bootswish()` (line 64) and touch-responsive audio wave (lines 1147-1155)
- **Audio synthesis**: `plinky.c` - `DoAudio()`, `RunVoice()`
- **MIDI handling**: `plinky.c` - `processmidimsg()`
- **Touch sensing**: `touch.h` - finger data structures and access functions
- **Arpeggiator logic**: `arp.h` - `nextup()`, `nextdown()`, `euclidstuff()`
- **Parameter evaluation**: `plinky.c` - `param_eval_float()`, `update_params()`
- **LED control**: `leds.h` - `led_update()`, `led_ram[][]`
- **Envelopes**: `plinky.c` - `UpdateEnvelope()`
- **Scales**: `enums.h` - scale definitions and `scalenames[]`
- **Main loop**: `main.c` - `main()` → `plinky_frame()` loop
