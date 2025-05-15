# MagSimFPSv2
# 3D Magnetic Physics Shooter Simulation (Three.js + Rapier.js)

## Overview

This HTML and JavaScript application demonstrates a 3D first-person interactive physics simulation. Users can navigate a 3D environment, shoot magnetic balls, and observe their physical interactions based on magnetic dipole forces and torques. The simulation is built using [Three.js](https://threejs.org/) for 3D rendering and graphics, and [Rapier.js](https://rapier.rs/) for the 3D physics engine.

The core of the simulation involves:
1.  A player-controlled character in a 3D world.
2.  The ability to shoot spherical objects ("magnetic balls").
3.  A physics-based model for the magnetic interaction (attraction, repulsion, and alignment torques) between these balls, simulating them as magnetic dipoles.
4.  Various pre-defined scenarios to spawn pairs of magnetic balls with specific orientations to observe different magnetic interactions.
5.  Controls for pausing, stepping, and toggling magnetism.

## Features

*   **First-Person Controls:**
    *   Standard W, A, S, D for movement.
    *   Mouse for looking around (aiming).
    *   Spacebar for jumping.
*   **Physics-Based Player:** The player character is a dynamic rigid body (capsule shape) interacting with the environment's physics (gravity, collisions).
*   **Shooting Mechanism:**
    *   Clicking the mouse shoots a "magnetic ball" from the player's viewpoint.
    *   Each ball is a dynamic rigid body and a magnetic dipole.
    *   The initial orientation of the ball's magnetic pole is aligned with the shooting direction.
*   **Magnetic Dipole Simulation:**
    *   When magnetism is enabled (default), shot balls interact with each other.
    *   The script calculates the force between any two magnetic balls using the formula for **magnetic dipole-dipole interaction**. This results in attraction or repulsion depending on their relative positions and orientations.
    *   It also calculates the **torque** that each magnetic dipole exerts on the other, causing them to try and align (or anti-align).
    *   The magnetic strength of the balls is a defined constant.
*   **Dynamic 3D Environment:**
    *   A flat ground plane.
    *   Several dynamic boxes that can be pushed around by the player or shot balls.
*   **Debug/Test Scenarios:**
    *   Dedicated keys allow the user to spawn pairs of magnetic balls in pre-defined positions and orientations to easily observe specific magnetic interactions (attraction, repulsion, side-by-side interactions, etc.).
    *   A scenario for spawning a pair with randomized orientations is also included.
*   **Simulation Controls:**
    *   **Toggle Magnetism (M):** Turn the magnetic force calculations on or off.
    *   **Pause/Play (P):** Pause or resume the physics simulation. Player camera can still be moved while paused.
    *   **Step Physics (O):** When paused, advance the physics simulation by a single fixed timestep.
*   **User Interface (HUD):**
    *   **Instructions Overlay:** Displayed on startup, showing controls. Disappears on click to play.
    *   **Info Display:** Shows the current status of magnetism (ON/OFF) and the number of active magnetic pairs interacting.
    *   **Simulation Status:** Shows whether the physics simulation is "Running" or "Paused".

## How It Works

### 1. Setup (HTML & CSS)
*   The `index.html` file sets up the webpage structure.
*   Basic CSS is used for a full-screen experience and for the overlays (instructions, info display, simulation status).
*   An `importmap` is used to conveniently load Three.js, Rapier.js, and PointerLockControls from CDN URLs.

### 2. Core Libraries
*   **Three.js:** Handles all 3D rendering, including scene setup, camera, lighting, materials, and geometries (spheres for magnetic balls, planes for the floor, boxes).
*   **Rapier.js:** A 3D physics engine responsible for:
    *   Creating rigid bodies (static for the ground, dynamic for player, boxes, and magnetic balls).
    *   Defining colliders for these bodies (shapes like cuboids, capsules, spheres).
    *   Simulating physics: gravity, collisions, restitution (bounciness), friction.
    *   Applying forces and torques calculated by our magnetic simulation.
    *   Stepping the physics world forward in time.

### 3. Initialization (`initThree`, `initRapier`, `initPlayerControls`)
*   **`initThree()`:**
    *   Sets up the Three.js perspective camera, scene, and WebGL renderer.
    *   Adds ambient and spotlight (with shadows) to the scene.
    *   Creates a large plane geometry for the floor and some decorative boxes.
*   **`initRapier()`:**
    *   Initializes the Rapier physics world with specified gravity.
    *   Creates a static rigid body and collider for the ground.
    *   Creates a dynamic rigid body (capsule) for the player, with properties like damping and locked rotations.
    *   Creates several dynamic box rigid bodies and colliders.
    *   Defines materials for the magnetic balls (`ballMaterialOriginal`, `ballMaterialRed`) which are used to visually distinguish poles (red hemisphere is North).
*   **`initPlayerControls()`:**
    *   Sets up `PointerLockControls` from Three.js addons for first-person mouse look.
    *   Adds event listeners for keyboard input (WASD, Space, M, P, O, T, R, G, V) to handle player movement, shooting, magnetism toggle, pause/step, and debug scenario spawning.
    *   Manages the display of the initial instructions overlay and handles pausing the simulation when pointer lock is lost (e.g., by pressing Escape).

### 4. Magnetic Ball Creation (`createMagneticBall`, `shootBall`)
*   **`createMagneticBall()`:**
    *   Creates a Three.js sphere mesh with two material groups. By default, one hemisphere is colored red (representing the North pole).
    *   Creates a corresponding Rapier dynamic rigid body (sphere collider).
    *   Sets `userData` on the Rapier body to store its `localMagneticPoleAxis` (defined as local +Y). This axis, when transformed by the ball's current rotation, gives the world-space direction of its magnetic North pole.
    *   Applies the given initial position, velocity, and rotation to the ball.
*   **`shootBall()`:**
    *   Called on mouse click when controls are locked.
    *   Determines the shooting direction based on the camera's current orientation.
    *   Calculates a spawn position slightly in front of the camera.
    *   Calculates an initial rotation for the ball so its local +Y axis (North pole) aligns with the shooting direction.
    *   Calls `createMagneticBall` with these parameters.

### 5. Debug Spawn Scenarios (`spawnScenarios`, `spawnDebugScenario`)
*   The `spawnScenarios` array holds definitions for various test setups. Each scenario defines:
    *   A `name` for logging.
    *   A keyboard `key` to trigger it.
    *   An array of `balls`, each specifying:
        *   `pos`: An array `[x, y, z]` for the ball's initial position.
        *   `rot`: An array `[dx, dy, dz]` representing a target world direction for the ball's North pole (local +Y axis), or the string `'random'`.
*   A helper function `R_HELPER` converts the compact `rot` data into a `THREE.Quaternion`.
*   `spawnDebugScenario()` is called when a scenario key is pressed. It clears any existing magnetic balls and then creates new ones based on the selected scenario's configuration.

### 6. Magnetic Interaction Logic (`processMagneticInteractions`)
*   This is the core physics calculation loop for magnetism. It runs every frame if magnetism is enabled and there are at least two magnetic balls.
*   **For each unique pair of magnetic balls (A and B):**
    1.  **Get World State:** Retrieves the current world position and rotation (as a quaternion) for both balls from Rapier.
    2.  **Calculate World Magnetic Moments (`mA_world`, `mB_world`):**
        *   Takes the `localMagneticPoleAxis` (e.g., `(0,1,0)`) of each ball.
        *   Applies the ball's current world rotation quaternion to this local axis.
        *   Scales the result by `MAGNETIC_STRENGTH_BALL` to get the world-space magnetic dipole moment vector.
    3.  **Calculate Separation Vector (`r_AB_vec`):** The vector pointing from the center of ball A to the center of ball B. Its magnitude `r_mag` and squared magnitude `r_mag_sq` are also computed. A minimum distance is enforced to prevent forces from becoming infinite.
    4.  **Calculate Torque:**
        *   The magnetic field `B_AatB` (field generated by ball A at ball B's position) is calculated using the dipole field formula: `B_A(r) = (μ₀ / (4π)) * [ (3(mA ⋅ r_hat)r_hat - mA) / |r|³ ]`.
        *   The torque on ball B is then `torqueOnB = mB_world × B_AatB`.
        *   A similar calculation is done for the torque on ball A due to ball B's field (`torqueOnA`).
    5.  **Calculate Force:**
        *   The force on ball B due to ball A (`forceOnB`) is calculated using the full vector formula for dipole-dipole interaction:
            `F_on_B = (3μ₀ / (4π|r|⁵)) * [ (mA⋅r)mB + (mB⋅r)mA + (mA⋅mB)r - 5(mA⋅r)(mB⋅r)r/|r|² ]`
            (where `r` is `r_AB_vec`, `mA` is `mA_world`, `mB` is `mB_world`).
    6.  **Apply Force/Torque Capping (Optional):** If `APPLY_FORCE_CAP` is true, the magnitudes of the calculated forces and torques are capped to `MAX_APPLIED_FORCE` and `MAX_APPLIED_TORQUE` respectively to prevent extreme values and potential instability.
    7.  **Apply to Physics Engine:**
        *   The calculated (and possibly capped) `forceOnB` is applied to ball B's Rapier rigid body.
        *   An equal and opposite force (`-forceOnB`) is applied to ball A's rigid body (Newton's Third Law).
        *   The calculated (and possibly capped) `torqueOnA` and `torqueOnB` are applied to their respective Rapier rigid bodies.
    8.  **Logging (Debug):** If `DEBUG_LOG_CALCULATIONS` is true, details of the calculated forces and torques for the first interacting pair are printed to the console.

### 7. Main Loop (`animate`)
*   Uses `requestAnimationFrame` for a smooth animation loop.
*   Calculates `dt_wall` (delta time based on wall clock).
*   **Player Update (`updatePlayer`):**
    *   Called if pointer lock is active.
    *   Reads keyboard input (`moveState`).
    *   Moves the camera directly based on input and `dt_wall` for immediate responsiveness, even if physics is paused.
    *   If physics is not paused or is being stepped:
        *   Calculates physics-based impulses for player movement based on `moveState`, camera direction, and a fixed `timeStep`.
        *   Applies these impulses to the player's Rapier rigid body.
        *   Handles jump impulses.
*   **Physics Simulation:**
    *   If the simulation is not paused or `stepPhysics` is true:
        *   Calls `processMagneticInteractions()` to calculate and apply magnetic forces/torques.
        *   Calls `physicsWorld.step()` to advance the Rapier physics simulation by one fixed `timeStep`.
        *   Resets `stepPhysics` flag if it was true.
*   **Camera-Body Sync:** If physics is running (not paused) and controls are locked, the camera's position is explicitly synced to the player's physics body's position to ensure they stay together.
*   **Mesh Synchronization:** Iterates through all `dynamicObjects` and updates the position and rotation of their Three.js meshes to match the state of their corresponding Rapier rigid bodies.
*   **Rendering:** Calls `renderer.render(scene, camera)` to draw the 3D scene.
*   Updates HUD elements.

## How to Run

1.  Save the entire code block as an HTML file (e.g., `magnetic_shooter.html`).
2.  Open this HTML file in a modern web browser that supports ES6 Modules and WebGL (e.g., Chrome, Firefox, Edge, Safari).
3.  The simulation should start, displaying the "Click to play" instructions.

## Controls Summary

*   **W, A, S, D:** Move player (forward, left, backward, right).
*   **Mouse:** Look around / Aim.
*   **Spacebar:** Jump.
*   **Left Mouse Click:** Shoot a magnetic ball.
*   **M:** Toggle magnetism ON/OFF for the shot balls.
*   **T:** Spawn "Attraction Head-Tail" debug pair.
*   **R:** Spawn "Repel Head-Head" debug pair.
*   **F:** Spawn "Side Repel (N away)" debug pair.
*   **G:** Spawn "Side Attract (N away, S away)" debug pair.
*   **B:** Spawn "Vertical Attract (N up, N down)" debug pair.
*   **V:** Spawn a pair of balls with randomized orientations.
*   **P:** Pause or Play the physics simulation.
*   **O:** If paused, step the physics simulation forward by one frame.
*   **Escape (Esc):** Release mouse pointer lock (shows instructions again, pauses simulation).

## Potential Future Enhancements (Not Implemented)

*   More complex magnetic materials or shapes.
*   Eddy current damping for more realistic energy loss.
*   More sophisticated visual feedback for magnetic fields.
*   Game objectives or interactive elements beyond shooting balls.
*   Sound effects.
*   Performance optimizations for a very large number of magnetic objects.
