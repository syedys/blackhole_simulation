# Black Hole Simulation Programming Guide

This guide describes how to build a C++ black hole visualization that combines
relativistic ray tracing, an accretion disk renderer, and a spacetime curvature
demonstration. The focus is on providing the core algorithms, recommended data
structures, and tips for achieving real-time interactivity.

## 1. Project Overview

1. **Language and libraries**
   - C++20 for modern language features and strong performance.
   - `glm` for vector/matrix math, `Eigen` as an alternative.
   - `OpenGL` (via GLFW + GLEW/GLAD) or `Vulkan` for GPU rendering.
   - `ImGui` for debug UI and parameter tweaking.
   - `OpenMP` or `TBB` if you need CPU parallelization.
2. **Executable layout**
   - `main.cpp`: application entry, window management, render loop.
   - `renderer/`: shader compilation, framebuffer management.
   - `simulation/`: physics logic, ray tracing kernels.
   - `assets/`: shader source, textures, and lookup tables.
3. **Core systems**
   - Scene graph with camera, black hole, accretion disk, background stars.
   - Parameter controllers for black hole mass/spin, disk inclination, etc.

## 2. Fundamental Physics Modeling

### 2.1 Metric selection
- Start with the Schwarzschild metric for a non-rotating black hole:
  \[
  ds^2 = -\left(1-\frac{2GM}{c^2r}\right)c^2dt^2 + \left(1-\frac{2GM}{c^2r}\right)^{-1}dr^2 + r^2(d\theta^2 + \sin^2\theta\,d\phi^2)
  \]
- Extend later to the Kerr metric for spinning black holes if needed.

### 2.2 Null geodesic integration
- Use Hamiltonian formulation to trace light rays through curved spacetime.
- Parameterize rays by constants of motion (energy, angular momentum) to
  initialize geodesic equations.
- Integrate using a 4th-order Runge-Kutta method with adaptive step size.
- Terminate rays when:
  1. `r < r_event_horizon` (absorbed)
  2. `r > r_max` (escaped to skybox)
  3. Integration exceeds iteration budget

### 2.3 Accretion disk emission
- Model thin-disk temperature profile `T(r) ∝ r^{-3/4}`.
- Convert temperature to spectral radiance using Planck's law.
- Include relativistic beaming: compute observed intensity `I_obs = g^3 I_emitted`
  where `g` is the redshift factor from ray tracing.

## 3. Ray Tracing Pipeline

1. **Primary ray generation**
   - From camera pixel `(x, y)`, compute direction in camera space.
   - Transform to Boyer-Lindquist coordinates.
   - Set initial conditions for null geodesic: `(t, r, θ, φ)` and conjugate momenta.
2. **Integrator**
   - Solve geodesic ODEs for `r`, `θ`, `φ`, and their derivatives.
   - Use `double` precision; store state in `struct RayState { glm::dvec4 x, p; }`.
3. **Intersection tests**
   - Check crossing with equatorial plane for accretion disk hit.
   - If hit, compute disk emissive color using texture lookup or procedural model.
   - Otherwise, sample background cubemap for escaped rays.
4. **Color accumulation**
   - Support multi-scattering halos by spawning secondary rays around disk plane.
   - For each scattering event, attenuate intensity to simulate optical depth.
5. **Optimization**
   - Precompute lookup tables for constants of motion.
   - Batch integrate rays in compute shader for GPU acceleration.

## 4. Accretion Disk and Halo Rendering

### 4.1 Disk geometry
- Represent disk as annulus `[r_in, r_out]` in the equatorial plane.
- Use textured quads or procedural shading in fragment shader.
- Apply Doppler beaming and gravitational redshift from ray tracing step.

### 4.2 Volumetric halo
- Add optically thin emission around disk:
  - Sample volume density `ρ(r, θ)` (e.g., exponential falloff).
  - Integrate emissive + absorptive contributions along ray segments.
  - Use ray marching with `N <= 64` steps for performance.

### 4.3 Temporal dynamics
- Animate disk turbulence by scrolling noise textures.
- Allow adjustable accretion rates to control brightness.

## 5. Spacetime Curvature Visualization

1. **Grid setup**
   - Create a uniform grid in Euclidean space (lines/planes).
   - Warp grid vertices by embedding Schwarzschild coordinates into 3D space using
     optical embedding diagrams (Flamm's paraboloid):
     \[
     z(r) = 2\sqrt{r_s(r - r_s)}
     \]
   - Render grid with transparency to emphasize curvature.
2. **Trapdoor effect**
   - Animate particle moving along grid sliding into horizon to illustrate the
     "trapdoor".
   - Use blending to show grid lines converging.
3. **Toggle**
   - Provide UI toggle between flat space and curved embedding for comparison.

## 6. Application Structure

### 6.1 Main loop
```cpp
int main() {
    App app;
    if (!app.init()) return -1;
    while (app.isRunning()) {
        app.handleInput();
        app.update(deltaTime);
        app.render();
    }
    app.shutdown();
}
```

### 6.2 Modules
- `Renderer`: manages framebuffers, post-processing, tone mapping.
- `RayTracer`: dispatches geodesic integration compute shader or CPU threads.
- `AccretionDisk`: stores disk parameters, emits radiance values.
- `SpacetimeGrid`: generates and animates embedding diagram mesh.
- `UIOverlay`: provides sliders for mass, spin, disk temperature, camera FOV.

### 6.3 Shader overview
- `raytrace.comp`: integrates geodesics, outputs radiance buffer.
- `disk.frag`: converts ray hit info to HDR color.
- `grid.vert/frag`: renders curvature grid.
- `post.frag`: bloom, tone mapping, gamma correction.

## 7. Optional Real-Time Optimizations

1. **GPU compute**
   - Implement ray tracing in compute shader; store state in SSBO.
   - Use wavefront path tracing to reduce divergence.
2. **Adaptive resolution**
   - Use dynamic resolution scaling based on frame time.
   - Render half-resolution background with upsampling.
3. **Temporal accumulation**
   - Accumulate radiance over multiple frames with reprojection.
   - Clamp history to avoid ghosting.
4. **Level-of-detail**
   - Simplify geodesic integration for rays far from black hole using analytic
     lensing approximations.

## 8. Testing and Validation

- Compare ray deflection angles with analytic predictions for Schwarzschild
  metric to verify integrator accuracy.
- Validate disk color vs. published images (e.g., EHT visuals).
- Profile CPU/GPU frame times to ensure stable real-time performance (16 ms
  budget).
- Provide automated tests for math utilities and integrator using Catch2 or
  GoogleTest.

## 9. Further Enhancements

- Extend to Kerr metric with frame dragging.
- Add photon ring caustics and polarization visuals.
- Incorporate VR mode using OpenXR.

Use this outline as a roadmap to implement a compelling black hole simulation
that showcases gravitational lensing, accretion dynamics, and spacetime
curvature within an interactive C++ application.
