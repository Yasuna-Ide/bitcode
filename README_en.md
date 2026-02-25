# BITCODE — Chiptune Live Coder

## 1. Concept

**BITCODE** is a browser-based chiptune live coding language that actively embraces the 4-channel architecture of the original Game Boy (DMG-01) sound chip LR35902 as a creative constraint.

### Design Core

Just as Strudel brought TidalCycles' pattern language to the Web, BITCODE **redefines the constraints of the DMG sound hardware as a pattern language**. Rather than offering flexible, general-purpose synthesis, the language's aesthetic and design identity lies in manipulating patterns within a deliberately limited parameter space: 4-bit volume, pulse wave duty cycles, 32-sample wave memory, and LFSR noise.

The UI is rendered in pure 1-bit monochrome (#fff / #000 only), embodying the "aesthetics of constraint" on the visual level as well.

### Relationship with Strudel

- **Shared**: Cycle-based pattern model, concise mini-notation inspired syntax, method-chaining transformations, browser-native execution
- **Distinct**: Sound sources are fixed to four DMG channels rather than arbitrary Web Audio API nodes. The physical constraint of monophonic channels (one note at a time per channel) directly influences pattern design. The parameter space is discrete and finite (4-bit 16-step volume, 4 duty cycle types, etc.)

## 2. Channel Architecture

```
┌─────────────────────────────────────────┐
│             BITCODE Engine              │
│                                         │
│  p1  ──▶  [CH1: Pulse+Sweep]  ──┐       │
│  p2  ──▶  [CH2: Pulse      ]  ──┤       │
│  p3  ──▶  [CH3: Wave Memory]  ──┼──▶ 🔊 │
│  n1  ──▶  [CH4: Noise      ]  ──┘       │
│                                         │
│  Each channel: 1 persistent oscillator  │
│  New event→ immediately overwrites prev │
│  Master gain: 2.5                       │
└─────────────────────────────────────────┘
```

Four functions map 1:1 to four channels. Each channel holds a single fixed sound generator; when a new event arrives, it forcibly overwrites the previous sound (even mid-release). This mirrors DMG hardware behavior and is an intentional design constraint that forces compositional thinking about "how to allocate four channels."

## 3. Language Syntax

### 3.1 Basics: Note Patterns

```javascript
// CH1: Pulse wave melody
p1("c4 e4 g4 c5")

// CH2: Pulse wave harmony / countermelody
p2("c3 ~ e3 ~")

// CH3: Wave memory bass
p3("c2 c2 g2 g2")

// CH4: Noise rhythm
n1("h l ~ h")   // h=high(short period/7bit), l=low(long period/15bit)
```

### 3.2 Mini-Notation

Adopts mini-notation syntax from TidalCycles / Strudel:

| Notation | Meaning |
|----------|---------|
| Space-separated | Equal subdivision within a cycle |
| `~` | Rest |
| `[ ]` | Subdivision (temporal subdivision) |
| `< >` | Alternate (rotate per cycle) |
| `*n` | Repeat |
| `?` | Probabilistic triggering (50% chance) |

```javascript
// Subdivision: 16th-note-like movement
p1("c4 [e4 g4] a4 [g4 e4]")

// Alternate: rotate each cycle
p1("<c4 e4 g4 c5> <e4 g4 b4 e5>")

// Repeat
n1("h*4 l*2 h l")

// Probabilistic triggering
n1("h l? h ~")   // l has 50% chance of sounding
```

### 3.3 Chiptune-Specific Parameters

#### Duty Cycle (CH1, CH2)

```javascript
p1("c4 e4 g4 c5").duty(50)            // Uniform 50% for all notes
p1("c4 e4 g4 c5").duty("12 25 50 75") // Patterned per note
// Values: 12(=12.5%), 25, 50, 75
```

#### Volume (All Channels)

```javascript
p1("c4 e4 g4 c5").vol(8)              // Uniform (0-15)
p1("c4 e4 g4 c5").vol("15 12 8 4")    // Patterned
```

#### Volume Envelope (All Channels)

DMG envelope: initial volume (0-15), direction (up/down), speed (1-7)

```javascript
p1("c4 e4 g4 c5").env(15, "down", 3)
// Start at volume 15, decay, speed 3
```

#### Wave Memory (CH3 Only)

```javascript
// Preset waveforms
p3("c3 e3 g3 c4").wave("tri")       // Triangle
p3("c3 e3 g3 c4").wave("saw")       // Sawtooth
p3("c3 e3 g3 c4").wave("sin")       // Sine-like
p3("c3 e3 g3 c4").wave("sqr")       // Square

// Direct hex waveform (32 chars = 32 samples × 4bit)
p3("c3").wave("0123456789abcdeffedcba9876543210")

// Waveform patterning (alternates per cycle)
p3("c3 e3 g3 c4").wave("<tri saw sin>")
```

#### Noise Mode (CH4)

```javascript
n1("h l h l")                // h=short period(7bit), l=long period(15bit)
n1("h l h l").env(15, "down", 1)  // With envelope
```

### 3.4 Pattern Transformations

| Transform | Syntax | Description |
|-----------|--------|-------------|
| Speed up | `.fast(n)` | Repeat pattern at n× speed |
| Slow down | `.slow(n)` | Play pattern n× slower |
| Reverse | `.rev()` | Play pattern in reverse |
| Periodic | `.every(n, fn)` | Apply fn every n cycles |
| Probabilistic | `.sometimes(fn)` | Apply fn with 50% probability |
| High probability | `.often(fn)` | Apply fn with 75% probability |
| Low probability | `.rarely(fn)` | Apply fn with 25% probability |
| Palindrome | `.palindrome()` | Reverse on even cycles |
| Rotate | `.rotate(n)` | Shift pattern by n elements |

```javascript
p1("c4 e4 g4 c5").fast(2)
p1("c4 e4 g4 c5").every(4, x => x.fast(2))
p1("c4 e4 g4 c5").sometimes(x => x.duty(12))
p1("c4 e4 g4 c5").palindrome()
p1("c4 e4 g4 c5").rotate(2)

// Composing transformations: reverse + double speed every 4 cycles
p1("c4 e4 g4 c5").every(4, x => x.rev().fast(2))
```

### 3.5 Chiptune-Specific Effects

```javascript
// Arpeggio: rapid pitch switching within a single note (semitone intervals)
p1("c4 e4").arp("0 4 7")          // Major triad arpeggio

// Sweep: continuous frequency change (recommended for CH1)
p1("c4 ~ c4 ~").sweep(3, "up")    // Upward sweep, speed 3
p1("c5 ~ c5 ~").sweep(3, "down")  // Downward sweep

// Echo: pseudo-delay via volume decay
p1("c4 e4 g4 c5").echo(3, 0.25)   // 3 repetitions, 0.25 interval

// Glissando: discrete slide to next note
p1("c4 ~ c5 ~").gliss(4)          // 4-step slide
```

### 3.6 Channel Control

```javascript
p1("c4 e4 g4 c5").solo()   // Play this channel only
p2("c3 e3 g3 c4").mute()   // Mute this channel
```

## 4. Global Controls

```javascript
bpm(140)        // Set tempo
hush()          // Stop all channels
```

## 5. UI Design

### Layout

```
┌──────────────────────────────────────────┐
│  BITCODE  v0.3      ▶ PLAY  ■ STOP  BPM  │  ← Header (fixed)
├──────────────────────────────────────────┤
│ 1│ bpm(104)                              │
│ 2│                                       │
│ 3│ p1("~ a4 c5 ~ e5 ~ d5 c5 ~ a4 ~ g4"). │
│ 4│   .duty("<50 25 50 25>")              │  ← CodeMirror editor
│ 5│   .env(11, "down", 4)                 │    with line numbers
│ 6│   .rotate(1)                          │    & highlighting
│ 7│   .every(4, x => x.rev())             │
│  │                                       │
│  │ Error message (inline display)        │
├──────────────────────────────────────────┤
│  [CH1]   [CH2]   [CH3]   [CH4]           │  ← Channel monitor
│  ------------ Waveform Scope ----------- │  ← AnalyserNode display
│  p1 p2 pulse · p3 wave · n1 noise │ ...  │  ← Help
└──────────────────────────────────────────┘
```

### Color Palette

Pure 1-bit monochrome:
- `#ffffff` — Background
- `#000000` — Text, borders, all UI elements

No intermediate colors whatsoever. This constraint forms BITCODE's visual identity.

### CodeMirror Editor

Integrates CodeMirror 5 with a custom `bitcode` mode for syntax highlighting. Within the 1-bit monochrome constraint, syntax elements are visually differentiated along three axes: **bold, italic, and opacity**.

| Syntax Element | Style |
|---------------|-------|
| `p1` `p2` `p3` `n1` channel names | **Bold** |
| `bpm` `hush` global functions | **Bold** |
| `.duty()` `.fast()` etc. methods | *Italic* |
| `//` comments | Dim (opacity 35%) |
| Mini-notation notes `c4` `e5` | **Bold** |
| Noise tokens `h` `l` | **Bold** |
| Rests `~` | Dim (opacity 30%) |
| `[ ]` `< >` `( )` brackets | **Bold** |
| `=>` arrow | **Bold** |
| Callback variable `x` | *Italic* |

Additional features:
- Line numbers
- Bracket matching (`matchBrackets`)
- Auto-close brackets (`autoCloseBrackets`)
- Active line highlight (`styleActiveLine`)

### Error Display

Errors are shown as CodeMirror inline widgets directly below the last line of the editor. For method name typos, Levenshtein distance-based "did you mean ...?" suggestions are provided.

```
  .evry(4, x => x.fast(2))
✕ x.evry is not a function — did you mean .every()?
```

### Channel Monitor

Displays each channel's active state with monochrome inversion. Shows "S" for `.solo()` and "M" for `.mute()`.

### Waveform Scope

Real-time waveform scope connected to `AnalyserNode`. HiDPI (`devicePixelRatio`) aware.

### Controls

- **Ctrl+Enter** (Mac: ⌘+Enter): Evaluate code and start playback / update patterns
- **▶ PLAY / ⟳ EVAL**: Button-triggered evaluation
- **■ STOP**: Stop all

## 6. Technical Architecture

### Sound Engine

- Web Audio API-based, one `OscillatorNode` per channel (persistent oscillator architecture)
- CH1/CH2: `OscillatorNode` (type: "square") — duty cycle controlled via `PeriodicWave` Fourier coefficients
- CH3: `OscillatorNode` (type: custom) — `PeriodicWave` from 32-sample wave memory
- CH4: `AudioBufferSourceNode` — LFSR-generated noise buffer (7-bit short period / 15-bit long period)
- Volume: `GainNode` per channel, 4-bit (0–15) mapped to 16 discrete levels
- Envelope: Attack/decay via `linearRampToValueAtTime`
- Master output: `GainNode` (gain: 2.5)
- Scheduling: `setValueAtTime` / `linearRampToValueAtTime` based on cycle timing

### Pattern Engine

- Cycle-based scheduler: divides time by BPM and distributes events equally within each cycle
- Mini-notation parser: tokenizer → AST → event list
- Pattern transformation: functional composition via method chaining
- Cycle counter: integer incremented per cycle, used for `.every()` / `.palindrome()` evaluation

### Evaluation Flow

```
User code (string)
  → new Function() wrapping
  → Pattern object construction
  → Scheduler registration
  → Per-cycle event generation
  → Web Audio API parameter scheduling
```

## 7. Implementation Status

### Phase 1: MVP ✅ Complete

- [x] Note patterns for p1–p3 + noise patterns for n1
- [x] Mini-notation (space-separated, `[ ]`, `~`)
- [x] `.duty()`, `.wave()`, `.vol()`, `.env()` basic parameters
- [x] `.fast()`, `.slow()`, `.rev()`, `.every()` basic transformations
- [x] String-based parameter patterning (duty, vol)
- [x] Textarea editor + Ctrl+Enter evaluation
- [x] Channel monitor + waveform scope
- [x] 1-bit monochrome UI
- [x] `bpm()` / `hush()` global controls

### Phase 2: Expressive Extensions ✅ Complete

- [x] `< >` alternate (within mini-notation)
- [x] `*n` repeat
- [x] `?` probabilistic triggering (50%)
- [x] `.sometimes()` / `.often()` / `.rarely()` probabilistic transforms
- [x] `.palindrome()` palindrome transform
- [x] `.rotate(n)` rotation transform
- [x] `.arp()` arpeggio (semitone interval, 35ms switching)
- [x] `.sweep()` frequency sweep (speed & direction)
- [x] `.echo()` pseudo-delay (repeat count & interval)
- [x] `.gliss()` glissando (discrete step transition to next note)
- [x] Hex direct wave memory input (32 characters)
- [x] Waveform patterning `.wave("<tri saw>")`
- [x] `.solo()` / `.mute()` channel control
- [x] AnalyserNode real-time waveform scope (HiDPI aware)
- [x] Cycle counter display
- [x] Persistent oscillator architecture (monophonic forced switching)

### Phase 3: Live Coding Environment ✅ Complete

- [x] CodeMirror 5 integration
- [x] Custom `bitcode` syntax highlighting mode
- [x] 1-bit monochrome theme (bold/italic/opacity)
- [x] Line numbers
- [x] Bracket matching (`matchBrackets`)
- [x] Auto-close brackets (`autoCloseBrackets`)
- [x] Active line highlight (`styleActiveLine`)
- [x] Inline error widget display
- [x] Typo "did you mean" suggestions (Levenshtein distance)

### Known Limitations

- Parameter strings (`.duty()`, `.vol()`) do not support `< >` alternate syntax; only space-separated numeric sequences
- `.gliss()` only works when the next note exists within the same pattern
- `.sweep()` works on all channels, but on actual hardware it is a CH1-only feature
- CH4 `fast()` at high multipliers (4+) may cause `setValueAtTime` congestion and glitches

### Phase 4: Sharing & Distribution (In Progress)

- [ ] Extended keyboard shortcuts
- [ ] Mobile support
- [ ] URL-encoded share functionality
- [ ] Preset / sample code library (in-UI selection)

## 8. Design Notes

### Constraint-Driven Creativity

The DMG's 4-channel limit embodies the principle that "constraints drive creativity" in live coding. While Strudel allows unlimited channel addition, BITCODE is physically limited to four. This constraint produces:

1. **Urgency of choice**: Use 2 channels for melody, or allocate one for bass?
2. **Channel allocation experimentation**: Redistribute the same 4 channels across different arrangements
3. **Pseudo-polyphony techniques**: Using `.arp()` for rapid pitch switching to simulate chords — rediscovering traditional chiptune techniques

### Aesthetics of Discrete Parameter Space

All DMG parameters are discrete:
- Volume: 16 levels (4-bit)
- Duty cycle: 4 types
- Waveform: 32 samples × 4-bit
- Noise: 2 modes × 8 rates

By making this "coarse" parameter space the subject of pattern manipulation, `.duty("12 25 50 75")` cycling through discrete values produces distinctly digital textures.

### Significance of the 1-Bit UI

The pure black-and-white UI is not merely a decorative choice. It is a visual constraint that echoes the audio constraints (4-bit, 4ch, 4 duty cycles), maintaining consistency with BITCODE's design philosophy of "expression within constraints" at the interface level. The CodeMirror syntax highlighting — which uses zero color, differentiating syntax through bold, italic, and opacity alone — is an extension of this philosophy.

### Paradigm Shift to Pattern-Based Composition

Chiptune production has traditionally used trackers (LSDj, FamiTracker, MML, etc.). These are fundamentally about direct sequence placement — composing "the resulting music." BITCODE shifts this to the TidalCycles-derived paradigm of "pattern transformation." A statement like `every(4, x => x.fast(2).rev())` manipulates transformation rules, not note placement. This cognitive shift introduces a new mode of thinking to chiptune production.

### Monophonic Thinking Enforced by Persistent Oscillators

Each channel holds a single fixed sound generator, and new events immediately overwrite the previous sound. This mirrors actual DMG behavior and produces the following design effects:

1. **Release truncation**: New notes ruthlessly cut off mid-decay envelopes. This "cutting" creates the articulation characteristic of chiptune
2. **Channel physicality**: The absolute constraint of 1 channel = 1 note strictly enforces the "only one sound at a time" principle that node-based implementations left ambiguous
3. **Performance**: No node creation/destruction — only `setValueAtTime` scheduling — ensures stability during extended live coding sessions

### Qualitative Shift from Phase 2 to Phase 3

With Phase 2 completing expressive extensions (probabilistic transforms, arpeggio, waveform patterning, etc.) and Phase 3 adding CodeMirror integration and error handling, BITCODE transitioned from an "experimental sound engine" to a "live coding environment." Syntax highlighting is not merely cosmetic — it is essential cognitive support for the live coding practice of instantly grasping code structure and rewriting in real time.

## 9. Sample Session

```javascript
// === "The Sealed Forest" ===
bpm(104)

// CH1: A winding melody through the trees
// rotate makes the path shift each cycle
p1("~ a4 c5 ~ e5 ~ d5 c5 ~ a4 ~ g4")
  .duty("<50 25 50 25>")
  .env(11, "down", 4)
  .rotate(1)
  .every(4, x => x.rev())
  .every(7, x => x.arp("0 3 7"))

// CH2: Distant bells, echoing through canopy
p2("~ ~ ~ e5 ~ ~ ~ ~ a5 ~ ~ ~")
  .duty(12)
  .vol(4)
  .echo(2, 0.2)
  .palindrome()
  .every(5, x => x.rotate(3))

// CH3: Ancient roots — Am-Em-F-Dm
p3("a3 a3 a3 e3 e3 e3 f3 f3 f3 d3 d3 d3")
  .wave("<tri tri saw tri>")
  .vol(6)
  .every(6, x => x.rev())
  .every(8, x => x.rotate(3))

// CH4: Leaves rustling, footsteps on moss
n1("~ l? ~ ~ h ~ ~ l? ~ h? ~ ~")
  .env(7, "down", 4)
  .every(3, x => x.rotate(1))
  .every(5, x => x.fast(2))
```

## 10. External Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| React | 18.2.0 | UI rendering |
| ReactDOM | 18.2.0 | UI rendering |
| Babel Standalone | 7.23.9 | JSX transpilation |
| CodeMirror | 5.65.7 | Code editor |
| CodeMirror matchbrackets | 5.65.7 | Bracket matching |
| CodeMirror closebrackets | 5.65.7 | Auto-close brackets |
| CodeMirror active-line | 5.65.7 | Active line highlight |

All loaded from cdnjs.cloudflare.com. Zero installation, fully browser-based.

## 11. Legal Considerations

### Legality of Sound Emulation

BITCODE recreates the **operating principles** of the DMG sound chip using the Web Audio API and does not use any Nintendo firmware, BIOS, or ROM. Basic waveforms such as pulse waves and LFSR noise are generated from natural principles and do not constitute copyrightable works. Related patents have expired (20+ years since filing). Semiconductor layout exclusivity rights have also lapsed (10-year term).

### Trademarks

"Game Boy" is a registered trademark of Nintendo and is avoided in product naming and marketing. "BITCODE" is a generic term with very low trademark risk. Technical references in documentation use chip model numbers such as "DMG-01" and "LR35902."
