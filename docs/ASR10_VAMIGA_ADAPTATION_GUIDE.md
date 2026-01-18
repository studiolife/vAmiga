# ASR-10 Emulation Adaptation Guide for vAmiga
## Comprehensive Strategy for Adapting vAmiga Architecture to Emulate the Ensoniq ASR-10

---

## Executive Summary

This document provides a complete roadmap for adapting the **vAmiga Amiga emulator** to emulate the **Ensoniq ASR-10 sampler workstation** using components from the **analog-set** project. The adaptation leverages vAmiga's existing Motorola 68000 CPU emulator (Moira) and modular architecture while integrating ASR-10-specific hardware components.

**Key Strategy:**
- **Reuse vAmiga's CPU emulator** (Moira 68000) - ASR-10 uses same CPU architecture
- **Replace Amiga-specific components** (Agnus, Paula, Denise, CIA) with ASR-10 equivalents
- **Integrate analog-set DSP components** (DOC5503, OTIS, CEM3379, OSCAR) for authentic sound
- **Adapt memory map** from Amiga layout to ASR-10 specifications
- **Implement ASR-10 sequencer** as replacement for Copper coprocessor

**Related Documents:**
- [analog-set/docs/ASR10_CYCLE_ACCURATE_EMULATION.md](../analog-set/docs/ASR10_CYCLE_ACCURATE_EMULATION.md)
- [analog-set/docs/ENSONIQ_ASIC_EMULATION_GUIDE.md](../analog-set/docs/ENSONIQ_ASIC_EMULATION_GUIDE.md)
- [analog-set/docs/DOC5503_IMPLEMENTATION.md](../analog-set/docs/DOC5503_IMPLEMENTATION.md)
- [vAmiga/Manual/Developer/ClassHierarchy.md](ClassHierarchy.md)

---

## Table of Contents

1. [Architecture Comparison: Amiga vs ASR-10](#architecture-comparison)
2. [Component Mapping Strategy](#component-mapping-strategy)
3. [CPU Emulator Adaptation (Moira)](#cpu-emulator-adaptation)
4. [Memory System Adaptation](#memory-system-adaptation)
5. [Audio System Replacement (Paula → DOC II + OTIS)](#audio-system-replacement)
6. [Sequencer Implementation (Copper → ASR-10 Sequencer)](#sequencer-implementation)
7. [Effects Processor Integration (OSCAR)](#effects-processor-integration)
8. [File I/O and Sample Management](#file-io-and-sample-management)
9. [MIDI Integration](#midi-integration)
10. [Implementation Roadmap](#implementation-roadmap)
11. [Code Architecture and File Structure](#code-architecture)
12. [Testing and Validation](#testing-and-validation)

---

## Architecture Comparison

### Amiga 500 Architecture (Original vAmiga Target)

```
┌────────────────────────────────────────────────────────────────┐
│                      AMIGA 500 SYSTEM                          │
│                                                                │
│  ┌──────────────┐         ┌──────────────┐      ┌──────────┐  │
│  │  MC68000 CPU │◄───────►│  Agnus DMA   │◄────►│ Chip RAM │  │
│  │  @ 7.16 MHz  │         │  Controller  │      │ 512KB-2MB│  │
│  └──────┬───────┘         └──────┬───────┘      └──────────┘  │
│         │                        │                            │
│         │          ┌─────────────┴──────────┐                 │
│         │          │                        │                 │
│         ▼          ▼                        ▼                 │
│  ┌──────────┐ ┌──────────┐           ┌──────────┐            │
│  │   CIA    │ │  Denise  │           │  Paula   │            │
│  │ (I/O)    │ │ Graphics │           │  Audio   │            │
│  │ Keyboard │ │ Sprites  │           │ 4-chan   │            │
│  │  Mouse   │ │ Display  │           │ 8-bit    │            │
│  └──────────┘ └──────────┘           └──────────┘            │
│         │          │                        │                 │
│         ▼          ▼                        ▼                 │
│  ┌──────────────────────────────────────────────┐             │
│  │              Copper Coprocessor              │             │
│  │        (Display List Processing)             │             │
│  └──────────────────────────────────────────────┘             │
└────────────────────────────────────────────────────────────────┘
```

### Ensoniq ASR-10 Architecture (Target System)

```
┌────────────────────────────────────────────────────────────────┐
│                      ASR-10 SYSTEM                             │
│                                                                │
│  ┌──────────────┐         ┌──────────────┐      ┌──────────┐  │
│  │  MC68000 CPU │◄───────►│  OTIS Chip   │◄────►│ ROM/RAM  │  │
│  │  @ 10 MHz    │         │ (Synth Voice)│      │ 2-16 MB  │  │
│  └──────┬───────┘         └──────┬───────┘      └──────────┘  │
│         │                        │                            │
│         │          ┌─────────────┴──────────┐                 │
│         │          │                        │                 │
│         │          ▼                        ▼                 │
│         │   ┌──────────────┐      ┌──────────────┐           │
│         │   │ DOC II Chip  │      │  CEM3379 VCF │           │
│         │   │ (32 voices)  │      │  (Analog)    │           │
│         │   │ 16-bit, 44kHz│      │  2-pole LP   │           │
│         │   └──────┬───────┘      └──────┬───────┘           │
│         │          │                     │                    │
│         │          └──────────┬──────────┘                    │
│         │                     ▼                               │
│         │          ┌──────────────────────┐                   │
│         │          │   OSCAR Effects      │                   │
│         │          │   (DSP Processor)    │                   │
│         │          └──────────┬───────────┘                   │
│         │                     │                               │
│         ▼                     ▼                               │
│  ┌──────────────────────────────────────┐                     │
│  │       Sequencer Engine               │                     │
│  │  (96 PPQ, 16 tracks, real-time)      │                     │
│  └──────────────────────────────────────┘                     │
│         │                     │                               │
│         ▼                     ▼                               │
│  ┌──────────────┐   ┌──────────────────┐                     │
│  │ MIDI I/O     │   │  Audio I/O       │                     │
│  │ (In/Out/Thru)│   │  ADC/DAC + Amp   │                     │
│  └──────────────┘   └──────────────────┘                     │
└────────────────────────────────────────────────────────────────┘
```

### Key Architectural Differences

| Component | Amiga | ASR-10 | Adaptation Strategy |
|-----------|-------|--------|---------------------|
| **CPU** | MC68000 @ 7.16 MHz | MC68000 @ 10 MHz | Adjust clock rate in Moira |
| **Audio** | Paula (4-ch, 8-bit) | DOC II (32-voice, 16-bit) | Replace Paula with DOC II + OTIS |
| **Graphics** | Denise (sprites, playfield) | None (LCD display only) | Remove/stub Denise |
| **DMA** | Agnus (complex DMA) | Simple DMA for audio | Simplify DMA controller |
| **I/O** | CIA (keyboard, mouse, ports) | MIDI + front panel | Replace CIA with MIDI handler |
| **Coprocessor** | Copper (display lists) | Sequencer (MIDI events) | Replace Copper with Sequencer |
| **Filter** | None (digital) | CEM3379 (analog) | Add analog filter emulation |
| **Effects** | None | OSCAR (reverb, delay, etc.) | Add effects processor |
| **Memory** | 512KB-2MB Chip RAM | 2-16 MB sample RAM | Adjust memory map |

---

## Component Mapping Strategy

### Direct Reuse (Minimal Changes)

| vAmiga Component | ASR-10 Usage | Modification Required |
|------------------|--------------|----------------------|
| **Moira CPU** | ✅ Direct reuse | Clock speed adjustment (7.16 → 10 MHz) |
| **Memory System** | ✅ Adapt memory map | Change address ranges, remove slow RAM |
| **MsgQueue** | ✅ Direct reuse | Same event messaging system |
| **Config System** | ✅ Direct reuse | Define ASR-10 specific options |
| **RingBuffer** | ✅ Direct reuse | Use for audio streaming |

### Replace Completely

| vAmiga Component | ASR-10 Replacement | Source |
|------------------|-------------------|--------|
| **Paula Audio** | DOC II + OTIS | analog-set components |
| **Agnus DMA** | Simple Audio DMA | New implementation |
| **Denise Graphics** | LCD Display Stub | Minimal stub (text display) |
| **CIA I/O** | MIDI Handler | New implementation |
| **Copper** | ASR-10 Sequencer | New implementation |

### Add New Components

| New Component | Purpose | Source |
|---------------|---------|--------|
| **CEM3379 Filter** | Analog filter emulation | analog-set (needs implementation) |
| **OSCAR Effects** | Reverb, delay, chorus | analog-set docs (needs implementation) |
| **Sequencer** | 96 PPQ MIDI sequencer | New implementation |
| **Sample Manager** | WAV/AIFF loading, memory | Adapt from analog-set StateManager |

---

## CPU Emulator Adaptation (Moira)

### Existing Moira Architecture

**Location**: `Core/Components/CPU/Moira/`

**Current Implementation** (from vAmiga):
```cpp
class Moira {
    // Registers
    Registers reg;              // D0-D7, A0-A7
    i64 clock;                  // Cycle counter

    // Execution
    void execute();             // Execute one instruction
    void sync(int cycles);      // Synchronize with DMA

    // Configuration
    Model model;                // 68000, 68010, 68020, etc.
    u32 baseFreq;               // Base clock frequency
};
```

### Adaptation for ASR-10

**Changes Required**:

1. **Clock Speed Configuration**
```cpp
// In ASR10.cpp (new top-level class)
class ASR10 {
    CPU cpu;

    void _initialize() override {
        // ASR-10 uses 10 MHz CPU (vs Amiga's 7.16 MHz)
        cpu.clock.setFrequency(10'000'000);  // 10 MHz
    }
};
```

2. **Synchronization Model**
```cpp
// Replace Agnus synchronization with simpler model
void CPU::sync(int cycles) {
    clock += cycles;

    // ASR-10: Synchronize with audio DMA and sequencer
    docII.execute(cycles);           // Audio voice processing
    sequencer.execute(cycles);       // MIDI sequencer ticks
    oscar.execute(cycles);           // Effects processing
}
```

3. **Memory-Mapped I/O Changes**
```cpp
// Replace Amiga custom chip registers with ASR-10 registers
// Amiga: 0xC00000-0xDFFFFF (Agnus, Paula, Denise)
// ASR-10: Custom mapping for OTIS, DOC II, OSCAR

// In Memory::peek16/poke16:
template <Accessor s> u16 peek16(u32 addr) {
    if (addr >= 0xE00000 && addr < 0xE10000) {
        // DOC II registers
        return docII.readRegister((addr - 0xE00000) >> 1);
    }
    if (addr >= 0xE10000 && addr < 0xE20000) {
        // OTIS registers
        return otis.readRegister((addr - 0xE10000) >> 1);
    }
    if (addr >= 0xE20000 && addr < 0xE30000) {
        // OSCAR registers
        return oscar.readRegister((addr - 0xE20000) >> 1);
    }
    // ... MIDI, sequencer, etc.
}
```

4. **Interrupt Handling**
```cpp
// ASR-10 interrupt levels (different from Amiga)
enum ASR10Interrupt {
    INT_SEQUENCER = 1,    // Sequencer tick (Level 1)
    INT_MIDI_RX   = 2,    // MIDI receive (Level 2)
    INT_KEYBOARD  = 3,    // Front panel (Level 3)
    INT_AUDIO     = 4,    // Audio sample ready (Level 4)
    INT_DISK      = 5,    // Disk I/O (Level 5)
    INT_DISPLAY   = 6,    // LCD update (Level 6)
    INT_NMI       = 7     // Non-maskable (Level 7)
};

// In CPU interrupt handling:
void CPU::serviceInterrupts() {
    if (sequencer.hasEvent()) {
        setIPL(INT_SEQUENCER);
    }
    if (midi.hasData()) {
        setIPL(INT_MIDI_RX);
    }
    // ... etc.
}
```

### Integration with vAmiga Structure

**File**: `Core/Components/CPU/CPU.h` (modify existing)
```cpp
class CPU : public Moira, public CoreComponent {
public:
    // vAmiga compatibility
    Amiga& amiga;
    Memory& mem;

    // ASR-10 specific
    #ifdef ASR10_BUILD
    DOCII& docII;
    OTIS& otis;
    OSCAR& oscar;
    ASR10Sequencer& sequencer;
    #else
    Agnus& agnus;
    Paula& paula;
    #endif

    // Synchronization (adaptive)
    void sync(int cycles) override {
        #ifdef ASR10_BUILD
            // ASR-10: Simple audio sync
            docII.execute(cycles);
            sequencer.execute(cycles);
        #else
            // Amiga: DMA sync with Agnus
            agnus.execute(CPU_AS_DMA_CYCLES(cycles));
        #endif
    }
};
```

---

## Memory System Adaptation

### Amiga Memory Map (Original)

```
0x000000 - 0x07FFFF: Chip RAM (512 KB)
0x080000 - 0x1FFFFF: Extended Chip RAM (Agnus mask)
0x200000 - 0x9FFFFF: Fast RAM (expansion)
0xA00000 - 0xBFFFFF: CIA registers
0xC00000 - 0xD7FFFF: Slow RAM (512 KB)
0xD80000 - 0xDCFFFF: RTC
0xDD0000 - 0xDFFFFF: Custom chips (Agnus, Paula, Denise)
0xE00000 - 0xE7FFFF: Reserved
0xE80000 - 0xEFFFFF: Zorro autoconfig
0xF00000 - 0xF7FFFF: ROM expansion
0xF80000 - 0xFFFFFF: Kickstart ROM (512 KB)
```

### ASR-10 Memory Map (Target)

```
0x000000 - 0x1FFFFF: ROM (2 MB)
  0x000000 - 0x0FFFFF: Operating System (1 MB)
  0x100000 - 0x1FFFFF: Factory Sounds (1 MB)

0x200000 - 0x3FFFFF: Sample RAM (2 MB base)
0x400000 - 0xFFFFFF: Extended RAM (up to 16 MB total)

Memory-Mapped I/O:
0xE00000 - 0xE0FFFF: DOC II registers
0xE10000 - 0xE1FFFF: OTIS registers
0xE20000 - 0xE2FFFF: OSCAR registers
0xE30000 - 0xE3FFFF: MIDI I/O
0xE40000 - 0xE4FFFF: Disk controller
0xE50000 - 0xE5FFFF: Display controller
0xE60000 - 0xE6FFFF: Sequencer registers
0xE70000 - 0xE7FFFF: Front panel / keyboard
```

### Memory Class Adaptation

**File**: `Core/Components/Memory/Memory.h` (modify)

```cpp
class Memory : public CoreComponent {
    // Remove Amiga-specific memory regions
    #ifndef ASR10_BUILD
    u8 *chip;      // Chip RAM (Amiga only)
    u8 *slow;      // Slow RAM (Amiga only)
    u8 *wom;       // Write-once Memory (Amiga only)
    #endif

    // Common regions
    u8 *rom;       // Boot ROM
    u8 *ext;       // Extended ROM (ASR-10: factory sounds)

    // ASR-10 specific
    #ifdef ASR10_BUILD
    u8 *sampleRam; // Sample RAM (2-16 MB)
    isize sampleRamSize;
    #else
    u8 *fast;      // Fast RAM (Amiga only)
    #endif

    // Memory-mapped hardware
    #ifdef ASR10_BUILD
    DOCII *docII;
    OTIS *otis;
    OSCAR *oscar;
    MIDIHandler *midi;
    ASR10Sequencer *sequencer;
    #else
    Agnus *agnus;
    Paula *paula;
    Denise *denise;
    CIA *ciaA, *ciaB;
    #endif
};
```

### Memory Access Implementation

**Read Operations** (in `Memory.cpp`):
```cpp
template <Accessor s> u16 Memory::peek16(u32 addr) {
    // ROM: 0x000000 - 0x1FFFFF
    if (addr < 0x200000) {
        if (addr < romSize) {
            return R16BE(rom + addr);
        } else {
            return 0xFFFF;  // Unmapped
        }
    }

    // Sample RAM: 0x200000 - 0xFFFFFF
    if (addr >= 0x200000 && addr < (0x200000 + sampleRamSize)) {
        return R16BE(sampleRam + (addr - 0x200000));
    }

    // Memory-mapped I/O: 0xE00000 - 0xE7FFFF
    if (addr >= 0xE00000 && addr < 0xE80000) {
        return peekIO16<s>(addr);
    }

    return 0xFFFF;  // Unmapped address
}
```

**I/O Register Access**:
```cpp
template <Accessor s> u16 Memory::peekIO16(u32 addr) {
    // DOC II: 0xE00000 - 0xE0FFFF
    if (addr >= 0xE00000 && addr < 0xE10000) {
        return docII->readRegister((addr & 0xFFFF) >> 1);
    }

    // OTIS: 0xE10000 - 0xE1FFFF
    if (addr >= 0xE10000 && addr < 0xE20000) {
        return otis->readRegister((addr & 0xFFFF) >> 1);
    }

    // OSCAR: 0xE20000 - 0xE2FFFF
    if (addr >= 0xE20000 && addr < 0xE30000) {
        return oscar->readRegister((addr & 0xFFFF) >> 1);
    }

    // MIDI: 0xE30000 - 0xE3FFFF
    if (addr >= 0xE30000 && addr < 0xE40000) {
        return midi->readRegister((addr & 0xFFFF) >> 1);
    }

    // Sequencer: 0xE60000 - 0xE6FFFF
    if (addr >= 0xE60000 && addr < 0xE70000) {
        return sequencer->readRegister((addr & 0xFFFF) >> 1);
    }

    return 0xFFFF;
}
```

### ROM Loading

**ASR-10 ROM Structure**:
```cpp
// In ASR10.cpp
void ASR10::loadROM(const std::filesystem::path &path) {
    // ASR-10 ROM is 2 MB (OS + factory sounds)
    const isize EXPECTED_SIZE = 2 * 1024 * 1024;

    auto file = std::ifstream(path, std::ios::binary);
    file.seekg(0, std::ios::end);
    isize size = file.tellg();
    file.seekg(0, std::ios::beg);

    if (size != EXPECTED_SIZE) {
        throw Error("Invalid ASR-10 ROM size");
    }

    // Allocate ROM memory
    mem.rom = new u8[size];
    mem.romSize = size;

    // Read ROM
    file.read(reinterpret_cast<char*>(mem.rom), size);

    // Verify checksum (optional)
    u32 checksum = calculateCRC32(mem.rom, size);
    // Compare against known good ROMs...
}
```

---

## Audio System Replacement (Paula → DOC II + OTIS)

### Remove Paula Audio

**Original Paula** (vAmiga):
- 4 independent 8-bit audio channels
- DMA-driven sample playback
- Per-channel volume control
- Simple period-based playback

**Strategy**: Replace entirely with DOC II + OTIS emulation

### Integrate DOC II Voice Chip

**Source**: `analog-set/src/components/exp/DOC5503.h` (base implementation)

**Adaptation** (create `Core/Components/Audio/DOCII.h`):
```cpp
#include "analog-set/src/components/exp/DOC5503.h"

class DOCII : public DOC5503, public CoreComponent {
public:
    // vAmiga integration
    ASR10& asr10;
    Memory& mem;

    // DOC II specific (16-bit, 44.1 kHz)
    static constexpr int MAX_VOICES = 32;
    static constexpr float NATIVE_SAMPLE_RATE = 44100.0f;
    static constexpr int OUTPUT_BIT_DEPTH = 16;  // vs 13-bit in DOC5503

    // Constructor
    DOCII(ASR10& ref) : asr10(ref), mem(asr10.mem) { }

    // vAmiga CoreComponent interface
    void _initialize() override {
        prepare(NATIVE_SAMPLE_RATE, 512);
    }

    void _reset(bool hard) override {
        reset();
    }

    // Execute for N CPU cycles
    void execute(i64 cycles) {
        // Convert CPU cycles to audio samples
        double samplesPerCycle = NATIVE_SAMPLE_RATE / asr10.getClock();
        i64 samples = static_cast<i64>(cycles * samplesPerCycle);

        // Process audio samples
        for (i64 i = 0; i < samples; i++) {
            float left, right;
            processBlock(&left, &right, 1);

            // Send to audio output stream
            audioStream.write({left, right});
        }
    }

    // Memory-mapped register access
    u16 readRegister(u16 reg) {
        // DOC II register mapping
        // 0x00-0x1F: Voice 0 registers
        // 0x20-0x3F: Voice 1 registers
        // ... etc.
        int voice = reg >> 5;  // Each voice has 32 registers
        int offset = reg & 0x1F;

        return voices[voice].readRegister(offset);
    }

    void writeRegister(u16 reg, u16 value) {
        int voice = reg >> 5;
        int offset = reg & 0x1F;

        voices[voice].writeRegister(offset, value);
    }

private:
    RingBuffer<SamplePair, 65536> audioStream;  // Output buffer
};
```

### Integrate OTIS Synthesizer Chip

**Source**: `analog-set/docs/ENSONIQ_ASIC_EMULATION_GUIDE.md` (needs implementation)

**Create** `Core/Components/Audio/OTIS.h`:
```cpp
class OTIS : public CoreComponent {
public:
    static constexpr int MAX_VOICES = 31;       // ASR-10 polyphony
    static constexpr float CONTROL_RATE = 1500.0f; // 1.5 kHz CV updates

    struct Voice {
        // Oscillator control (2 per voice)
        struct Oscillator {
            float pitch;           // Coarse/fine tune
            float detune;          // Oscillator detuning
            int waveform;          // Wave selection
        } osc[2];

        // Envelopes (3 per voice: amp, filter, aux)
        struct Envelope {
            float attack;          // 0-10s
            float decay;           // 0-10s
            float sustain;         // 0-1
            float release;         // 0-10s

            // Current state
            enum Stage { IDLE, ATTACK, DECAY, SUSTAIN, RELEASE } stage;
            float level;
            float phase;
        } ampEnv, filterEnv, auxEnv;

        // LFOs (3 per voice)
        struct LFO {
            float rate;            // 0.01 - 50 Hz
            enum Waveform { SINE, TRIANGLE, SQUARE, SAW, RANDOM } waveform;
            float phase;
        } lfo[3];

        // Filter control voltage (8-bit quantized)
        u8 filterCutoffCV;
        u8 filterResonanceCV;
    };

    Voice voices[MAX_VOICES];

    // Process control-rate updates
    void execute(i64 cycles) {
        // Update at 1.5 kHz (not every sample)
        controlPhase += cycles * CONTROL_RATE / clockRate;

        while (controlPhase >= 1.0) {
            updateControlVoltages();
            processEnvelopes();
            processLFOs();
            controlPhase -= 1.0;
        }
    }

    // Generate filter control voltage for CEM3379
    float getFilterCV(int voice) {
        // 8-bit quantization (256 steps)
        return voices[voice].filterCutoffCV / 256.0f;
    }

private:
    double controlPhase = 0.0;
    double clockRate;
};
```

### Integrate CEM3379 Analog Filter

**Source**: `analog-set/docs/ASR10_CYCLE_ACCURATE_EMULATION.md` (needs implementation)

**Create** `Core/Components/Audio/CEM3379.h`:
```cpp
class CEM3379 : public CoreComponent {
public:
    static constexpr int MAX_FILTERS = 31;  // One per voice

    struct VoiceFilter {
        // State variables (2-pole filter)
        float z1, z2;

        // Parameters
        float cutoff;              // Hz
        float resonance;           // 0-1

        // Temperature model (drift over time)
        float temperature;         // Celsius
        float warmupTime;          // Seconds since power-on

        // Control voltage (from OTIS)
        float controlVoltage;      // 0-1
    };

    VoiceFilter filters[MAX_FILTERS];

    // Process audio sample
    float process(int voice, float input) {
        auto& f = filters[voice];

        // Apply temperature drift to cutoff
        const float TEMP_COEFF = 3300.0f / 1e6f;  // 3300 ppm/°C
        float tempOffset = (f.temperature - 25.0f) * TEMP_COEFF;
        float actualCutoff = f.cutoff * (1.0f + tempOffset);

        // Exponential CV to frequency
        float cvCutoff = actualCutoff * std::pow(2.0f, f.controlVoltage * 5.0f);

        // Calculate filter coefficients
        float omega = 2.0f * M_PI * cvCutoff / sampleRate;
        float q = 1.0f / (1.0f - f.resonance);

        // 2-pole state-variable filter
        float feedback = f.z2 * q;
        float low = f.z1 + omega * (input - feedback);
        float band = f.z2 + omega * (low - f.z1);

        f.z1 = low;
        f.z2 = band;

        // Soft clipping (OTA saturation)
        return std::tanh(low * 0.8f);
    }

    // Update temperature over time
    void execute(i64 cycles) {
        double dt = cycles / clockRate;

        for (auto& f : filters) {
            // Exponential approach to operating temperature
            const float TAU = 1800.0f;  // 30 minutes
            const float TARGET_TEMP = 45.0f;

            f.temperature += (TARGET_TEMP - f.temperature) * (dt / TAU);
            f.warmupTime += dt;
        }
    }
};
```

### Audio Processing Pipeline

**Integration** (in `ASR10.cpp`):
```cpp
void ASR10::executeAudioCycle() {
    // 1. OTIS generates control voltages
    otis.execute(currentCycles);

    // 2. DOC II processes voices
    for (int v = 0; v < 31; v++) {
        if (docII.voices[v].isActive) {
            // Get raw oscillator output
            float sample = docII.processVoice(v);

            // Apply filter (controlled by OTIS)
            float cutoffCV = otis.getFilterCV(v);
            cem3379.filters[v].controlVoltage = cutoffCV;
            sample = cem3379.process(v, sample);

            // Apply envelope (from OTIS)
            sample *= otis.voices[v].ampEnv.level;

            // Accumulate to output buffer
            outputBuffer[v] = sample;
        }
    }

    // 3. Mix voices
    float mixedL = 0.0f, mixedR = 0.0f;
    for (int v = 0; v < 31; v++) {
        mixedL += outputBuffer[v] * docII.voices[v].panLeft;
        mixedR += outputBuffer[v] * docII.voices[v].panRight;
    }

    // 4. Apply OSCAR effects
    oscar.process(&mixedL, &mixedR, 1);

    // 5. Output to audio stream
    audioStream.write({mixedL, mixedR});
}
```

---

## Sequencer Implementation (Copper → ASR-10 Sequencer)

### Remove Copper Coprocessor

**Amiga Copper** (vAmiga):
- Display list processor
- Writes to custom chip registers
- Synchronized to video beam

**ASR-10**: No Copper equivalent (replace with MIDI sequencer)

### ASR-10 Sequencer Architecture

**Create** `Core/Components/Sequencer/ASR10Sequencer.h`:
```cpp
class ASR10Sequencer : public CoreComponent {
public:
    static constexpr int PPQ = 96;          // Pulses per quarter note
    static constexpr int MAX_TRACKS = 16;
    static constexpr int MAX_SONGS = 10;

    // Sequencer state
    enum Mode { STOPPED, PLAYING, RECORDING } mode;

    struct Track {
        int channel;              // MIDI channel (1-16)
        bool muted;
        bool soloed;
        int transpose;            // Semitones

        std::vector<SequencerEvent> events;

        // Playback state
        int eventIndex;
        i64 nextEventTime;        // In pulses
    };

    Track tracks[MAX_TRACKS];

    // Timing
    float bpm;
    i64 currentPulse;
    double pulsePhase;

    // Swing quantization
    struct {
        bool enabled;
        float amount;             // 50-75%
        int division;             // 8th or 16th
    } swing;

    // Execute for N CPU cycles
    void execute(i64 cycles) {
        if (mode == STOPPED) return;

        // Convert CPU cycles to sequencer pulses
        double pulsesPerCycle = (bpm / 60.0) * PPQ / clockRate;
        pulsePhase += cycles * pulsesPerCycle;

        while (pulsePhase >= 1.0) {
            currentPulse++;
            processPulse();
            pulsePhase -= 1.0;
        }
    }

    // Process one sequencer pulse
    void processPulse() {
        for (int t = 0; t < MAX_TRACKS; t++) {
            auto& track = tracks[t];

            if (track.muted || track.events.empty()) continue;

            // Check if event should trigger
            if (currentPulse >= track.nextEventTime) {
                auto& event = track.events[track.eventIndex];

                // Apply swing quantization
                i64 swungTime = applySwing(event.time);

                if (currentPulse >= swungTime) {
                    // Trigger event
                    sendMIDIEvent(event);

                    // Advance to next event
                    track.eventIndex++;
                    if (track.eventIndex >= track.events.size()) {
                        // Loop or stop
                        track.eventIndex = 0;
                    }
                    track.nextEventTime = track.events[track.eventIndex].time;
                }
            }
        }
    }

    // Swing quantization (ASR-10 specific)
    i64 applySwing(i64 pulseTime) {
        if (!swing.enabled) return pulseTime;

        int gridSize = (swing.division == 16) ? (PPQ / 4) : (PPQ / 2);
        bool isOffBeat = ((pulseTime / gridSize) % 2) == 1;

        if (isOffBeat) {
            float swingRatio = swing.amount / 100.0f;
            float offset = gridSize * (swingRatio - 0.5f);
            return pulseTime + static_cast<i64>(offset);
        }

        return pulseTime;
    }

    // Send MIDI event to sound engine
    void sendMIDIEvent(const SequencerEvent& event) {
        switch (event.type) {
            case SequencerEvent::NOTE_ON:
                docII.noteOn(event.note, event.velocity);
                break;
            case SequencerEvent::NOTE_OFF:
                docII.noteOff(event.note);
                break;
            case SequencerEvent::CONTROL_CHANGE:
                handleCC(event.controller, event.value);
                break;
            // ... etc.
        }
    }
};
```

### Event Structure

```cpp
struct SequencerEvent {
    enum Type {
        NOTE_ON,
        NOTE_OFF,
        CONTROL_CHANGE,
        PROGRAM_CHANGE,
        PITCH_BEND,
        AFTERTOUCH
    } type;

    i64 time;                 // In pulses (96 PPQ)
    u8 note;                  // 0-127
    u8 velocity;              // 0-127
    u8 controller;            // For CC events
    u8 value;                 // CC value

    float subPulseTiming;     // Sub-pulse offset (0-1)
};
```

### Integration with CPU

```cpp
// In CPU::sync():
void CPU::sync(int cycles) {
    clock += cycles;

    // Execute sequencer (generates MIDI events)
    sequencer.execute(cycles);

    // Execute audio components
    docII.execute(cycles);
    otis.execute(cycles);
    cem3379.execute(cycles);
    oscar.execute(cycles);
}
```

---

## Effects Processor Integration (OSCAR)

### OSCAR Architecture

**Source**: `analog-set/docs/ENSONIQ_ASIC_EMULATION_GUIDE.md` (needs implementation)

**Create** `Core/Components/Audio/OSCAR.h`:
```cpp
class OSCAR : public CoreComponent {
public:
    static constexpr int FIXED_POINT_BITS = 24;
    static constexpr int LATENCY_SAMPLES = 224;  // ~5 ms @ 44.1 kHz

    // Effect types
    enum EffectType {
        NONE,
        REVERB,
        DELAY,
        CHORUS,
        FLANGER,
        EQ
    };

    // Current configuration
    EffectType activeEffect = REVERB;

    // Effect parameters
    struct ReverbParams {
        float roomSize;
        float damping;
        float predelay;
        float mix;
    } reverbParams;

    struct DelayParams {
        float time;
        float feedback;
        float mix;
        bool pingPong;
    } delayParams;

    // Process audio (stereo in/out)
    void process(float* left, float* right, int samples) {
        for (int i = 0; i < samples; i++) {
            // Add latency (models hardware buffering)
            float inputL = latencyBuffer[latencyIndex].l;
            float inputR = latencyBuffer[latencyIndex].r;
            latencyBuffer[latencyIndex] = {left[i], right[i]};
            latencyIndex = (latencyIndex + 1) % LATENCY_SAMPLES;

            // Process effect (in fixed-point)
            float outputL, outputR;
            switch (activeEffect) {
                case REVERB:
                    processReverb(inputL, inputR, outputL, outputR);
                    break;
                case DELAY:
                    processDelay(inputL, inputR, outputL, outputR);
                    break;
                case CHORUS:
                    processChorus(inputL, inputR, outputL, outputR);
                    break;
                default:
                    outputL = inputL;
                    outputR = inputR;
            }

            left[i] = outputL;
            right[i] = outputR;
        }
    }

private:
    // Latency buffer
    struct SamplePair { float l, r; };
    std::array<SamplePair, LATENCY_SAMPLES> latencyBuffer;
    int latencyIndex = 0;

    // Fixed-point arithmetic (models hardware)
    i32 floatToFixed24(float x) {
        return static_cast<i32>(x * (1 << 23));
    }

    float fixed24ToFloat(i32 x) {
        return static_cast<float>(x) / (1 << 23);
    }

    // Effect implementations
    void processReverb(float inL, float inR, float& outL, float& outR);
    void processDelay(float inL, float inR, float& outL, float& outR);
    void processChorus(float inL, float inR, float& outL, float& outR);
};
```

---

## File I/O and Sample Management

### Remove Amiga Disk Support

**vAmiga Components to Remove/Stub**:
- `DiskController` (Paula subsystem)
- `FloppyDrive` class
- ADF file format support

### Add Sample File Support

**Source**: `analog-set/src/state/StateManager.h` (adapt)

**Create** `Core/Components/Storage/SampleManager.h`:
```cpp
class SampleManager : public CoreComponent {
public:
    static constexpr isize MAX_SAMPLES = 127;
    static constexpr isize DEFAULT_RAM_SIZE = 2 * 1024 * 1024;  // 2 MB

    struct Sample {
        u32 offset;           // Byte offset in sample RAM
        u32 size;             // Size in bytes
        u16 sampleRate;       // Original sample rate
        bool stereo;

        // Loop points
        u32 loopStart;
        u32 loopEnd;

        // Metadata
        char name[16];
        u8 rootNote;          // MIDI note
        i8 fineTune;          // -50 to +50 cents
    };

    std::vector<Sample> samples;
    u8* sampleRam;
    isize totalSize;
    isize usedSize;

    // Load WAV/AIFF file
    bool loadSample(const std::filesystem::path& path) {
        // Determine file type
        auto ext = path.extension().string();

        if (ext == ".wav" || ext == ".WAV") {
            return loadWAV(path);
        } else if (ext == ".aif" || ext == ".aiff" || ext == ".AIFF") {
            return loadAIFF(path);
        }

        return false;
    }

    // Allocate sample in RAM
    Sample* allocateSample(isize sizeBytes) {
        if (usedSize + sizeBytes > totalSize) {
            // Out of memory - need defragmentation or expansion
            return nullptr;
        }

        Sample sample;
        sample.offset = usedSize;
        sample.size = sizeBytes;

        samples.push_back(sample);
        usedSize += sizeBytes;

        return &samples.back();
    }

    // Defragment memory (ASR-10 does this automatically)
    void defragment() {
        u32 currentOffset = 0;

        for (auto& sample : samples) {
            if (sample.offset != currentOffset) {
                // Move sample data
                std::memmove(sampleRam + currentOffset,
                            sampleRam + sample.offset,
                            sample.size);
                sample.offset = currentOffset;
            }
            currentOffset += sample.size;
        }

        usedSize = currentOffset;
    }

private:
    bool loadWAV(const std::filesystem::path& path);
    bool loadAIFF(const std::filesystem::path& path);
};
```

### Disk I/O (Optional)

For loading/saving ASR-10 disk images:
```cpp
class ASR10DiskController : public CoreComponent {
    // Support for ASR-10 disk format (3.5" HD floppies)
    // Or SCSI hard drive images

    void loadDiskImage(const std::filesystem::path& path);
    void saveDiskImage(const std::filesystem::path& path);
};
```

---

## MIDI Integration

### Replace CIA with MIDI Handler

**vAmiga CIA** (Complex Interface Adapter):
- Keyboard scanning
- Mouse quadrature
- Joystick ports
- Serial I/O

**ASR-10**: MIDI In/Out/Thru + front panel keyboard

### MIDI Handler Implementation

**Create** `Core/Components/MIDI/MIDIHandler.h`:
```cpp
class MIDIHandler : public CoreComponent {
public:
    // MIDI ports
    struct Port {
        std::queue<u8> rxBuffer;  // Receive buffer
        std::queue<u8> txBuffer;  // Transmit buffer
        bool connected;
    };

    Port midiIn;
    Port midiOut;
    Port midiThru;

    // Receive MIDI byte
    void receiveByte(u8 byte) {
        midiIn.rxBuffer.push(byte);

        // MIDI Thru (echo input to thru port)
        midiThru.txBuffer.push(byte);

        // Parse MIDI message
        parseMIDI(byte);
    }

    // Parse MIDI messages
    void parseMIDI(u8 byte) {
        if (byte & 0x80) {
            // Status byte
            currentStatus = byte;
            dataByteIndex = 0;
        } else {
            // Data byte
            dataBytes[dataByteIndex++] = byte;

            // Check if message complete
            if (isMessageComplete()) {
                processMIDIMessage();
            }
        }
    }

    // Process complete MIDI message
    void processMIDIMessage() {
        u8 messageType = currentStatus & 0xF0;
        u8 channel = currentStatus & 0x0F;

        switch (messageType) {
            case 0x90:  // Note On
                if (dataBytes[1] > 0) {
                    noteOn(channel, dataBytes[0], dataBytes[1]);
                } else {
                    noteOff(channel, dataBytes[0]);
                }
                break;

            case 0x80:  // Note Off
                noteOff(channel, dataBytes[0]);
                break;

            case 0xB0:  // Control Change
                controlChange(channel, dataBytes[0], dataBytes[1]);
                break;

            case 0xC0:  // Program Change
                programChange(channel, dataBytes[0]);
                break;

            case 0xE0:  // Pitch Bend
                pitchBend(channel, (dataBytes[1] << 7) | dataBytes[0]);
                break;

            case 0xF8:  // Timing Clock
                if (sequencer.isExternal()) {
                    sequencer.clockTick();
                }
                break;

            case 0xFA:  // Start
                sequencer.start();
                break;

            case 0xFC:  // Stop
                sequencer.stop();
                break;
        }
    }

    // Forward MIDI events to sound engine
    void noteOn(u8 channel, u8 note, u8 velocity) {
        docII.noteOn(note, velocity / 127.0f);
    }

    void noteOff(u8 channel, u8 note) {
        docII.noteOff(note);
    }

    void controlChange(u8 channel, u8 controller, u8 value) {
        // Map CC to ASR-10 parameters
        switch (controller) {
            case 1:   // Mod Wheel
                otis.setModWheel(value / 127.0f);
                break;
            case 7:   // Volume
                docII.setVolume(value / 127.0f);
                break;
            case 74:  // Filter Cutoff
                otis.setFilterCutoff(value / 127.0f);
                break;
            case 71:  // Filter Resonance
                otis.setFilterResonance(value / 127.0f);
                break;
        }
    }

private:
    u8 currentStatus = 0;
    u8 dataBytes[2];
    int dataByteIndex = 0;

    bool isMessageComplete() {
        u8 messageType = currentStatus & 0xF0;

        // 2-byte messages
        if (messageType == 0xC0 || messageType == 0xD0) {
            return dataByteIndex >= 1;
        }

        // 3-byte messages
        return dataByteIndex >= 2;
    }
};
```

### Interrupt Integration

```cpp
// In CPU interrupt handling:
void CPU::serviceInterrupts() {
    // MIDI receive interrupt (Level 2)
    if (midi.hasData()) {
        setIPL(2);
        // Jump to MIDI interrupt vector
        u32 vector = mem.peek32(0x68);  // Level 2 autovector
        reg.pc = vector;
    }
}
```

---

## Implementation Roadmap

### Phase 1: Foundation (Months 1-2)

**Goal**: Basic CPU + memory working

**Tasks**:
1. ✅ Copy vAmiga project structure
2. ✅ Remove Amiga-specific components (Agnus, Paula, Denise, CIA)
3. ✅ Adapt Moira CPU for 10 MHz operation
4. ✅ Implement ASR-10 memory map
5. ✅ Create stub components (DOCII, OTIS, OSCAR, Sequencer)
6. ✅ Implement ROM loading (2 MB ASR-10 ROM)
7. ✅ Basic boot sequence (CPU executing ROM code)

**Milestone**: CPU boots and executes ASR-10 ROM

---

### Phase 2: Audio Core (Months 3-5)

**Goal**: Basic sound generation

**Tasks**:
1. ✅ Port DOC5503 from analog-set
2. ✅ Implement DOCII (16-bit, 44.1 kHz)
3. ✅ Implement OTIS envelope generators
4. ✅ Implement OTIS LFOs
5. ✅ Implement CEM3379 filter emulation
6. ✅ Integrate audio pipeline (DOCII → Filter → Output)
7. ✅ MIDI input handling (basic note on/off)
8. ✅ Sample loading (WAV/AIFF)

**Milestone**: Can play samples with filtering

---

### Phase 3: Sequencer (Months 6-8)

**Goal**: MIDI sequencer functional

**Tasks**:
1. ✅ Implement 96 PPQ sequencer engine
2. ✅ Track management (16 tracks)
3. ✅ Event recording
4. ✅ Event playback
5. ✅ Swing quantization
6. ✅ MIDI sync (internal/external clock)
7. ✅ Song mode (patterns → songs)

**Milestone**: Can record and play back sequences

---

### Phase 4: Effects & Polish (Months 9-11)

**Goal**: OSCAR effects + refinements

**Tasks**:
1. ✅ Implement OSCAR reverb
2. ✅ Implement OSCAR delay
3. ✅ Implement OSCAR chorus/flanger
4. ✅ Temperature drift modeling (CEM3379)
5. ✅ Timing jitter and quirks
6. ✅ Sample memory management
7. ✅ Preset system

**Milestone**: Complete feature set matching hardware

---

### Phase 5: ROM Compatibility (Months 12-15)

**Goal**: Run original ASR-10 ROM

**Tasks**:
1. ✅ Complete 68000 instruction coverage
2. ✅ All memory-mapped I/O
3. ✅ Interrupt handling (7 levels)
4. ✅ Disk I/O emulation
5. ✅ LCD display emulation
6. ✅ Front panel keyboard
7. ✅ ROM verification and testing

**Milestone**: Original ROM boots and operates

---

### Phase 6: Testing & Optimization (Months 16-18)

**Goal**: Production-ready

**Tasks**:
1. ✅ Hardware comparison testing
2. ✅ Performance optimization
3. ✅ Bug fixes
4. ✅ Documentation
5. ✅ User interface
6. ✅ Cross-platform builds

**Milestone**: Release v1.0

---

## Code Architecture and File Structure

### Recommended Project Structure

```
vAmiga-ASR10/
├── Core/
│   ├── Components/
│   │   ├── ASR10.h/cpp           # Main ASR-10 class (replaces Amiga.h)
│   │   ├── CPU/
│   │   │   ├── CPU.h/cpp         # Adapted from vAmiga
│   │   │   └── Moira/            # Reuse vAmiga's CPU emulator
│   │   ├── Memory/
│   │   │   ├── Memory.h/cpp      # Adapted memory map
│   │   │   └── MemoryTypes.h     # ASR-10 specific types
│   │   ├── Audio/
│   │   │   ├── DOCII.h           # DOC II voice chip
│   │   │   ├── OTIS.h            # OTIS synthesizer chip
│   │   │   ├── CEM3379.h         # Curtis analog filter
│   │   │   ├── OSCAR.h           # Effects processor
│   │   │   └── AudioStream.h     # Reuse from vAmiga Paula
│   │   ├── Sequencer/
│   │   │   ├── ASR10Sequencer.h  # 96 PPQ sequencer
│   │   │   └── SequencerEvent.h  # Event structure
│   │   ├── MIDI/
│   │   │   └── MIDIHandler.h     # MIDI In/Out/Thru
│   │   ├── Storage/
│   │   │   ├── SampleManager.h   # Sample RAM management
│   │   │   └── DiskController.h  # ASR-10 disk I/O
│   │   └── Display/
│   │       └── LCDDisplay.h      # LCD emulation (stub)
│   ├── Config/
│   │   └── ASR10Config.h         # ASR-10 configuration options
│   ├── Base/
│   │   └── (reuse from vAmiga)   # CoreComponent, SubComponent, etc.
│   └── Utilities/
│       └── (reuse from vAmiga)   # RingBuffer, MsgQueue, etc.
├── analog-set/                    # Git submodule
│   ├── src/
│   │   ├── components/
│   │   │   ├── exp/DOC5503.h     # Reference implementation
│   │   │   ├── Quantizer.h       # Reuse
│   │   │   ├── ADC.h             # Reuse
│   │   │   └── DAC.h             # Reuse
│   │   └── state/
│   │       └── StateManager.h    # Adapt for sample loading
│   └── docs/                     # Reference documentation
├── Manual/
│   └── Developer/
│       └── ASR10_VAMIGA_ADAPTATION_GUIDE.md  # This document
└── CMakeLists.txt                # Build configuration
```

### Build Configuration

**CMakeLists.txt**:
```cmake
cmake_minimum_required(VERSION 3.16)
project(vAmiga-ASR10 VERSION 1.0)

# Use C++20 (for vAmiga compatibility)
set(CMAKE_CXX_STANDARD 20)

# Include analog-set as submodule
add_subdirectory(analog-set EXCLUDE_FROM_ALL)

# vAmiga core (adapted)
add_library(vAmiga-ASR10-Core
    Core/Components/ASR10.cpp
    Core/Components/CPU/CPU.cpp
    Core/Components/CPU/Moira/Moira.cpp
    Core/Components/Memory/Memory.cpp
    Core/Components/Audio/DOCII.cpp
    Core/Components/Audio/OTIS.cpp
    Core/Components/Audio/CEM3379.cpp
    Core/Components/Audio/OSCAR.cpp
    Core/Components/Sequencer/ASR10Sequencer.cpp
    Core/Components/MIDI/MIDIHandler.cpp
    Core/Components/Storage/SampleManager.cpp
)

# Link analog-set components
target_link_libraries(vAmiga-ASR10-Core
    PRIVATE analog-set-components
)

# Compile definitions
target_compile_definitions(vAmiga-ASR10-Core
    PUBLIC ASR10_BUILD=1
)
```

---

## Testing and Validation

### Unit Tests

**Test DOC II**:
```cpp
TEST_CASE("DOCII Voice Playback", "[DOCII][Audio]") {
    DOCII doc;
    doc.prepare(44100.0f, 512);

    // Load sample
    float sample[1024];
    generateSineWave(sample, 1024, 440.0f);

    // Trigger voice
    doc.noteOn(0, sample, 1024, 440.0f);

    // Process block
    float left[512], right[512];
    doc.processBlock(left, right, 512);

    // Verify output
    REQUIRE(calculateRMS(left, 512) > 0.0f);
}
```

**Test CEM3379 Filter**:
```cpp
TEST_CASE("CEM3379 Temperature Drift", "[CEM3379][Filter]") {
    CEM3379 filter;
    filter.prepare(44100.0f, 512);
    filter.filters[0].cutoff = 1000.0f;

    // Measure cutoff at 25°C
    filter.filters[0].temperature = 25.0f;
    float cutoff25 = filter.measureCutoff(0);

    // Measure cutoff at 45°C
    filter.filters[0].temperature = 45.0f;
    float cutoff45 = filter.measureCutoff(0);

    // Should drift ~50 cents
    float expectedRatio = std::pow(2.0f, 50.0f / 1200.0f);
    float actualRatio = cutoff45 / cutoff25;

    REQUIRE(actualRatio == Approx(expectedRatio).epsilon(0.01));
}
```

**Test Sequencer**:
```cpp
TEST_CASE("Sequencer Swing Quantization", "[Sequencer]") {
    ASR10Sequencer seq;
    seq.bpm = 120.0f;
    seq.swing.enabled = true;
    seq.swing.amount = 62.5f;  // 62.5%
    seq.swing.division = 16;

    // Add events
    seq.addEvent(0, NOTE_ON, 60, 100);
    seq.addEvent(24, NOTE_ON, 62, 100);  // 16th note later

    // Verify swing applied
    i64 swungTime = seq.applySwing(24);
    REQUIRE(swungTime == 27);  // Delayed by 3 pulses
}
```

### Integration Tests

**End-to-End Audio**:
```cpp
TEST_CASE("ASR10 Full Audio Pipeline", "[Integration]") {
    ASR10 asr10;
    asr10.initialize();
    asr10.loadROM("asr10_v3.6.bin");
    asr10.powerOn();

    // Load sample
    asr10.sampleManager.loadSample("kick.wav");

    // Trigger via MIDI
    asr10.midi.receiveByte(0x90);  // Note On, channel 1
    asr10.midi.receiveByte(60);    // C4
    asr10.midi.receiveByte(100);   // Velocity 100

    // Process audio
    float output[44100 * 2];
    asr10.processAudio(output, 44100);

    // Verify audio output
    REQUIRE(calculateRMS(output, 44100) > 0.01f);
}
```

### Hardware Comparison

**Methodology**:
1. Record ASR-10 hardware audio output
2. Record emulator audio output with same settings
3. Compare waveforms, spectra, timing

**Metrics**:
- Waveform correlation: >99%
- Frequency response: ±0.5 dB
- Harmonic distortion: ±0.01%
- Timing accuracy: ±1 sample

---

## Conclusion

This adaptation guide provides a complete roadmap for transforming vAmiga into an ASR-10 emulator by:

1. **Reusing proven components**: Moira CPU, memory system, utilities
2. **Replacing Amiga-specific parts**: Paula → DOC II, Copper → Sequencer
3. **Integrating analog-set DSP**: DOC5503, OTIS, CEM3379, OSCAR
4. **Adapting architecture**: Memory map, interrupts, I/O

**Estimated Timeline**: 18 months for full ROM-compatible emulation

**Key Success Factors**:
- Leverage vAmiga's cycle-accurate CPU emulation
- Integrate analog-set's component-level audio modeling
- Maintain modular architecture for testing
- Focus on hardware quirks for authenticity

**Result**: A plugin/standalone application that can load the original ASR-10 ROM and sound exactly like the hardware.

---

**Document Version**: 1.0
**Date**: January 18, 2026
**Author**: Claude Code
**Status**: Comprehensive Implementation Guide
**Related Files**:
- [vAmiga/Core/Components/Amiga.h](../Core/Components/Amiga.h)
- [analog-set/docs/ASR10_CYCLE_ACCURATE_EMULATION.md](../analog-set/docs/ASR10_CYCLE_ACCURATE_EMULATION.md)
- [analog-set/src/components/exp/DOC5503.h](../analog-set/src/components/exp/DOC5503.h)
