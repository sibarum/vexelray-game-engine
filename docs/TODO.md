# TODO

Known gaps. An entry is deleted when it is done, not ticked. The order of work is in
[architecture.md](architecture.md); this is what is noticed along the way.

- [ ] **`TechniqueView.PARAMS` is 32 floats** (vexelray-gui-core). A camera is 16. Decide whether it grows or the camera
      crosses as a bound buffer (architecture.md, open question 3) before game-pane's first picture.
- [ ] **No `Gui` tree as a technique.** A game drawing its own frame (window plan 2) needs vexelray-gui to drive layout,
      input and the frame from a host other than `GuiApp`. Not needed for plan 1; write it up in vexelray-gui if plan 2
      is taken.
- [ ] **The field contract with vexelray-sim-physics**: its TODO's static SDF geometry (a Menger-carved tunnel system) is
      this repo's fractal CPU twin. One contract, one owner (open question 5).
