# Flappy Kiro - Product Overview

Flappy Kiro is a browser-based endless scroller game where players guide a ghost character through vertically oriented pipes. The game runs entirely in the browser using the Canvas API with no external dependencies.

## Game Description

- **Genre**: Arcade endless scroller
- **Platform**: Web browser (Canvas API)
- **Primary Mechanic**: Tap/jump to navigate a ghost through pipe obstacles
- **Visual Style**: Retro sketchbook aesthetic with sky blue background, green pipes, and hand-drawn textures

## Core Features

- **60 FPS Game Loop**: Uses requestAnimationFrame for smooth rendering
- **State Machine**: Menu, Playing, and Game Over states (with Paused state support)
- **Physics System**: Gravity-based movement with terminal velocity limits
- **Scoring**: Real-time score tracking with localStorage high score persistence
- **Audio**: Sound effects for jumping and game over using Web Audio API
- **Visual Effects**: Particle trails, screen shake, floating indicators, and parallax clouds

## Target Audience

- Casual gamers looking for quick arcade gameplay
- Developers interested in learning Canvas-based game development
- Educational purposes for understanding game loop patterns and state management

## Current Status

The project is in active development. Implementation plans are tracked via the Spec workflow in the `.kiro/specs/flappy-kiro/` directory.