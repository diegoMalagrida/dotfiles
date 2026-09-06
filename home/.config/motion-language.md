# Lenguaje de movimiento y forma del sistema

> 🇬🇧 [In English](motion-language.en.md)

Fuente de verdad única para **todo lo que se mueve o tiene forma** en este
escritorio: Hyprland, Quickshell (barra, notch, paneles, Ajustes), GTK y kitty.

Regla de oro: **si algo se mueve y no está en esta escala, es un bug.**
No se inventan duraciones ni curvas nuevas. Si hace falta una, se añade aquí
primero y luego se usa.

Escrito 2026-08-05. Decisiones de forma tomadas por Diego explícitamente.

---

## Las dos leyes de forma (decididas por Diego)

### Ley 1 — La geometría dice de quién es la capa

|                        | Capa de CONTENIDO        | Capa de SHELL              |
|------------------------|--------------------------|----------------------------|
| Qué es                 | kitty, chrome, nautilus… | notch, barra, paneles      |
| Quién la dibuja        | el compositor            | Quickshell                 |
| Esquinas               | **rectas, 0 px**         | **redondas** (10/14/18/30) |
| Borde                  | ninguno                  | hairline cuando hace falta |

No es un descuido: es el contraste el que informa. Lo recto es tuyo y está
anclado a la rejilla; lo redondo flota por encima y es del sistema. **Nunca
redondear ventanas** ni cuadrar el shell "por coherencia" — la coherencia aquí
la da el movimiento, no la forma.

### Ley 2 — El foco se marca con luz, no con contorno

`border_size = 0`. La ventana enfocada está **plenamente presente** (opacidad 1);
las demás **se hunden** (algo de opacidad menos y `dim_inactive`). La jerarquía
se lee como profundidad, no como un marco de color.

Corolario: **la sombra dice "esto flota"**. Las ventanas en mosaico están
pegadas a la rejilla y no llevan sombra; las flotantes y los diálogos sí. La
sombra es información sobre el plano, no adorno.

---

## La escala de duraciones

Cuatro escalones, más una excepción con nombre. Ningún número fuera de aquí.

| Escalón       | Entra  | Sale   | Para qué                                            |
|---------------|-------:|-------:|-----------------------------------------------------|
| **RESPUESTA** | 130 ms | 130 ms | hover, pulsación, color, un toggle: "te he oído"    |
| **CONTENIDO** | 210 ms | 110 ms | texto, iconos, filas dentro de algo que ya está ahí |
| **PANEL**     | 320 ms | 170 ms | una superficie que aparece: panel, notificación, menú |
| **FORMA**     | 440 ms | 220 ms | la forma misma cambia, o se mueve una ventana entera |
|               |        |        | (mover una ventana ya no es una duración: es el muelle `mover`) |
| **RECORRIDO** | 500 ms |    —   | lo que **atraviesa la pantalla entera** (escritorios) |

En Hyprland la velocidad va en decisegundos: **1.3 / 2.1 / 3.2 / 4.4** y las
salidas **1.1 / 1.7 / 2.2**. Son literalmente los mismos números que los tokens
de `Appearance.qml`. Cuando una ventana se abre en 440 ms con la misma curva con
la que el notch cambia de forma en 440 ms, el escritorio se lee como **una sola
cosa**. Eso es todo el objetivo.

**RECORRIDO va aparte** porque atravesar la pantalla entera es el viaje más
largo que existe aquí, y cobrarlo al precio de abrir una ventana lo deja en
nada. Va con la curva `apple`.

> **Dos intentos fallidos antes de este, y los dos enseñan algo.**
> El primero: 320 ms (PANEL), razonando que cambiar de escritorio se hace cien
> veces al día y no puede pesar. Salió **soso**.
> El segundo: 700 ms con una curva `frontal` (0.1,1)(0,1) copiada de end-4, con
> la teoría de que una curva brutalmente adelantada permite duraciones largas.
> Salió **raro** — se leía como un tirón seguido de una deriva lenta. La lección
> es que una curva tan extrema no da "rápido Y con peso": da dos gestos pegados
> que no parecen el mismo movimiento. Curva normal y duración honesta.

### Ley 3 — Los eventos son asimétricos; los estados, simétricos

Lo que **entra** merece atención: tarda, y llega con algo de peso.
Lo que **sale** es un descarte: se va en la mitad de tiempo y sin ceremonia.
Ratio 2:1, siempre.

Pero el hover no es un evento, es un **estado** que sigue al puntero: entra y
sale igual (130/130). Animar la salida del hover más rápido que la entrada hace
que el puntero se sienta "pegajoso".

### Ley 4 — La frecuencia baja el peso un escalón

Lo que haces cien veces al día no puede pesar. Abrir una ventana es un
acontecimiento → se queda en FORMA (440).

**Matiz aprendido después:** "no puede pesar" no quiere decir "tiene que durar
poco". Quiere decir que **tiene que arrancar ya**. Son cosas distintas, y las
confundí: el cambio de escritorio estuvo en PANEL (320) por esta ley y salió
soso. Por eso ahora está en RECORRIDO (500) con `apple`.

**Y el matiz del matiz, que es el que más costó:** ninguna de las dos cosas
—duración ni curva— era el problema de verdad en el cambio de escritorio. Era
la **distancia**. Ver la ley 10.

---

## La familia de curvas

Cinco curvas. Los nombres son los mismos en Hyprland (`bezier =`) y en
Quickshell (comentados junto a cada `easing`).

| Nombre      | Cúbica                    | Carácter                                        |
|-------------|---------------------------|-------------------------------------------------|
| `respuesta` | `0.2, 0, 0, 1`            | arranca ya, frena en seco. Solo microinteracción |
| `entra`     | `0.16, 1.3, 0.3, 1`       | sale disparada, **se pasa** y se asienta. Solo en escalas |
| `sale`      | `0.3, 0, 1, 1`            | acelera y se va. Nunca frena: ya no te interesa  |
| `forma`     | `0.23, 1, 0.32, 1`        | quint: 90 % del recorrido en el primer tercio y aterrizaje largo |
| `apple`     | `0.32, 0.72, 0, 1`        | la de iOS/macOS: arranca con algo de aceleración (parece que algo con masa se pone en marcha) y frena durante casi todo el viaje |

`linear` queda **prohibida** para cualquier cosa espacial. Solo se admite en
bucles continuos (un gradiente girando), donde justamente no debe haber acento.

---

## Los muelles — lo que no es una curva

Una bézier no sabe dónde está, solo cuánto le queda: tiene duración fija. Si le
cambias el destino a medio vuelo tiene que **cortar y empezar de cero**. Eso es
lo que hacía que el shell se sintiera de goma al hacer dos cosas seguidas
rápido, y no se arregla con ninguna curva, porque el problema no es la forma del
recorrido sino que no hay memoria.

Un muelle tiene **estado**: posición y velocidad. Cambias el destino a medio
vuelo y continúa desde donde iba, a la velocidad que llevaba. No hay corte
porque no hay nada que reiniciar.

**En Qt existe desde siempre y no lo usa nadie en el mundo rice** — así que el
shell, que es lo que más se mira, fue lo primero que llevó muelles.

**En Hyprland ya no está pendiente (2026-08-05).** Exigía migrar la config a
Lua, y se hizo: `~/.config/hypr/hyprland.lua`. La `.conf` se queda al lado como
respaldo — si borras el `.lua`, Hyprland vuelve a leerla sola. Las dos no
conviven: si existe el `.lua`, manda el `.lua`.

| Token       | spring / damping | Para qué |
|-------------|------------------|----------|
| `sprTight`  | 5.2 / 0.58 | llega y se queda. El borde de delante del indicador, el ancho del notch |
| `sprPanel`  | 4.0 / 0.42 | una superficie que aparece. Caras del notch, cascadas |
| `sprLoose`  | 3.1 / 0.34 | recorrido de sobra y cola. El alto del notch, el borde de detrás del indicador |

### Los muelles de Hyprland van en otras unidades

Ojo al traducir: los tokens de arriba son de Qt (`SpringAnimation`, donde
`spring` y `damping` son números de andar por casa). Hyprland pide **física de
verdad** — `stiffness`, `dampening` y `mass` — así que no se copian los números,
se convierten:

    w0   = sqrt(stiffness / mass)              -> frecuencia propia, rad/s
    zeta = dampening / (2*sqrt(stiffness*mass)) -> amortiguación

`zeta` es el único número que importa de verdad:

- `zeta = 1` — **criticamente amortiguado**: llega lo más rápido que se puede
  sin pasarse. Es el sitio de todo lo que se DESPLAZA, por la ley 5.
- `zeta < 1` — rebota. Va en las escalas, nunca en los desplazamientos.
- `zeta > 1` — se arrastra. No hace falta nunca.

| Token       | stiffness / dampening | zeta | Para qué |
|-------------|----------------------:|-----:|----------|
| `mover`     | 130 / 23   | 1.01 | `windowsMove` — mover una ventana en el mosaico |

Y el asentamiento (banda del 2 %) sale de ahí: con `zeta ≈ 1`, `ts ≈ 5.83/w0`.
Para `mover`: `w0 = 11.4`, o sea unos **510 ms** de cola completa, de los que la
mitad final es invisible porque `epsilon` la corta. Si lo quieres más seco, sube
`stiffness` y sube `dampening` **con él**, manteniendo `dampening ≈ 2*sqrt(stiffness)`,
o dejarás de estar en el crítico sin darte cuenta.

Se calibra en vivo, sin salir de la sesión, con
`~/.config/hypr/scripts/probar-muelle.sh` (lo engancha por `hyprctl eval`;
`hyprctl reload` lo deshace).

Y dos reglas que se pagan caras si se olvidan:

- **Nunca un muelle en la opacidad.** Se pasaría de 1 y parpadea. Los fundidos
  se quedan en bézier; los muelles son para escala y posición.
- **`epsilon` según la unidad.** En píxeles, `0.25` (un cuarto de píxel es
  invisible y corta la cola muerta). En escala 0..1 hace falta `0.001`, o el
  muelle se planta a un 1 % del destino — que en algo de 300 px se ve.

## Lo líquido — la dirección del sistema

Decidido por Diego el 2026-08-05: *"el indicador podría ser más líquido en
general, creo que el sistema podría seguir eso muchas veces"*.

Lo líquido **no se consigue animando figuras**. Se puede animar una escala, una
opacidad y un rebote todo lo bien que se quiera y seguirán siendo dos objetos
sólidos moviéndose cerca. Lo que el ojo reconoce como líquido es la **tensión
superficial**: que dos cuerpos que se acercan se fundan con un *cuello* antes de
tocarse, y que al separarse el hilo se estire y se rompa.

Eso obliga a cambiar de herramienta: no se dibujan figuras y se mueven, se
resuelve **un solo campo de distancia** con todos los cuerpos dentro, unidos con
`smin()` (unión suave). El primero está en
`~/.config/quickshell/shaders/liquid.frag`, y lo usa el indicador de escritorio.

Reglas aprendidas montándolo, que valen para el siguiente:

- **Los cuerpos necesitan aire o el efecto no existe.** La píldora medía 22 px y
  los puntos iban a 8 de separación: quedaba 1 px entre ambos, o sea que estaban
  fundidos *en reposo*. Si ya están pegados no hay nada que fundir al pasar. Se
  bajó la píldora a 18 y se subió la separación a 12 → ~7 px de aire. **Antes de
  tocar el shader, mirar la geometría.**
- **`k` es cuánto moja.** Por debajo de ~4 los cuerpos se ignoran hasta chocar
  (y entonces es un choque, no una fusión); muy por encima queda todo pegado
  permanentemente y se pierde la fila. 4.5 con estas distancias.
- **Antialiasing por cobertura: `clamp(0.5 - d/fwidth(d), 0, 1)`.** Da **un**
  píxel de transición. Un `smoothstep(-fwidth, +fwidth, d)` abarca **dos**, y
  con cuerpos de 8 px eso no es un borde suave: es un borrón. Es la fórmula
  estándar de SDF y no hay que inventarse otra.
- **El color se reparte por cobertura, nunca por distancia.** "Cuánto de este
  píxel es píldora" en el borde libre vale 0.5 de píldora y 0 de punto → color
  de píldora limpio. "Como de cerca está la píldora" en ese mismo borde tira
  hacia el color del punto y **pinta un cerco claro alrededor** — un halo pegado
  al borde es exactamente lo que hace que algo se vea sucio.
- **Un cuerpo dentro de otro se disuelve.** El punto activo queda bajo la
  píldora con el mismo centro y el mismo radio: los dos campos *empatan*, y con
  un empate no hay fórmula de color que valga. No se compensa al mezclar — se
  hace que el empate no exista, disolviendo el punto según la profundidad a la
  que lo tenga tragado la píldora. Por profundidad y no por índice, para que no
  dé un salto al cambiar de escritorio.
- **Margen alrededor.** El cuello abulta fuera de la caja de los cuerpos; sin
  aire, se recorta justo la parte que hace el efecto.
- **Los uniformes de color llegan YA premultiplicados.** Volver a multiplicar
  por el alfa deja los cuerpos translúcidos a un cuarto de intensidad: grises
  sucios en vez de blancos.
- **Qt cachea el shader compilado por URL.** Recompilar el `.qsb` con el mismo
  nombre y recargar el QML **no** lo recarga: sigue pintando el viejo sin ningún
  aviso. Nombre versionado (`liquid.v4.frag.qsb`) o reiniciar `qs`.

### El ligamento — cómo se estira algo de verdad

El indicador de escritorio empezó con dos muelles de dureza distinta en los dos
extremos, con la idea de que el más blando se retrasaría y esa separación sería
el estiramiento. **Medido: en un salto de dos escritorios (40 px de recorrido)
la píldora se estiraba 4 px.** Nada. Lo que parecía estiramiento en las capturas
era casi todo la fusión del metaball con el punto de al lado.

Dos muelles distintos no producen un ligamento: producen **dos cosas que llegan
casi a la vez**. Un líquido no se estira porque su cola sea más lenta — se
estira porque la cola **sigue pegada donde estaba** hasta que la tensión la
vence, y entonces se suelta de golpe.

O sea: **una pausa y después un tirón**, no una dureza distinta.

- Borde de delante: sale disparado en cuanto cambia el destino (`sprTight`).
- Borde de detrás: `PauseAnimation` de `mStagger` y luego `sprSquash` con
  `dmpPanel`. Durante la pausa el estiramiento es **el recorrido entero**.

Con eso el ciclo medido pasa a ser: reposo 18×8 → vuelo **44×5.6** → aterrizaje
18×8 → temblor 21×7.5 → reposo. Se estira a 2,4 veces su largo y adelgaza un
30 %.

Y la amortiguación del tirón importa: con `dmpSquash` (0.20) el borde se quedaba
oscilando y la píldora latía después de llegar. Un temblor está bien en el
**grosor**, donde se lee como materia, y es un fallo en la **posición**, donde
se lee como que no sabe dónde pararse.

> **Regla de QML que costó cara: nunca un muelle sobre una señal continua.**
> Puse un `Behavior { SpringAnimation }` sobre el grosor, que está *bindeado* al
> estiramiento y por tanto cambia cada fotograma. El Behavior reinicia el muelle
> 60 veces por segundo, así que no llega a simular nada — y cuando el binding
> deja de cambiar, la propiedad se queda **congelada en el último valor**. En
> pantalla: una píldora que adelgazaba al volar y no recuperaba el grosor nunca.
> Los muelles son para destinos que cambian por **sucesos**, no para seguir una
> señal continua. El grosor sale directo del estiramiento, y el temblor de
> llegada aparece solo, porque el borde de detrás se pasa al frenar.

### El truco de los dos muelles

Un solo destino con **dos muelles de carácter distinto** es de dónde sale casi
todo lo bueno:

- **El morfeo del notch.** El ancho `sprTight`, el alto `sprLoose`. El notch se
  ensancha primero y **cae después**, pasándose un poco al aterrizar. Antes los
  dos ejes iban con la misma duración y la misma curva, y por eso se reescalaba
  como una caja en vez de deformarse. Esto es squash & stretch de manual, y
  además el redondeo de las esquinas de abajo se calcula desde la altura, así
  que la forma se curva sola durante el recorrido.

### Ley 5 — El rebote va en las escalas, nunca en los desplazamientos

Algo que **se desliza** y se pasa de largo enseña el borde de la pantalla: un
hueco negro más allá del límite que delata el truco. Por eso los deslizamientos
(cambio de escritorio, capas entrando por un borde) usan `forma`, que aterriza
y no rebota.

Pero una **escala** no tiene nada que enseñar. Una ventana que se infla
pasándose un pelo de tamaño, o un panel que crece desde el 90 %, no revelan
ningún borde: solo se sienten vivos. Ahí es donde va el carácter, y ahí se usa
`entra`.

- **Rebota**: lo que crece o encoge — ventanas al abrirse (`popin 78%`), el
  morfeo del notch, los paneles, chips, tarjetas.
- **No rebota**: lo que viaja — escritorios, deslizamientos de capa, mover y
  redimensionar ventanas en el mosaico (ahí un rebote solaparía los vecinos).

> **Corregido el 2026-08-05.** La primera versión de esta ley prohibía el
> rebote en todo lo grande, razonando lo del borde de la pantalla. Eso solo
> valía para los desplazamientos; aplicarlo también a las escalas dejó el
> sistema coherente y **soso**. Diego lo dijo en tres palabras: "demasiado
> sutil". La coherencia no es lo mismo que la timidez: si algo hay que subir,
> se sube el **vocabulario** (los tokens de amplitud), no los sitios sueltos.

### Ley 6 — El movimiento tiene origen

Nada aparece "en el aire": crece desde donde está o entra desde el borde al que
pertenece. Los paneles del notch **bajan del notch**. Las notificaciones entran
por donde van a quedarse. Una ventana en mosaico se **infla en su hueco**
(`popin`), no vuela desde un borde: su sitio ya estaba decidido.

**Aplicado a las caras del notch** (`NotchLayer.qml`, `origin`). Las once caras
entraban todas igual — el mismo funde-y-escala para el reloj, el lanzador y el
menú de apagado. Once cosas distintas con un solo gesto es no tener gesto.
Ahora cada cara nace **del lado del botón que la invoca**: el lanzador por la
izquierda, que es donde está el icono de Arch; control, red, bluetooth y
apagado por la derecha, que es donde están los suyos. Lo ambiental (reloj,
música, notificaciones) no viaja y crece en el centro, porque no lo has pedido
tú desde ningún sitio.

Funciona porque el notch **recorta** (`clip`): el contenido viaja por dentro de
la ranura, así que no lo ves aparecer, lo ves llegar. Y al salir vuelve hacia su
lado, para que el gesto se lea igual de ida que de vuelta.

### Ley 7 — Nada se mueve porque sí

Ni pulsos infinitos, ni brillos que recorren nada, ni animar un dato que se
refresca solo. Si una barra de CPU se actualiza cada segundo, su transición dura
**menos** que el intervalo de refresco o se ve permanentemente en movimiento sin
que tú hayas hecho nada. Una animación que no responde a un acto tuyo es ruido.

**Excepción registrada (2026-09-06): el aviso de grabación.** El punto rojo de
«te están capturando la pantalla» guiña una vez cada 2 s: bajada y vuelta de
opacidad, 2×130 ms (RESPUESTA, OutQuad). No es adorno, es un aviso de privacidad
que debe distinguirse de un icono quieto. Antes latía en bucle continuo (900 ms,
fuera de escala) y eso mantenía la ventana del shell repintándose a 60 fps justo
mientras el codificador grababa; el guiño conserva la señal viva con ~1% del
coste. Bucles permitidos: este guiño, mTick y los spinners de tarea en curso.

### Ley 8 — Un solo hueco en la capa de contenido

`gaps_in 3` / `gaps_out 6` → entre dos ventanas quedan 6 px y hasta el borde de
la pantalla, 6 px. **El mismo hueco**. Antes eran 6 y 5: un fallo de un píxel,
invisible de mirar y perfectamente audible de sentir.

La capa de shell tiene su propio ritmo (6/10/14/16) y no tiene por qué coincidir:
son capas distintas (ley 1).

### Ley 9 — Una animación, un dueño

Si el compositor anima la aparición de una superficie **y** la app anima su
propio contenido a la vez, se ven dos animaciones encadenadas y el resultado es
un borrón. El reparto:

- El **compositor** anima superficies que aparecen y desaparecen (ventanas y
  capas efímeras: los paneles del notch, el lanzador, los avisos).
- La **app** anima lo de dentro (y las capas persistentes que solo cambian de
  tamaño, como el notch: ahí el compositor no debe meterse).

**Cómo se reparte en la práctica** (aplicado el 2026-08-05). Cada superficie de
Quickshell declara `WlrLayershell.namespace`, y eso es lo que permite darle a
cada una su propia regla en vez de un fundido genérico para todas. El criterio
no es estético, es de propiedad: mira qué anima ya la app y dale al compositor
**solo el canal que quede libre**.

| Superficie              | Regla                    | Por qué |
|-------------------------|--------------------------|---------|
| `quickshell:bar`        | `no_anim`                | capa permanente que se redimensiona sola; la anima Quickshell entera |
| `quickshell:media`      | `animation slide bottom` | vive en el borde de abajo, así que sube desde ahí. `slide` mueve posición y **no funde**, y la tarjeta ya se encarga de escala y opacidad: canales distintos |
| `quickshell:wallpaper`  | `no_anim`                | pantalla completa (deslizarla enseñaría el borde) y su `stage` ya se funde solo |
| `quickshell:overview`   | `animation fade`         | las miniaturas entran solas en cascada, pero **el velo negro no lo anima nadie por dentro**: sin esto aparecería de golpe |

La trampa a evitar es el fundido doble: si el compositor funde la superficie y
la app funde su contenido, las dos opacidades se multiplican y lo que se ve no
es el doble de suave, es que **llega tarde**.

### Ley 10 — Lo que se siente soso casi nunca es la duración: es la distancia

La ley que más cara ha salido, porque tardé tres intentos en verla.

El cambio de escritorio se sentía sutil y probé lo obvio: acortar la duración
(soso), alargarla con una curva extrema (raro). Ninguna de las dos era el
problema. El problema es que estaba en `slidefade 20%`: los escritorios
**recorrían un quinto de la pantalla** y el resto del trabajo lo hacía un
fundido. No es que el movimiento fuera flojo — es que casi no había movimiento.

Y encima lo había empeorado en nombre del realismo: puse "paralaje" (la que
entra 20 %, la que sale 50 %) razonando que dos planos a distinta distancia dan
profundidad. Sobre el papel es cierto. En pantalla ese porcentaje es **un
recorte del viaje**.

macOS no hace nada de eso, y por eso se siente contundente: los dos escritorios
están pegados como dos habitaciones contiguas y la pantalla viaja el **100 %**
de lado a lado, a la vez, **sin fundido**. El fundido es justo lo que delata
que son dos imágenes superpuestas; sin él, son un sitio del que te vas y otro al
que llegas.

Corolario práctico: antes de tocar milisegundos o curvas, comprobar **cuánto se
mueve de verdad la cosa**. Un porcentaje de `slidefade`, un `mScaleFrom` de 0.96
o un desplazamiento de 4 px son la causa habitual de que algo "no se note", y
ninguna curva del mundo lo arregla.

---

## Dónde vive cada cosa

| Fichero                              | Qué define                                    |
|--------------------------------------|-----------------------------------------------|
| `~/.config/motion-language.md`       | esto: la spec, la fuente de verdad            |
| `~/.config/hypr/hyprland.lua`        | las curvas, el muelle y las animaciones (ACTIVO) |
| `~/.config/hypr/hyprland.conf`       | la versión legacy, de respaldo. No se lee mientras exista el `.lua` |
| `~/.config/quickshell/Appearance.qml`| los mismos números como tokens QML            |

Si cambias un número, cámbialo **aquí primero** y propágalo. Dos sesiones
distintas tocando duraciones sueltas es exactamente cómo se llegó al desorden
que esto viene a arreglar.
