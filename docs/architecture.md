# Architecture

Status: **Proposed 2026-11-15.** The modules exist and are empty. This is the plan for what goes in them, in what
order, and what each needs from the rest of the stack. Nothing here is decided until it is built or ruled; the open
questions are at the end.

## The ask

The author's, 2026-11-15:

- A game engine on the VexelRay stack.
- **It runs as a view inside a vexelray-gui application, and a vexelray-gui interface runs as an overlay inside a
  game.** Neither host is the real one and the other a port.
- **The same patterns the GUI already lives by:** an uber shader extended with shadelets, the framework's compile-time
  wiring, kronometer for clocks running at different rates, and lanes that talk only through mailboxes.
- **The headline is fractals:** how they are worked into the visuals and the 3D art, not bolted on as a special
  effect.

## Where this sits

| Module | What it is | Depends on |
| --- | --- | --- |
| `vexelray-game-core` | The world as data: what is in it, a step as messages, the clocks. No device, no window | kronometer, atchung |
| `vexelray-game-shadelets` | The 3D host for shadelets: the world's uber shader, its roles and injection points | vastir, vexelray-surface, vexelray-shader |
| `vexelray-game-fractal` | Fractals as shadelets, their level of detail, and their CPU twin | game-shadelets, vexelray-fields-fractal |
| `vexelray-game-render` | The world drawn, as `RenderTechnique`s. Knows neither a pane nor a window | game-core, game-shadelets, technique-sdf, technique-mesh |
| `vexelray-game-pane` | A game in a vexelray-gui application | game-render, vexelray-gui, framework-api |
| `vexelray-game-window` | A game in a window of its own, with a vexelray-gui overlay | game-render, technique-canvas, vexelray-gui, framework-api |

Dependencies point downward only. **`-core` and `-shadelets` never see a device**, so a world can be stepped, saved
and judged in a test with no GPU, and every shadelet runs on the CPU. **`-render` never sees a host**: the two hosts are
thin, and anything both need is in `-render` or below.

Modules that will come, and are not made until there is code for them:

| Later | What for |
| --- | --- |
| `vexelray-game-physics` | The bridge to vexelray-sim-physics: bodies and fluids meeting the world's fields, fractals included (*Fractals*, the CPU twin) |
| `vexelray-game-audio` | A mixer on vexelray-audio, its voices on the world's clock |
| `vexelray-game-assets` | Fractals, meshes and materials as files: a record of parameters, not a bake |
| `vexelray-game-demo` | The first game, in both hosts, behind a `demos` profile until it builds |

## Two hosts, one world

**A world is drawn as a list of techniques**, with the buffers they bind and the numbers they read each frame. That
is the only thing a host is given, and both hosts already know how to run one.

### In a pane (game-pane)

vexelray-gui 0.2 has this today: a `TechniqueView` in a `TechniquePane` (`vexelray-gui-core`, `-widget`). The host
owns the target and its resize, records the techniques into the frame's pre-pass through `SampledTechniqueHost`, binds
device feeds by name, and gives the technique `params()` set by the view's `@Draw`. A technique is never called by
application code (the processor's T7.6). So the pane host is: a `@View` whose `@Draw` chooses the world's snapshot
and sets the camera, a `TechniquePane` holding the world's techniques, and input through `Seams`.

One limit shows at once: **`params()` is 32 floats.** A camera is 16 of them before the time, the blend and the
exposure. Either the camera crosses as a bound buffer in a feed, or `TechniqueView.PARAMS` grows (open question 3).

### In a window of its own (game-window)

Two ways, and the plan is to take the first and measure before taking the second:

1. **A full-bleed pane.** The window is a framework application whose root is one `TechniquePane` the size of the
   window, and the overlay is ordinary nodes laid over it. Everything in it exists: the HUD, menus and debug panels
   are widgets, input reaches the overlay first and what it does not take goes to the game, and automation
   (`ottermate`) drives a game as it drives any application. The cost is one extra pass: the world is drawn into a
   target and the canvas samples it.
2. **The engine owns the frame.** `vexelray-engine` runs the window and swapchain, the world's techniques draw
   straight into it, and the GUI is the last technique (`CanvasTechnique`, depth `NONE`, as the HUD case in
   vexelray's `techniques.md`). This buys exclusive fullscreen, present-mode control, render scale and upscaling, and
   no sampling pass. It needs what does not exist: **a `Gui` tree whose layout, input and frame are driven by a host
   other than `GuiApp`.** Today only the canvas is a technique, not the widget tree. That is a plan for vexelray-gui
   to own, not this repo.

`game-window` starts as (1), so the overlay works on day one and both hosts share every line below them. (2) is
worth building only when the sampling pass measures as a cost, or when exclusive fullscreen is wanted.

**A third case comes free:** a GUI drawn *into* the world, on a surface in it (a terminal on a wall, a map on a
table). That is `technique-panel`'s canvas-on-a-plane, depth-tested, and wants nothing new but input routed through
the plane (a ray from the camera, hit against the panel's quad).

## One uber shader for the world

The canvas has one uber shader (`CanvasShader`) extended by shadelets: pure functions in SupirVast IR, each placed at
a named injection point, chained `source → modifier* → painter+ → composite → clip`, composed once per application
and compiled to SPIR-V at build time under a name that is the manifest's hash (`supirvast/docs/shadelets.md`). The
world gets the same shape in three dimensions:

| Role | Canvas | World |
| --- | --- | --- |
| source | `vec2(distance, reach)` to an edge | distance to a surface, and its **step factor**: how far a march may trust it |
| modifier | reshapes a distance | domain operations: fold, repeat, twist, smooth union, displacement |
| painter | premultiplied colour | **material**: albedo, roughness, metal, emission, from the point and the trap values its source left |
| — | — | **light**: `vexelray-shader`'s `Shading`, or `vastir-pbr`'s `PbrShader`, per the composition |
| composite | layers | fog, volume, glow along the ray, tone |

As on the canvas:

- **Per-object numbers are a style table**, one float buffer, written by records the `@Style` processor
  (`vexelray-shadelet-codegen`) generates writers for. Changing a fractal's parameters, animating it, or placing a
  thousand of them is a table write, never a recompile.
- **Every role is pure, so every role runs on the CPU** through vast. The canvas uses that for agreement tests. The
  world uses it for gameplay as well (*Fractals*, the CPU twin).
- **No function values, so a generated dispatcher**, `distanceAt(id, p)`, over the shadelets the game composed. The
  world's distance is that dispatcher over the objects a ray may meet, culled by their bounds first (`Bounds`, `Cones`
  in `vexelray-surface`).
- **A composition is the game's, once.** A game names its shadelets as an application names `DemoShader`; one not
  compiled in is dropped and reported (`Diagnostics.dropped`), not lowered at startup by surprise.

**Meshes share the material and light roles.** A rasterised mesh's fragment shader is the same composition from
*material* on, given the point and normal the rasteriser found. So a mesh can wear a fractal material, and a fractal
and a mesh next to each other are lit by the same code. Depth agrees across techniques through `ClipDepth`, as the
built-in SDF and mesh techniques already agree.

**Where the 3D host lives** is open question 1: here first, and the shared machinery (manifest, style tables,
dispatcher generation) lifted into `vexelray-canvas` or `vastir` once there are two hosts to lift it from.

## Fractals

The headline. A fractal is a shadelet like any other, so everything above applies, and what follows is what makes
them art and not a screensaver.

- **Distance estimators as sources.** Mandelbulb, Mandelbox, Menger, Sierpinski and kaleidoscopic IFS, Kleinian and
  other inversion fractals, Julia sets in quaternions. Each declares its step factor, since many overestimate. That is
  the question `vexelray/docs/clip-shells.md` left open, and it is answered here per shadelet.
- **Bounded by clip shells.** A fractal is marched only inside a mesh that holds it, rasterised first, so it is culled,
  depth-sorted and sized like a mesh. `clip-shells.md` prototyped this with a Menger sponge
  (`MengerShellTechnique`, test code in `vexelray-technique-mesh`); here it is promoted to the way every fractal is
  placed. A shell is also the level of detail's far step: past some distance the shell *is* the fractal, with a baked
  texture.
- **Detail from the pixel's footprint.** Iterations follow the cone a pixel covers, so a fractal never shimmers and
  never spends iterations below a pixel. `vexelray-fields-fractal`'s `Lod`, `Footprint` and prefiltered `Coverage`
  are the start.
- **Fractals as materials.** Orbit traps (the point's closest approach to a plane, a point, a shape, over the
  iteration) are what a source leaves for the material role, so colour comes from the fractal's own structure. A
  mesh can wear one too.
- **Fractal detail on other surfaces.** A modifier that carves or lays fractal detail onto a surface: rock that is a
  box with a Menger erosion, bark that is a cylinder with an IFS displacement, a terrain whose small scale is
  fractal. Bounded by the cone, so it costs nothing where it is below a pixel.
- **The CPU twin.** The same IR a fractal is drawn with runs on the CPU, so the game asks the *same* function where a
  surface is: collision, picking, line of sight, navigation, footstep sounds. Never a separate collision mesh that
  drifts from what is drawn, and the agreement is a test. This is also what vexelray-sim-physics wants (its TODO's
  *static SDF geometry both simulations meet*, a Menger-carved tunnel system), so the world's fields and the
  simulations' boundaries are one contract (open question 5).
- **Fractals that move.** Parameters are kronometer cells on a clock of their own, so a fractal breathes, morphs and
  unfolds in time with the world, slows with it, and reaches the GPU as a table write.
- **Scale.** A fractal invites zooming in without end. Floating point runs out long before the fractal does: a
  floating origin per region, doubles on the CPU, and what a deep zoom needs on the GPU are open question 6.

## The world and its lanes

The framework's rules (`vexelray-framework/docs/threading.md`, T1 to T7) are the engine's rules. Nothing here
invents a thread.

- **The world is a `@Component` on a lane of its own** (`lane = "world"`), stepped by a fixed rate. It hears input,
  and its own and other parts' messages, as messages, and publishes each finished step as a deeply immutable snapshot
  on a state topic. It holds no device and draws nothing.
- **The view chooses.** The pane's (or window's) `@View` runs on the GUI thread inside the frame: its `@Draw` takes the
  snapshot due, blends it with the one before by the clock's phase, and sets the camera and per-frame numbers for the
  techniques. It decides nothing about the world.
- **What lives on the GPU stays there.** Particles, instance transforms, a fluid: a `@Source` holding a lent
  `DeviceQueue` writes them into a device feed, and the techniques bind the slot the view chose. That is
  vexelray-gui's `device-feed.md`, built for the simulations, and the engine takes it as it is.
- **Slow work is a `@Job`**: loading, baking a shell's far texture, building a world chunk.

**Clocks are kronometer's**, each a rate domain on one timeline:

| Clock | Rate | What keeps it |
| --- | --- | --- |
| the world | fixed, `kron.fixed(hz(60)).maxCatchUp(n)` | the world's lane; reaches it as a message |
| the display | whatever the display runs at | the frame; `FrameStage.CLOCK` |
| animation | the display's, sampled between world steps | `kronometer-anim`, `Sampled`, `handoff` |
| fractal motion | its own tempo, a child of the world's | cells the composition reads as table writes |
| slow motion | the world's, dilated | `Dilated`, or a child tempo per region (`tempo().child(Ratio.of(1, 4))`) |
| the overlay | the GUI's | `vexelray-gui-krono`, as any application's |

Components never touch kronometer's cells (T3.2): a step reaches a component as a message, and what it changes
returns through `KronBridge`.

**Input** in the pane is the GUI's (`Seams`); in a window of its own it is still the GUI's under plan (1). Controllers
come through tactroller.

## The order of work

Each step ends in a test or a picture. The first two need no device.

1. **game-core: a world that steps headless.** Entities as records, a step as messages, the world lane on a fixed
   kronometer rate, snapshots published. A test steps it and reads them.
2. **game-shadelets: the 3D roles on the CPU.** Source, modifier, material, light, composite as vastir functions; one
   composition; the generated dispatcher; agreement of a sphere and a box on the CPU.
3. **game-fractal: one fractal, both ways.** A Menger sponge and a Mandelbulb as sources, with step factors and an
   orbit trap; the CPU twin answering where the surface is, judged against known geometry.
4. **game-render and game-pane: the first picture.** The composition lowered to SPIR-V at build time, marched by a
   technique in a `TechniquePane` in a small framework application; the camera through `@Draw`.
5. **Meshes and fractals in one depth.** A mesh technique sharing material and light; a fractal in a clip shell beside
   a mesh, `ClipDepth` agreeing.
6. **game-window (plan 1): a game with an overlay.** A full-bleed pane, a HUD and a menu over it, driven by
   `ottermate`.
7. **Level of detail and motion.** Footprint-driven iterations, a shell's far texture baked by a job, a fractal whose
   parameters move on its own clock.
8. **The physics bridge**, once vexelray-sim-physics' boundary contract exists.

## Open questions

1. **Where the 3D shadelet host lives.** Here first, then the shared machinery lifted into `vexelray-canvas` or
   `vastir` once there are two hosts. Or generalise canvas `Composition` now, before there is a second host to
   generalise from.
2. **An engine-owned frame (window plan 2).** It needs a `Gui` tree driven by a host other than `GuiApp`; that is
   vexelray-gui's to plan, and only worth it if the sampling pass costs.
3. **Per-frame numbers.** `TechniqueView.PARAMS` is 32 floats. Grow it, or put the camera and lights in a bound buffer.
4. **A technique of many objects.** One march over a dispatcher of every object, culled by bounds, or one technique
   per kind of object; and where the bounding structure is built (a job, the world lane, the GPU).
5. **Who owns the field contract.** The world's fields and vexelray-sim-physics' static boundaries are the same idea.
   One contract, in vexelray-surface or a shared module, rather than one per repo.
6. **Precision at depth.** A floating origin per region is cheap; a fractal zoomed past single precision needs more
   (perturbation, doubles on the CPU feeding the GPU), and it is not settled how much the engine promises.
7. **Shell far textures.** Baked at load by a job, baked offline into an asset, or rendered on demand into an atlas.
