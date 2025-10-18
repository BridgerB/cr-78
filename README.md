# CR-78 Drum Machine

A web-based emulator of the classic Roland CR-78 drum machine, built with
SvelteKit 5 and the Web Audio API.

## Features

- 15 authentic CR-78 drum sounds
- 16-step sequencer (adjustable from 3-32 steps)
- Adjustable BPM (20-200)
- 6-level swing control with precise drift compensation timing
- Individual mute controls for each drum sound
- Real-time pattern editing
- Visual playback indicator

## Technologies

- **SvelteKit 5** with Svelte 5's new runes-based reactivity system
- **Web Audio API** for low-latency audio playback
- **TypeScript** with strict type checking
- **Deno** for modern JavaScript runtime
- **Vite** for fast development and building

## Development

Start the development server:

```bash
deno task dev
```

The app will be available at `http://localhost:5173/`

## Building

Create a production build:

```bash
npm run build
```

Preview the production build:

```bash
npm run preview
```

## How to Use

1. Click the green **Play** button to start the drum machine
2. Click on the checkboxes in the grid to program your drum pattern
3. Adjust **BPM** to change the tempo
4. Adjust **Swing** (1-6) to add groove to your pattern
5. Adjust **Length** to change the number of steps in your sequence
6. Use the **Mute** checkboxes to silence individual drum sounds
7. Click **Stop** to pause playback
8. Click **Clear** to erase the current pattern

## Audio Samples

The drum machine uses WAV samples of the Roland CR-78, sourced from the original
drummer.js project. Audio files are located in `src/lib/assets/audio/`.

## Attribution

This project is a SvelteKit 5 port of [Drummer](https://github.com/glynnbird/drummer) by [Glynn Bird](https://github.com/glynnbird).

**Adapted from the original:**
- Timing algorithm with drift compensation (`newSetInterval`)
- Swing implementation (6 levels)
- CR-78 WAV audio samples (originally from [boxedear.com](http://www.boxedear.com/free.html))
- Sequencer architecture
- Default pattern

**New in this version:**
- SvelteKit 5 with Svelte 5 runes
- TypeScript
- Web Audio API (instead of Howler.js)
- CR-78 only (original supported CR-78, R8, Tempest)
- Simplified UI

The original was built with Vue.js 2, Vuetify, and Howler.js.

## License

Apache 2.0 - This project uses the same license as the original [Drummer](https://github.com/glynnbird/drummer) project.
