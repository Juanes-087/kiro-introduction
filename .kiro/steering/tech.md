# Flappy Kiro - Technical Stack

## Build System

**No build system required** - This is a vanilla JavaScript project using plain HTML, CSS, and JavaScript that runs directly in the browser.

## Tech Stack

### Core Technologies

- **HTML5 Canvas** - For rendering game graphics
- **Vanilla JavaScript (ES6+)** - No framework or transpilation required
- **Web Audio API** - For sound effect playback
- **localStorage API** - For high score persistence

### Game Technologies

- **requestAnimationFrame** - For smooth 60 FPS game loop
- **Canvas 2D Context** - For all rendering operations

## File Structure

```
kiro-introduction/
├── index.html              # Main HTML file with Canvas element
├── config.js              # Single source of truth for all game constants
├── game.js                # Main game implementation (to be created)
├── assets/
│   ├── ghosty.png         # Ghost sprite image
│   ├── jump.wav           # Jump sound effect
│   └── game_over.wav      # Game over sound effect
└── img/                   # Screenshots and documentation images
```

## Common Commands

### Development

```bash
# No build commands required - open index.html directly in browser
# For local development server:
python -m http.server 8000
# Then open http://localhost:8000 in browser
```

### Testing

```bash
# No test runner configured yet
# Tests should be run manually in browser or added via Jest/Mocha if needed
```

## Key Libraries

**None** - The project uses only native browser APIs.

## Configuration

All game constants are centralized in `config.js` as a single exported `CONFIG` object. This file:
- Contains all tunable values (physics, dimensions, colors, timing, etc.)
- Is loaded via script tag before all other game scripts
- Is treated as read-only at runtime
- Is organized by category: Canvas, Physics, Ghost, Pipes, Clouds, Collision, Particles, Audio, UI