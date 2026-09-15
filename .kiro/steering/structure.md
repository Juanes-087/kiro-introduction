# Flappy Kiro - Project Structure

## Directory Layout

```
kiro-introduction/
├── .kiro/                      # Kiro configuration and specs
│   ├── steering/              # Steering rules (this directory)
│   │   ├── product.md         # Product overview
│   │   ├── tech.md            # Technical stack and conventions
│   │   └── structure.md       # This file
│   └── specs/                 # Spec workflow files
│       └── flappy-kiro/
│           ├── requirements.md    # User requirements with acceptance criteria
│           ├── design.md          # Technical design and architecture
│           ├── tasks.md           # Implementation task list
│           └── .config.kiro       # Kiro spec configuration
├── assets/                     # Game assets
│   ├── ghosty.png             # Ghost sprite image
│   ├── jump.wav               # Jump sound effect
│   └── game_over.wav          # Game over sound effect
├── img/                        # Screenshots and documentation images
├── index.html                  # Main HTML file
├── config.js                   # Game configuration constants
└── game.js                     # Main game implementation
```

## Coding Conventions

### JavaScript Style

- **File Organization**: Each major component in its own file or as part of game.js
- **Naming**: camelCase for variables/functions, PascalCase for classes
- **Comments**: JSDoc-style for public APIs, inline for complex logic
- **Imports**: No module bundler - use script tags in index.html

### Game Architecture

Components follow a clean separation of concerns:

```
Game Loop (game.js)
    ├── InputHandler    # User input processing
    ├── PhysicsEngine   # Ghost movement and gravity
    ├── PipeSpawner     # Pipe generation and management
    ├── CollisionDetector # Collision detection logic
    ├── ScoreManager    # Score tracking and persistence
    ├── CloudManager    # Cloud system management
    ├── AudioManager    # Sound effect playback
    ├── AssetManager    # Asset loading and caching
    ├── Renderer        # Canvas rendering
    └── StateMachine    # Game state transitions
```

### File Responsibilities

| File | Purpose |
|------|---------|
| `index.html` | Canvas element, basic layout, script loading |
| `config.js` | All tunable game constants (exported CONFIG object) |
| `game.js` | Main game loop, component initialization, orchestration |
| `components/*.js` | Individual component implementations (optional organization) |

## Configuration Conventions

- **config.js**: Single source of truth for all constants
- **No magic numbers**: All numeric values imported from CONFIG
- **Read-only CONFIG**: Never mutate configuration at runtime
- **Categories**: Constants grouped by concern (Physics, Canvas, Audio, etc.)

## Asset Conventions

- **Images**: PNG format with transparency support
- **Audio**: WAV format for simplicity
- **Loading**: Preload all assets before Game State transitions to Menu
- **Location**: All assets in `assets/` directory relative to index.html