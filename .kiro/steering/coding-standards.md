# Flappy Kiro - JavaScript Coding Standards

## Class Naming Conventions

### Naming Patterns

- **Classes**: PascalCase (e.g., `PhysicsEngine`, `CloudManager`, `PipeSpawner`)
- **Interfaces**: PascalCase with "I" prefix (e.g., `IInputHandler`, `IRenderer`) or no prefix
- **Functions**: camelCase (e.g., `updatePhysics`, `spawnCloud`, `checkCollision`)
- **Variables**: camelCase (e.g., `ghostVelocity`, `pipeSpeed`, `deltaTime`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `GRAVITY`, `JUMP_VELOCITY`, `TARGET_FPS`)
- **Private members**: Prefix with underscore (e.g., `_clouds`, `_spawnTimer`)

### Class Structure

```javascript
class ComponentName {
    constructor(config) {
        this.publicProperty = config.value;
        this._privateProperty = 0;
    }

    // Public methods
    update(deltaTime) { /* ... */ }

    // Private methods (private methods should use underscore prefix)
    _helperMethod() { /* ... */ }

    // Getters and setters
    get property() { return this._property; }
    set property(value) { this._property = value; }
}
```

## JavaScript Game Patterns

### Game Loop Pattern

```javascript
let lastTimestamp = 0;

function gameLoop(timestamp) {
    const deltaTime = timestamp - lastTimestamp;
    lastTimestamp = timestamp;
    
    // Frame time validation (prevent spiral of death)
    if (deltaTime > 17.67) {
        console.error('Frame rate too slow');
    }
    
    update(deltaTime);
    render();
    
    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

### Component Pattern

```javascript
class Component {
    constructor(config = {}) {
        this.config = { ...this.defaultConfig, ...config };
        this.state = this.initialState;
    }

    update(deltaTime) {
        // Update component state
    }

    render(ctx) {
        // Render component
    }

    reset() {
        // Reset component to initial state
    }
}
```

### Event Handling Pattern

```javascript
class InputHandler {
    constructor() {
        this.listeners = [];
    }

    addListener(callback) {
        this.listeners.push(callback);
    }

    removeListener(callback) {
        this.listeners = this.listeners.filter(cb => cb !== callback);
    }

    // Trigger events
    trigger(action) {
        this.listeners.forEach(cb => cb(action));
    }
}
```

### Factory Pattern for Object Creation

```javascript
class PipeSpawner {
    spawnPipePair() {
        return {
            topPipe: this.createPipe(),
            bottomPipe: this.createPipe(),
            spawnTime: Date.now()
        };
    }

    createPipe() {
        return {
            x: 800,
            width: CONFIG.PIPE_WIDTH,
            scored: false
        };
    }
}
```

## Performance Optimization Guidelines

### Frame Time Management

- **Always validate frame time** to prevent performance degradation
- **Log errors** if frame exceeds 17.67ms (60 FPS + tolerance)
- **Continue to next frame** rather than skipping frames
- **Use requestAnimationFrame** for smooth rendering (never setInterval)

### Memory Management

- **Remove off-screen objects** from active arrays immediately
- **Reset objects** instead of creating new ones where possible
- **Preload assets** before gameplay begins
- **Clear intervals** and remove event listeners on game reset

```javascript
// Good: Remove off-screen pipes
pipes = pipes.filter(pipe => pipe.x + pipe.width > 0);

// Good: Reuse particle objects (object pooling concept)
let particles = [];
for (let i = 0; i < MAX_PARTICLES; i++) {
    particles.push({ x: 0, y: 0, age: 0 });
}
```

### Rendering Optimization

- **Batch draw calls** when possible (e.g., draw all clouds in one loop)
- **Use save/restore** for canvas state to avoid state pollution
- **Minimize alpha blending** operations (expensive on some devices)
- **Draw order matters**: background → entities → foreground → UI

```javascript
// Efficient rendering with state management
ctx.save();
ctx.globalAlpha = cloud.opacity;
// Draw cloud
ctx.restore();
```

### Physics Optimization

- **Pre-calculate constants** (e.g., `deltaTimeSeconds = deltaTime / 1000`)
- **Clamp values** to prevent extreme physics behavior
- **Use simple bounding boxes** for collision detection (AABB)
- **Early exit** collision checks when possible

```javascript
// Pre-calculate for performance
const deltaTimeSeconds = deltaTime / 1000;

// Early exit collision check
if (ghost.x + ghost.width < pipe.x || ghost.x > pipe.x + pipe.width) {
    return false; // No collision possible
}
```

### Asset Loading

- **Preload all assets** before transitioning to Menu state
- **Use Promises** for async asset loading with error handling
- **Retry failed loads** up to 3 times with 100ms delays
- **Validate assets** before marking as loaded

### Code Organization

- **Separate concerns**: Input, Physics, Rendering, Audio, State
- **Single responsibility**: Each class should do one thing well
- **Configuration centralization**: All tunable values in `config.js`
- **No magic numbers**: Import all constants from CONFIG