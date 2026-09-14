# Requirements Document

## Introduction

Flappy Kiro is a retro browser-based endless scroller game where players guide a ghost character through a series of vertically oriented pipes. The game features increasing difficulty as the player progresses, with the ghost automatically falling due to gravity and requiring player input to jump upward. The game ends when the ghost collides with a pipe or the ground.

## Glossary

- **Flappy Kiro**: The browser-based endless scroller game
- **Ghost**: The player-controlled character (ghosty.png)
- **Pipe**: Vertical obstacle that the Ghost must pass through
- **Score**: Number of pipes successfully passed
- **Game State**: Current mode of the game (Menu, Playing, Game Over)
- **Jump**: Upward movement of the Ghost triggered by player input, defined as the period when the Ghost's vertical velocity is negative
- **Gravity**: Constant downward force affecting the Ghost
- **Score Counter**: UI element displaying the current score
- **High Score**: Best score achieved in previous games
- **Screen Dimensions**: 800x600 pixels for coordinate reference
- **Velocity**: Measured in screen Y-axis pixels per second where negative values indicate upward movement
- **Cloud**: Background decorative element that creates depth and parallax effect
- **Terminal Velocity**: Maximum speed limit for the Ghost during freefall or ascent
- **Interpolation**: Smooth movement calculation between frames using momentum conservation
- **Hitbox**: Bounding box used for collision detection between game entities
- **Invincibility Frames**: Temporary period during which collision responses are disabled
- **Screen Shake**: Visual effect that displaces the camera briefly on collision
- **Particle Trail**: Visual trail effect trailing behind the Ghost's movement
- **Floating Indicator**: Temporary UI element showing score increments
- **CONFIG**: The exported constants object from config.js; the single source of truth for all tunable game values

## Requirements

### Requirement 1: Main Game Loop

**User Story:** As a player, I want the game to run continuously, so that I can play without manual restarts between frames.

#### Acceptance Criteria

1. THE Game Loop SHALL update the game state at 60 frames per second with a timeout of 16.67ms ± 1ms per frame
2. WHILE the Game State is Playing, THE Game Loop SHALL move the Ghost downward due to gravity with a constant acceleration of -9.8 pixels/second²
3. WHILE the Game State is Playing, THE Game Loop SHALL scroll pipes from right to left at 100 pixels/second
4. IF a collision occurs, THEN THE Game State SHALL transition to Game Over, preserving the Ghost's coordinates at collision position
5. IF the frame update exceeds the timeout, THEN THE Game Loop SHALL log an error and continue to the next frame

### Requirement 2: Player Control

**User Story:** As a player, I want to control the Ghost's movement, so that I can navigate through pipes.

#### Acceptance Criteria

1. WHEN the player presses the spacebar, THE Ghost SHALL jump upward with initial velocity of -500 pixels per second
2. WHEN the player clicks the mouse, THE Ghost SHALL jump upward with initial velocity of -500 pixels per second
3. WHILE the Ghost is jumping, THE Gravity SHALL apply at -1500 pixels/second² to reduce upward velocity over time
4. WHERE the Game State is Menu, THE spacebar press or mouse click SHALL transition the Game State to Playing
5. Velocity is measured in screen Y-axis pixels per second where negative values indicate upward movement
6. The jump state SHALL be defined as the period when the Ghost's vertical velocity is negative

### Requirement 3: Pipe Generation

**User Story:** As a player, I want pipes to appear continuously, so that the game provides endless obstacles.

#### Acceptance Criteria

1. WHEN a pipe exits the left side of the screen, THE Spawner SHALL create a new pipe pair at the right edge
2. WHERE the Game State is Playing, THE Spawner SHALL generate a pipe pair every 1.5 seconds
3. THE pipe pair SHALL consist of a top pipe and bottom pipe with a gap of 150 pixels ± 5 pixels between them
4. WHERE the Game State is Menu, THE Spawner SHALL NOT generate pipes
5. WHERE the Game State is Game Over, THE Spawner SHALL NOT generate pipes
6. IF a pipe pair fails to spawn, THEN THE Spawner SHALL retry up to 3 times with a 100 millisecond delay between attempts

### Requirement 4: Collision Detection

**User Story:** As a player, I want the game to detect collisions, so that I lose when hitting obstacles.

#### Acceptance Criteria

1. WHEN a collision occurs between the Ghost and a pipe, THE Game State SHALL transition to Game Over and preserve coordinates
2. WHEN a collision occurs between the Ghost and the top (y=0) or bottom (y=screen_height) boundary of the screen, THE Game State SHALL transition to Game Over
3. WHEN a collision occurs, THE Game Over Sound SHALL play once with a timeout of 200ms maximum for sound initiation
4. THE Collision Detector SHALL check for overlaps between the Ghost's bounding box and pipe bounding boxes, as well as screen boundaries
5. THE collision box SHALL be defined as the pixel-perfect overlap area between two sprites

### Requirement 5: Scoring System

**User Story:** As a player, I want my score to be tracked, so that I can measure my progress.

#### Acceptance Criteria

1. WHEN the Ghost passes through a pipe gap with center x-coordinate between pipe left and right edges, THE Score Counter SHALL increment by 1 and mark that pipe as scored
2. WHERE the Game State is Game Over, THE Score Counter SHALL display the final score in the center of the screen
3. THE High Score SHALL store the maximum score achieved across game sessions using browser localStorage
4. WHERE the Game State is Playing, THE Score Counter SHALL be visible on screen with 24-point font
5. A pipe SHALL be scored only once per game session, preventing score duplication from revisiting the same pipe

### Requirement 6: Game States

**User Story:** As a player, I want distinct game states, so that I understand the current phase of the game.

#### Acceptance Criteria

1. THE Game State Machine SHALL have three states: Menu, Playing, and Game Over
2. WHEN the game loads, THE Game State SHALL be Menu
3. WHEN the player presses spacebar or clicks mouse in Menu state, THE Game State SHALL transition to Playing
4. WHEN a collision is detected per Requirement 4, THE Game State SHALL transition from Playing to Game Over
5. WHEN the player presses spacebar in Game Over state, THE Game State SHALL reset to Menu
6. THE Game State Machine SHALL validate transitions to prevent invalid state changes

### Requirement 7: Audio System

**User Story:** As a player, I want sound effects, so that the game feels more engaging.

#### Acceptance Criteria

1. WHEN the Ghost jumps, THE Sound Manager SHALL play the Jump Sound for its full duration (0.3 seconds)
2. WHEN a collision occurs between Ghost and pipe OR Ghost and ground, THE Sound Manager SHALL play the Game Over Sound for its full duration (1.5 seconds)
3. WHERE the Game State is Playing, THE Sound Manager SHALL play at most one sound effect per frame
4. THE Audio System SHALL preload all sound assets before the Game State transitions to Menu
5. IF a sound asset fails to load, THEN THE Sound Manager SHALL log an error and continue without that sound effect

### Requirement 8: Asset Loading

**User Story:** As a developer, I want assets to load properly, so that the game displays correctly.

#### Acceptance Criteria

1. WHEN the Game State transitions to Menu, THE Asset Manager SHALL load ghosty.png, jump.wav, and game_over.wav
2. IF any asset fails to load after 3 retry attempts, THEN THE Game SHALL display an error message and halt execution
3. All assets SHALL be loaded from the assets/ directory relative to the game HTML file
4. The Asset Manager SHALL validate asset format and dimensions before marking assets as loaded

### Requirement 9: Visual Display

**User Story:** As a player, I want to see the game elements, so that I can navigate and play.

#### Acceptance Criteria

1. WHILE the Game State is Menu, THE Renderer SHALL display the title "Flappy Kiro" at center X, y=25% screen height
2. WHILE the Game State is Playing, THE Renderer SHALL display the Ghost at its current x=50px, y=50% screen height position
3. WHILE the Game State is Playing, THE Renderer SHALL display pipes at their current positions with z-order behind the Ghost
4. WHILE the Game State is Playing, THE Renderer SHALL display the Score Counter in the upper center of the screen with 24-point font
5. WHILE the Game State is Game Over, THE Renderer SHALL display "Game Over" at center X, y=30% and final score at center X, y=50%
6. The Renderer SHALL maintain a minimum frame rate of 30 FPS for visual smoothness

### Requirement 10: Game Reset

**User Story:** As a player, I want to restart the game after losing, so that I can try again.

#### Acceptance Criteria

1. WHEN the player presses spacebar in Game Over state, THE Game State SHALL reset to Menu after a 100ms debounce period
2. WHEN transitioning from Game Over to Menu, THE Score Counter SHALL reset to 0
3. WHEN transitioning from Game Over to Menu, THE Ghost position SHALL reset to x=50 pixels, y=50% of screen height
4. WHEN transitioning from Game Over to Menu, THE pipes SHALL be removed from the active pipe array
5. Screen dimensions SHALL be assumed as 800x600 pixels for coordinate reference

### Requirement 11: Cloud System

**User Story:** As a player, I want background clouds to create depth, so that the game world feels more immersive and dynamic.

#### Acceptance Criteria

1. WHERE the Game State is Playing, THE Cloud System SHALL render clouds in the background layer behind pipes and the Ghost
2. WHERE the Game State is Playing, THE Cloud System SHALL generate clouds at random horizontal positions on the right edge of the screen
3. WHILE the Game State is Playing, THE Cloud System SHALL scroll clouds from right to left at varying speeds: 30 pixels/second for distant clouds, 60 pixels/second for closer clouds
4. WHERE the Game State is Menu, THE Cloud System SHALL NOT render clouds
5. WHERE the Game State is Game Over, THE Cloud System SHALL NOT render clouds
6. CLOUDS SHALL have semi-transparent opacity between 0.3 and 0.7 to create depth effect
7. THE Cloud System SHALL generate at least 3 clouds and at most 8 clouds active on screen simultaneously
8. WHEN a cloud exits the left side of the screen, THE Cloud System SHALL remove it from the active cloud array
9. WHEN a cloud exits the left side of the screen, THE Cloud System SHALL spawn a new cloud at the right edge with random vertical position and speed

### Requirement 12: Enhanced Physics System

**User Story:** As a player, I want smoother and more realistic movement, so that the Ghost feels more responsive and natural to control.

#### Acceptance Criteria

1. THE Ghost SHALL have a maximum downward terminal velocity of 800 pixels/second
2. THE Ghost SHALL have a maximum upward terminal velocity of 600 pixels/second
3. WHILE the Game State is Playing, THE Physics System SHALL apply constant gravity of -9.8 pixels/second² to the Ghost's vertical velocity
4. WHEN the player triggers a jump, THE Physics System SHALL apply initial upward velocity of -500 pixels/second to the Ghost
5. WHILE the Ghost is moving, THE Physics System SHALL use momentum conservation to calculate velocity changes incrementally each frame
6. WHILE the Game State is Playing, THE Physics System SHALL interpolate the Ghost's position between frames using linear interpolation (lerp) for smooth visual movement
7. THE Physics System SHALL update velocity using the formula: new_velocity = current_velocity + (gravity * delta_time)
8. THE Physics System SHALL update position using the formula: new_position = current_position + (velocity * delta_time)
9. WHERE the Game State is Menu, THE Physics System SHALL maintain the Ghost at x=50 pixels, y=50% screen height with zero velocity
10. WHERE the Game State is Game Over, THE Physics System SHALL maintain the Ghost's final position and velocity at collision
### Requirement 14: Enhanced Collision Detection

**User Story:** As a player, I want precise collision detection with feedback, so that I get fair gameplay and visual confirmation when collisions occur.

#### Acceptance Criteria

1. THE Hitbox Definition SHALL be a bounding box of 30x30 pixels centered on the Ghost's sprite position
2. WHEN a collision occurs between the Ghost and the top boundary (y = 0), THE Collision Detector SHALL register the collision and prevent the Ghost from moving above the ceiling
3. WHEN a collision occurs between the Ghost and the bottom boundary (y + height = screen_height), THE Collision Detector SHALL register the collision and prevent the Ghost from moving below the ground
4. WHEN a collision is detected, THE Collision Response System SHALL trigger a 500ms invincibility frame period during which no additional collision responses occur
5. WHILE in invincibility frames, THE Collision Response System SHALL apply a 50% opacity effect to the Ghost sprite to indicate invulnerability
6. WHEN a collision with a pipe occurs, THE Collision Response System SHALL trigger a 20-pixel screen shake effect lasting 300ms
7. THE Collision Response System SHALL use pixel-perfect bounding box overlap detection between the Ghost hitbox and pipe hitboxes
8. WHEN invincibility frames expire, THE Collision Response System SHALL restore the Ghost to full opacity

### Requirement 15: Comprehensive Game State Management

**User Story:** As a player, I want complete game state handling with persistent scores, so that I can navigate the game smoothly and see my progress across sessions.

#### Acceptance Criteria

1. THE Main Menu State SHALL display the title "Flappy Kiro" at center screen, y=25%, and the High Score at center screen, y=50%
2. THE Gameplay State SHALL track real-time scoring with score increments when the Ghost successfully passes through pipes
3. WHEN the player presses the Escape key or spacebar during Gameplay State, THE Game State SHALL transition to Paused State
4. WHERE the Game State is Paused, THE Game Loop SHALL update physics but SHALL NOT update game entities or render animations
5. WHERE the Game State is Paused, THE Renderer SHALL display "PAUSED" at center screen, y=40%
6. WHEN the player presses Escape key or spacebar in Paused State, THE Game State SHALL resume Gameplay State from the paused position
7. THE Game Over Screen SHALL display "Game Over" at center screen, y=30%, the final Score at center screen, y=50%, and a "Press Space to Restart" prompt at center screen, y=70%
8. WHERE the Game State is Game Over, WHEN the player presses spacebar, THE Game State SHALL reset to Menu State after a 100ms debounce period
9. THE High Score Storage System SHALL persist the maximum score achieved using browser localStorage with key "flappyKiro_highScore"
10. WHEN loading the game, THE High Score Storage System SHALL load the stored high score value on initialization

### Requirement 16: Audio and Visual Feedback System

**User Story:** As a player, I want rich audio-visual feedback, so that the game feels responsive and engaging.

#### Acceptance Criteria

1. WHEN the Ghost flaps/jumps, THE Audio System SHALL play the Jump Sound effect at full volume
2. WHEN the Ghost successfully passes through a pipe, THE Audio System SHALL play a Scoring Sound effect (0.1 second duration)
3. WHEN a collision occurs, THE Audio System SHALL play the Game Over Sound effect at full volume
4. WHERE the Game State is Menu, THE Audio System SHALL play a 30-second background music loop at 50% volume
5. WHERE the Game State is Playing, THE Audio System SHALL pause background music and play only sound effects
6. WHEN a collision with a pipe occurs, THE Visual System SHALL trigger a screen shake effect with 20-pixel amplitude for 300ms
7. WHILE the Game State is Playing, THE Visual System SHALL generate particle trails behind the Ghost's position, creating a fading tail effect with 50-pixel maximum length
8. WHERE the Game State is Playing, WHEN the score increments, THE Visual System SHALL display a floating "+1" indicator at the Ghost's current position for 500ms
9. THE Particle Trail System SHALL generate one particle every 3 frames during gameplay
10. THE Particle Trail System SHALL fade particles linearly over 1 second from full opacity to transparent

### Requirement 17: Game Configuration File

**User Story:** As a developer, I want all game constants centralized in a single configuration file, so that I can tune gameplay parameters without searching through logic files.

#### Acceptance Criteria

1. THE Game SHALL include a dedicated `config.js` file at the project root containing all tunable constants as a single exported `CONFIG` object
2. WHERE any game module requires a numeric or string constant (physics values, dimensions, colors, timing, audio volumes), THE module SHALL import that value from `config.js` rather than hardcoding it
3. THE `config.js` file SHALL be plain JavaScript (no build step required) and SHALL load directly in the browser via a script tag before all other game scripts
4. THE `CONFIG` object SHALL contain constants organized by category: Canvas, Physics, Ghost, Pipes, Clouds, Collision, Particles, Floating Indicators, Score UI, Audio, and Game Over/Pause UI
5. WHEN a developer changes a constant in `config.js`, THE change SHALL take effect across all game modules without modifying any other file
6. THE `CONFIG` object SHALL be treated as read-only at runtime - no game module SHALL mutate its values