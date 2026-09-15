# Flappy Kiro - Canvas API Patterns

## Rendering Order (Z-Index)

Canvas rendering order from back to front:

```
1. Background fill (sky blue #87CEEB)
2. Background texture overlay (sketch pattern, low opacity)
3. Clouds (semi-transparent, parallax scrolling)
4. Pipes (green columns with darker caps)
5. Particle trail (behind Ghost)
6. Ghost sprite (player character)
7. Floating indicators ("+1" score popup)
8. UI overlay (score bar at bottom, Game Over screen)
```

## Canvas Context Setup

```javascript
// Canvas initialization
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Set canvas dimensions
canvas.width = CONFIG.SCREEN_WIDTH;  // 800
canvas.height = CONFIG.SCREEN_HEIGHT; // 600

// Canvas border
ctx.lineWidth = CONFIG.CANVAS_BORDER_WIDTH; // 3
ctx.strokeStyle = CONFIG.CANVAS_BORDER_COLOR; // '#1A1A1A'
ctx.strokeRect(0, 0, canvas.width, canvas.height);
```

## Drawing Primitives

### Rectangles (Pipes, Background)

```javascript
// Fill rectangle
ctx.fillStyle = '#2E8B22'; // Pipe color
ctx.fillRect(x, y, width, height);

// Stroke rectangle
ctx.strokeStyle = '#000000';
ctx.lineWidth = 1;
ctx.strokeRect(x, y, width, height);
```

### Images (Ghost Sprite)

```javascript
const ghostSprite = assetManager.getAsset('ghosty');
ctx.drawImage(
    ghostSprite,
    ghost.x,
    ghost.y,
    CONFIG.GHOST_SPRITE_SIZE,
    CONFIG.GHOST_SPRITE_SIZE
);
```

### Clouds (Rounded Rectangles)

```javascript
// Draw cloud as rounded rectangle
function drawCloud(ctx, cloud) {
    ctx.save();
    ctx.globalAlpha = cloud.opacity;
    ctx.fillStyle = CONFIG.CLOUD_COLOR; // '#D6EEF8'
    
    // Use ellipse for organic cloud shape
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
}
```

### Text (Score, UI)

```javascript
// Draw score
ctx.font = `${CONFIG.SCORE_FONT}`; // '18px Arial'
ctx.fillStyle = CONFIG.SCORE_TEXT_COLOR; // '#FFFFFF'
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(`Score: ${score} | High: ${highScore}`, canvas.width / 2, canvas.height - 20);

// Draw Game Over text
ctx.font = '30px Arial';
ctx.fillStyle = '#FFFFFF';
ctx.textAlign = 'center';
ctx.fillText('GAME OVER', canvas.width / 2, canvas.height * 0.3);
```

## Animation Frame Handling

### Game Loop with Frame Timing

```javascript
let lastTimestamp = 0;
let frameCount = 0;

function gameLoop(timestamp) {
    const deltaTime = timestamp - lastTimestamp;
    
    // Frame rate protection
    if (deltaTime > CONFIG.FRAME_TIMEOUT_MS) {
        console.error('Frame rate too slow:', deltaTime);
        // Continue to next frame instead of skipping
    }
    
    // Update all systems
    update(deltaTime);
    
    // Render all systems
    render();
    
    // Increment frame counter for particle spawns
    frameCount++;
    
    lastTimestamp = timestamp;
    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

### Delta Time Calculation

```javascript
// Convert deltaTime to seconds for physics calculations
const deltaTimeSeconds = deltaTime / 1000;

// Apply gravity: velocity += gravity * deltaTimeSeconds
ghost.velocity += CONFIG.GRAVITY * deltaTimeSeconds;

// Update position: position += velocity * deltaTimeSeconds
ghost.y += ghost.velocity * deltaTimeSeconds;
```

## Efficient Collision Detection

### Axis-Aligned Bounding Box (AABB)

```javascript
function checkAABB(box1, box2) {
    return (
        box1.x < box2.x + box2.width &&
        box1.x + box1.width > box2.x &&
        box1.y < box2.y + box2.height &&
        box1.y + box1.height > box2.y
    );
}

// Ghost hitbox
const ghostHitbox = {
    x: ghost.x + CONFIG.GHOST_HITBOX_SIZE / 2,
    y: ghost.y + CONFIG.GHOST_HITBOX_SIZE / 2,
    width: CONFIG.GHOST_HITBOX_SIZE,
    height: CONFIG.GHOST_HITBOX_SIZE
};

// Pipe hitbox
const pipeHitbox = {
    x: pipe.x,
    y: pipe.topPipeHeight,
    width: pipe.width,
    height: pipe.bottomPipeY - pipe.topPipeHeight
};

const collision = checkAABB(ghostHitbox, pipeHitbox);
```

### Early Exit Optimization

```javascript
// Check if objects are far apart before detailed collision
function quickReject(ghost, pipe) {
    // X-axis check - if ghost is completely to the left or right of pipe
    if (ghost.x + ghost.width < pipe.x || ghost.x > pipe.x + pipe.width) {
        return true; // No collision possible
    }
    
    // Y-axis check - if ghost is completely above or below pipe gap
    if (ghost.y + ghost.height < pipe.topPipeHeight || 
        ghost.y > pipe.bottomPipeY) {
        return true; // No collision possible
    }
    
    return false; // Collision possible, check detailed AABB
}
```

### Boundary Collision

```javascript
function checkTopBoundary(ghost) {
    return ghost.y <= 0;
}

function checkBottomBoundary(ghost, screen_height) {
    return ghost.y + ghost.height >= screen_height;
}
```

## Screen Shake Effect

```javascript
class ScreenShake {
    constructor() {
        this.active = false;
        this.amplitude = 0;
        this.duration = 0;
        this.elapsed = 0;
    }

    start(amplitude, duration) {
        this.active = true;
        this.amplitude = amplitude;
        this.duration = duration;
        this.elapsed = 0;
    }

    update(deltaTime) {
        if (!this.active) return;
        
        this.elapsed += deltaTime;
        if (this.elapsed >= this.duration) {
            this.active = false;
        }
    }

    getOffset() {
        if (!this.active) return { x: 0, y: 0 };
        
        const progress = this.elapsed / this.duration;
        const decay = 1 - progress; // Linear decay
        
        const randomX = (Math.random() - 0.5) * 2 * this.amplitude * decay;
        const randomY = (Math.random() - 0.5) * 2 * this.amplitude * decay;
        
        return { x: randomX, y: randomY };
    }
}
```

## Particle Trail System

```javascript
class ParticleTrail {
    constructor() {
        this.particles = [];
        this.maxParticles = 50;
    }

    update(ghost, deltaTime) {
        // Spawn new particle every 3 frames
        if (frameCount % 3 === 0) {
            this.particles.push({
                x: ghost.x,
                y: ghost.y,
                age: 0,
                maxAge: 1000 // 1 second
            });
        }

        // Update existing particles
        for (let i = this.particles.length - 1; i >= 0; i--) {
            const p = this.particles[i];
            p.age += deltaTime;
            
            // Remove expired particles
            if (p.age >= p.maxAge) {
                this.particles.splice(i, 1);
            }
        }
    }

    render(ctx) {
        this.particles.forEach(p => {
            const opacity = 1 - (p.age / p.maxAge);
            const size = CONFIG.PARTICLE_RADIUS * opacity;
            
            ctx.save();
            ctx.globalAlpha = opacity;
            ctx.fillStyle = '#FFFFFF';
            ctx.beginPath();
            ctx.arc(p.x, p.y, size, 0, Math.PI * 2);
            ctx.fill();
            ctx.restore();
        });
    }
}
```

## Floating Indicator System

```javascript
class FloatingIndicators {
    constructor() {
        this.indicators = [];
    }

    add(x, y) {
        this.indicators.push({
            x: x,
            y: y,
            age: 0,
            maxAge: 500 // 0.5 seconds
        });
    }

    update(deltaTime) {
        this.indicators.forEach(indicator => {
            indicator.age += deltaTime;
            indicator.y -= 100 * (deltaTime / 1000); // Float upward
        });
        
        // Remove expired indicators
        this.indicators = this.indicators.filter(i => i.age < i.maxAge);
    }

    render(ctx) {
        ctx.font = CONFIG.INDICATOR_FONT; // 'bold 20px Arial'
        ctx.fillStyle = '#FFFFFF';
        
        this.indicators.forEach(indicator => {
            const opacity = 1 - (indicator.age / indicator.maxAge);
            ctx.globalAlpha = opacity;
            ctx.fillText('+1', indicator.x, indicator.y);
        });
        
        ctx.globalAlpha = 1.0; // Reset
    }
}
```

## Background Parallax Scrolling

```javascript
class CloudManager {
    constructor() {
        this.clouds = [];
    }

    update(deltaTime) {
        const deltaTimeSeconds = deltaTime / 1000;
        
        // Move clouds at different speeds for parallax effect
        this.clouds.forEach(cloud => {
            cloud.x -= cloud.speed * deltaTimeSeconds;
        });
        
        // Remove clouds that have exited screen
        this.clouds = this.clouds.filter(cloud => cloud.x + cloud.width > 0);
        
        // Spawn new clouds if count is below maximum
        if (this.clouds.length < CONFIG.CLOUD_MAX_COUNT) {
            if (Math.random() < 0.02) { // Small chance per frame
                this.clouds.push(this.spawnCloud());
            }
        }
    }

    spawnCloud() {
        const screen_height = CONFIG.SCREEN_HEIGHT;
        
        return {
            x: CONFIG.SCREEN_WIDTH, // Start off-screen right
            y: Math.random() * (screen_height - 100) + 50,
            opacity: Math.random() * (CONFIG.CLOUD_MAX_OPACITY - CONFIG.CLOUD_MIN_OPACITY) + CONFIG.CLOUD_MIN_OPACITY,
            speed: Math.random() * (CONFIG.CLOUD_MAX_SPEED - CONFIG.CLOUD_MIN_SPEED) + CONFIG.CLOUD_MIN_SPEED,
            width: CONFIG.CLOUD_MIN_WIDTH + Math.random() * (CONFIG.CLOUD_MAX_WIDTH - CONFIG.CLOUD_MIN_WIDTH),
            height: CONFIG.CLOUD_MIN_HEIGHT + Math.random() * (CONFIG.CLOUD_MAX_HEIGHT - CONFIG.CLOUD_MIN_HEIGHT)
        };
    }
}
```