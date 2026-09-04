# Motion and shape language of the system

> 🇪🇸 [En castellano](motion-language.md)

The single source of truth for **everything that moves or has a shape** in this
desktop: Hyprland, Quickshell (bar, notch, panels, Settings), GTK and kitty.

Golden rule: **if something moves and it is not on this scale, it is a bug.**
No inventing new durations or curves. If one is needed, it gets added here
first and used afterwards.

Written 2026-08-05. The shape decisions were taken by Diego explicitly.

---

## The two laws of shape (decided by Diego)

### Law 1 — Geometry says whose layer it is

|                        | CONTENT layer            | SHELL layer                |
|------------------------|--------------------------|----------------------------|
| What it is             | kitty, chrome, nautilus… | notch, bar, panels         |
| Who draws it           | the compositor           | Quickshell                 |
| Corners                | **square, 0 px**         | **rounded** (10/14/18/30)  |
| Border                 | none                     | hairline when it is needed |

It is not an oversight: the contrast is what tells you. What is square is yours
and anchored to the grid; what is rounded floats above it and belongs to the
system. **Never round the windows** and never square off the shell “for
consistency” — consistency here comes from the motion, not from the shape.

### Law 2 — Focus is marked with light, not with an outline

`border_size = 0`. The focused window is **fully present** (opacity 1); the
rest **sink back** (a bit less opacity and `dim_inactive`). The hierarchy reads
as depth, not as a coloured frame.

Corollary: **the shadow says “this floats”**. Tiled windows are stuck to the
grid and carry no shadow; floating ones and dialogs do. The shadow is
information about the plane, not decoration.

---

## The duration scale

Four steps, plus one exception with a name of its own. No number outside this.
The names of the steps and of the curves are kept in Spanish, because they are
the names the code uses (`Appearance.qml`, `hyprland.lua`).

| Step          | In     | Out    | What for                                            |
|---------------|-------:|-------:|-----------------------------------------------------|
| **RESPUESTA** | 130 ms | 130 ms | hover, press, colour, a toggle: “I heard you”       |
| **CONTENIDO** | 210 ms | 110 ms | text, icons, rows inside something already there    |
| **PANEL**     | 320 ms | 170 ms | a surface that appears: panel, notification, menu |
| **FORMA**     | 440 ms | 220 ms | the shape itself changes, or a whole window moves |
|               |        |        | (moving a window is no longer a duration: it is the `mover` spring) |
| **RECORRIDO** | 500 ms |    —   | whatever **crosses the whole screen** (workspaces) |

In Hyprland the speed goes in deciseconds: **1.3 / 2.1 / 3.2 / 4.4**, and the
exits **1.1 / 1.7 / 2.2**. They are literally the same numbers as the tokens in
`Appearance.qml`. When a window opens in 440 ms with the same curve the notch
uses to change shape in 440 ms, the desktop reads as **one single thing**. That
is the whole point.

**RECORRIDO stands apart** because crossing the whole screen is the longest
journey there is here, and charging it at the price of opening a window leaves
it as nothing. It goes with the `apple` curve.

> **Two failed attempts before this one, and both teach something.**
> The first: 320 ms (PANEL), on the grounds that you switch workspace a hundred
> times a day and it cannot feel heavy. It came out **flat**.
> The second: 700 ms with a `frontal` curve (0.1,1)(0,1) copied from end-4, on
> the theory that a brutally front-loaded curve allows long durations.
> It came out **odd** — it read as a jerk followed by a slow drift. The lesson
> is that a curve that extreme does not give you “fast AND weighty”: it gives
> two gestures stuck together that do not look like the same movement. Normal
> curve and an honest duration.

### Law 3 — Events are asymmetric; states are symmetric

What **comes in** deserves attention: it takes its time, and it arrives with
some weight. What **goes out** is a dismissal: it leaves in half the time and
without ceremony. Ratio 2:1, always.

But hover is not an event, it is a **state** that follows the pointer: it comes
in and goes out the same (130/130). Animating the hover out faster than in
makes the pointer feel “sticky”.

### Law 4 — Frequency drops the weight by one step

What you do a hundred times a day cannot feel heavy. Opening a window is an
occasion → it stays at FORMA (440).

**A nuance learned later:** “it cannot feel heavy” does not mean “it has to be
short”. It means **it has to set off at once**. Those are different things, and
I confused them: workspace switching sat at PANEL (320) because of this law and
came out flat. That is why it is now at RECORRIDO (500) with `apple`.

**And the nuance of the nuance, the one that cost the most:** neither of the
two things — duration or curve — was the real problem with workspace switching.
It was the **distance**. See law 10.

---

## The family of curves

Five curves. The names are the same in Hyprland (`bezier =`) and in Quickshell
(in the comments next to each `easing`).

| Name        | Cubic                     | Character                                       |
|-------------|---------------------------|-------------------------------------------------|
| `respuesta` | `0.2, 0, 0, 1`            | sets off at once, brakes dead. Microinteraction only |
| `entra`     | `0.16, 1.3, 0.3, 1`       | shoots off, **overshoots** and settles. Scales only |
| `sale`      | `0.3, 0, 1, 1`            | accelerates and is gone. It never brakes: you have stopped caring |
| `forma`     | `0.23, 1, 0.32, 1`        | quint: 90 % of the travel in the first third and a long landing |
| `apple`     | `0.32, 0.72, 0, 1`        | the iOS/macOS one: sets off with a bit of acceleration (as if something with mass were getting going) and brakes for almost the whole journey |

`linear` is **banned** for anything spatial. It is only allowed in continuous
loops (a gradient spinning), which is precisely where there must be no accent.

---

## Springs — the thing that is not a curve

A bézier does not know where it is, only how much is left: it has a fixed
duration. Change its target mid-flight and it has to **cut and start from
scratch**. That is what made the shell feel rubbery when you did two things
quickly one after the other, and no curve fixes it, because the problem is not
the shape of the path but that there is no memory.

A spring has **state**: position and velocity. Change the target mid-flight and
it carries on from where it was, at the speed it had. There is no cut because
there is nothing to restart.

**In Qt it has been there forever and nobody in the ricing world uses it** — so
the shell, which is the thing you look at most, was the first to get springs.

**In Hyprland this is no longer pending (2026-08-05).** It called for migrating
the config to Lua, and that was done: `~/.config/hypr/hyprland.lua`. The `.conf`
stays alongside as a backup — delete the `.lua` and Hyprland goes back to
reading it by itself. The two do not coexist: if the `.lua` exists, the `.lua`
rules.

| Token       | spring / damping | What for |
|-------------|------------------|----------|
| `sprTight`  | 5.2 / 0.58 | gets there and stays. The leading edge of the indicator, the width of the notch |
| `sprPanel`  | 4.0 / 0.42 | a surface that appears. The faces of the notch, cascades |
| `sprLoose`  | 3.1 / 0.34 | travel to spare and a tail. The height of the notch, the trailing edge of the indicator |

### Hyprland springs use different units

Careful when converting: the tokens above are Qt's (`SpringAnimation`, where
`spring` and `damping` are rough-and-ready numbers). Hyprland asks for **real
physics** — `stiffness`, `dampening` and `mass` — so you do not copy the
numbers across, you convert them:

    w0   = sqrt(stiffness / mass)              -> natural frequency, rad/s
    zeta = dampening / (2*sqrt(stiffness*mass)) -> damping

`zeta` is the only number that really matters:

- `zeta = 1` — **critically damped**: it arrives as fast as it possibly can
  without overshooting. It is the place for everything that TRAVELS, by law 5.
- `zeta < 1` — it bounces. It goes in the scales, never in the travelling.
- `zeta > 1` — it drags. Never needed.

| Token       | stiffness / dampening | zeta | What for |
|-------------|----------------------:|-----:|----------|
| `mover`     | 130 / 23   | 1.01 | `windowsMove` — moving a window in the tiling layout |

And the settling time (2 % band) comes out of that: with `zeta ≈ 1`,
`ts ≈ 5.83/w0`. For `mover`: `w0 = 11.4`, which is about **510 ms** of full
tail, of which the last half is invisible because `epsilon` cuts it off. If you
want it drier, raise `stiffness` and raise `dampening` **with it**, keeping
`dampening ≈ 2*sqrt(stiffness)`, or you will drop out of critical damping
without noticing.

It is calibrated live, without leaving the session, with
`~/.config/hypr/scripts/probar-muelle.sh` (it hooks it in through
`hyprctl eval`; `hyprctl reload` undoes it).

And two rules that cost you dearly if you forget them:

- **Never a spring on opacity.** It would overshoot past 1 and flicker. Fades
  stay on béziers; springs are for scale and position.
- **`epsilon` depends on the unit.** In pixels, `0.25` (a quarter of a pixel is
  invisible and it cuts the dead tail off). On a 0..1 scale you need `0.001`,
  or the spring stops 1 % short of the target — which on something 300 px wide
  is visible.

## The liquid thing — the direction the system is going

Decided by Diego on 2026-08-05: *“the indicator could be more liquid in
general, I think the system could follow that in a lot of places”*.

Liquid **is not something you get by animating shapes**. You can animate a
scale, an opacity and a bounce as well as you like and they will still be two
solid objects moving near each other. What the eye recognises as liquid is
**surface tension**: that two bodies coming together merge through a *neck*
before they touch, and that on separating the thread stretches and snaps.

That forces a change of tool: you do not draw shapes and move them, you solve
**a single distance field** with every body inside it, joined with `smin()`
(smooth union). The first one is in
`~/.config/quickshell/shaders/liquid.frag`, and the workspace indicator uses it.

Rules learned while building it, good for the next one:

- **The bodies need air or the effect does not exist.** The pill was 22 px and
  the dots were spaced 8 apart: that left 1 px between them, which is to say
  they were already merged *at rest*. If they are stuck together to begin with
  there is nothing to merge as you pass. The pill came down to 18 and the
  spacing went up to 12 → ~7 px of air. **Before touching the shader, look at
  the geometry.**
- **`k` is how much it wets.** Below ~4 the bodies ignore each other until they
  collide (and then it is a collision, not a merge); far above it everything
  stays permanently stuck together and you lose the row. 4.5 with these
  distances.
- **Coverage antialiasing: `clamp(0.5 - d/fwidth(d), 0, 1)`.** It gives **one**
  pixel of transition. A `smoothstep(-fwidth, +fwidth, d)` spans **two**, and
  with 8 px bodies that is not a soft edge: it is a smudge. It is the standard
  SDF formula and there is no need to invent another one.
- **Colour is shared out by coverage, never by distance.** “How much of this
  pixel is pill” on the free edge is worth 0.5 of pill and 0 of dot → clean
  pill colour. “How close the pill is” on that same edge pulls towards the
  colour of the dot and **paints a light ring all around** — a halo stuck to
  the edge is exactly what makes something look dirty.
- **A body inside another one dissolves.** The active dot ends up under the
  pill with the same centre and the same radius: the two fields *tie*, and with
  a tie there is no colour formula that works. You do not compensate for it in
  the blend — you make the tie not exist, dissolving the dot according to how
  deep the pill has swallowed it. By depth and not by index, so that it does
  not jump when you switch workspace.
- **Margin all around.** The neck bulges outside the bounding box of the
  bodies; without air, what gets clipped is precisely the part that makes the
  effect.
- **The colour uniforms arrive ALREADY premultiplied.** Multiplying by the
  alpha again leaves the bodies translucent at a quarter of the intensity:
  dirty greys instead of whites.
- **Qt caches the compiled shader by URL.** Recompiling the `.qsb` under the
  same name and reloading the QML does **not** reload it: it carries on
  painting the old one without a word of warning. Versioned name
  (`liquid.v4.frag.qsb`) or restart `qs`.

### The ligament — how something actually stretches

The workspace indicator started out with two springs of different stiffness at
either end, on the idea that the softer one would lag behind and that gap would
be the stretch. **Measured: on a two-workspace jump (40 px of travel) the pill
stretched 4 px.** Nothing. What looked like stretch in the screenshots was
almost all of it the metaball merging with the dot next door.

Two different springs do not produce a ligament: they produce **two things that
arrive almost at the same time**. A liquid does not stretch because its tail is
slower — it stretches because the tail **stays stuck where it was** until the
tension beats it, and then it lets go all at once.

That is: **a pause and then a yank**, not a different stiffness.

- Leading edge: it shoots off the moment the target changes (`sprTight`).
- Trailing edge: a `PauseAnimation` of `mStagger` and then `sprSquash` with
  `dmpPanel`. During the pause the stretch is **the whole travel**.

With that the measured cycle becomes: rest 18×8 → flight **44×5.6** → landing
18×8 → wobble 21×7.5 → rest. It stretches to 2.4 times its length and thins by
30 %.

And the damping of the yank matters: with `dmpSquash` (0.20) the edge was left
oscillating and the pill throbbed after arriving. A wobble is fine in the
**thickness**, where it reads as material, and is a fault in the **position**,
where it reads as not knowing where to stop.

> **A QML rule that cost dearly: never a spring on a continuous signal.**
> I put a `Behavior { SpringAnimation }` on the thickness, which is *bound* to
> the stretch and therefore changes every frame. The Behavior restarts the
> spring 60 times a second, so it never gets to simulate anything — and when
> the binding stops changing, the property is left **frozen at the last
> value**. On screen: a pill that thinned as it flew and never got its
> thickness back. Springs are for targets that change on **events**, not for
> following a continuous signal. The thickness comes straight out of the
> stretch, and the wobble on arrival appears by itself, because the trailing
> edge overshoots as it brakes.

### The two-spring trick

A single target with **two springs of different character** is where almost
everything good comes from:

- **The morph of the notch.** Width on `sprTight`, height on `sprLoose`. The
  notch widens first and **drops afterwards**, overshooting a little as it
  lands. Before, both axes ran on the same duration and the same curve, and
  that is why it rescaled like a box instead of deforming. This is textbook
  squash & stretch, and on top of that the rounding of the bottom corners is
  worked out from the height, so the shape curves by itself along the way.

### Law 5 — Bounce goes in the scales, never in the travelling

Something that **slides** and overshoots shows you the edge of the screen: a
black gap beyond the limit that gives the trick away. That is why slides
(workspace switching, layers coming in from an edge) use `forma`, which lands
and does not bounce.

But a **scale** has nothing to show. A window that swells a hair past its size,
or a panel that grows from 90 %, reveal no edge: they simply feel alive. That
is where the character goes, and that is where `entra` is used.

- **Bounces**: what grows or shrinks — windows opening (`popin 78%`), the morph
  of the notch, the panels, chips, cards.
- **Does not bounce**: what travels — workspaces, layer slides, moving and
  resizing windows in the tiling layout (a bounce there would overlap the
  neighbours).

> **Corrected on 2026-08-05.** The first version of this law banned bounce on
> anything big, on the screen-edge argument. That only held for the travelling;
> applying it to the scales as well left the system consistent and **flat**.
> Diego put it in a nutshell: “too subtle”. Consistency is not the same as
> timidity: if something has to be turned up, you turn up the **vocabulary**
> (the amplitude tokens), not the individual places.

### Law 6 — Motion has an origin

Nothing appears “out of thin air”: it grows from where it is or comes in from
the edge it belongs to. The panels of the notch **come down out of the notch**.
Notifications come in from where they are going to stay. A tiled window
**swells into its slot** (`popin`), it does not fly in from an edge: its place
had already been decided.

**Applied to the faces of the notch** (`NotchLayer.qml`, `origin`). All eleven
faces used to come in the same way — the same fade-and-scale for the clock, the
launcher and the power menu. Eleven different things with a single gesture is
having no gesture. Now each face is born **from the side of the button that
calls it**: the launcher from the left, which is where the Arch icon is;
control, network, bluetooth and power from the right, which is where theirs
are. The ambient ones (clock, music, notifications) do not travel and grow in
the centre, because you did not ask for them from anywhere.

It works because the notch **clips** (`clip`): the content travels inside the
slot, so you do not see it appear, you see it arrive. And on the way out it
goes back towards its own side, so the gesture reads the same going as coming.

### Law 7 — Nothing moves just because

No infinite pulses, no shimmers running across anything, no animating a figure
that refreshes on its own. If a CPU bar updates every second, its transition
lasts **less** than the refresh interval or it looks permanently in motion
without you having done anything. An animation that does not answer an act of
yours is noise.

### Law 8 — One single gap in the content layer

`gaps_in 3` / `gaps_out 6` → between two windows there are 6 px, and to the
edge of the screen, 6 px. **The same gap**. It used to be 6 and 5: a one-pixel
mistake, invisible to look at and perfectly audible to feel.

The shell layer has its own rhythm (6/10/14/16) and does not have to match:
they are different layers (law 1).

### Law 9 — One animation, one owner

If the compositor animates a surface appearing **and** the app animates its own
content at the same time, you see two animations chained together and the
result is a smudge. The split:

- The **compositor** animates surfaces that appear and disappear (windows and
  ephemeral layers: the panels of the notch, the launcher, the notifications).
- The **app** animates what is inside (and the persistent layers that only
  change size, like the notch: the compositor must not get involved there).

**How it is split in practice** (applied on 2026-08-05). Every Quickshell
surface declares `WlrLayershell.namespace`, and that is what makes it possible
to give each one its own rule instead of a generic fade for all of them. The
criterion is not aesthetic, it is one of ownership: look at what the app
already animates and give the compositor **only the channel that is left
free**.

| Surface                 | Rule                     | Why |
|-------------------------|--------------------------|---------|
| `quickshell:bar`        | `no_anim`                | permanent layer that resizes itself; Quickshell animates the whole of it |
| `quickshell:media`      | `animation slide bottom` | it lives on the bottom edge, so it comes up from there. `slide` moves position and **does not fade**, and the card already takes care of scale and opacity: different channels |
| `quickshell:wallpaper`  | `no_anim`                | full screen (sliding it would show the edge) and its `stage` already fades on its own |
| `quickshell:overview`   | `animation fade`         | the thumbnails come in staggered on their own, but **nobody animates the black veil from the inside**: without this it would appear all at once |

The trap to avoid is the double fade: if the compositor fades the surface and
the app fades its content, the two opacities multiply and what you see is not
twice as smooth, it is that it **arrives late**.

### Law 10 — What feels flat is almost never the duration: it is the distance

The law that came dearest, because it took me three attempts to see it.

Workspace switching felt subtle and I tried the obvious: shorten the duration
(flat), lengthen it with an extreme curve (odd). Neither of the two was the
problem. The problem is that it was on `slidefade 20%`: the workspaces
**travelled a fifth of the screen** and a fade did the rest of the work. It was
not that the movement was weak — it was that there was hardly any movement.

And on top of that I had made it worse in the name of realism: I put in
“parallax” (the one coming in at 20 %, the one going out at 50 %) reasoning
that two planes at different distances give depth. On paper it is true. On
screen that percentage is **a cut taken out of the journey**.

macOS does none of that, and that is why it feels solid: the two workspaces are
stuck together like two adjoining rooms and the screen travels **100 %** from
side to side, all at once, **with no fade**. The fade is precisely what gives
away that they are two stacked images; without it, they are a place you leave
and another one you arrive at.

Practical corollary: before touching milliseconds or curves, check **how much
the thing actually moves**. A `slidefade` percentage, an `mScaleFrom` of 0.96
or a 4 px shift are the usual reason something “does not show”, and no curve in
the world fixes it.

---

## Where each thing lives

| File                                 | What it defines                               |
|--------------------------------------|-----------------------------------------------|
| `~/.config/motion-language.md`       | the spec, the source of truth (this, in Spanish) |
| `~/.config/hypr/hyprland.lua`        | the curves, the spring and the animations (ACTIVE) |
| `~/.config/hypr/hyprland.conf`       | the legacy version, kept as a backup. Not read while the `.lua` exists |
| `~/.config/quickshell/Appearance.qml`| the same numbers as QML tokens                |

If you change a number, change it **here first** and propagate it. Two
different sessions each touching durations of their own is exactly how the mess
this document exists to fix came about.
