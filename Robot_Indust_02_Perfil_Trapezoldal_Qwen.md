
# 📋 DOCUMENTACIÓN COMPLETA DEL PROYECTO: Simulador 3D de Brazo Robótico 7 GDL (Babylon.js)

> **Versión:** 2.1 (Perfil Trapezoidal Ajustado)
> **Última actualización:** tras integrar control de velocidad, 4 presets, combinaciones, mapa de control data‑driven, nodo TCP, robustez (A1‑A3, B1‑B4), perfil de movimiento trapezoidal y ajuste fino de parámetros.

---

## 1. RESUMEN EJECUTIVO

- **Nombre del Proyecto:** Simulador 3D Interactivo de Brazo Robótico 7 GDL
- **Tecnología:** Babylon.js (CDN) + HTML5 + JavaScript Vanilla
- **Formato:** Archivo HTML único autocontenido (`robot.html`)
- **Estado Actual:** Funcional y estable. Brazo robótico con 6 articulaciones + pinza paralela, control híbrido teclado/ratón con combinaciones y modificadores de velocidad, 4 presets de posición, perfil de movimiento trapezoidal con ajuste fino, nodo TCP, suelo de madera procedural y HUD de telemetría optimizado.

### Novedades de la versión 2.1
- ✅ Modificadores de velocidad **Shift** (precisión ×0.25) y **Ctrl** (rápido ×2.0).
- ✅ **4 Presets de posición** (F1 / F2 / F3 / F4) + reset **R** independiente.
- ✅ Combinaciones de teclas **Z / X / C / V** para mover varios ejes a la vez.
- ✅ Corrección del bug **Shift + tecla numérica** (ahora se usa `e.code`).
- ✅ **A1‑A3:** robustez de foco, clamp de `deltaTime`, bloqueo de zoom de página con Ctrl+rueda.
- ✅ **B1‑B4:** mapa de control data‑driven (`KEY_MAP`), nodo **TCP**, separación R/F1, throttle del HUD.
- ✅ **Perfil de movimiento trapezoidal** (aceleración → crucero → desaceleración) con fix de `speedingUp` para evitar rebotes.
- ✅ Parámetros ajustados: ACCEL=15, VMAX=4, DECEL=20 (articulaciones y pinza unificados).

---

## 2. ARQUITECTURA DEL CÓDIGO

El proyecto es un único archivo HTML con la siguiente estructura interna:

```
robot.html
 ├── <head>
 │   ├── Estilos CSS (layout, HUD de instrucciones)
 │   └── CDN de Babylon.js (core + GUI)
 ├── <body>
 │   ├── <canvas id="renderCanvas"> (viewport 3D)
 │   ├── <div id="instructions"> (panel de controles)
 │   └── <script> (toda la lógica)
 │       ├── createPineWoodTexture() → Textura procedural del suelo
 │       ├── createScene() → Función principal
 │       │   ├── Cámara (ArcRotateCamera)
 │       │   ├── Iluminación (Hemispheric + Directional)
 │       │   ├── Suelo (madera de pino procedural)
 │       │   ├── Sombras (BlurExponentialShadowMap)
 │       │   ├── Materiales (StandardMaterial x3)
 │       │   ├── Cinemática 6 articulaciones (TransformNode jerárquicos)
 │       │   ├── Muñeca 3 (J6 - giro destornillador)
 │       │   ├── Pinza paralela (J7 - traslación lineal)
 │       │   ├── Nodo TCP (Tool Center Point) ── [B2]
 │       │   ├── KEY_MAP (mapa de control data-driven) ── [B1]
 │       │   ├── presets{} + applyPreset() (F1-F4)
 │       │   ├── velocities[] + parámetros trapezoidales (ACCEL/VMAX/DECEL)
 │       │   ├── Input Manager (teclado e.code + ratón)
 │       │   ├── Robustez (blur, dt-clamp, wheel) ── [A1-A3]
 │       │   ├── HUD (Babylon.GUI + throttle) ── [B4]
 │       │   └── Bucle de renderizado (perfil trapezoidal + límites)
 │       └── engine.runRenderLoop()
```

**Dependencias externas (CDN):**
- `https://cdn.babylonjs.com/babylon.js` (Core)
- `https://cdn.babylonjs.com/gui/babylon.gui.min.js` (Interfaz)

---

## 3. CONFIGURACIÓN CINEMÁTICA (7 GDL)

### Jerarquía de nodos
```
Mundo
 └── J1 (Base) ── rotación Y (Yaw)
      └── J2 (Hombro) ── rotación Z (Pitch)
           └── J3 (Codo) ── rotación Z (Pitch)
                └── J4 (Muñeca 1) ── rotación X (Roll)
                     └── J5 (Muñeca 2) ── rotación Z (Pitch)
                          └── J6 (Muñeca 3) ── rotación Y (Yaw/Destornillador)
                               ├── [Brida + Pinza Paralela J7]
                               └── [Nodo TCP] ── punto de herramienta (B2)
```

### Tabla de Articulaciones
| Eje | Nombre | Tipo de Movimiento | Eje de Rotación | Rango (grados) | Control individual |
| --- | --- | --- | --- | --- | --- |
| J1 | Base | Yaw (giro horizontal) | Y | ±170° | `1` + Ratón X |
| J2 | Hombro | Pitch (elevación) | Z | ±120° | `2` + Ratón Y |
| J3 | Codo | Pitch (flexión) | Z | ±170° | `3` + Ratón Y |
| J4 | Muñeca 1 | Roll (giro antebrazo) | X | ±120° | `4` + Ratón X |
| J5 | Muñeca 2 | Pitch (inclinación) | Z | ±120° | `5` + Ratón Y |
| J6 | Muñeca 3 | Yaw (giro destornillador) | Y | ±360° | `6` + Ratón X |
| J7 | Pinza | Traslación lineal X | N/A (posición) | 0.08 a 0.88 u | `7` + Ratón Y |

> **Nota:** El control de J7 pasó de Ratón X a **Ratón Y** en esta versión.

---

## 4. SISTEMA DE CONTROL (INPUT MANAGER)

### Lógica de funcionamiento
- Se usa un `Set` de JavaScript (`keysPressed`) para rastrear teclas presionadas, **almacenando `e.code`** (no `e.key`) para que los modificadores no alteren la identificación de la tecla.
- El ratón actúa como joystick analógico: se mide el delta del movimiento (`evt.movementX`, `evt.movementY`), no la posición absoluta.
- **Conflicto cámara/robot resuelto:** solo se capturan deltas del ratón cuando `evt.buttons === 0` (sin botones presionados). Así, clic+arrastrar orbita la cámara sin mover el robot.
- **Acordes (simultaneidad):** si se presionan múltiples teclas (ej: `1` y `4`), el delta del ratón se aplica a ambos ejes en el mismo frame.
- **Límites de software:** `BABYLON.Scalar.Clamp()` impide que las articulaciones excedan sus rangos físicos.

### 4.1 Teclas individuales
| Tecla | Eje del Ratón | Acción |
| --- | --- | --- |
| `1` | X | Rotar Base (J1) |
| `2` | Y | Rotar Hombro (J2) |
| `3` | Y | Rotar Codo (J3) |
| `4` | X | Rotar Muñeca 1 (J4) |
| `5` | Y | Rotar Muñeca 2 (J5) |
| `6` | X | Rotar Muñeca 3 (J6) |
| `7` | Y | Abrir/Cerrar Pinza (J7) |

### 4.2 Combinaciones (varios ejes a la vez)
| Tecla | Ejes afectados | Ratón |
| --- | --- | --- |
| `Z` | J1 + J2 (Base + Hombro) | X → J1, Y → J2 |
| `X` | J3 (Codo) | Y → J3 |
| `C` | J4 + J5 (Muñecas 1‑2) | X → J4, Y → J5 |
| `V` | J6 + J7 (Muñeca 3 + Pinza) | X → J6, Y → J7 |

> Mayúsculas y minúsculas son indistintas (`e.code` es idéntico para `z` y `Z`).

### 4.3 Modificadores de velocidad
| Modificador | Multiplicador | Uso |
| --- | --- | --- |
| *(ninguno)* | ×1.0 | Velocidad normal |
| `Shift` | ×0.25 | Precisión / ajustes finos |
| `Ctrl` | ×2.0 | Movimientos rápidos |

Implementación: `effSensitivity = sensitivity * speedMultiplier`, donde `sensitivity = 0.005`. El modificador escala la velocidad a la que el usuario desplaza los **objetivos** (`targets`); el perfil trapezoidal (Sección 5) gobierna cómo las posiciones actuales persiguen a esos objetivos. Ambos mecanismos son independientes y se combinan.

### 4.4 Presets y reset
| Tecla | Nombre | Ángulos [J1..J6] (°) | Pinza | Descripción |
| --- | --- | --- | --- | --- |
| `F1` | Posición Inicial | `[-20, -30, -60, 0, -90, 0]` | 0.5 | Pose estética "listo para trabajar" |
| `F2` | Posición 2 | `[35, 10, 105, 0, -25, 90]` | 0.5 | Brazo elevado, muñeca rotada |
| `F3` | Posición 3 | `[-45, 30, 115, 0, -55, 90]` | 1.0 | Brazo girado izquierda, pinza abierta |
| `F4` | Posición 4 | `[-75, 30, 75, -60, -25, -90]` | 1.0 | Pose compleja con rotación de muñecas |
| `R` | Reset (Home) | `[0, 0, 0, 0, 0, 0]` | 0.5 | Vuelve todo a posición vertical |

- Los presets se definen en un objeto `presets` (ángulos en grados + apertura normalizada 0‑1) y se aplican con `applyPreset()`, que convierte a radianes y escribe en `targets[]` y `gripperTarget`. El movimiento hacia el preset es suave gracias al perfil trapezoidal.
- Se llama a `e.preventDefault()` para `F1‑F4` y así evitar que el navegador abra la ayuda.
- **`R` y `F1` son funciones independientes** (`R` = home absoluto; `F1` = pose inicial estética). *[B3]*

### 4.5 Mapa de control data‑driven (`KEY_MAP`) *[B1]*
Las teclas ya no se resuelven con una cadena de `if`, sino con una tabla que mapea cada `e.code` a una lista de acciones. Añadir o re‑mapear una tecla = editar la tabla, no tocar la lógica del bucle.

```javascript
const KEY_MAP = {
    Digit1: [{ j: 0, axis: "x" }],
    Digit2: [{ j: 1, axis: "y" }],
    Digit3: [{ j: 2, axis: "y" }],
    Digit4: [{ j: 3, axis: "x" }],
    Digit5: [{ j: 4, axis: "y" }],
    Digit6: [{ j: 5, axis: "x" }],
    Digit7: [{ g: true, axis: "y", scale: 2 }],
    KeyZ:   [{ j: 0, axis: "x" }, { j: 1, axis: "y" }],
    KeyX:   [{ j: 2, axis: "y" }],
    KeyC:   [{ j: 3, axis: "x" }, { j: 4, axis: "y" }],
    KeyV:   [{ j: 5, axis: "x" }, { g: true, axis: "y", scale: 2 }]
};
// j: índice de articulación | g: gripper | axis: eje del ratón | scale: multiplicador
```

### 4.6 Identificación de teclas con `e.code` (corrección Shift+número)
- **Problema:** con `e.key`, presionar `Shift + 1` producía `"!"`, `Shift + 2` producía `"\""`, etc., bloqueando las teclas numéricas al combinarse con Shift/Ctrl.
- **Solución:** usar `e.code` (identificador físico de la tecla): `Digit1`, `Digit2`, …, `ShiftLeft`, `ControlLeft`, `KeyZ`, etc. `keysPressed` y `KEY_MAP` operan sobre `e.code`, por lo que los modificadores ya no alteran la tecla detectada.

---

## 5. PERFIL DE MOVIMIENTO TRAPEZOIDAL

Reemplaza al suavizado Lerp simple (`Lerp` con factor `dt * 8`) por un perfil de velocidad realista: **aceleración → velocidad crucero → desaceleración**, similar al de los servodrives industriales.

### Estado y parámetros
```javascript
const velocities = [0, 0, 0, 0, 0, 0];  // velocidad angular por articulación (rad/s)
let gripperVelocity = 0;                 // velocidad lineal de la pinza (u/s)

// Articulaciones (rad y rad/s)
const ACCEL = 15;    // Rampa de subida: más bajo = inicio más lento
const VMAX  = 4;     // Tope de velocidad crucero: más bajo = más limitado
const DECEL = 20;    // Frenado: más alto = llegada más firme y rápida

// Pinza (unidades lineales u y u/s) - unificados con articulaciones
const G_ACCEL = 15;
const G_VMAX  = 4;
const G_DECEL = 20;
```

### Algoritmo (por articulación y por frame)
```javascript
const error = targets[i] - currents[i];
const dist  = Math.abs(error);

if (dist < 0.0020) {
    currents[i] = targets[i];        // Snap final: llegada exacta
    velocities[i] = 0;
} else {
    // Velocidad permitida por la distancia restante (frenado en raíz cuadrada)
    const vBrake  = Math.sqrt(2 * DECEL * dist);
    const vTarget = Math.sign(error) * Math.min(VMAX, vBrake);
    // Limitar aceleración (subida) o frenado (bajada)
    const dv = vTarget - velocities[i];
    // Fix de speedingUp: evita rebotes al cambiar de dirección
    const speedingUp = (velocities[i] === 0) || 
                       (Math.sign(velocities[i]) === Math.sign(vTarget) && 
                        Math.abs(vTarget) > Math.abs(velocities[i]));
    const dvMax = (speedingUp ? ACCEL : DECEL) * dt;
    velocities[i] += Math.sign(dv) * Math.min(Math.abs(dv), dvMax);
    // Integrar posición
    currents[i] += velocities[i] * dt;
}
```

### Fases del perfil
| Fase | Comportamiento | Parámetro que la gobierna |
| --- | --- | --- |
| Arranque | La velocidad sube desde 0 con pendiente limitada | `ACCEL` |
| Crucero | Velocidad constante topeada | `VMAX` |
| Frenado | La velocidad cae según `√(2·DECEL·dist)` | `DECEL` |
| Llegada | Snap exacto cuando `dist < 0.0020` | umbral de snap |

- El frenado en raíz cuadrada garantiza llegar en **tiempo finito** (sin la cola lenta del Lerp exponencial), y el snap final asegura que el HUD y la pose queden clavados en el valor exacto.
- La pinza aplica el mismo perfil con `G_ACCEL / G_VMAX / G_DECEL` (unificados con las articulaciones para consistencia).
- **Fix de `speedingUp`:** la condición original `Math.abs(vTarget) > Math.abs(velocities[i])` causaba rebotes al cambiar de dirección. La nueva condición verifica que velocidad y objetivo apunten en la misma dirección antes de acelerar, evitando oscilaciones.
- **Guía de ajuste:** para inicio más lento bajar `ACCEL`; para limitar el crucero bajar `VMAX`; para llegada más firme subir `DECEL`.

---

## 6. NODO TCP (Tool Center Point) *[B2]*

Punto de referencia en la punta de la herramienta, preparado para cinemática inversa y agarre de objetos.

```javascript
const tcp = new BABYLON.TransformNode("TCP", scene);
tcp.parent = j6;
tcp.position = new BABYLON.Vector3(0, 1.3, 0);  // punta de los dedos
```

- **Usos futuros:** objetivo de IK (`tcp.getAbsolutePosition()`), punto de anclaje al agarrar cubos (`cube.parent = tcp`), cálculo de orientación de la herramienta.
- Al ser hijo de J6, hereda automáticamente todos los movimientos del brazo.

---

## 7. COMPONENTES VISUALES

### 7.1 Materiales (StandardMaterial)
| Material | Color Difuso | Specular | Uso |
| --- | --- | --- | --- |
| matOrange | (0.95, 0.5, 0.1) | (0.3, 0.3, 0.3) | Brazos, base superior, anillo J6 |
| matDarkGrey | (0.25, 0.25, 0.25) | (0.4, 0.4, 0.4) | Base, muñeca 2, puntas de dedos, guías |
| matMetal | (0.75, 0.75, 0.8) | (0.8, 0.8, 0.8) | Articulaciones, carrier, dedos |

> **Nota:** se usan `StandardMaterial` en lugar de `PBRMetallicRoughnessMaterial` por compatibilidad y visibilidad garantizada sin necesidad de HDRI.

### 7.2 Iluminación
- **HemisphericLight:** intensidad 0.9, color de suelo (0.2, 0.2, 0.3). Iluminación ambiental uniforme.
- **DirectionalLight:** intensidad 0.8, dirección (-1, -2, -1), posición (10, 20, 10). Genera sombras.

### 7.3 Sombras
- `ShadowGenerator` de 1024×1024
- `useBlurExponentialShadowMap = true`
- `blurKernel = 32`

### 7.4 Suelo de Madera de Pino (Procedural)
- Función `createPineWoodTexture()` que genera un `DynamicTexture` de 512×512.
- Color base: `#d4a574` (amarillo‑marrón claro de pino).
- 80 vetas verticales con variaciones de tono y ondulaciones.
- 3 nudos ocasionales con gradiente radial.
- Textura repetida 4×4 (`uScale = 4, vScale = 4`).
- Specular bajo para simular madera barnizada.

### 7.5 Cámara
- `ArcRotateCamera` apuntando a (0, 3, 0)
- Alpha: -π/2, Beta: π/3, Radio: 12
- Límites de zoom: 5 a 25

---

## 8. PINZA PARALELA MECÁNICA (J7)

### Diseño
- **Tipo:** pinza paralela de 2 dedos (movimiento lineal, sin rotación).
- **Mecanismo:** los dedos se trasladan en el eje X manteniéndose perfectamente paralelos.

### Componentes
- Base adaptadora (0.5 × 0.15 × 0.4)
- Carrier/riel alargado (1.05 × 0.1 × 0.35) — cubre todo el rango de apertura
- Guías laterales izquierda y derecha
- Dedo izquierdo (0.08 × 0.6 × 0.2) + punta (0.1 × 0.15 × 0.22)
- Dedo derecho (0.08 × 0.6 × 0.2) + punta (0.1 × 0.15 × 0.22)

### Rangos
- **Cerrado:** 0.08 unidades entre dedos (casi se tocan, sin atravesarse)
- **Abierto:** 0.88 unidades entre dedos (suficiente para cubos de 0.8 u)
- **Posición inicial:** 50% de apertura (0.48 u entre dedos)
- **Reset:** vuelve a 50% de apertura

### Implementación técnica
```javascript
const openingDistance = BABYLON.Scalar.Lerp(gripperMinOpening, gripperMaxOpening, gripperCurrent);
leftFinger.position.x  = -openingDistance;
rightFinger.position.x =  openingDistance;
```
> El valor `gripperCurrent` (0‑1) ahora es conducido por el **perfil trapezoidal** de la pinza (`G_ACCEL/G_VMAX/G_DECEL`), no por un Lerp simple.

---

## 9. HUD / INTERFAZ DE USUARIO

- Creado con `BABYLON.GUI.AdvancedDynamicTexture.CreateFullscreenUI()`.
- Panel `StackPanel` alineado a la derecha superior.
- 7 líneas de texto:
  - 6 líneas blancas para ángulos de J1‑J6 (en grados, 1 decimal).
  - 1 línea cian para apertura de pinza J7 (en unidades, 2 decimales).
- Fondo semitransparente negro en cada línea.
- **Throttle de actualización *[B4]*:** el HUD se refresca a ~10 Hz (cada 0.1 s) en lugar de cada frame, reduciendo la generación de strings y la presión sobre el garbage collector sin pérdida perceptible.

---

## 10. ROBUSTEZ Y OPTIMIZACIONES

| Ref | Cambio | Propósito |
| --- | --- | --- |
| **A1** | `window.addEventListener("blur", () => keysPressed.clear())` | Evita teclas "pegadas" si la ventana pierde el foco con teclas presionadas (Alt+Tab, clic fuera). |
| **A2** | `dt = Math.min(engine.getDeltaTime()/1000, 0.1)` | Clamp del delta time: evita saltos bruscos del brazo tras cambiar de pestaña. |
| **A3** | `canvas.addEventListener("wheel", e => { if (e.ctrlKey) e.preventDefault(); }, {passive:false})` | Evita que Ctrl+rueda haga zoom de página del navegador (la cámara sigue recibiendo el evento). |
| **B1** | `KEY_MAP` data‑driven | Añadir/re‑mapear teclas sin tocar la lógica del bucle. |
| **B2** | Nodo `TCP` | Punto de herramienta para IK y agarre futuro. |
| **B3** | `R` y `F1` separados; `F1` redefinido como pose inicial estética | Home absoluto vs. pose de presentación. |
| **B4** | Throttle del HUD a ~10 Hz | Menos basura de strings, menor carga de GC. |

---

## 11. ERRORES CONOCIDOS Y SOLUCIONES APLICADAS

| Error | Causa | Solución |
| --- | --- | --- |
| `CreateCone is not a function` | `BABYLON.MeshBuilder.CreateCone` no existe | Usar `CreateCylinder` con `diameterTop: 0` |
| Pantalla negra / brazo invisible | Materiales PBR sin HDRI + iluminación insuficiente | Cambiar a `StandardMaterial` + añadir `HemisphericLight` + colores más brillantes |
| Dedos flotando al abrir pinza | Carrier demasiado corto | Alargar carrier a 1.05 unidades + añadir guías laterales |
| Teclas numéricas bloqueadas con Shift/Ctrl | `e.key` devuelve `"!"`, `"\""`, `"#"`… al combinar con Shift | Usar `e.code` (`Digit1`, `ShiftLeft`, `KeyZ`…) en `keydown`, `keyup` y `KEY_MAP` |
| El brazo no llegaba al destino (Lerp dinámico) | Cola exponencial del Lerp cerca del objetivo | Reemplazo por perfil trapezoidal con frenado `√(2·DECEL·dist)` + snap final |
| Rebote al cambiar de dirección | Condición `speedingUp` incorrecta | Fix: verificar dirección antes de acelerar |
| Vibración cerca del objetivo | Umbral de snap demasiado estricto (0.0005) | Ajustado a 0.0020 (≈0.11°) para llegada limpia |
| Teclas pegadas tras Alt+Tab | El `keyup` no llega si se pierde el foco | Listener de `blur` que limpia `keysPressed` (A1) |
| Salto brusco al volver de otra pestaña | `getDeltaTime()` enorme | Clamp de `dt` a 0.1 s (A2) |
| Ctrl+rueda hace zoom de página | Comportamiento por defecto del navegador | `preventDefault()` en `wheel` si `ctrlKey` (A3) |

---

## 12. ROADMAP / PRÓXIMOS PASOS SUGERIDOS

### Fase A: Interacción con Objetos (Prioridad Alta)
- [ ] Añadir cubos de prueba en la escena (sobre la mesa de madera)
- [ ] Implementar detección de colisiones entre los dedos de la pinza y los cubos
- [ ] Lógica de agarre: cuando la pinza se cierra sobre un cubo, el cubo se "attacha" al nodo **TCP** (`cube.parent = tcp`) y se mueve con ella
- [ ] Lógica de soltar: cuando la pinza se abre, el cubo cae con gravedad (usar plugin de física Havok o Cannon)

### Fase B: Fotorrealismo Avanzado (Prioridad Media)
- [ ] Migrar de `StandardMaterial` a `PBRMetallicRoughnessMaterial`
- [ ] Cargar HDRI de entorno industrial (`.env`)
- [ ] Añadir `DefaultRenderingPipeline` con Bloom, DoF, SSR, Tonemapping ACES
- [ ] Texturas PBR con mapas de rugosidad, normal y metálico

### Fase C: Cinemática Inversa (Prioridad Media)
- [ ] Implementar IK para que el usuario pueda arrastrar el **TCP** y que el brazo calcule automáticamente los ángulos de las 6 articulaciones
- [ ] Usar el solver CCD (Cyclic Coordinate Descent) o Jacobiano
- [ ] *El nodo TCP (B2) ya está preparado como objetivo/efector.*

### Fase D: Mejoras de UX (Prioridad Baja)
- [ ] Modo de control alternativo: botones en pantalla para móviles (aprovechando `KEY_MAP`)
- [ ] Grabación y reproducción de trayectorias
- [ ] Exportar posiciones a formato G-code o ROS
- [ ] Modo de colisiones con auto-parada

### Fase E: Arquitectura (Prioridad Baja)
- [ ] Migrar de HTML único a proyecto Vite + TypeScript modular
- [ ] Separar en módulos: `SceneManager`, `RobotKinematics`, `InputManager`, `GripperController`, `HUDManager`, `MotionProfile`
```

---

## Resumen de cambios respecto a la documentación anterior

1. **Versión actualizada a 2.1** (antes era 2.0)
2. **4 presets** en lugar de 3 (se agregó F4 con ángulos `[-75, 30, 75, -60, -25, -90]`)
3. **Parámetros trapezoidales actualizados:** ACCEL=15, VMAX=4, DECEL=20 (antes eran 4.0, 2.5, 10.0)
4. **Parámetros de pinza unificados:** G_ACCEL=15, G_VMAX=4, G_DECEL=20 (antes eran diferentes)
5. **Umbral de snap ajustado:** 0.0020 (antes era 0.0005)
6. **Sección 5 actualizada** con el fix de `speedingUp` y explicación del rebote
7. **Sección 11 ampliada** con los nuevos errores resueltos (rebote, vibración)

La documentación está lista para copiar y pegar en tu archivo `.md`.