# vexelray-game-engine

A game engine on the VexelRay stack, whose worlds are drawn as much with **fractals** as with meshes.

Part of the [VexelRay](https://sibarum.github.io/) stack. Building an application on it? Start at the
[developer guide](https://sibarum.github.io/vexelray-framework/developers/).

- **A game is a view, and a GUI is an overlay.** A world is drawn as a list of techniques, so it runs in a pane of
  any vexelray-gui application, and a game in a window of its own has a vexelray-gui interface over it.
- **One uber shader, extended with shadelets**, as the GUI's canvas is: pure functions in SupirVast IR, composed once
  per game and compiled at build time, that run on the CPU too.
- **Fractals are first-class shadelets**: marched in bounded shells, detailed by the pixel's footprint, coloured by
  their own orbit traps, laid onto other surfaces, and asked on the CPU where they are, so what the player collides
  with is exactly what is drawn.
- **The framework's rules for the machinery, not the content**: compile-time wiring, lanes that talk by messages, and
  kronometer's clocks for a world stepped at a fixed rate and shown at the display's. What the world holds (scenes,
  entities, assets) is loaded progressively at run time and is the world's own data, never a declared part.

| Module | What it is |
| --- | --- |
| `vexelray-game-core` | The world as data, a step as messages, the clocks. No device, no window |
| `vexelray-game-scene` | Scenes and progressive loading: regions, residency, budgets. No device |
| `vexelray-game-shadelets` | The 3D host for shadelets: the world's uber shader |
| `vexelray-game-fractal` | Fractals as shadelets, their level of detail, and their CPU twin |
| `vexelray-game-render` | The world drawn, as techniques; knows no host |
| `vexelray-game-pane` | A game in a vexelray-gui application |
| `vexelray-game-window` | A game in a window of its own, with a vexelray-gui overlay |

## Status

**A scaffold.** The modules exist and are empty. [docs/architecture.md](docs/architecture.md) is the plan: what goes
in each, the order of work, and the open questions. [docs/TODO.md](docs/TODO.md) is what is known to be missing.

## Building

The stack's other repositories must be installed to the local Maven repository first: `vexelray-build`, `supirvast`,
`vexelray`, `vexelray-fields`, `kronometer`, `atchung`, `vexelray-gui`, `vexelray-framework`
(`vexelray-framework/bootstrap.sh` installs them in order).

```bash
mvn install
```
