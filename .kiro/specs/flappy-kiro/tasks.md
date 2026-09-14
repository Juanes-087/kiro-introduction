# Implementation Plan: Flappy Kiro Cloud and Physics Enhancement

## Overview

This implementation plan covers two major enhancements to Flappy Kiro:
1. **Cloud System**: Adds background clouds with parallax scrolling for depth effect
2. **Enhanced Physics**: Implements terminal velocity limits and frame interpolation for smoother movement

## Tasks

- [ ] 1. Implement Cloud System
  - [ ] 1.1 Create Cloud data structure with x, y, opacity, speed, width, height properties
    - **Requirements: 11.1, 11.2, 11.3**
    - Define Cloud interface with all required fields
    - Use TypeScript or JavaScript depending on project conventions

  - [ ] 1.2 Create CloudConfig interface with minOpacity (0.3), maxOpacity (0.7), minSpeed (30), maxSpeed (60)
    - **Requirements: 11.6, 11.3**
    - Define configuration for cloud generation parameters
    - Set default values for spawnInterval (3 seconds) and cloudCount (5-8)

  - [ ] 1.3 Implement CloudManager class with update(), render(), spawnCloud()
    - **Requirements: 11.1, 11.2, 11.3, 11.4, 11.5**
    - Create class that manages cloud array
    - Implement update() to move clouds based on speed
    - Implement render() to draw clouds with ctx.globalAlpha
    - Implement spawnCloud() to generate random clouds

  - [ ] 1.4 Add cloud spawning logic (one cloud every 3 seconds)
    - **Requirements: 11.2, 11.9**
    - Track time since last cloud spawn
    - Spawn new cloud at right edge when interval elapsed
    - Ensure maximum cloud count (3-8 active clouds)

  - [ ] 1.5 Implement parallax scrolling by varying cloud speeds (30-60 px/s)
    - **Requirements: 11.3**
    - Assign random speed between 30-60 px/s to each cloud
    - Update cloud positions based on deltaTime and speed
    - Create depth effect through variable speeds

  - [ ] 1.6 Implement cloud rendering with ctx.globalAlpha for semi-transparency
    - **Requirements: 11.6**
    - Use ctx.save() and ctx.restore() for alpha context
    - Set ctx.globalAlpha to cloud.opacity before drawing
    - Reset alpha to 1.0 after drawing

- [ ] 2. Implement Enhanced Physics System
  - [ ] 2.1 Update Ghost interface with terminalVelocity (upward: -600, downward: +800), previousPosition, interpolationFactor
    - **Requirements: 12.1, 12.2**
    - Add terminalVelocity field (use separate values for up/down or single max)
    - Add previousPosition for interpolation calculation
    - Add interpolationFactor for smooth movement

  - [ ] 2.2 Implement velocity clamping function clampVelocity(velocity) with terminal limits
    - **Requirements: 12.1, 12.2**
    - Clamp upward velocity to -600 px/s maximum
    - Clamp downward velocity to +800 px/s maximum
    - Use conditional logic to enforce limits

  - [ ] 2.3 Implement frame interpolation calculation: position = previousPosition + (currentPosition - previousPosition) * interpolationFactor
    - **Requirements: 12.6**
    - Store previous position before updating
    - Calculate interpolation factor based on frame timing
    - Apply linear interpolation formula for smooth position

  - [ ] 2.4 Update PhysicsEngine.update() to apply gravity, clamp velocity, and interpolate position
    - **Requirements: 12.3, 12.4, 12.5, 12.7, 12.8**
    - Order of operations: gravity → clamp → interpolate
    - Update ghost.velocity after gravity application
    - Update ghost.y with interpolated position

  - [ ] 2.5 Test that jump physics still work with new terminal velocity and interpolation
    - **Requirements: 12.4**
    - Verify jump velocity (-500 px/s) is within limits
    - Confirm gravity still affects jump arc
    - Ensure smooth visual movement with interpolation

- [ ] 3. Integration Tasks
  - [ ] 3.1 Update Renderer to draw clouds before pipes (z-order: clouds, pipes, ghost, score)
    - **Requirements: 9.3**
    - Modify renderPlaying() to accept clouds array
    - Draw clouds first using renderClouds() helper
    - Then draw pipes, ghost, and score in order

  - [ ] 3.2 Update Game Loop to include CloudManager.update() call
    - **Requirements: 11.1**
    - Add CloudManager instance to game loop
    - Call CloudManager.update(deltaTime) before rendering
    - Pass clouds array to Renderer

  - [ ] 3.3 Verify all cloud and physics requirements are satisfied
    - **Requirements: 11.x, 12.x**
    - Check all acceptance criteria from Requirements 11 and 12
    - Test in different game states (Menu, Playing, Game Over)
    - Verify z-order and rendering performance

  - [ ] 3.4 Run property-based tests for physics interpolation and cloud parallax effect
    - **Requirements: 12.6, 11.3**
    - Test interpolation preserves velocity changes smoothly
    - Verify parallax effect works with varying cloud speeds
    - Test edge cases (very high deltaTime, zero deltaTime)

- [ ] 4. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "2.1"] },
    { "id": 1, "tasks": ["1.2", "2.2"] },
    { "id": 2, "tasks": ["1.3", "2.3"] },
    { "id": 3, "tasks": ["1.4", "2.4"] },
    { "id": 4, "tasks": ["1.5", "2.5"] },
    { "id": 5, "tasks": ["1.6", "3.1"] },
    { "id": 6, "tasks": ["3.2", "3.3"] },
    { "id": 7, "tasks": ["3.4"] }
  ]
}
```

## Notes

- Tasks 1.x implement the Cloud System (Requirements 11)
- Tasks 2.x implement the Enhanced Physics System (Requirements 12)
- Tasks 3.x handle integration and testing
- All tasks are required for complete feature implementation
- Property-based tests validate correctness of physics interpolation and cloud parallax scrolling