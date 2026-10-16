# Piano con las manos

Piano virtual controlado con la cámara web. Cada uno de los 10 dedos actúa como una tecla: al "bajar" un dedo frente a la cámara suena una nota. Proyecto de Ciencia de Datos basado en un modelo preentrenado de visión por computadora (**MediaPipe Hands**, 21 landmarks por mano).

## Características

- **Modo canción:** la pantalla indica qué mano y qué dedo bajar; la nota suena solo si aciertas. Incluye 5 canciones (Jingle Bells, Himno a la Alegría, Estrellita, Cumpleaños feliz, Mary tenía un corderito).
- **Celebración:** al terminar una canción aparece confeti, una fanfarria y un resumen con tiempo y fallos.
- **Modo libre:** los 10 dedos tocan 10 notas consecutivas; la octava es ajustable.
- **Sin archivos de audio:** el sonido se sintetiza en tiempo real con NumPy.
- **Calibración en vivo** para adaptarse a cada mano y a cada cámara.

## Requisitos

| Componente | Detalle |
|---|---|
| Python | 3.11 (entorno con el que se desarrolló el notebook) |
| Librerías | `opencv-python`, `mediapipe`, `numpy`, `pygame` |
| Hardware | Cámara web y salida de audio |
| Entorno | Ejecución local con interfaz gráfica (usa `cv2.imshow`; **no funciona en Colab ni en servidores sin pantalla**) |

> El código usa la API clásica `mp.solutions.hands`. Si una versión reciente de `mediapipe` no la incluye, instala una versión anterior de la serie 0.10.x.

## Instalación y ejecución

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install opencv-python mediapipe numpy pygame jupyter
jupyter notebook piano.ipynb
```

Ejecuta todas las celdas en orden. Se abrirá una ventana con la cámara: muestra **ambas manos abiertas con las palmas al frente** y pulsa `C` para calibrar.

## Controles

Con la ventana de la cámara seleccionada:

| Tecla | Acción |
|---|---|
| `1`–`5` | Elegir canción |
| `L` | Alternar modo libre |
| `R` | Reiniciar canción |
| `C` | Calibrar con las manos abiertas |
| `,` / `.` | Bajar / subir una octava (modo libre) |
| `D` | Mostrar valores de depuración (flexión y umbral por dedo) |
| `Q` / `Esc` | Salir |

## Estructura del notebook

| Sección | Contenido |
|---|---|
| 1. Librerías | Imports |
| 2. Configuración y canciones | Parámetros de sensibilidad y lista `CANCIONES` |
| 3. Notas y sonido | Parseo de notas, frecuencias y síntesis de audio |
| 4. Detección de dedos | Cálculo de flexión y detección de "dedo bajado" |
| 5. Dibujo y celebración | Interfaz sobre el frame (OpenCV) y confeti |
| 6. Programa principal | Bucle de captura, lógica de juego y teclado |
| 7. Ejecutar | Llamada a `main()` con limpieza de ventanas |

## Cómo funciona

### Pipeline por frame

```
Cámara → flip (espejo) → MediaPipe Hands → asignar_manos → medir (flexión)
       → actualizar (suavizado + histéresis) → evento "dedo bajó"
       → nota → sonido / validación → dibujo en pantalla
```

### Detección de dedos bajados

Para cada dedo se calcula una **razón de distancias** (en coordenadas de imagen, con `x`, `y`, `z`), por lo que es independiente del tamaño de la mano y de la distancia a la cámara:

- **Dedos 2–5:** `dist(punta, muñeca) / dist(nudillo, muñeca)` (grande = extendido, pequeño = doblado).
- **Pulgar:** `dist(punta, nudillo del meñique) / dist(nudillo del índice, nudillo del meñique)`.

Cada razón se compara con la **línea base de la mano abierta** (`BASE_ABIERTA`, recalibrable con `C`):

1. **Suavizado EMA:** `s = 0.5·s_prev + 0.5·r` para reducir el ruido del tracking.
2. **Histéresis:** el dedo se considera abajo si `s < 0.70 · base` y vuelve a estar arriba solo si `s > 0.82 · base`, lo que evita disparos múltiples por temblor.
3. **Armado y cooldown:** un dedo solo dispara una nota si estaba "armado" (había subido antes) y pasaron más de 0.10 s desde su último toque.
4. Una mano que aparece en pantalla no dispara notas por sí sola (se inicializa su estado con la postura actual).

### Asignación de manos

- Con **2 manos**, se ordenan por la coordenada `x` de la muñeca (la imagen está en espejo, así que izquierda = tu izquierda).
- Con **1 mano**, se usa la etiqueta `handedness` que entrega MediaPipe.

### Mapeo dedo → nota

Las notas se guardan como enteros diatónicos (`0 = Do4`, `7 = Do5`, `-1 = Si3`). Los 10 dedos cubren 10 notas consecutivas:

```
Izquierda:  meñique → pulgar   = notas 0–4
Derecha:    pulgar → meñique   = notas 5–9
```

`calcular_inicio` centra automáticamente cada canción sobre los 10 dedos según su rango de notas.

### Síntesis de audio

Cada nota se genera con síntesis aditiva y se precarga al inicio (rango de −10 a 20):

- Fundamental + 3 armónicos (amplitudes 1, 0.5, 0.25, 0.12).
- Ataque de 5 ms y caída exponencial `exp(-3t)`, duración de 1.4 s.
- `pygame.mixer` a 44.1 kHz, 16 bits, estéreo, buffer de 256 muestras (baja latencia) y 32 canales para polifonía.

## Configuración

Parámetros en la sección 2 del notebook:

| Parámetro | Valor | Descripción |
|---|---|---|
| `CAMARA` | `0` | Índice de la cámara (prueba `1` si no abre) |
| `SUAVIZADO` | `0.5` | Peso del valor previo en el EMA (0 = sin suavizar) |
| `FACTOR_ABAJO` | `0.70` | Umbral para considerar un dedo bajado |
| `FACTOR_ARRIBA` | `0.82` | Umbral para considerar un dedo levantado |
| `COOLDOWN` | `0.10` s | Tiempo mínimo entre dos toques del mismo dedo |

MediaPipe se inicializa con `max_num_hands=2`, `model_complexity=1`, `min_detection_confidence=0.6` y `min_tracking_confidence=0.5`. La captura se pide a 960×540.

## Agregar canciones

Añade una cadena de notas (nota + octava, `C4` = Do central) y regístrala en `CANCIONES`:

```python
MI_CANCION = "C4 D4 E4 C4  C4 D4 E4 C4"

CANCIONES = [
    ...,
    ("Mi canción", MI_CANCION),
]
```

Restricciones: solo **notas blancas** (Do mayor) y un rango máximo de **10 notas consecutivas**; de lo contrario `calcular_inicio` lanza un `ValueError`. Las teclas `1`–`5` están pensadas para 5 canciones; ajusta la ayuda en pantalla si agregas más.

## Limitaciones

- Solo notas naturales; no hay sostenidos/bemoles, velocidad (dinámica) ni pedal.
- La detección depende de la iluminación, el fondo y de mostrar las palmas hacia la cámara.
- Los dedos deben doblarse de forma clara; una flexión parcial puede no superar el umbral (se ajusta con `C` y `FACTOR_ABAJO`).
- El modelo estima profundidad (`z`) de forma aproximada, por lo que la detección del pulgar es la más sensible a la postura.

## Tecnologías

[MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker) · OpenCV · NumPy · Pygame
