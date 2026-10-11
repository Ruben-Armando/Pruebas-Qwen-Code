# 🤖 Simulador 3D — Brazo Robótico Industrial 7 GDL

Implementación del proyecto **Simulador 3D Interactivo de Brazo Robótico 7 GDL** según la
especificación [`Robot_Indust_02_Perfil_Trapezoldal_Qwen.md`](./Robot_Indust_02_Perfil_Trapezoldal_Qwen.md)
(v2.1, prioridad absoluta), que refina la propuesta general
[`Proyecto Robot Industrial - Brazo Mecánico.md`](./Proyecto%20Robot%20Industrial%20-%20Brazo%20Mec%C3%A1nico.md).

## Archivo principal

**[`robot.html`](./robot.html)** — SPA autocontenida (un solo archivo HTML):

- Babylon.js + Babylon.GUI vía CDN (`https://cdn.babylonjs.com/...`).
- Cinemática jerárquica de 6 articulaciones (J1–J6) + pinza paralela J7 + nodo **TCP**.
- **Perfil de movimiento trapezoidal** (ACCEL=15, VMAX=4, DECEL=20; pinza unificada; snap 0.0020; fix `speedingUp`).
- Input Manager data-driven (`KEY_MAP`, `e.code`), teclas `1–7`, combinaciones `Z/X/C/V`,
  modificadores `Shift ×0.25` / `Ctrl ×2.0`, presets `F1–F4` y reset `R` independiente.
- Robustez A1–A3 (blur, clamp de `dt`, Ctrl+rueda) y HUD con throttle ~10 Hz (B4).
- Visuales: StandardMaterials (naranja/gris/metal), luz hemisférica + direccional,
  sombras BlurExponential (1024, kernel 32), suelo de madera de pino procedural (DynamicTexture 512², 80 vetas, 3 nudos, repetición 4×4), cámara ArcRotate (zoom 5–25).

## Cómo ejecutar

Abrir `robot.html` directamente en un navegador moderno (requiere conexión a internet para los CDN),
o servir la carpeta localmente:

```bash
python3 -m http.server 8080
# → http://localhost:8080/robot.html
```

## Controles

| Tecla | Acción | Ratón |
| --- | --- | --- |
| `1`…`7` | J1 Base, J2 Hombro, J3 Codo, J4 Muñeca 1, J5 Muñeca 2, J6 Muñeca 3, J7 Pinza | X o Y según eje |
| `Z` / `X` / `C` / `V` | Combinaciones J1+J2 / J3 / J4+J5 / J6+Pinza | X e Y simultáneos |
| `Shift` / `Ctrl` | Precisión ×0.25 / Rápido ×2.0 | — |
| `F1`–`F4` | Presets de pose | — |
| `R` | Reset Home absoluto | — |
| Clic+arrastrar | Orbitar cámara (no mueve el robot) | — |

## Verificación

La lógica del bucle (convergencia trapezoidal exacta sin rebotes, respeto de VMAX, límites ±rangos,
modificadores de velocidad sobre `targets`, acordes, KEY_MAP, presets F1–F4/R, pinza 0.08–0.88 u,
conflicto cámara/robot y limpieza de teclas por `blur`) fue validada con una suite de pruebas
unitarias en Node.js usando un stub de la API de Babylon (13 casos, todos PASS).
