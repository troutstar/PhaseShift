# Plan: Guitar/Bass Phase VST Plugin (JUCE 8)

## Context
Building a single VST3 plugin targeting guitar/bass that combines two distinct phase-based effects under one mode switch:
1. **Phase Vocoder** — independent pitch shift and time stretch controls
2. **Phasiness** — two sub-modes: classic all-pass phaser sweep (MXR-style) and FFT phase randomization (lo-fi/underwater)

JUCE 8.0.13 is installed at `C:\JUCE`. Build target: Windows VST3 + Standalone.

---

## Project Setup

**Use CMake** (not Projucer) — JUCE 8 recommends CMake and it's easier to version-control.

Directory: `C:\Users\Troutstar\Desktop\Harness\PhaseShift\`

```
PhaseShift/
  CMakeLists.txt
  Source/
    PluginProcessor.h / .cpp
    PluginEditor.h / .cpp
    dsp/
      PhaseVocoder.h / .cpp
      PhaserChain.h / .cpp
      PhaseRandomizer.h / .cpp
  Resources/  (fonts, images if needed)
```

**CMakeLists.txt** key config:
```cmake
cmake_minimum_required(VERSION 3.22)
project(PhaseShift VERSION 1.0.0)
add_subdirectory(C:/JUCE juce)

juce_add_plugin(PhaseShift
    FORMATS VST3 Standalone
    PLUGIN_NAME "PhaseShift"
    PLUGIN_MANUFACTURER_CODE Trst
    PLUGIN_CODE Pshf
    IS_SYNTH FALSE
    NEEDS_MIDI_INPUT FALSE
)

target_sources(PhaseShift PRIVATE Source/PluginProcessor.cpp Source/PluginEditor.cpp
    Source/dsp/PhaseVocoder.cpp Source/dsp/PhaserChain.cpp Source/dsp/PhaseRandomizer.cpp)

target_link_libraries(PhaseShift PRIVATE juce::juce_audio_processors juce::juce_dsp juce::juce_gui_basics)
```

Build: `cmake -B build -S . && cmake --build build --config Release`

---

## Parameter Layout (AudioProcessorValueTreeState)

| ID | Type | Range | Description |
|---|---|---|---|
| `mode` | int | 0–1 | 0 = Vocoder, 1 = Phasiness |
| `pitch` | float | -24 to +24 st | Semitone pitch shift (Vocoder) |
| `timeStretch` | float | 0.25–4.0 | Time stretch ratio (Vocoder) |
| `phasinessMode` | int | 0–1 | 0 = AllPass sweep, 1 = FFT randomize |
| `phaserRate` | float | 0.1–10 Hz | LFO rate (AllPass mode) |
| `phaserDepth` | float | 0–1 | LFO depth (AllPass mode) |
| `phaserStages` | int | 2–12 (even) | All-pass filter count |
| `randomAmt` | float | 0–1 | Phase randomization amount (FFT mode) |
| `mix` | float | 0–1 | Dry/wet blend |

---

## DSP Implementation

### 1. PhaseVocoder (`dsp/PhaseVocoder.h/.cpp`)

Overlap-add phase vocoder using `juce::dsp::FFT`.

**Key design decisions for guitar/bass:**
- FFT size: 2048 (good transient resolution at typical 44.1/48 kHz)
- Overlap factor: 4 (75% overlap) — reduces artifacts on sustained notes
- Window: Hann

**Core loop:**
```
prepare(sampleRate, blockSize)
  → allocate circular input buffer, FFT buffers, phase accumulator arrays

processBlock(buffer)
  → feed samples into circular input buffer
  → when hopSize samples accumulated: run analysis frame
      fft.performRealOnlyForwardTransform(frame)
      for each bin: compute instantaneous freq from phase difference
      scale frequencies by pitchRatio
      set output hop = inputHop * stretchRatio
      reconstruct phases for output frame
      fft.performRealOnlyInverseTransform(frame)
      overlap-add into output buffer
  → read output buffer with appropriate delay compensation

Parameters updated per block:
  pitchRatio = pow(2, semitones/12)
  stretchRatio = timeStretch param value
```

**Phase accumulation (the critical part):**
```cpp
float expectedPhaseDiff = 2.0f * pi * bin * analysisHop / fftSize;
float phaseDiff = inputPhase[bin] - lastAnalysisPhase[bin] - expectedPhaseDiff;
// wrap to [-pi, pi]
float trueFreq = bin + phaseDiff * fftSize / (2.0f * pi * analysisHop);
outputPhase[bin] += 2.0f * pi * trueFreq * synthesisHop / fftSize;
```

### 2. PhaserChain (`dsp/PhaserChain.h/.cpp`)

Classic all-pass phaser for the "sweep" sub-mode.

- Use `juce::dsp::ProcessorChain` with a chain of `juce::dsp::IIR::Filter<float>` configured as all-pass
- LFO: a simple sine wave modulating the all-pass center frequency
- `phaserStages` controls how many all-pass sections (2, 4, 6, 8, 10, 12)
- Mix dry + processed to create the comb filtering that gives the phaser sound

**All-pass coefficient (first-order):**
```
a = (tan(pi * fc / sampleRate) - 1) / (tan(pi * fc / sampleRate) + 1)
H(z) = (a + z^-1) / (1 + a*z^-1)
```

Update `fc` each block from LFO: `fc = centerFreq + depth * lfoValue * sweepRange`

For guitar/bass, center frequency sweep range: 200 Hz – 2000 Hz.

### 3. PhaseRandomizer (`dsp/PhaseRandomizer.h/.cpp`)

FFT-based phase smearing for the lo-fi sub-mode.

- Same FFT size as vocoder (2048) for consistency
- Per frame: take FFT, for each bin add `randomAmt * random(-pi, pi)` to phase, keep magnitude, IFFT
- Overlap-add output
- At `randomAmt = 0`: transparent. At `randomAmt = 1`: fully randomized, "underwater" texture.

---

## PluginProcessor

```cpp
// PluginProcessor.h
class PhaseShiftProcessor : public juce::AudioProcessor {
    juce::AudioProcessorValueTreeState apvts;
    PhaseVocoder vocoder;
    PhaserChain phaserChain;
    PhaseRandomizer phaseRandomizer;

    void processBlock(juce::AudioBuffer<float>&, juce::MidiBuffer&) override;
};
```

**processBlock routing:**
```cpp
int mode = (int)*apvts.getRawParameterValue("mode");
float mix  = *apvts.getRawParameterValue("mix");

auto dry = buffer; // copy for wet/dry blend

if (mode == 0) {
    vocoder.setParams(pitch, timeStretch);
    vocoder.process(buffer);
} else {
    int pMode = (int)*apvts.getRawParameterValue("phasinessMode");
    if (pMode == 0) {
        phaserChain.setParams(rate, depth, stages);
        phaserChain.process(buffer);
    } else {
        phaseRandomizer.setAmount(randomAmt);
        phaseRandomizer.process(buffer);
    }
}

// dry/wet blend
buffer.applyGain(mix);
dry.applyGain(1.0f - mix);
buffer.addFrom(0, 0, dry, 0, 0, buffer.getNumSamples());
```

---

## PluginEditor (UI)

Simple single-page UI using JUCE's built-in `Slider` and `ComboBox`:

- Top: **Mode** toggle (VOCODER | PHASINESS) — `ComboBox` or two `TextButton`s
- Middle panel swaps content based on mode:
  - Vocoder: two large knobs — PITCH (semitones) and TIME
  - Phasiness: sub-mode toggle + conditional knobs (Rate/Depth/Stages or Randomize Amount)
- Bottom: DRY/WET knob, always visible
- Size: 400 × 300 px — compact for a pedalboard-style plugin

Use `apvts.createAndAddParameter` + `SliderAttachment` / `ComboBoxAttachment` for all controls.

---

## Guitar/Bass Specific Considerations

- **Latency**: Phase vocoder introduces latency = FFT size / sampleRate (~46ms at 2048/44100). Report this via `getLatencySamples()` so the DAW can compensate.
- **Mono-friendly**: Process both channels independently OR sum to mono internally (guitar is typically mono). Add a MONO button or just process channel 0 and copy to channel 1.
- **Pitch range**: -24 to +24 semitones covers all practical guitar/bass needs (drop tuning, harmonics, octave up).
- **Transient handling**: At extreme time stretch ratios (<0.5 or >2.0), transients will smear. This is acceptable/expected for this use case.

---

## Build & Verification Steps

1. `cmake -B build -S . -DCMAKE_BUILD_TYPE=Release`
2. `cmake --build build --config Release`
3. VST3 output lands in `build/PhaseShift_artefacts/Release/VST3/PhaseShift.vst3`
4. Copy to `C:\Program Files\Common Files\VST3\`
5. Load in Reaper (or standalone):
   - Test vocoder: feed guitar DI, pitch shift -12st (octave down), verify tuning with a tuner plugin
   - Test vocoder: time stretch to 0.5x, verify pitch unchanged
   - Test phaser sweep: enable, set rate 0.5Hz, listen for sweep on sustained chord
   - Test phase randomize: set amount 1.0, verify lo-fi underwater texture
   - Test dry/wet: blend 50%, verify dry signal audible underneath
   - Test mode switch mid-playback: no clicks or crashes
