# Referencias de movimiento — lo que hacen los repos buenos

> 🇬🇧 [In English](motion-references.en.md)

Investigado el 2026-08-05 sobre los repos que te gustan. Guardado aquí porque
sacarlo costó una tanda de búsqueda entera y no quiero que se pierda entre
sesiones. Complementa a `~/.config/motion-language.md`, que es la spec.

---

## 1. Aviso urgente: tu `hyprland.conf` tiene fecha de caducidad

Hyprland **0.56.1** (el que tienes) ya muestra este aviso:

> *You are using the .conf config format, support for which will be removed in
> Hyprland 0.57.*

**El formato `.conf` desaparece en 0.57**, o sea la versión siguiente a la
tuya. El sustituto es `~/.config/hypr/hyprland.lua`.

Y es **todo o nada**: si existe `hyprland.lua`, Hyprland ignora
`hyprland.conf` por completo. La comprobación se hace una sola vez al arrancar.
No se puede tener el `.conf` de siempre y meter un `.lua` pequeñito al lado
solo para una cosa.

Esto es un trabajo de sesión propia, y conviene hacerlo **antes** de que la
actualización te obligue.

## 2. Los muelles (springs) — el motivo real para migrar

Hyprland 0.55 añadió **curvas de muelle**, y solo se pueden definir desde Lua:

```lua
hl.curve("easy", { type = "spring", mass = 1, stiffness = 238.1191, dampening = 24.21279333 })
hl.animation({ leaf = "windows",   enabled = true, speed = 4.79, spring = "easy" })
hl.animation({ leaf = "windowsIn", enabled = true, speed = 4.1,  spring = "easy", style = "popin 87%" })
```

Un muelle no es una curva más bonita: es **otra cosa**. Una bézier tiene una
duración fija, así que si la interrumpes a media animación tiene que cortar y
empezar de nuevo. Un muelle tiene estado (posición y velocidad), así que al
cambiar de destino a medio vuelo **continúa desde donde iba y a la velocidad
que llevaba**. Eso es exactamente lo que hace que algo se sienta físico en vez
de programado — y es lo que más se nota cuando cambias de escritorio dos veces
seguidas rápido.

Otro ejemplo real (de una discusión del repo), más pesado y flotante:

```lua
hl.curve("workspaceSpring", { type = "spring", mass = 2.4, stiffness = 38, dampening = 8 })
hl.curve("windowSpring",    { type = "spring", mass = 2.5, stiffness = 40, dampening = 10 })
```

**Pega conocida**: los muelles calculan su progreso integrando el tiempo entre
tics del compositor, mientras que las béziers usan el tiempo total de reloj. Si
la CPU va apurada y se pierden tics, un muelle se ve **a cámara lenta** en vez
de simplemente saltarse fotogramas. En una Iris Xe eso puede pasar. Hay que
probarlo, no darlo por bueno.

Dato que sorprende: **ningún** repo famoso usa muelles todavía. Ni end-4 ni
caelestia, y eso que ya están los dos en Lua. Siguen todos en bézier.

## 3. El truco de end-4, que es el mejor de todos

Su config no es más animada que la tuya. Es más **asimétrica**, y ahí está todo:

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

Tres cosas que copiar:

1. **Abrir del 80 %, cerrar solo al 90 %.** La apertura recorre el doble de
   escala que el cierre. Abrir es un acontecimiento; cerrar es un descarte.
2. **`menu_decel` = (0.1, 1)(0, 1)**, con las dos Y clavadas en 1. Es una curva
   brutalmente frontal: el recorrido se come casi entero al principio y luego
   solo hay un aterrizaje largo. Por eso puede permitirse **speed 7 (700 ms)**
   en el cambio de escritorio y aun así sentirse instantáneo. Esta es
   probablemente la mejor idea de todo el estudio, y es lo contrario de lo que
   hice yo: yo acorté la duración; ellos alargan la duración y adelantan la
   curva. Se siente rápido Y tiene peso, en vez de una cosa u otra.
3. **Nunca usan overshoot fuerte.** Tienen definida `expressiveFastSpatial`
   (0.42, **1.67**)(0.21, 0.90) y **no la enchufan a nada**. Su expresividad
   sale de la asimetría, no del rebote.

## 4. Movimiento a medida por cada superficie del shell

Esto es lo que más separa a end-4 de un rice normal: **cada ventana del shell
entra por donde le corresponde**, en vez de un estilo global para todas.

```lua
hl.layer_rule({ match = { namespace = "quickshell:sidebarRight" },  animation = "slide right"})
hl.layer_rule({ match = { namespace = "quickshell:cheatsheet" },    animation = "slide bottom"})
hl.layer_rule({ match = { namespace = "quickshell:wallpaperSelector" }, animation = "slide top"})
hl.layer_rule({ match = { namespace = "quickshell:screenCorners" }, animation = "popin 120%"})
hl.layer_rule({ match = { namespace = "quickshell:notificationPopup" }, animation = "fade"})
hl.layer_rule({ match = { namespace = "quickshell:overview" },      no_anim = true})
hl.layer_rule({ match = { namespace = "gtk4-layer-shell" },         no_anim = true}) -- "los lanzadores tienen que ser RÁPIDOS"
```

Fíjate en el criterio: al **overview** y a los **lanzadores** les quitan la
animación del compositor a propósito, porque tienen que sentirse instantáneos
o los animan ellos por dentro.

**Esto es aplicable a tu sistema hoy mismo**, en `.conf`, sin migrar a Lua:

```ini
layerrule = animation slide top, match:namespace ^(quickshell:loquesea)$
```

Lo único que falta es que tus superficies **declaren un namespace**. Ahora
mismo solo `TopShell.qml` lo hace (`quickshell:bar`); Overview, Settings,
MediaControls y WallpaperPicker no declaran ninguno, así que no hay nada a lo
que agarrarse. Ese es el primer paso, y es una línea por fichero:

```qml
WlrLayershell.namespace: "quickshell:overview"
```

## 5. Cosas de Hyprland que nadie usa y podrían valer

Todo confirmado en la wiki de la versión actual:

```lua
decoration = {
  motion_blur = { enabled = true, samples = 7 },
  wobble      = { enabled = true, mesh = 12, stiffness = 200, damping = 12, mass = 1, intensity = 0.2 },
}
```

`wobble` es una simulación de muelle aplicada a la malla de la ventana al
moverla o redimensionarla — gelatina, literalmente. `motion_blur` es desenfoque
de movimiento real. Ninguno de los repos estudiados los usa. En una Iris Xe hay
que medirlos antes de creérselos, pero son justo el tipo de cosa que se sale de
lo básico.

También real y sin usar por nadie: `animations { workspace_wraparound = true }`,
que hace que el último y el primer escritorio se animen como si fueran vecinos.

## 6. Trampas de rendimiento, para una Iris Xe sin VRR

- **`borderangle` / `shadowangle` / `glowangle` con estilo `loop`** fuerzan
  repintado continuo a 60 Hz aunque no se vean y aunque las animaciones estén
  apagadas. Es el coste fijo más tonto que se puede pagar. Los bordes arcoíris
  de medio r/unixporn son esto. Si alguna vez los quieres, `once`, no `loop`.
- **Si usas TLP**, mira `/etc/tlp.conf`: subir `INTEL_GPU_MIN_FREQ_ON_AC` y
  `INTEL_GPU_MIN_FREQ_ON_BAT` de ~300 a ~500 quita tirones en iGPU Intel. La
  wiki lo describe como que en el mejor caso los elimina del todo. **Esto es lo
  primero que hay que mirar antes de tocar nada de config.**
- El coste real del blur no es la curva, es que **cada fotograma de una
  transición vuelve a calcular el desenfoque** de la capa que se mueve. end-4
  va con `passes = 3, size = 10`; tú estás en `6 / 2`, que para tu GPU está
  bien. Si notas tirones, eso es lo primero que hay que bajar.
- Sin VRR no hay mitigación por parte del compositor para lo que se anima en
  reposo. Lo que se mueve solo, se paga entero.

## 7. Curvas para robar, con su carácter

| Nombre | Puntos | Rebote | Para qué |
|---|---|---|---|
| `emphasizedDecel` | (0.05, 0.7)(0.1, 1) | no | MD3. Todo lo que entra, en end-4 y caelestia |
| `emphasizedAccel` | (0.3, 0)(0.8, 0.15) | no | todo lo que sale |
| `menu_decel` | (0.1, 1)(0, 1) | no | brutalmente frontal; deja usar duraciones largas |
| `wind` | (0.05, 0.9)(0.1, 1.05) | +5 % | la más copiada del mundo rice |
| `winOut` | (0.3, **-0.3**)(0, 1) | anticipación | se echa atrás antes de salir. Muy bonito, poco usado |
| `smoothIn` | (0.5, **-0.5**)(0.68, **1.5**) | ambos | anticipa y se pasa. Lo más expresivo sin ser ridículo |
| `expressiveFastSpatial` | (0.42, **1.67**)(0.21, 0.90) | +67 % | de la spec Material 3 Expressive. end-4 la define y no la usa |
| `OutBack` | (0.34, 1.56)(0.64, 1) | +56 % | el rebote clásico |
| `crazyshot` | (0.1, 1.5)(0.76, 0.92) | +50 % | todos la traen, nadie la enchufa |

Y una advertencia: la curva `nice` = (0, **6.9**)(0.5, **-4.20**) que circula en
un preset viejo de JaKooLit es una **broma** (6.9 y 4.20, ya ves). Se ve rota,
no expresiva. No la copies aunque la encuentres.
