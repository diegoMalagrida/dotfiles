# Motion references — what the good repos do

> 🇪🇸 [En castellano](motion-referencias.md)

Researched on 2026-08-05 across the repos you like. Kept here because digging
it out cost a whole round of searching and I do not want it lost between
sessions. It goes with `~/.config/motion-language.en.md`, which is the spec.

---

## 1. Urgent notice: your `hyprland.conf` has an expiry date

Hyprland **0.56.1** (the one you have) already shows this warning:

> *You are using the .conf config format, support for which will be removed in
> Hyprland 0.57.*

**The `.conf` format goes away in 0.57**, which is the version right after
yours. The replacement is `~/.config/hypr/hyprland.lua`.

And it is **all or nothing**: if `hyprland.lua` exists, Hyprland ignores
`hyprland.conf` entirely. The check is done once, at startup. You cannot keep
the `.conf` you have always had and drop a small `.lua` next to it just for one
thing.

This is a job for a session of its own, and it is worth doing **before** the
update forces you into it.

## 2. Springs — the real reason to migrate

Hyprland 0.55 added **spring curves**, and they can only be defined from Lua:

```lua
hl.curve("easy", { type = "spring", mass = 1, stiffness = 238.1191, dampening = 24.21279333 })
hl.animation({ leaf = "windows",   enabled = true, speed = 4.79, spring = "easy" })
hl.animation({ leaf = "windowsIn", enabled = true, speed = 4.1,  spring = "easy", style = "popin 87%" })
```

A spring is not a prettier curve: it is **a different thing**. A bézier has a
fixed duration, so if you interrupt it halfway through the animation it has to
cut and start again. A spring has state (position and velocity), so when the
target changes mid-flight it **carries on from where it was and at the speed it
had**. That is exactly what makes something feel physical instead of
programmed — and it shows most when you switch workspace twice quickly in a
row.

Another real example (from a discussion in the repo), heavier and more floaty:

```lua
hl.curve("workspaceSpring", { type = "spring", mass = 2.4, stiffness = 38, dampening = 8 })
hl.curve("windowSpring",    { type = "spring", mass = 2.5, stiffness = 40, dampening = 10 })
```

**Known catch**: springs work out their progress by integrating the time
between compositor ticks, whereas béziers use total clock time. If the CPU is
stretched and ticks get dropped, a spring looks like **slow motion** instead of
simply skipping frames. On an Iris Xe that can happen. It has to be tested, not
taken on trust.

A surprising fact: **no** famous repo uses springs yet. Not end-4, not
caelestia, and that is with both of them already on Lua. They are all still on
béziers.

## 3. end-4's trick, which is the best one of the lot

Their config is not more animated than yours. It is more **asymmetric**, and
that is where it all is:

```lua
hl.curve("emphasizedDecel", { type = "bezier", points = {{0.05, 0.7}, {0.1, 1}} })
hl.curve("emphasizedAccel", { type = "bezier", points = {{0.3, 0}, {0.8, 0.15}} })
hl.curve("menu_decel",      { type = "bezier", points = {{0.1, 1}, {0, 1}} })

hl.animation({ leaf = "windowsIn",  speed = 3,   bezier = "emphasizedDecel", style = "popin 80%" })
hl.animation({ leaf = "windowsOut", speed = 2,   bezier = "emphasizedDecel", style = "popin 90%" })
hl.animation({ leaf = "layersIn",   speed = 2.7, bezier = "emphasizedDecel", style = "popin 93%" })
hl.animation({ leaf = "layersOut",  speed = 2.4, bezier = "menu_accel",      style = "popin 94%" })
hl.animation({ leaf = "workspaces", speed = 7,   bezier = "menu_decel",      style = "slide" })
```

Three things to copy:

1. **Open from 80 %, close only to 90 %.** The opening covers twice as much
   scale as the closing. Opening is an occasion; closing is a dismissal.
2. **`menu_decel` = (0.1, 1)(0, 1)**, with both Y values nailed to 1. It is a
   brutally front-loaded curve: it eats almost the whole travel at the start
   and after that there is only a long landing. That is why it can afford
   **speed 7 (700 ms)** on workspace switching and still feel instant. This is
   probably the best idea in the whole study, and it is the opposite of what I
   did: I shortened the duration; they lengthen the duration and bring the
   curve forward. It feels fast AND it has weight, instead of one or the other.
3. **They never use strong overshoot.** They have `expressiveFastSpatial`
   (0.42, **1.67**)(0.21, 0.90) defined and they **do not plug it into
   anything**. Their expressiveness comes from the asymmetry, not from bounce.

## 4. Motion tailored to each surface of the shell

This is what separates end-4 from a normal rice the most: **each window of the
shell comes in from where it should**, instead of one global style for all of
them.

```lua
hl.layer_rule({ match = { namespace = "quickshell:sidebarRight" },  animation = "slide right"})
hl.layer_rule({ match = { namespace = "quickshell:cheatsheet" },    animation = "slide bottom"})
hl.layer_rule({ match = { namespace = "quickshell:wallpaperSelector" }, animation = "slide top"})
hl.layer_rule({ match = { namespace = "quickshell:screenCorners" }, animation = "popin 120%"})
hl.layer_rule({ match = { namespace = "quickshell:notificationPopup" }, animation = "fade"})
hl.layer_rule({ match = { namespace = "quickshell:overview" },      no_anim = true})
hl.layer_rule({ match = { namespace = "gtk4-layer-shell" },         no_anim = true}) -- "los lanzadores tienen que ser RÁPIDOS"
```

Look at the criterion: the **overview** and the **launchers** have the
compositor animation taken off them on purpose, because they have to feel
instant or because they animate themselves from the inside.

**This can be applied to your system today**, in `.conf`, without migrating to
Lua:

```ini
layerrule = animation slide top, match:namespace ^(quickshell:loquesea)$
```

The only thing missing is for your surfaces to **declare a namespace**. Right
now only `TopShell.qml` does (`quickshell:bar`); Overview, Settings,
MediaControls and WallpaperPicker declare none, so there is nothing to grab
hold of. That is the first step, and it is one line per file:

```qml
WlrLayershell.namespace: "quickshell:overview"
```

## 5. Hyprland things nobody uses that could be worth it

All confirmed in the wiki for the current version:

```lua
decoration = {
  motion_blur = { enabled = true, samples = 7 },
  wobble      = { enabled = true, mesh = 12, stiffness = 200, damping = 12, mass = 1, intensity = 0.2 },
}
```

`wobble` is a spring simulation applied to the mesh of the window when you move
or resize it — jelly, literally. `motion_blur` is real motion blur. None of the
repos studied use them. On an Iris Xe you have to measure them before believing
them, but they are exactly the kind of thing that goes beyond the basics.

Also real and used by nobody: `animations { workspace_wraparound = true }`,
which makes the last and the first workspace animate as if they were
neighbours.

## 6. Performance traps, for an Iris Xe with no VRR

- **`borderangle` / `shadowangle` / `glowangle` with the `loop` style** force
  continuous repainting at 60 Hz even when they cannot be seen and even with
  animations turned off. It is the silliest fixed cost you can pay. The rainbow
  borders on half of r/unixporn are this. If you ever want them, `once`, not
  `loop`.
- **If you use TLP**, look at `/etc/tlp.conf`: raising
  `INTEL_GPU_MIN_FREQ_ON_AC` and `INTEL_GPU_MIN_FREQ_ON_BAT` from ~300 to ~500
  takes the stutter out of Intel iGPUs. The wiki describes it as removing it
  altogether in the best case. **This is the first thing to look at before
  touching any config.**
- The real cost of blur is not the curve, it is that **every frame of a
  transition recalculates the blur** of the layer that is moving. end-4 runs
  `passes = 3, size = 10`; you are on `6 / 2`, which is fine for your GPU. If
  you notice stutter, that is the first thing to turn down.
- With no VRR there is no mitigation from the compositor for whatever animates
  while idle. Whatever moves on its own, you pay for in full.

## 7. Curves worth stealing, with their character

| Name | Points | Bounce | What for |
|---|---|---|---|
| `emphasizedDecel` | (0.05, 0.7)(0.1, 1) | no | MD3. Everything that comes in, in end-4 and caelestia |
| `emphasizedAccel` | (0.3, 0)(0.8, 0.15) | no | everything that goes out |
| `menu_decel` | (0.1, 1)(0, 1) | no | brutally front-loaded; lets you get away with long durations |
| `wind` | (0.05, 0.9)(0.1, 1.05) | +5 % | the most copied one in the ricing world |
| `winOut` | (0.3, **-0.3**)(0, 1) | anticipation | it pulls back before leaving. Very pretty, rarely used |
| `smoothIn` | (0.5, **-0.5**)(0.68, **1.5**) | both | anticipates and overshoots. The most expressive without being ridiculous |
| `expressiveFastSpatial` | (0.42, **1.67**)(0.21, 0.90) | +67 % | from the Material 3 Expressive spec. end-4 defines it and does not use it |
| `OutBack` | (0.34, 1.56)(0.64, 1) | +56 % | the classic bounce |
| `crazyshot` | (0.1, 1.5)(0.76, 0.92) | +50 % | everybody ships it, nobody plugs it in |

And a warning: the `nice` curve = (0, **6.9**)(0.5, **-4.20**) doing the rounds
in an old JaKooLit preset is a **joke** (6.9 and 4.20, quite). It looks broken,
not expressive. Do not copy it even if you come across it.
