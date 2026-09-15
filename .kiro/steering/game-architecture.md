# Flappy Kiro - Game Architecture Standards

## Modular Systems Architecture

### Component Separation

Each system should be encapsulated in its own module with clear responsibilities:

```
┌─────────────────────────────────────────────────────────────┐
│                      Game Core                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Game Loop   │  │ StateMachine │  │   Input    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                         │      │            │
                         ▼      ▼            ▼
┌────────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   Physics      │  │  Collision   │  │   Score      │      │
│   Engine       │  │  Detector    │  │  Manager     │      │
└────────────────┘  └──────────────┘  └──────────────┘      │
                         │                     │              │
                         ▼                     ▼              │
┌────────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   Pipe         │  │  Cloud       │  │   Audio      │      │
│   Spawner      │  │  Manager     │  │  Manager     │      │
└────────────────┘  └──────────────┘  └──────────────┘      │
                         │                     │              │
                         ▼                     ▼              │
┌────────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   Renderer     │  │   Asset      │  │   High Score │      │
│                │  │  Manager     │  │  Storage     │      │
└────────────────┘  └──────────────┘  └──────────────┘      │
```

### Component Interface Pattern

```javascript
// Base component interface
class Component {
    constructor(config) {
        this.config = config;
        this.enabled = true;
    }

    update(deltaTime) {
        if (!this.enabled) return;
    }

    render(ctx) {
        if (!this.enabled) return;
    }

    reset() {
        // Reset to initial state
        this.enabled = true;
    }

    enable() {
        this.enabled = true;
    }

    disable() {
        this.enabled = false;
    }
}
```

### Module Organization

```javascript
// game.js - Main orchestration
import { GameStateMachine } from './state.js';
import { PhysicsEngine } from './physics.js';
import { Renderer } from './renderer.js';
import { InputHandler } from './input.js';
// ... other imports

// Initialize components
const gameState = new GameStateMachine(CONFIG);
const physics = new PhysicsEngine(CONFIG);
const renderer = new Renderer(CONFIG, assetManager);
const input = new InputHandler();
// ... other initializations

// Game loop
function gameLoop(timestamp) {
    input.update();
    physics.update(ghost, deltaTime);
    gameState.update(deltaTime);
    renderer.render();
    
    requestAnimationFrame(gameLoop);
}
```

## Event Handling Patterns

### Event Bus Pattern

```javascript
class EventBus {
    constructor() {
        this.events = {};
    }

    on(event, callback) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(callback);
    }

    off(event, callback) {
        if (!this.events[event]) return;
        this.events[event] = this.events[event].filter(cb => cb !== callback);
    }

    emit(event, data) {
        if (!this.events[event]) return;
        this.events[event].forEach(cb => cb(data));
    }
}

// Usage
const eventBus = new EventBus();

// Listen for input events
eventBus.on('jump', () => {
    physics.jump(ghost);
});

// Emit events from input handler
input.addEventListener((action) => {
    if (action === 'jump') {
        eventBus.emit('jump');
    }
});
```

### Callback Pattern for State Changes

```javascript
class GameStateMachine {
    constructor(config) {
        this.config = config;
        this.currentState = 'Menu';
        this.listeners = [];
    }

    addStateChangeListener(callback) {
        this.listeners.push(callback);
    }

    transition(newState) {
        if (!this.isValidTransition(this.currentState, newState)) {
            console.warn(`Invalid transition: ${this.currentState} → ${newState}`);
            return false;
        }

        const oldState = this.currentState;
        this.currentState = newState;
        
        // Notify listeners
        this.listeners.forEach(cb => cb(newState, oldState));
        
        return true;
    }
}
```

### Observer Pattern for Score Changes

```javascript
class ScoreManager {
    constructor(config) {
        this.config = config;
        this.score = 0;
        this.observers = [];
    }

    addObserver(callback) {
        this.observers.push(callback);
    }

    incrementScore() {
        this.score++;
        this.observers.forEach(cb => cb(this.score));
        this.saveHighScore();
    }

    saveHighScore() {
        const currentHigh = this.getHighScore();
        if (this.score > currentHigh) {
            localStorage.setItem(this.config.HIGH_SCORE_KEY, this.score);
        }
    }
}
```

## State Management

### Finite State Machine (FSM) Design

```javascript
class GameStateMachine {
    constructor(config) {
        this.config = config;
        this.states = {
            Menu: { enter: this.enterMenu.bind(this), exit: this.exitMenu.bind(this) },
            Playing: { enter: this.enterPlaying.bind(this), exit: this.exitPlaying.bind(this) },
            Paused: { enter: this.enterPaused.bind(this), exit: this.exitPaused.bind(this) },
            GameOver: { enter: this.enterGameOver.bind(this), exit: this.exitGameOver.bind(this) }
        };
        this.currentState = 'Menu';
        this.validTransitions = {
            Menu: ['Playing'],
            Playing: ['Paused', 'GameOver'],
            Paused: ['Playing', 'Menu'],
            GameOver: ['Menu']
        };
    }

    transition(newState) {
        if (!this.validTransitions[this.currentState]?.includes(newState)) {
            console.warn(`Invalid state transition: ${this.currentState} → ${newState}`);
            return false;
        }

        // Exit current state
        if (this.states[this.currentState].exit) {
            this.states[this.currentState].exit();
        }

        // Update state
        const oldState = this.currentState;
        this.currentState = newState;

        // Enter new state
        if (this.states[newState].enter) {
            this.states[newState].enter();
        }

        return true;
    }

    isValidTransition(from, to) {
        return this.validTransitions[from]?.includes(to) || false;
    }

    // State-specific methods
    enterMenu() {
        // Reset game state
        this.scoreManager.reset();
        this.cloudManager.reset();
    }

    enterPlaying() {
        // Resume physics and spawning
        this.physics.enabled = true;
        this.pipeSpawner.enabled = true;
    }

    enterPaused() {
        // Pause game but continue physics for smooth resume
        this.pipeSpawner.enabled = false;
        this.cloudManager.enabled = false;
    }

    enterGameOver() {
        // Stop spawning and physics
        this.pipeSpawner.enabled = false;
        this.cloudManager.enabled = false;
        this.audioManager.playGameOverSound();
    }

    // Accessors
    getCurrentState() {
        return this.currentState;
    }

    isState(state) {
        return this.currentState === state;
    }
}
```

### State Transition Table

| Current State | Trigger | Next State | Condition |
|---------------|---------|------------|-----------|
| Menu | Input | Playing | Spacebar or mouse click |
| Playing | Collision | Game Over | Ghost hits pipe or boundary |
| Playing | Input | Paused | Spacebar or Escape key |
| Paused | Input | Playing | Spacebar or Escape key |
| Paused | Input | Menu | Reset requested |
| Game Over | Input | Menu | Spacebar after debounce |

### Game Context Object

```javascript
// Central game context passed to components
class GameContext {
    constructor() {
        this.ghost = { x: 50, y: 300, velocity: 0 };
        this.pipes = [];
        this.clouds = [];
        this.score = 0;
        this.highScore = this.loadHighScore();
        this.gameState = 'Menu';
        this.deltaTime = 0;
        this.timestamp = 0;
        this.inputBuffer = [];
    }

    getScreenDimensions() {
        return {
            width: this.config.SCREEN_WIDTH,
            height: this.config.SCREEN_HEIGHT
        };
    }

    isPlaying() {
        return this.gameState === 'Playing';
    }

    isPaused() {
        return this.gameState === 'Paused';
    }
}
```

## Component Communication

### Component Dependency Injection

```javascript
// Create components with dependencies injected
const config = CONFIG;
const assetManager = new AssetManager(config);
const audioManager = new AudioManager(config, assetManager);
const scoreManager = new ScoreManager(config);
const stateMachine = new GameStateMachine(config);
const physics = new PhysicsEngine(config);
const collisionDetector = new CollisionDetector(config);
const pipeSpawner = new PipeSpawner(config);
const cloudManager = new CloudManager(config);
const renderer = new Renderer(config, assetManager);
const input = new InputHandler(config);

// Inject dependencies
scoreManager.addHighScoreObserver((score) => {
    renderer.updateScoreUI(score, stateMachine.getHighScore());
});

pipeSpawner.setCollisionDetector(collisionDetector);
physics.setCollisionDetector(collisionDetector);
```

### Component Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      Game Loop                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 1. InputHandler processes user input                 │  │
│  │    - Spacebar, mouse click, keyboard events          │  │
│  │    - Updates input buffer or triggers callbacks      │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 2. PhysicsEngine updates Ghost position              │  │
│  │    - Apply gravity to velocity                       │  │
│  │    - Update position based on velocity               │  │
│  │    - Clamp velocity to terminal velocity             │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 3. PipeSpawner generates/updates pipes               │  │
│  │    - Spawn new pipes at intervals                    │  │
│  │    - Update pipe positions                           │  │
│  │    - Remove off-screen pipes                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 4. CollisionDetector checks for collisions           │  │
│  │    - Check Ghost vs pipes                            │  │
│  │    - Check Ghost vs screen boundaries                │  │
│  │    - Return collision state                          │  │
│  └───────────────────────────────────────────��──────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 5. ScoreManager updates score (if applicable)        │  │
│  │    - Check if Ghost passed pipe center               │  │
│  │    - Increment score if not already scored           │  │
│  │    - Update high score if needed                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 6. Renderer draws all elements                       │  │
│  │    - Draw background, clouds, pipes, ghost, UI       │  │
│  │    - Handle screen shake offset                      │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 7. StateMachine validates and updates game state     │  │
│  │    - Check for state transition triggers             │  │
│  │    - Execute state enter/exit callbacks              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Component Reset Pattern

```javascript
class Game {
    constructor(config) {
        this.config = config;
        this.components = [
            this.physics = new PhysicsEngine(config),
            this.pipeSpawner = new PipeSpawner(config),
            this.cloudManager = new CloudManager(config),
            this.collisionDetector = new CollisionDetector(config),
            this.scoreManager = new ScoreManager(config),
            this.audioManager = new AudioManager(config),
            this.renderer = new Renderer(config),
            this.input = new InputHandler(config)
        ];
    }

    reset() {
        // Reset each component
        this.components.forEach(component => component.reset());
        
        // Reset game-specific state
        this.scoreManager.reset();
        this.stateMachine.transition('Menu');
        this.cloudManager.reset();
    }

    start() {
        this.stateMachine.transition('Playing');
    }

    gameOver() {
        this.stateMachine.transition('GameOver');
    }
}
```

## Configuration and Constants

### Central Configuration Object

All game constants should be in `config.js`:

```javascript
export const CONFIG = {
    // Canvas
    SCREEN_WIDTH: 800,
    SCREEN_HEIGHT: 600,
    TARGET_FPS: 60,
    FRAME_TIME_MS: 16.67,
    FRAME_TIMEOUT_MS: 17.67,

    // Physics
    GRAVITY: -9.8,
    JUMP_VELOCITY: -500,
    TERMINAL_VELOCITY_UP: -600,
    TERMINAL_VELOCITY_DOWN: 800,

    // Ghost
    GHOST_X: 50,
    GHOST_INITIAL_Y_RATIO: 0.5,
    GHOST_SPRITE_SIZE: 32,
    GHOST_HITBOX_SIZE: 30,

    // Pipes
    PIPE_SPEED: 100,
    PIPE_GAP: 150,
    PIPE_GAP_VARIANCE: 5,
    PIPE_WIDTH: 60,
    PIPE_SPAWN_INTERVAL: 1500,

    // Clouds
    CLOUD_MIN_SPEED: 30,
    CLOUD_MAX_SPEED: 60,
    CLOUD_MIN_OPACITY: 0.3,
    CLOUD_MAX_OPACITY: 0.7,
    CLOUD_MIN_COUNT: 3,
    CLOUD_MAX_COUNT: 8,

    // Collision & Response
    INVINCIBILITY_MS: 500,
    SCREEN_SHAKE_AMPLITUDE: 20,
    SCREEN_SHAKE_DURATION: 300,

    // Audio
    JUMP_SOUND_DURATION: 0.3,
    GAME_OVER_SOUND_DURATION: 1.5,

    // UI
    SCORE_BAR_HEIGHT: 40,
    HIGH_SCORE_KEY: 'flappyKiro_highScore'
};
```