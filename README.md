# Orbital Gravity Simulator

A real-time 3D gravitational N-body simulation with spacetime grid warping, built in C++ with OpenGL.

---

## Setup & Running

### Dependencies

Install on Ubuntu/Debian:
```bash
sudo apt install libglew-dev libglfw3-dev libglm-dev g++
```

### Compile
```bash
g++ solar_sim.cpp -o solar_sim -lGL -lGLEW -lglfw
```

### Run
```bash
./solar_sim
```

Press **K** to unpause the simulation after launch.

---

## Controls

| Input | Action |
|---|---|
| `W / A / S / D` | Move camera |
| `Space` / `Left Shift` | Move camera up / down |
| Mouse | Look around |
| Scroll wheel | Zoom in / out |
| `K` (hold) | Pause / unpause simulation |
| `Q` | Quit |
| Left-click (hold) | Spawn a new body, use arrow keys to position it |
| Right-click (while spawning) | Increase spawned body's mass |

---



## Current Solar System Bodies

| Body | Sim Radius | Orbital Velocity | Color |
|---|---|---|---|
| Star | — (stationary) | — | Yellow (glowing) |
| Mercury | 1500 | 931 | Gray |
| Venus | 2800 | 681 | Pale yellow |
| Earth | 3876 | 579 | Blue |

---

## Physics Engine

### 1. Newtonian Gravity (N-body)

Every pair of bodies attracts each other each frame using Newton's law of universal gravitation:

$$F = \frac{G m_1 m_2}{r^2}$$

This gives an acceleration applied to each body toward every other:

$$a = \frac{F}{m} = \frac{G m_2}{r^2}$$

Implemented in the inner loop of the main render loop — every body interacts with every other body every frame ($O(n^2)$ complexity).

### 2. Forward Euler Integration

Velocity and position are updated each frame using explicit (forward) Euler integration with fixed divisors acting as a time-scale:

$$v_{new} = v + \frac{a}{96}, \quad x_{new} = x + \frac{v}{94}$$

The constants `94` and `96` are simulation time-scale factors — they are not physical timesteps but tune how fast the simulation runs relative to real time.

> **Note:** This is frame-rate dependent. At higher FPS orbits are more stable; at lower FPS they can drift. A proper implementation would multiply by `deltaTime`.

### 3. Physical Radius from Mass and Density

Planet sphere sizes are derived from real density using the volume formula for a sphere:

$$V = \frac{m}{\rho} = \frac{4}{3}\pi r^3 \implies r = \left(\frac{3m}{4\pi\rho}\right)^{1/3}$$

This radius is then divided by `sizeRatio` to fit the simulation's coordinate space. A separate `displayScale` multiplier allows visual resizing without changing the collision hitbox.

### 4. Collision Response

When two bodies overlap (sum of radii > distance between centers), velocities are reversed and heavily damped:

$$v_{after} = -0.2 \cdot v_{before}$$

This produces a bouncy, energy-dissipating collision (20% restitution coefficient).

### 5. Spacetime Grid Warping — Flamm's Paraboloid

The grid is a visual representation of spacetime curvature using **Flamm's paraboloid**, derived from the Schwarzschild metric of General Relativity. For a mass $M$, the embedding depth at coordinate distance $r$ from the mass is:

$$z = 2\sqrt{r_s \left(r - r_s\right)}, \quad r_s = \frac{2GM}{c^2}$$

where $r_s$ is the **Schwarzschild radius** — the radius at which escape velocity equals the speed of light.

The grid uses a **two-pass inversion** so the center sinks (near the star) and the edges stay flat:

1. Compute displacement $z_i$ for every grid vertex from all masses
2. Find $z_{max}$ (the largest displacement, occurring at the far edges)
3. Set each vertex height to $y_i = z_i - z_{max} + \text{gridRaise}$

This guarantees the outer rim of the grid sits at $y = \text{gridRaise}$ and the star sits at the deepest point of the funnel — matching the classic "bowling ball on a rubber sheet" visualization.

> **Important:** This grid is purely visual. It does not affect the physics — orbits are computed entirely by Newtonian gravity regardless of grid shape.

### 6. Rendering

- **Vertex shader:** transforms sphere vertices through model/view/projection matrices and computes a simple diffuse light intensity using the dot product of the surface normal and a direction toward the world origin (the star).
- **Fragment shader:** applies smooth lighting to planets, a flat color to the grid, and a bloom-like overexposure (`rgb * 100000`) to glowing bodies like the star.
- Spheres are generated procedurally as stacked triangle strips using spherical coordinates.
