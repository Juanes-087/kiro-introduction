# Design Document: Flappy Kiro

## Overview

Flappy Kiro is a browser-based endless scroller game where players guide a ghost character through vertically oriented pipes. The game runs at 60 FPS using the Canvas API and requestAnimationFrame, featuring a state machine with Menu, Playing, and Game Over states.

**Technical Stack:**
- Canvas API for rendering
- requestAnimationFrame for game loop
- localStorage for high score persistence
- Web Audio API for sound playback
- Pure JavaScript with no external dependencies

---

## Architecture

```mermaid
graph TB
    subgraph "Game Core"
        GameLoop[Game Loop]
        StateMachine[State Machine]
    end
    
    subgraph "Components"
        Renderer[Renderer]
        InputHandler[Input Handler]
        PhysicsEngine[Physics Engine]
        PipeSpawner[Pipe Spawner]
        CollisionDetector[Collision Detector]
        ScoreManager[Score Manager]
        AudioManager[Audio Manager]
        AssetManager[Asset Manager]
    end
    
    GameLoop --> StateMachine
    GameLoop -->|updates| PhysicsEngine
    GameLoop -->|triggers| PipeSpawner
    GameLoop -->|checks| CollisionDetector
    GameLoop -->|calls| Renderer
    
    InputHandler --> PhysicsEngine
    InputHandler --> StateMachine
    
    CollisionDetector --> StateMachine
    
    PipeSpawner --> CollisionDetector
    
    ScoreManager --> Renderer
    ScoreManager --> StateMachine
    
    AudioManager --> StateMachine
    AssetManager --> Audio
    AssetManager --> Renderer
```

### Component Interactions

```
┌─────────────────────────────────────────────────────────────────┐
│                         Game Loop (60 FPS)                      │
├─────────────────────────────────────────────────────────────────┤
│  1. InputHandler processes user input                           │
│  2. PhysicsEngine updates Ghost position                        │
│  3. PipeSpawner generates new pipes (if needed)                 │
│  4. CollisionDetector checks for collisions                     │
│  5. ScoreManager updates score (if passed pipes)                │
│  6. Renderer draws all elements                                 │
│  7. StateMachine validates state transitions                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Game Loop Implementation

The game loop uses `requestAnimationFrame` for smooth 60 FPS rendering:

```javascript
function gameLoop(timestamp) {
    const deltaTime = timestamp - lastTimestamp;
    
    // Limit frame time to prevent spiral of death
    if (deltaTime > 16.67 + 1) {
        console.error('Frame rate too slow');
        lastTimestamp = timestamp;
        requestAnimationFrame(gameLoop);
        return;
    }
    
    update(deltaTime);
    render();
    
    lastTimestamp = timestamp;
    requestAnimationFrame(gameLoop);
}
```

**Frame Timing:**
- Target: 16.67ms per frame (60 FPS)
- Threshold: 17.67ms (60 FPS - 1ms tolerance)
- If exceeded: Log error and continue to next frame

---

## State Machine Design

```mermaid
stateDiagram-v2
    [*] --> Menu
    Menu --> Playing: Spacebar/Click
    Playing --> GameOver: Collision detected
    GameOver --> Menu: Spacebar
    
    state Menu {
        [*] --> DisplayTitle
        DisplayTitle --> WaitInput
    }
    
    state Playing {
        [*] --> UpdatePhysics
        UpdatePhysics --> CheckCollisions
        CheckCollisions --> UpdateScore
        UpdateScore --> RenderFrame
    }
    
    state GameOver {
        [*] --> PlayGameOverSound
        PlayGameOverSound --> DisplayGameOver
        DisplayGameOver --> WaitReset
    }
```

### State Transitions

| Current State | Trigger | Next State | Condition |
|---------------|---------|------------|-----------|
| Menu | Spacebar/Click | Playing | Valid transition |
| Playing | Collision | Game Over | Ghost touches pipe or boundary |
| Game Over | Spacebar | Menu | 100ms debounce passed |

### State Validation

- Invalid transitions are rejected and logged
- State is validated before each game loop iteration
- Default state on load: Menu

---

## Component Breakdown

### 1. AssetManager

**Responsibilities:**
- Load and cache game assets (images, audio)
- Validate asset format and dimensions
- Handle retry logic for failed loads

**Interface:**
```typescript
interface AssetManager {
    loadAssets(): Promise<void>
    getAsset(name: string): HTMLImageElement | HTMLAudioElement
    preload(): Promise<void>
}
```

**Asset Loading Strategy:**
- Load ghosty.png as HTMLImageElement
- Load jump.wav and game_over.wav as HTMLAudioElement
- Retry failed loads up to 3 times with 100ms delay
- Halt execution if assets fail to load after 3 retries

---

### 2. InputHandler

**Responsibilities:**
- Listen for spacebar and mouse click events
- Map input to game actions
- Debounce Game Over to Menu transition

**Interface:**
```typescript
interface InputHandler {
    addEventListener(callback: (action: 'jump' | 'start') => void): void
    removeEventListener(callback: () => void): void
}
```

**Event Mapping:**
- Spacebar (keydown) → 'jump' action
- Mouse click → 'jump' action
- Spacebar in Game Over → 'start' action (with 100ms debounce)

---

### 3. PhysicsEngine

**Responsibilities:**
- Calculate Ghost position based on gravity and velocity
- Apply constant gravity acceleration
- Update positions for all game entities

**Interface:**
```typescript
interface PhysicsEngine {
    update(ghost: Ghost, deltaTime: number): void
    jump(ghost: Ghost): void
    applyGravity(ghost: Ghost, deltaTime: number): void
}
```

**Physics Constants:**
- Gravity: -9.8 pixels/second² (downward acceleration)
- Jump velocity: -500 pixels/second (upward initial velocity)
- Pipe scroll speed: 100 pixels/second (right to left)

**Velocity Convention:**
- Negative values = upward movement
- Positive values = downward movement
- Jump state = period when vertical velocity is negative

---

### 4. PipeSpawner

**Responsibilities:**
- Generate pipe pairs at regular intervals
- Spawn pipes off-screen to the right
- Remove pipes that exit the left side of the screen

**Interface:**
```typescript
interface PipeSpawner {
    update(deltaTime: number, pipes: Pipe[]): void
    spawnPipePair(): PipePair
    shouldSpawn(): boolean
}
```

**Spawn Configuration:**
- Spawn interval: 1.5 seconds (only in Playing state)
- Pipe gap: 150 pixels ± 5 pixels
- Retry attempts: 3 with 100ms delay on failure
- No spawning in Menu or Game Over states

**Pipe Generation Logic:**
1. Check if spawn interval has elapsed
2. Generate random vertical offset for pipe gap
3. Create top and bottom pipe pair
4. Position pipes at screen right edge (x = 800)

---

### 5. CollisionDetector

**Responsibilities:**
- Detect collisions between Ghost and pipes
- Detect collisions between Ghost and screen boundaries
- Mark collision events for game state transition

**Interface:**
```typescript
interface CollisionDetector {
    checkGhostToPipes(ghost: Ghost, pipes: Pipe[]): boolean
    checkGhostToBoundaries(ghost: Ghost): boolean
    getCollisionBox(sprite: any): BoundingBox
}
```

**Collision Detection Strategy:**
- Pixel-perfect bounding box overlap detection
- Check all active pipes for collision
- Check top boundary (y = 0) and bottom boundary (y = screen_height)
- Preserve Ghost coordinates at collision position

---

### 6. ScoreManager

**Responsibilities:**
- Track score during gameplay
- Update high score in localStorage
- Manage score display

**Interface:**
```typescript
interface ScoreManager {
    incrementScore(pipe: Pipe): void
    getHighScore(): number
    saveScore(score: number): void
    reset(): void
}
```

**Scoring Logic:**
- Increment when Ghost center x-coordinate passes pipe center x-coordinate
- Mark pipe as scored to prevent duplication
- High score persisted to localStorage across sessions

---

### 7. AudioManager

**Responsibilities:**
- Play sound effects at appropriate times
- Preload audio assets before game starts
- Limit sound effects to one per frame

**Interface:**
```typescript
interface AudioManager {
    playJumpSound(): void
    playGameOverSound(): void
    preload(): Promise<void>
    playOneSoundPerFrame(): void
}
```

**Audio Configuration:**
- Jump sound duration: 0.3 seconds
- Game over sound duration: 1.5 seconds
- At most one sound effect per frame
- Continue operation if sound fails to load (with error log)

---

### 8. Renderer

**Responsibilities:**
- Draw all game elements to Canvas
- Handle rendering for each game state
- Maintain minimum 30 FPS for visual smoothness

**Interface:**
```typescript
interface Renderer {
    renderMenu(): void
    renderPlaying(ghost: Ghost, pipes: Pipe[], score: number): void
    renderGameOver(score: number, highScore: number): void
    clearCanvas(): void
}
```

**Rendering Order (Z-index):**
1. Background (optional)
2. Pipes
3. Ghost
4. Score display (overlay)

**Visual Elements:**
- Title "Flappy Kiro" at center X, y = 25% screen height (Menu)
- Ghost at x = 50px, y = 50% screen height (Playing)
- Score counter at upper center, 24-point font (Playing)
- "Game Over" at center X, y = 30% and final score at y = 50% (Game Over)

---

## Data Structures

### Ghost

```typescript
interface Ghost {
    x: number;              // Horizontal position (fixed at 50px)
    y: number;              // Vertical position (50% screen height initially)
    velocity: number;       // Vertical velocity in pixels/second
    width: number;          // Sprite width
    height: number;         // Sprite height
    isJumping: boolean;     // Current jump state
}
```

### Pipe

```typescript
interface Pipe {
    id: string;             // Unique identifier for scoring
    x: number;              // Horizontal position
    topPipeHeight: number;  // Height of top pipe
    bottomPipeY: number;    // Y position of bottom pipe
    width: number;          // Pipe width (40-60 pixels)
    gap: number;            // Gap between pipes (150 pixels ± 5)
    scored: boolean;        // Whether this pipe has been scored
}
```

### PipePair

```typescript
interface PipePair {
    topPipe: Pipe;
    bottomPipe: Pipe;
    spawnTime: number;
}
```

### GameConfig

```typescript
interface GameConfig {
    screen_width: number;   // 800 pixels
    screen_height: number;  // 600 pixels
    gravity: number;        // -9.8 pixels/second²
    jump_velocity: number;  // -500 pixels/second
    pipe_scroll_speed: number; // 100 pixels/second
    spawn_interval: number; // 1.5 seconds
    pipe_gap: number;       // 150 pixels
    ghost_x: number;        // 50 pixels
    ghost_initial_y: number; // 50% screen height
}
```

### BoundingBox

```typescript
interface BoundingBox {
    x: number;              // Top-left x coordinate
    y: number;              // Top-left y coordinate
    width: number;          // Width of box
    height: number;         // Height of box
}
```

---

## Sequence Diagrams

### Jump Sequence

```mermaid
sequenceDiagram
    participant Player
    participant InputHandler
    participant PhysicsEngine
    participant GameLoop
    
    Player->>InputHandler: Press spacebar or click
    InputHandler->>InputHandler: Validate game state
    InputHandler->>PhysicsEngine: jump(ghost)
    PhysicsEngine->>PhysicsEngine: Set velocity = -500 px/s
    PhysicsEngine->>PhysicsEngine: Set isJumping = true
    PhysicsEngine-->>InputHandler: Acknowledge
    InputHandler-->>Player: Visual feedback
    Note over GameLoop: PhysicsEngine.applyGravity() runs each frame
```

### Pipe Generation Sequence

```mermaid
sequenceDiagram
    participant GameLoop
    participant PipeSpawner
    participant GameConfig
    
    GameLoop->>PipeSpawner: update(deltaTime, pipes)
    PipeSpawner->>PipeSpawner: Check spawn interval elapsed
    PipeSpawner->>GameConfig: Get spawn_interval (1.5s)
    alt Spawn Interval Elapsed
        PipeSpawner->>PipeSpawner: Generate random gap position
        PipeSpawner->>PipeSpawner: Create top pipe
        PipeSpawner->>PipeSpawner: Create bottom pipe
        PipeSpawner-->>GameLoop: Return new PipePair
        GameLoop->>GameLoop: Add PipePair to active pipes array
    end
    PipeSpawner->>PipeSpawner: Remove pipes that exited screen
```

### Collision Detection Sequence

```mermaid
sequenceDiagram
    participant GameLoop
    participant CollisionDetector
    participant Ghost
    participant Pipe
    participant StateMachine
    
    GameLoop->>CollisionDetector: checkGhostToPipes(ghost, pipes)
    CollisionDetector->>CollisionDetector: For each pipe in pipes
    alt Collision Detected
        CollisionDetector->>CollisionDetector: Check bounding box overlap
        CollisionDetector-->>GameLoop: Return true
        GameLoop->>StateMachine: transition(Playing, GameOver)
        StateMachine->>StateMachine: Validate transition
        StateMachine-->>GameLoop: Transition confirmed
        GameLoop->>AudioManager: playGameOverSound()
    else No Collision
        CollisionDetector-->>GameLoop: Return false
    end
    GameLoop->>CollisionDetector: checkGhostToBoundaries(ghost)
    alt Ghost at boundary
        CollisionDetector-->>GameLoop: Return true
        GameLoop->>StateMachine: transition(Playing, GameOver)
    end
```

---

## Technical Implementation Details

### Canvas API

```javascript
// Canvas setup
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
canvas.width = 800;
canvas.height = 600;

// Rendering
ctx.drawImage(image, x, y, width, height);
ctx.font = '24px Arial';
ctx.fillStyle = '#FFFFFF';
ctx.fillText(text, x, y);
```

### requestAnimationFrame

```javascript
let lastTimestamp = 0;

function gameLoop(timestamp) {
    const deltaTime = timestamp - lastTimestamp;
    
    // Frame time validation
    if (deltaTime > 17.67) {
        console.error('Frame rate too slow');
    }
    
    update(deltaTime);
    render();
    
    lastTimestamp = timestamp;
    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

### localStorage

```javascript
// High score persistence
const HIGH_SCORE_KEY = 'flappyKiro_highScore';

function getHighScore() {
    const stored = localStorage.getItem(HIGH_SCORE_KEY);
    return stored ? parseInt(stored, 10) : 0;
}

function saveScore(score) {
    const currentHigh = getHighScore();
    if (score > currentHigh) {
        localStorage.setItem(HIGH_SCORE_KEY, score.toString());
    }
}
```

### Web Audio API

```javascript
// Audio playback
const jumpSound = new Audio('assets/jump.wav');
const gameOverSound = new Audio('assets/game_over.wav');

function playJumpSound() {
    jumpSound.currentTime = 0;
    jumpSound.play().catch(err => console.error('Audio play failed:', err));
}
```

### Asset Loading

```javascript
// Image loading with retry
function loadImage(src, retries = 3): Promise<HTMLImageElement> {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.src = src;
        img.onload = () => resolve(img);
        
        let attempts = 0;
        img.onerror = () => {
            attempts++;
            if (attempts >= retries) {
                reject(new Error(`Failed to load ${src}`));
            } else {
                setTimeout(() => img.src = src, 100);
            }
        };
    });
}
```

---

## Error Handling

### Asset Loading Errors
- Retry up to 3 times with 100ms delay
- Display error message and halt if all retries fail
- Log error details to console

### Frame Rate Errors
- Log error if frame exceeds 17.67ms
- Continue to next frame (do not skip)
- Prevents spiral of death

### Audio Errors
- Log error if sound fails to play
- Continue operation without that sound effect
- Allow game to function without audio

### Collision Detection Errors
- Gracefully handle missing collision data
- Default to no collision if detection fails
- Log unexpected error conditions

---

## Testing Strategy

### Unit Tests
- PhysicsEngine: Test gravity calculations and jump physics
- PipeSpawner: Test spawn timing and pipe generation
- CollisionDetector: Test bounding box overlap detection
- ScoreManager: Test score increment and high score persistence

### Integration Tests
- Full game loop execution (100+ iterations)
- State transitions (Menu → Playing → Game Over → Menu)
- Asset loading and audio playback

### Property-Based Tests
- **Property 1**: Ghost position after jump follows parabolic trajectory
- **Property 2**: All pipes scroll at constant speed
- **Property 3**: Score increments exactly once per pipe pass
- **Property 4**: High score is always >= current score

---

## Implementation Notes

1. **Frame Time Management**: Always validate frame time to prevent performance degradation

2. **Coordinate System**: Canvas origin (0,0) is top-left; positive Y is downward

3. **Z-Order**: Render pipes before ghost, score as overlay

4. **Memory Management**: Remove off-screen pipes from active array

5. **Performance**: Use object pooling for pipes if needed

6. **Accessibility**: Consider keyboard-only navigation support

7. **Cross-Browser**: Test with requestAnimationFrame vendor prefixes if needed

---

## File Structure

```
kiro-introduction/
├── assets/
│   ├── ghosty.png
│   ├── jump.wav
│   └── game_over.wav
├── index.html
└── game.js (main game implementation)
```

### index.html
- Canvas element with id="gameCanvas"
- Basic styling and layout
- Script reference to game.js

### game.js
- Game loop implementation
- Component classes and interfaces
- Asset loading and management
- Input handling
- State machine logic
- Rendering functions

---

## Cloud System Design

### Cloud Interface

```typescript
interface Cloud {
    x: number;              // Horizontal position (pixels)
    y: number;              // Vertical position (pixels)
    opacity: number;        // Opacity (0.3-0.7 for visibility)
    speed: number;          // Scroll speed (30-60 pixels/second)
    width: number;          // Cloud width (pixels)
    height: number;         // Cloud height (pixels)
}
```

### Cloud Configuration

```typescript
interface CloudConfig {
    minOpacity: number;     // Minimum opacity (0.3)
    maxOpacity: number;     // Maximum opacity (0.7)
    minSpeed: number;       // Minimum scroll speed (30 px/s)
    maxSpeed: number;       // Maximum scroll speed (60 px/s)
    spawnInterval: number;  // Time between cloud spawns (seconds)
    cloudCount: number;     // Maximum number of clouds on screen
}
```

### CloudManager Component

**Responsibilities:**
- Manage cloud array with spawn, update, and render operations
- Implement parallax scrolling with different speed layers
- Generate clouds with random Y positions and varying opacities
- Handle cloud removal when they exit the screen

**Interface:**
```typescript
interface CloudManager {
    update(deltaTime: number): void
    render(ctx: CanvasRenderingContext2D): void
    spawnCloud(): Cloud
    getClouds(): Cloud[]
}
```

**Cloud Generation Strategy:**
1. Randomize Y position within screen bounds (preserving pipe space)
2. Randomize opacity between 0.3-0.7
3. Randomize speed between 30-60 pixels/second
4. Randomize cloud width and height for variety
5. Position clouds off-screen to the right initially

**Parallax Scrolling Implementation:**
- Clouds move slower than pipes (30-60 px/s vs 100 px/s)
- Creates depth perception effect
- Different cloud layers can have different speeds

**Rendering Order:**
1. Clouds (background layer)
2. Pipes
3. Ghost
4. Score (overlay layer)

---

## Enhanced Physics System

### Updated Ghost Interface

```typescript
interface Ghost {
    x: number;              // Horizontal position (fixed at 50px)
    y: number;              // Vertical position (50% screen height initially)
    velocity: number;       // Vertical velocity in pixels/second
    terminalVelocity: number; // Maximum velocity (downward: 800, upward: -600)
    previousPosition: number; // Previous frame position for interpolation
    width: number;          // Sprite width
    height: number;         // Sprite height
    isJumping: boolean;     // Current jump state
    interpolationFactor: number; // Interpolation factor (0-1)
}
```

### PhysicsEngine Enhancement

**Responsibilities:**
- Calculate Ghost position with terminal velocity limits
- Implement momentum conservation with velocity smoothing
- Frame interpolation for smooth movement

**Updated Interface:**
```typescript
interface PhysicsEngine {
    update(ghost: Ghost, deltaTime: number): void
    jump(ghost: Ghost): void
    applyGravity(ghost: Ghost, deltaTime: number): void
    updateWithInterpolation(ghost: Ghost, deltaTime: number): number
    clampVelocity(velocity: number): number
}
```

**Terminal Velocity Limits:**
- Upward: -600 pixels/second (maximum jump speed)
- Downward: +800 pixels/second (maximum fall speed)

**Velocity Clamping Logic:**
```javascript
function clampVelocity(velocity: number): number {
    const minVelocity = -600; // Terminal velocity upward
    const maxVelocity = 800;  // Terminal velocity downward
    
    if (velocity < minVelocity) return minVelocity;
    if (velocity > maxVelocity) return maxVelocity;
    return velocity;
}
```

**Frame Interpolation:**
```javascript
// Calculate interpolation factor based on frame timing
function calculateInterpolationFactor(deltaTime: number, targetFPS: number = 16.67): number {
    return Math.min(deltaTime / targetFPS, 1.0);
}

// Update position with interpolation
function updateWithInterpolation(ghost: Ghost, deltaTime: number): number {
    // Store previous position
    ghost.previousPosition = ghost.y;
    
    // Calculate current position
    const currentY = ghost.y + ghost.velocity * (deltaTime / 1000);
    
    // Calculate interpolation factor
    ghost.interpolationFactor = calculateInterpolationFactor(deltaTime);
    
    // Apply interpolation: position = previous + (current - previous) * factor
    const interpolatedPosition = ghost.previousPosition + 
                                  (currentY - ghost.previousPosition) * 
                                  ghost.interpolationFactor;
    
    return interpolatedPosition;
}
```

**Updated Update Method:**
```javascript
function update(ghost: Ghost, deltaTime: number): void {
    // Apply gravity
    applyGravity(ghost, deltaTime);
    
    // Clamp velocity to terminal velocity limits
    ghost.velocity = clampVelocity(ghost.velocity);
    
    // Update position with interpolation
    const interpolatedY = updateWithInterpolation(ghost, deltaTime);
    ghost.y = interpolatedY;
}
```

---

## Updated Component Architecture

### CloudManager Integration

**New Architecture:**
```mermaid
graph TB
    subgraph "Game Core"
        GameLoop[Game Loop]
        StateMachine[State Machine]
    end
    
    subgraph "Components"
        Renderer[Renderer]
        InputHandler[Input Handler]
        PhysicsEngine[Physics Engine]
        PipeSpawner[Pipe Spawner]
        CollisionDetector[Collision Detector]
        ScoreManager[Score Manager]
        AudioManager[Audio Manager]
        AssetManager[Asset Manager]
        CloudManager[Cloud Manager]
    end
    
    GameLoop --> StateMachine
    GameLoop -->|updates| PhysicsEngine
    GameLoop -->|updates| CloudManager
    GameLoop -->|triggers| PipeSpawner
    GameLoop -->|checks| CollisionDetector
    GameLoop -->|calls| Renderer
    
    InputHandler --> PhysicsEngine
    InputHandler --> StateMachine
    
    CollisionDetector --> StateMachine
    
    PipeSpawner --> CollisionDetector
    
    ScoreManager --> Renderer
    ScoreManager --> StateMachine
    
    AudioManager --> StateMachine
    AssetManager --> Audio
    AssetManager --> Renderer
    CloudManager --> Renderer
```

### Updated Component Interactions

```
┌─────────────────────────────────────────────────────────────────┐
│                         Game Loop (60 FPS)                      │
├─────────────────────────────────────────────────────────────────┤
│  1. InputHandler processes user input                           │
│  2. PhysicsEngine updates Ghost position                        │
│  3. CloudManager updates cloud positions                        │
│  4. PipeSpawner generates new pipes (if needed)                 │
│  5. CollisionDetector checks for collisions                     │
│  6. ScoreManager updates score (if passed pipes)                │
│  7. Renderer draws all elements                                 │
│     a. Clouds (background)                                      │
│     b. Pipes                                                    │
│     c. Ghost                                                    │
│     d. Score (overlay)                                          │
│  8. StateMachine validates state transitions                    │
└─────────────────────────────────────────────────────────────────┘
```

### Cloud Generation and Rendering Sequence

```mermaid
sequenceDiagram
    participant GameLoop
    participant CloudManager
    participant Renderer
    participant Canvas
    
    GameLoop->>CloudManager: update(deltaTime)
    CloudManager->>CloudManager: For each cloud in array
    CloudManager->>CloudManager: x = x - speed * deltaTime
    CloudManager->>CloudManager: Check if cloud exited screen
    alt Cloud Exited Screen
        CloudManager->>CloudManager: Remove cloud from array
        CloudManager->>CloudManager: Optionally spawn new cloud
    end
    CloudManager-->>GameLoop: Updated cloud array
    
    GameLoop->>Renderer: renderPlaying(ghost, pipes, clouds, score)
    Renderer->>Renderer: ctx.globalAlpha = cloud.opacity
    Renderer->>Canvas: Draw cloud at (cloud.x, cloud.y)
    Renderer->>Renderer: ctx.globalAlpha = 1.0 (reset)
    Renderer->>Renderer: Draw pipes
    Renderer->>Renderer: Draw ghost
    Renderer->>Renderer: Draw score overlay
```

---

## Technical Implementation

### CloudManager Class

```typescript
class CloudManager implements CloudManager {
    private clouds: Cloud[] = [];
    private config: CloudConfig;
    
    constructor(config: CloudConfig) {
        this.config = {
            minOpacity: 0.3,
            maxOpacity: 0.7,
            minSpeed: 30,
            maxSpeed: 60,
            spawnInterval: 3, // seconds
            cloudCount: 5,
            ...config
        };
    }
    
    update(deltaTime: number): void {
        const deltaTimeSeconds = deltaTime / 1000;
        
        // Update all clouds
        this.clouds.forEach(cloud => {
            cloud.x -= cloud.speed * deltaTimeSeconds;
        });
        
        // Remove clouds that have exited screen
        this.clouds = this.clouds.filter(cloud => cloud.x + cloud.width > 0);
        
        // Spawn new clouds if needed
        this.spawnCloudsIfNecessary();
    }
    
    render(ctx: CanvasRenderingContext2D): void {
        this.clouds.forEach(cloud => {
            ctx.save();
            ctx.globalAlpha = cloud.opacity;
            ctx.fillStyle = '#FFFFFF';
            
            // Draw cloud as ellipse
            ctx.beginPath();
            ctx.ellipse(
                cloud.x + cloud.width / 2,
                cloud.y + cloud.height / 2,
                cloud.width / 2,
                cloud.height / 2,
                0,
                0,
                Math.PI * 2
            );
            ctx.fill();
            ctx.restore();
        });
    }
    
    spawnCloud(): Cloud {
        const screen_height = 600;
        const cloud_width = 60 + Math.random() * 40;
        const cloud_height = 30 + Math.random() * 20;
        
        return {
            x: 800, // Start off-screen right
            y: Math.random() * (screen_height - 100) + 50,
            opacity: Math.random() * (this.config.maxOpacity - this.config.minOpacity) + this.config.minOpacity,
            speed: Math.random() * (this.config.maxSpeed - this.config.minSpeed) + this.config.minSpeed,
            width: cloud_width,
            height: cloud_height
        };
    }
    
    private spawnCloudsIfNecessary(): void {
        if (this.clouds.length < this.config.cloudCount) {
            if (Math.random() < 0.02) { // Small chance per frame
                this.clouds.push(this.spawnCloud());
            }
        }
    }
    
    getClouds(): Cloud[] {
        return this.clouds;
    }
}
```

### PhysicsEngine with Interpolation

```typescript
class PhysicsEngine implements PhysicsEngine {
    private readonly GRAVITY = -9.8;
    private readonly JUMP_VELOCITY = -500;
    private readonly TERMINAL_VELOCITY_UPWARD = -600;
    private readonly TERMINAL_VELOCITY_DOWNWARD = 800;
    
    update(ghost: Ghost, deltaTime: number): void {
        // Apply gravity
        this.applyGravity(ghost, deltaTime);
        
        // Clamp velocity to terminal velocity limits
        ghost.velocity = this.clampVelocity(ghost.velocity);
        
        // Update position with interpolation
        const interpolatedY = this.updateWithInterpolation(ghost, deltaTime);
        ghost.y = interpolatedY;
    }
    
    jump(ghost: Ghost): void {
        ghost.velocity = this.JUMP_VELOCITY;
    }
    
    applyGravity(ghost: Ghost, deltaTime: number): void {
        const deltaTimeSeconds = deltaTime / 1000;
        ghost.velocity += this.GRAVITY * deltaTimeSeconds;
    }
    
    updateWithInterpolation(ghost: Ghost, deltaTime: number): number {
        // Store previous position
        ghost.previousPosition = ghost.y;
        
        // Calculate current position
        const deltaTimeSeconds = deltaTime / 1000;
        const currentY = ghost.y + ghost.velocity * deltaTimeSeconds;
        
        // Calculate interpolation factor
        const targetFPS = 16.67;
        ghost.interpolationFactor = Math.min(deltaTime / targetFPS, 1.0);
        
        // Apply interpolation
        const interpolatedPosition = ghost.previousPosition + 
                                      (currentY - ghost.previousPosition) * 
                                      ghost.interpolationFactor;
        
        return interpolatedPosition;
    }
    
    clampVelocity(velocity: number): number {
        if (velocity < this.TERMINAL_VELOCITY_UPWARD) {
            return this.TERMINAL_VELOCITY_UPWARD;
        }
        if (velocity > this.TERMINAL_VELOCITY_DOWNWARD) {
            return this.TERMINAL_VELOCITY_DOWNWARD;
        }
        return velocity;
    }
}
```

### Renderer with Cloud Support

```typescript
interface Renderer {
    renderMenu(): void
    renderPlaying(ghost: Ghost, pipes: Pipe[], clouds: Cloud[], score: number): void
    renderGameOver(score: number, highScore: number): void
    clearCanvas(): void
}
```

```javascript
renderPlaying(ghost: Ghost, pipes: Pipe[], clouds: Cloud[], score: number): void {
    // 1. Draw clouds (background layer)
    clouds.forEach(cloud => {
        ctx.save();
        ctx.globalAlpha = cloud.opacity;
        ctx.fillStyle = '#FFFFFF';
        
        // Draw cloud shape
        ctx.beginPath();
        ctx.ellipse(
            cloud.x + cloud.width / 2,
            cloud.y + cloud.height / 2,
            cloud.width / 2,
            cloud.height / 2,
            0,
            0,
            Math.PI * 2
        );
        ctx.fill();
        ctx.restore();
    });
    
    // 2. Draw pipes
    pipes.forEach(pipe => {
        ctx.fillStyle = '#228B22';
        ctx.fillRect(pipe.x, 0, pipe.width, pipe.topPipeHeight);
        ctx.fillRect(pipe.x, pipe.bottomPipeY, pipe.width, screen_height - pipe.bottomPipeY);
    });
    
    // 3. Draw ghost
    ctx.drawImage(
        assetManager.getAsset('ghosty'),
        ghost.x,
        ghost.y,
        ghost.width,
        ghost.height
    );
    
    // 4. Draw score (overlay)
    ctx.font = '24px Arial';
    ctx.fillStyle = '#FFFFFF';
    ctx.fillText(`Score: ${score}`, 10, 30);
}
```
---

## Components and Interfaces

This section defines all component interfaces with their method signatures for the Flappy Kiro game.

### Core Game Components

#### AssetManager

**Responsibilities:**
- Load and cache game assets (images, audio)
- Validate asset format and dimensions
- Handle retry logic for failed loads

**Interface:**
```typescript
interface AssetManager {
    loadAssets(): Promise<void>
    getAsset(name: string): HTMLImageElement | HTMLAudioElement
    preload(): Promise<void>
    hasAsset(name: string): boolean
}
```

#### InputHandler

**Responsibilities:**
- Listen for spacebar and mouse click events
- Map input to game actions
- Debounce Game Over to Menu transition

**Interface:**
```typescript
interface InputHandler {
    addEventListener(callback: (action: 'jump' | 'start') => void): void
    removeEventListener(callback: () => void): void
    isJumping(): boolean
    isStarted(): boolean
}
```

#### PhysicsEngine

**Responsibilities:**
- Calculate Ghost position based on gravity and velocity
- Apply constant gravity acceleration
- Update positions for all game entities
- Implement terminal velocity limits and frame interpolation

**Interface:**
```typescript
interface PhysicsEngine {
    update(ghost: Ghost, deltaTime: number): void
    jump(ghost: Ghost): void
    applyGravity(ghost: Ghost, deltaTime: number): void
    updateWithInterpolation(ghost: Ghost, deltaTime: number): number
    clampVelocity(velocity: number): number
    getPosition(ghost: Ghost): number
}
```

#### PipeSpawner

**Responsibilities:**
- Generate pipe pairs at regular intervals
- Spawn pipes off-screen to the right
- Remove pipes that exit the left side of the screen

**Interface:**
```typescript
interface PipeSpawner {
    update(deltaTime: number, pipes: Pipe[]): void
    spawnPipePair(): PipePair
    shouldSpawn(deltaTime: number): boolean
    getSpawnTimer(): number
    resetSpawnTimer(): void
}
```

#### CollisionDetector

**Responsibilities:**
- Detect collisions between Ghost and pipes
- Detect collisions between Ghost and screen boundaries
- Mark collision events for game state transition

**Interface:**
```typescript
interface CollisionDetector {
    checkGhostToPipes(ghost: Ghost, pipes: Pipe[]): boolean
    checkGhostToBoundaries(ghost: Ghost, screen_height: number): boolean
    getCollisionBox(sprite: Ghost | Pipe): BoundingBox
    checkCollision boxes(box1: BoundingBox, box2: BoundingBox): boolean
}
```

#### ScoreManager

**Responsibilities:**
- Track score during gameplay
- Update high score in localStorage
- Manage score display

**Interface:**
```typescript
interface ScoreManager {
    incrementScore(pipe: Pipe): void
    getHighScore(): number
    saveScore(score: number): void
    reset(): void
    getScore(): number
    isPipeScored(pipeId: string): boolean
    markPipeScored(pipeId: string): void
}
```

#### AudioManager

**Responsibilities:**
- Play sound effects at appropriate times
- Preload audio assets before game starts
- Limit sound effects to one per frame

**Interface:**
```typescript
interface AudioManager {
    playJumpSound(): void
    playGameOverSound(): void
    preload(): Promise<void>
    playOneSoundPerFrame(): void
    enable(): void
    disable(): void
    isEnabled(): boolean
}
```

#### Renderer

**Responsibilities:**
- Draw all game elements to Canvas
- Handle rendering for each game state
- Maintain minimum 30 FPS for visual smoothness

**Interface:**
```typescript
interface Renderer {
    renderMenu(): void
    renderPlaying(ghost: Ghost, pipes: Pipe[], clouds: Cloud[], score: number): void
    renderGameOver(score: number, highScore: number): void
    clearCanvas(): void
    drawCloud(cloud: Cloud): void
    drawPipe(pipe: Pipe): void
    drawGhost(ghost: Ghost): void
    drawScore(score: number, highScore: number): void
    drawGameOver(score: number, highScore: number): void
    drawTitle(): void
}
```

#### CloudManager

**Responsibilities:**
- Manage cloud array with spawn, update, and render operations
- Implement parallax scrolling with different speed layers
- Generate clouds with random Y positions and varying opacities
- Handle cloud removal when they exit the screen

**Interface:**
```typescript
interface CloudManager {
    update(deltaTime: number): void
    render(ctx: CanvasRenderingContext2D): void
    spawnCloud(): Cloud
    getClouds(): Cloud[]
    addCloud(cloud: Cloud): void
    removeCloud(cloud: Cloud): void
}
```

#### StateMachine

**Responsibilities:**
- Manage game state transitions (Menu, Playing, GameOver)
- Validate state transitions
- Trigger state-specific entry/exit actions

**Interface:**
```typescript
interface StateMachine {
    getCurrentState(): GameState
    transition(newState: GameState): boolean
    isValidTransition(current: GameState, next: GameState): boolean
    executeStateActions(): void
    reset(): void
}
```

### Configuration Interfaces

#### GameConfig

```typescript
interface GameConfig {
    screen_width: number;           // 800 pixels
    screen_height: number;          // 600 pixels
    gravity: number;                // -9.8 pixels/second²
    jump_velocity: number;          // -500 pixels/second
    pipe_scroll_speed: number;      // 100 pixels/second
    spawn_interval: number;         // 1.5 seconds
    pipe_gap: number;               // 150 pixels
    ghost_x: number;                // 50 pixels
    ghost_initial_y: number;        // 50% screen height
    terminal_velocity_upward: number;   // -600 pixels/second
    terminal_velocity_downward: number; // 800 pixels/second
    target_fps: number;             // 60 FPS
}
```

#### CloudConfig

```typescript
interface CloudConfig {
    minOpacity: number;         // Minimum opacity (0.3)
    maxOpacity: number;         // Maximum opacity (0.7)
    minSpeed: number;           // Minimum scroll speed (30 px/s)
    maxSpeed: number;           // Maximum scroll speed (60 px/s)
    spawnInterval: number;      // Time between cloud spawns (seconds)
    cloudCount: number;         // Maximum number of clouds on screen
}
```

---

## Data Models

This section defines all data structures with TypeScript interfaces used in the Flappy Kiro game.

### Game Entities

#### Ghost

```typescript
interface Ghost {
    x: number;                  // Horizontal position (fixed at 50px)
    y: number;                  // Vertical position (50% screen height initially)
    velocity: number;           // Vertical velocity in pixels/second
    terminalVelocity: number;   // Maximum velocity (downward: 800, upward: -600)
    previousPosition: number;   // Previous frame position for interpolation
    width: number;              // Sprite width
    height: number;             // Sprite height
    isJumping: boolean;         // Current jump state
    interpolationFactor: number; // Interpolation factor (0-1)
}
```

#### Cloud

```typescript
interface Cloud {
    x: number;                  // Horizontal position (pixels)
    y: number;                  // Vertical position (pixels)
    opacity: number;            // Opacity (0.3-0.7 for visibility)
    speed: number;              // Scroll speed (30-60 pixels/second)
    width: number;              // Cloud width (pixels)
    height: number;             // Cloud height (pixels)
}
```

#### Pipe

```typescript
interface Pipe {
    id: string;                 // Unique identifier for scoring
    x: number;                  // Horizontal position
    topPipeHeight: number;      // Height of top pipe
    bottomPipeY: number;        // Y position of bottom pipe
    width: number;              // Pipe width (40-60 pixels)
    gap: number;                // Gap between pipes (150 pixels ± 5)
    scored: boolean;            // Whether this pipe has been scored
}
```

#### PipePair

```typescript
interface PipePair {
    topPipe: Pipe;
    bottomPipe: Pipe;
    spawnTime: number;
}
```

### Collision and Bounds

#### BoundingBox

```typescript
interface BoundingBox {
    x: number;                  // Top-left x coordinate
    y: number;                  // Top-left y coordinate
    width: number;              // Width of box
    height: number;             // Height of box
}
```

### Game State

#### GameState

```typescript
type GameState = 'Menu' | 'Playing' | 'GameOver';
```

#### GameContext

```typescript
interface GameContext {
    ghost: Ghost;
    pipes: Pipe[];
    clouds: Cloud[];
    score: number;
    highScore: number;
    gameState: GameState;
    deltaTime: number;
    timestamp: number;
}
```

### Utility Types

#### Callback Types

```typescript
type JumpCallback = (ghost: Ghost) => void;
type CollisionCallback = (ghost: Ghost, pipes: Pipe[]) => boolean;
type ScoreCallback = (score: number) => void;
type StateChangeCallback = (newState: GameState) => void;
```

#### Input Actions

```typescript
type InputAction = 'jump' | 'start';
```

---

## Correctness Properties

This section defines the correctness properties that should hold true for the Flappy Kiro game implementation. These properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Physics System Properties

#### Property 1: Gravity application produces correct acceleration

*For any* valid Ghost entity and deltaTime value between 10ms and 100ms, applying gravity should increase the ghost's downward velocity by exactly `-9.8 * deltaTime / 1000` pixels/second.

**Validates: Requirements Physics-1, Physics-2**

#### Property 2: Terminal velocity limits are enforced

*For any* Ghost entity, after applying gravity and before updating position, the ghost's velocity should always be clamped between -600 pixels/second (upward) and +800 pixels/second (downward).

**Validates: Requirements Physics-3**

#### Property 3: Jump action resets velocity to initial value

*For any* Ghost entity, calling the jump action should set the ghost's velocity to exactly -500 pixels/second, regardless of current velocity.

**Validates: Requirements Physics-4**

#### Property 4: Position interpolation preserves continuity

*For any* Ghost entity, after updating with interpolation, the interpolated position should always be between the previous position and current position values.

**Validates: Requirements Physics-5**

### Cloud System Properties

#### Property 5: Cloud movement produces parallax effect

*For any* Cloud entity, the cloud's horizontal position should decrease by exactly `cloud.speed * deltaTime / 1000` pixels each frame, where cloud.speed is between 30-60 pixels/second (slower than pipe speed of 100 pixels/second).

**Validates: Requirements Cloud-1**

#### Property 6: Cloud opacity remains within valid range

*For any* Cloud entity, the opacity value should always be between 0.3 and 0.7 (inclusive) throughout the entity's lifetime.

**Validates: Requirements Cloud-2**

#### Property 7: Cloud removal preserves screen boundaries

*For any* Cloud entity that exits the left side of the screen (x + width ≤ 0), the CloudManager should remove it from the active clouds array.

**Validates: Requirements Cloud-3**

#### Property 8: Cloud count respects maximum limit

*For any* CloudManager instance, the number of clouds in the active array should never exceed the configured cloudCount limit (default: 5).

**Validates: Requirements Cloud-4**

### Pipe System Properties

#### Property 9: Pipe gap is within acceptable range

*For any* PipePair generated by PipeSpawner, the gap between top and bottom pipes should be between 145 and 155 pixels (150 ± 5 pixels).

**Validates: Requirements Pipe-1**

#### Property 10: Pipe scrolling speed is constant

*For any* Pipe entity, the horizontal position should decrease by exactly `100 * deltaTime / 1000` pixels each frame (100 pixels/second scroll speed).

**Validates: Requirements Pipe-2**

#### Property 11: Pipe scoring is unique per pipe

*For any* Pipe entity, the score should increment by exactly 1 when the Ghost passes the pipe's center, and the pipe's scored flag should be set to prevent duplicate scoring.

**Validates: Requirements Pipe-3**

### Collision Detection Properties

#### Property 12: Collision detection identifies all boundary violations

*For any* Ghost entity at or beyond screen boundaries (y ≤ 0 or y + height ≥ screen_height), the collision detector should return true for boundary collision.

**Validates: Requirements Collision-1**

#### Property 13: Collision detection identifies all pipe collisions

*For any* Ghost entity whose bounding box overlaps with any active pipe's bounding box, the collision detector should return true.

**Validates: Requirements Collision-2**

#### Property 14: Bounding box collision check is symmetric

*For any* two entities with bounding boxes A and B, if A overlaps with B, then B overlaps with A.

**Validates: Requirements Collision-3**

### Score System Properties

#### Property 15: Score increments exactly once per pipe

*For any* Pipe entity that has not been scored, when the Ghost's center x-coordinate passes the pipe's center x-coordinate, the score should increment by exactly 1.

**Validates: Requirements Score-1**

#### Property 16: High score is always non-decreasing

*For any* sequence of game sessions, the high score stored in localStorage should always be greater than or equal to any previously recorded high score.

**Validates: Requirements Score-2**

#### Property 17: Score reset preserves high score

*For any* GameSession, when the game transitions to Menu state, the current score should reset to 0 while the high score remains unchanged.

**Validates: Requirements Score-3**

### State Machine Properties

#### Property 18: State transitions are valid only for defined transitions

*For any* StateMachine instance, attempting to transition from state A to state B should return true only if the transition is explicitly defined in the valid transitions table.

**Validates: Requirements State-1**

#### Property 19: Invalid state transitions are rejected

*For any* StateMachine instance, attempting to transition from Playing to Menu directly should return false and the current state should remain Playing.

**Validates: Requirements State-2**

#### Property 20: Game over state only transitions to Menu

*For any* StateMachine in GameOver state, the only valid transition is to Menu state via spacebar input.

**Validates: Requirements State-3**

### Round-Trip Properties

#### Property 21: Pipe serialization round-trip preserves identity

*For any* valid Pipe object, serializing to JSON and then deserializing should produce an equivalent Pipe object with identical field values.

**Validates: Requirements Data-1**

#### Property 22: Ghost serialization round-trip preserves state

*For any* valid Ghost object, serializing to JSON and then deserializing should produce an equivalent Ghost object with identical field values.

**Validates: Requirements Data-2**

### Error Handling Properties

#### Property 23: Frame time validation logs slow frames

*For any* game loop iteration where frame time exceeds 17.67ms, the system should log an error message to the console.

**Validates: Requirements Error-1**

#### Property 24: Asset loading retry mechanism works correctly

*For any* asset loading attempt that fails, the system should retry up to 3 times with 100ms delays between attempts before rejecting with an error.

**Validates: Requirements Error-2**

### Testing Strategy

- **Property Tests**: Implement property-based tests for all 24 properties above
- **Unit Tests**: Test specific edge cases and error conditions not covered by properties
- **Integration Tests**: Test full game loops and state transition sequences

Each property-based test should run a minimum of 100 iterations and be tagged with the property number and requirements it validates.


