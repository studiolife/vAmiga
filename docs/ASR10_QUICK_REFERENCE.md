# ASR-10 Emulation Quick Reference
## Fast lookup guide for adapting vAmiga to ASR-10

---

## Component Quick Map

| What to Change | vAmiga Component | ASR-10 Replacement | Action |
|----------------|------------------|-------------------|--------|
| **CPU** | Moira @ 7.16 MHz | Moira @ 10 MHz | Change clock rate |
| **Audio** | Paula (4ch, 8-bit) | DOC II (32-voice, 16-bit) | Replace completely |
| **Synthesis** | None | OTIS + CEM3379 | Add new |
| **Effects** | None | OSCAR | Add new |
| **Sequencer** | Copper | ASR-10 Sequencer | Replace completely |
| **I/O** | CIA (keyboard/mouse) | MIDI Handler | Replace completely |
| **Graphics** | Denise | LCD stub | Remove/stub |
| **DMA** | Agnus | Simple audio DMA | Simplify |
| **Memory** | 512KB-2MB chip RAM | 2-16MB sample RAM | Remap addresses |

---

## Memory Map Quick Reference

### Amiga (Original)
```
0x000000: Chip RAM
0xA00000: CIA
0xC00000: Slow RAM
0xDD0000: Custom chips
0xF80000: ROM
```

### ASR-10 (Target)
```
0x000000: ROM (2 MB)
0x200000: Sample RAM (2-16 MB)
0xE00000: DOC II registers
0xE10000: OTIS registers
0xE20000: OSCAR registers
0xE30000: MIDI I/O
0xE60000: Sequencer
```

---

## File Structure Mapping

```
OLD: Core/Components/Amiga.h
NEW: Core/Components/ASR10.h

OLD: Core/Components/Paula/
NEW: Core/Components/Audio/
     ├── DOCII.h         (from analog-set/DOC5503.h)
     ├── OTIS.h          (new)
     ├── CEM3379.h       (new)
     └── OSCAR.h         (new)

OLD: Core/Components/Agnus/Copper/
NEW: Core/Components/Sequencer/ASR10Sequencer.h

OLD: Core/Components/CIA/
NEW: Core/Components/MIDI/MIDIHandler.h

REMOVE:
- Core/Components/Denise/
- Core/Components/Agnus/Blitter/
- Core/Components/CIA/
```

---

## Clock Timing Quick Reference

| Component | Amiga | ASR-10 | Ratio |
|-----------|-------|--------|-------|
| CPU | 7.16 MHz | 10 MHz | 1.4x |
| Audio Sample Rate | Variable | 44.1 kHz | Fixed |
| Control Rate | N/A | 1.5 kHz | OTIS CV updates |
| Sequencer PPQ | N/A | 96 | MIDI timing |

---

## Code Snippets

### Change CPU Clock
```cpp
// In ASR10::_initialize()
cpu.clock.setFrequency(10'000'000);  // 10 MHz
```

### Memory Map Peek
```cpp
template <Accessor s> u16 Memory::peek16(u32 addr) {
    // ROM: 0x000000-0x1FFFFF
    if (addr < 0x200000) return R16BE(rom + addr);

    // Sample RAM: 0x200000+
    if (addr >= 0x200000)
        return R16BE(sampleRam + (addr - 0x200000));

    // I/O: 0xE00000-0xE7FFFF
    if (addr >= 0xE00000) return peekIO16<s>(addr);
}
```

### Audio Pipeline
```cpp
void ASR10::executeAudioCycle() {
    // 1. OTIS control voltages
    otis.execute(cycles);

    // 2. DOC II voices
    float sample = docII.processVoice(voice);

    // 3. CEM3379 filter
    sample = cem3379.process(voice, sample);

    // 4. OTIS envelope
    sample *= otis.voices[voice].ampEnv.level;

    // 5. OSCAR effects
    oscar.process(&sample, 1);
}
```

### MIDI Note Handling
```cpp
void MIDIHandler::noteOn(u8 note, u8 velocity) {
    docII.noteOn(note, velocity / 127.0f);
    otis.voices[note].ampEnv.trigger();
}
```

---

## Build Steps

1. **Clone vAmiga**
```bash
git clone https://github.com/dirkwhoffmann/vAmiga.git vAmiga-ASR10
cd vAmiga-ASR10
```

2. **Add analog-set submodule**
```bash
git submodule add https://github.com/concept10/analog-set.git
git submodule update --init --recursive
```

3. **Create ASR-10 components**
```bash
mkdir -p Core/Components/Audio
mkdir -p Core/Components/Sequencer
mkdir -p Core/Components/MIDI
```

4. **Build**
```bash
cmake -B build -DASR10_BUILD=ON
cmake --build build
```

---

## Integration Checklist

### Phase 1: Foundation
- [ ] Copy vAmiga project
- [ ] Add analog-set submodule
- [ ] Create ASR10.h/cpp (based on Amiga.h)
- [ ] Adapt CPU clock (7.16 → 10 MHz)
- [ ] Implement ASR-10 memory map
- [ ] Create component stubs

### Phase 2: Audio Core
- [ ] Port DOC5503 → DOCII
- [ ] Implement OTIS envelopes
- [ ] Implement OTIS LFOs
- [ ] Implement CEM3379 filter
- [ ] Sample loading (WAV/AIFF)
- [ ] MIDI note on/off

### Phase 3: Sequencer
- [ ] 96 PPQ sequencer engine
- [ ] 16 tracks
- [ ] Event recording/playback
- [ ] Swing quantization
- [ ] MIDI clock sync

### Phase 4: Effects
- [ ] OSCAR reverb
- [ ] OSCAR delay
- [ ] OSCAR chorus
- [ ] Fixed-point DSP

### Phase 5: ROM
- [ ] Full 68000 instruction set
- [ ] All memory-mapped I/O
- [ ] Interrupt handling
- [ ] ROM verification

---

## Common Pitfalls

### ❌ Don't Do This
```cpp
// Using Amiga DMA cycles
agnus.execute(CPU_AS_DMA_CYCLES(cycles));
```

### ✅ Do This Instead
```cpp
// ASR-10: Direct audio cycle execution
docII.execute(cycles);
otis.execute(cycles);
```

---

### ❌ Don't Do This
```cpp
// Paula 8-bit audio
paula.channel[0].volume = 64;
```

### ✅ Do This Instead
```cpp
// DOC II 16-bit voices
docII.voices[0].setVolume(0.5f);
```

---

### ❌ Don't Do This
```cpp
// Copper display list
copper.addInstruction(WAIT, 0x100);
```

### ✅ Do This Instead
```cpp
// ASR-10 sequencer event
sequencer.addEvent(0, NOTE_ON, 60, 100);
```

---

## Key Files to Study

### vAmiga (Understand These)
1. `Core/Components/Amiga.h` - Main container (copy structure)
2. `Core/Components/CPU/Moira/Moira.h` - CPU emulator (reuse)
3. `Core/Components/Memory/Memory.h` - Memory system (adapt)
4. `Core/Components/Paula/Paula.h` - Audio (understand to replace)

### analog-set (Integrate These)
1. `src/components/exp/DOC5503.h` - Voice chip (extend to DOCII)
2. `docs/ASR10_CYCLE_ACCURATE_EMULATION.md` - Requirements
3. `docs/ENSONIQ_ASIC_EMULATION_GUIDE.md` - OTIS/OSCAR specs
4. `src/state/StateManager.h` - Sample loading (adapt)

---

## Testing Commands

### Run Unit Tests
```bash
./build/test/vAmiga-ASR10-Tests
```

### Load ROM
```bash
./build/vAmiga-ASR10 --rom asr10_v3.6.bin
```

### Record Audio
```bash
./build/vAmiga-ASR10 --rom asr10.bin --record output.wav
```

### Compare to Hardware
```bash
python scripts/compare_audio.py hardware.wav emulator.wav
```

---

## Useful Macros

```cpp
// Endianness (from vAmiga)
#define R16BE(x) (((x)[0] << 8) | (x)[1])
#define W16BE(x, y) { (x)[0] = (y) >> 8; (x)[1] = (y); }

// Cycle conversion
#define CPU_CYCLES_TO_SAMPLES(c) ((c) * 44100.0 / 10000000.0)
#define SAMPLES_TO_CPU_CYCLES(s) ((s) * 10000000.0 / 44100.0)

// MIDI
#define MIDI_NOTE_ON(ch)  (0x90 | (ch))
#define MIDI_NOTE_OFF(ch) (0x80 | (ch))
#define MIDI_CC(ch)       (0xB0 | (ch))
```

---

## Performance Tips

1. **Use SIMD for audio**: Process 4 voices at once
2. **Optimize filter**: Oversample only when needed
3. **Voice culling**: Don't process inactive voices
4. **Batch MIDI events**: Process per-block, not per-sample
5. **Fixed-point math**: Use integer math where possible

---

## Debugging Tips

### Enable Cycle Logging
```cpp
#define ASR10_LOG_CYCLES 1
```

### Memory Access Tracing
```cpp
#define ASR10_TRACE_MEMORY 1
```

### MIDI Event Logging
```cpp
#define ASR10_LOG_MIDI 1
```

### Audio Underrun Detection
```cpp
if (audioStream.fillLevel() < 128) {
    warn("Audio buffer underrun!");
}
```

---

## Resources

### Documentation
- [Full Adaptation Guide](ASR10_VAMIGA_ADAPTATION_GUIDE.md)
- [analog-set Project](../analog-set/README.md)
- [vAmiga Developer Manual](../Manual/Developer/)

### Hardware References
- ASR-10 Service Manual
- Motorola 68000 Programmer's Manual
- Curtis CEM3379 Datasheet
- Ensoniq DOC II Specifications

### Community
- vAmiga GitHub: https://github.com/dirkwhoffmann/vAmiga
- analog-set GitHub: https://github.com/concept10/analog-set

---

**Version**: 1.0
**Date**: January 18, 2026
**See Also**: [ASR10_VAMIGA_ADAPTATION_GUIDE.md](ASR10_VAMIGA_ADAPTATION_GUIDE.md)
