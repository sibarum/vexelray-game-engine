# Working in this repo

`vexelray-game-engine` is a game engine on the VexelRay stack, with fractals as its headline. Since 2026-11-15 it is a
scaffold: [docs/architecture.md](docs/architecture.md) is the plan, and the modules are empty until its order of work
fills them.

**The layering is the design; keep it.**

- `vexelray-game-core`, `vexelray-game-scene` and `vexelray-game-shadelets` never depend on a device, a window or a
  GUI. A world steps, a scene streams and a shadelet runs in a plain JUnit test.
- **The framework declares the machinery; the world holds the content.** Lanes, topics, services and the overlay's
  screens are framework parts, declared at compile time. Scenes, entities, assets and their events are the world's
  data, loaded at run time. Never make an entity, a region or a loaded thing a `@Component` or give it a topic.
- `vexelray-game-render` never depends on a host (`-pane`, `-window`, vexelray-gui). Anything both hosts need goes in
  `-render` or below, never in both hosts.
- A shadelet is pure vastir IR, so it runs on the CPU as well. A fractal's CPU twin and its GPU form are the same
  function; agreement between them is a test, not a hope.

Anything you notice and do not fix goes in [docs/TODO.md](docs/TODO.md). An entry is deleted when it is done, not ticked.

The rest of the stack is in sibling checkouts under `C:\Users\User\Documents\GitHub\`, all the author's own and all
modifiable:

| Repo | What it gives this one |
| --- | --- |
| **vexelray** | `RenderTechnique` (`docs/techniques.md`), the canvas shadelets this repo's 3D host mirrors (`vexelray-canvas`, `vexelray-shadelet-codegen`), `vexelray-surface`, `technique-sdf`, `-mesh`, `-canvas`, `-panel`, `ClipDepth`; the Menger clip-shell prototype (`docs/clip-shells.md`) |
| **supirvast** | The IR every shadelet is written in, the CPU backend (vast), `vastir-pbr`; `docs/shadelets.md` |
| **vexelray-fields** | `vexelray-fields-fractal`: fractal coverage, `Lod`, `Footprint` |
| **vexelray-gui** | `TechniqueView` and `TechniquePane`, the device feed (`docs/plans/device-feed.md`), the overlay's widgets |
| **vexelray-framework** | The annotation processor and its rules: `docs/threading.md` (T1 to T7), `docs/components.md` |
| **kronometer** | The clocks: fixed and dynamic rates, `Dilated`, tempos |
| **vexelray-sim-physics** | The simulations a world will couple to; its TODO's static SDF geometry is this repo's fractal twin |

## Building

- `mvn -o install` at the root builds everything. A change in a sibling is seen here only once it is `mvn install`ed
  there.
- Other sessions may be reinstalling sibling jars mid-build; a sudden `NoClassDefFoundError` from a stack class that
  did not change is that, not a regression. Run again once it is quiet.
- There is no Python on this machine; script with bash, PowerShell or a single-file `java Foo.java`.
