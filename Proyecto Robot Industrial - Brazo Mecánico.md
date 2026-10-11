Aquí tienes una propuesta detallada y profesional para el desarrollo de este proyecto. Está estructurada como un documento de especificaciones técnicas y de diseño, ideal para presentarlo a un cliente, un equipo de desarrollo o como guía para tu propio trabajo.

---

# Propuesta de Proyecto: Simulador 3D Interactivo de Brazo Robótico de 6 GDL (RoboArm3D Pro)

## 1. Resumen del Proyecto
Desarrollo de una Aplicación de Página Única (SPA) web basada en **Babylon.js** que simula un brazo robótico industrial de 6 Grados de Libertad (GDL). El proyecto se enfocará en dos pilares fundamentales: **fotorrealismo extremo** aprovechando el motor de renderizado de Babylon.js, y un **sistema de control híbrido (Teclado + Ratón)** innovador que permita la manipulación directa e intuitiva de las articulaciones.

## 2. Configuración Estándar del Brazo Robótico (Cinemática)
Para garantizar un comportamiento industrial realista, se utilizará la configuración cinemática estándar de un brazo articulado de 6 ejes (similar a los modelos de KUKA, ABB o FANUC). 

Los puntos de pivote (origen de rotación) en el modelo 3D deben ser exactos:
*   **Eje 1 (Base):** Rotación en el eje Y (Yaw / Giro horizontal de la base). Rango: ±170°
*   **Eje 2 (Hombro):** Rotación en el eje Z (Pitch / Elevación del brazo). Rango: +120° a -120°
*   **Eje 3 (Codo):** Rotación en el eje Z (Pitch / Flexión del antebrazo). Rango: +170° a -170°
*   **Eje 4 (Muñeca 1):** Rotación en el eje X (Roll / Giro del antebrazo). Rango: ±120°
*   **Eje 5 (Muñeca 2):** Rotación en el eje Z (Pitch / Inclinación de la muñeca). Rango: ±120°
*   **Eje 6 (Brida/Herramienta):** Rotación en el eje Y (Yaw / Giro final de la herramienta). Rango: ±360°

## 3. Sistema de Control Interactivo (UX/UI)
El control utiliza un sistema de "acordes" (combinación de teclas modificadoras + movimiento del ratón) para permitir un control fluido sin necesidad de menús complejos. El ratón actúa como un joystick virtual de alta precisión.

### Esquema de Mapeo:
El movimiento del ratón se divide en **Eje X (Horizontal)** y **Eje Y (Vertical)**.

*   **Tecla `1` + Mover Ratón en X:** Controla **Articulación 1** (Base).
*   **Tecla `2` + Mover Ratón en Y:** Controla **Articulación 2** (Hombro).
*   **Tecla `3` + Mover Ratón en X:** Controla **Articulación 3** (Codo).
*   **Tecla `4` + Mover Ratón en Y:** Controla **Articulación 4** (Muñeca 1).
*   **Tecla `5` + Mover Ratón en X:** Controla **Articulación 5** (Muñeca 2).
*   **Tecla `6` + Mover Ratón en Y:** Controla **Articulación 6** (Brida).

### Control Simultáneo (Modo Avanzado):
Tal como se solicitó, el sistema soporta la interpolación de inputs. 
*   *Ejemplo:* Si el usuario mantiene presionadas las teclas **`1` y `4`** simultáneamente:
    *   El movimiento horizontal (X) del ratón rotará la **Base (Eje 1)**.
    *   El movimiento vertical (Y) del ratón rotará la **Muñeca 1 (Eje 4)**.
*   *Reset:* Presionar la tecla `R` (Reset) devuelve el brazo a su posición "Home" (cero) de manera animada y suave.

## 4. Fotorrealismo en Babylon.js (Rendering & Visuals)
Para cumplir con el requisito de que el proyecto se vea "lo más realista posible", se implementará el siguiente pipeline gráfico:

### A. Materiales y Texturas (PBR - Physically Based Rendering)
*   Uso exclusivo de materiales **PBRMetallicRoughness**.
*   **Metales:** Pintura industrial con micro-rayones (usando *DetailMap* o mapas de rugosidad desgastados), acero inoxidable cepillado en las articulaciones.
*   **Plásticos y Cables:** Gomas mate, plásticos texturizados y "cables" (usando *Tube* o *Ribbon* con dinámica física suave o *Dynamic Texture*) que se deformen según el ángulo de las articulaciones.

### B. Iluminación y Entorno
*   **HDRI (High Dynamic Range Imaging):** Uso de un mapa de entorno `.env` de alta resolución (ej. una fábrica industrial o un laboratorio de robótica) para reflejos realistas en las partes metálicas.
*   **Iluminación Volumétrica y de Área:** Uso de *Area Lights* para simular paneles LED industriales.
*   **Sombras:** Sombras de alta calidad usando *PCF (Percentage Closer Filtering)* o *Poisson Sampling* con un mapa de sombras de alta resolución (2048x2048 o 4096x4096).

### C. Post-Procesado (Image Processing)
*   **SSR (Screen Space Reflections):** Para reflejos en el suelo de epoxi o superficies pulidas.
*   **Depth of Field (DoF):** Efecto de profundidad de campo sutil para enfocar el brazo y desenfocar el fondo, dando un look cinematográfico.
*   **Bloom y Lens Dirt:** Efectos sutiles en las luces LED de estado del robot (ej. luz verde/roja en la base).
*   **Anti-aliasing:** SMAA o FXAA para bordes perfectos.

### D. Detalles Superficiales (Decals)
*   Uso del sistema de *Decals* de Babylon.js para superponer calcomanías realistas: logos de la marca, etiquetas de advertencia (alto voltaje), números de serie y tornillería visible.

## 5. Stack Tecnológico
*   **Motor 3D:** Babylon.js (v6.x o v7.x).
*   **Lenguaje:** TypeScript (indispensable para manejar la lógica compleja de estados del teclado/ratón y la cinemática).
*   **Build Tool:** Vite (para un entorno de desarrollo SPA rápido y moderno).
*   **Modelado 3D:** Blender (para crear el modelo, riggear los pivotes exactos y exportar a formato **.glTF / .glb**).
*   **UI (Interfaz de Usuario):** Babylon.GUI o un overlay de HTML/CSS (React/Vue) para mostrar el HUD (ángulos actuales, estado de las teclas presionadas, FPS).

## 6. Consideraciones Técnicas Críticas (Riesgos y Soluciones)
1.  **Gimbal Lock (Bloqueo de cardán):** Al usar ángulos de Euler, las articulaciones pueden bloquearse. **Solución:** Babylon.js maneja las rotaciones internamente con Quaternions. Debemos asegurarnos de aplicar las rotaciones relativas al nodo padre correcto.
2.  **Intersección de Geometría (Self-Collision):** El brazo podría atravesarse a sí mismo. **Solución:** Para esta versión (Cinemática Directa), se implementarán "colisiones virtuales" (límites de ángulo estrictos en el código). Si se requiere detección de colisiones realista, se puede integrar el motor de físicas de Havok/Cannon, aunque impactará el rendimiento.
3.  **Fuga de Ratón (Mouse Capture):** Para que el ratón no salga de la ventana al girar infinitamente, se usará la API `Pointer Lock` del navegador, o se limitará el movimiento del ratón a un área virtual, calculando el *delta* del movimiento en lugar de la posición absoluta.

