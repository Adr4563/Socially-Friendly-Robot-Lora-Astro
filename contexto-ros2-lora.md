# Contexto técnico completo — robot "Lora" (para diseñar la migración a ROS 2 Jazzy)

> Documento preparado para pegar en ChatGPT y que ayude a estructurar los
> paquetes/nodos/tópicos de ROS 2 Jazyy. Cubre TODO lo relevante del proyecto
> actual (monolítico en Python) que hoy vive en
> `Arquitecture-Agentic-RAG/deploy-raspberry-standalone/` + los dos proyectos
> hermanos de hardware (`ai-camera/`, `carrito-mecanum-esp32/`).

## 1. Qué es el robot y qué hace hoy

Robot educativo llamado **Lora**: conversa (Chat libre) y juega Trivia por
voz/texto con un usuario, con una carita animada en pantalla, música,
movimiento físico (motores) y reconocimiento de emociones faciales por
cámara. Todo corre HOY como **un solo proceso Python de larga vida**
(`Orchestrator_Management.py`) en una **Raspberry Pi 4 (8GB RAM, sin GPU,
ARM Cortex-A72)**, con **Ollama** como único servicio externo de red (LLM
local).

## 2. Hardware real y cómo se controla hoy

### 2.1 Motores (carrito mecanum, 4 ruedas)
- Chasis mecanum de 4 ruedas independientes, driver **L298N**, controlado por
  un **ESP32-S3 (DevKitC-1)**.
- Conexión: **Serial/USB directo** a la Raspberry Pi (antes era WiFi/HTTP,
  se cambió a cable para simplicidad y no depender de la LAN).
- Firmware: `carrito-mecanum-esp32/2-l298n-mecanum/mecanum_car_esp32s3.ino`
  (Arduino/C++). Protocolo: **un comando de texto por línea** vía Serial a
  115200 baud:
  - `F` adelante, `B` atrás
  - `SL` / `SR` desplazamiento lateral (strafe, propio de ruedas mecanum,
    sin rotar el chasis)
  - `RL` / `RR` rotar sobre el propio eje
  - `FL` / `FR` / `BL` / `BR` diagonales
  - `S` (o cualquier texto no reconocido) = stop
- Pines motor (8 GPIO, puente H por rueda): `IN1..IN8` = 4, 5, 6, 7, 15, 16,
  17, 18 (Front Left, Front Right, Back Left, Back Right).
- **Watchdog de seguridad en el firmware**: si no llega OTRO comando dentro
  de 500ms, el ESP32 frena los motores solo (protección ante cable
  desconectado o proceso colgado del lado Pi).
- No hay "girar 360°" como comando atómico: el lado Python (`mover_360()`)
  lo aproxima mandando `RR` 6 veces con pausa de 0.4s entre cada una — **sin
  calibrar contra el hardware real** (grados reales por pulso desconocidos).
- Lado Python: `Clients/Carrito_Client.py` — conexión persistente a nivel de
  módulo (abrir el puerto resetea el ESP32 vía DTR, así que no se
  abre/cierra por comando), reconecta sola si se cae, puerto configurable
  por `CARRITO_PORT` (default `/dev/ttyACM0`). Si el ESP32 no está
  conectado, **no lanza excepción**: solo loguea y sigue (mismo patrón que
  el resto de los clientes de hardware del proyecto).
- Quién dispara movimiento: `Agents/Agent_Behavior.py::expresar_desplazamiento()`,
  a partir de la columna `desplazamiento` del dataset de preguntas
  (valores: Adelante/Atrás/Izquierda/Derecha/"Girar 360°"), como parte de la
  reacción al veredicto de una pregunta de Trivia.

### 2.2 Cámara (reconocimiento de emociones)
- Sensor **Sony IMX500** (Raspberry Pi AI Camera) con NPU integrada — pero
  **NO se usa el NPU para esto**: no existe ningún modelo de
  emociones/expresión facial publicado en el model zoo oficial de
  Sony/Raspberry Pi para el formato `.rpk` del sensor (se investigó a
  fondo, ver `ai-camera/TODO-emociones-imx500.txt`). Convertir uno propio
  requeriría el "IMX500 Converter" de Sony, que solo corre en Linux x86_64
  (no en esta Pi ARM).
- Camino elegido: **CPU de la Pi**, con `onnxruntime` (liviano, se descartó
  TensorFlow/Keras por pesado) y dos modelos ONNX ya entrenados (nada se
  entrena acá):
  - `face_detection_yunet_2023mar.onnx` (~230KB) — detector de caras YuNet
    (reemplazo moderno de Haar cascades, que OpenCV 5.x sacó del build).
  - `emotion-ferplus-8.onnx` (~35MB) — clasificador FER+ (ONNX Model Zoo).
    Entrada: cara recortada 64x64 escala de grises. Salida: 8 clases
    (neutral/felicidad/sorpresa/tristeza/enojo/asco/miedo/desprecio),
    de las cuales solo 4 importan para el dataset de trivia:
    `{"felicidad":"Feliz","tristeza":"Triste","enojo":"Enojado","neutral":"Neutral"}`.
- Captura real: `picamera2.Picamera2()`, `create_preview_configuration()`,
  2s de espera para que asiente AE/AWB, hasta 15 intentos de captura+detección
  (0.2s entre intentos) antes de rendirse. **Importante**: hay que llamar
  `picam2.stop()` Y `picam2.close()` — sin el `close()` la cámara queda en
  estado "Configured" y el siguiente `Picamera2()` del mismo proceso falla
  al `acquire()` (bug real ya encontrado y arreglado, ver TODO-mantenimiento.md).
- Código: `ai-camera/reconocer_emocion.py` (detección pura, reusable) +
  `deploy-raspberry-standalone/Clients/Camara_Client.py` (puente perezoso —
  importa `ai-camera/` vía `sys.path` DENTRO de la función, no al tope, para
  que el proceso principal arranque igual en una Pi sin cámara/onnxruntime/
  opencv/picamera2 instalados).
- Benchmark real medido end-to-end de un turno del Juego de emociones (ver
  TODO-emociones-imx500.txt, "ACTUALIZACIÓN 2"):
  - cámara (captura+detección): 8.72s (52% del turno)
  - voz (edge-tts + mpv): 4.78s (28.5%)
  - LLM (reacción, TRIVIA_MODEL): 3.25s (19.4%)
  - TOTAL: ~16.76s
  - Pero en **CPU real consumida** (no reloj de pared) el LLM es el que más
    pesa: una sola reacción del LLM satura >3 de los 4 cores por ~4s
    (~12.66s de CPU), contra <1 core de la cámara incluso en su peor caso.
    Ollama (`llama-server`) es el proceso más pesado del sistema, con
    ~545MB RAM residente todo el tiempo que el modelo esté cargado.
- `ai-camera/` también tiene `snapshot_deteccion.py`, `live_stream.py`,
  `hdmi_live.py` — detección de OBJETOS (no emociones) usando el NPU real
  del IMX500 (modelo COCO SSD MobileNetV2 ya instalado en
  `/usr/share/imx500-models/`). Es un uso distinto del mismo sensor, no
  integrado al flujo del robot hoy.

### 2.3 Pantalla (carita del robot)
- LCD/HDMI conectado directo a la Pi, **headless** (sin sesión gráfica).
- Backend: `mpv --vo=drm` — un solo proceso `mpv --idle` se lanza una vez y
  queda vivo toda la sesión; cambiar de cara es un comando `loadfile` por su
  socket IPC Unix (`--input-ipc-server`), no un relanzamiento.
- Caras: `happy`, `sad`, `angry`, `content`, `speaking`, `countdown` — como
  archivos `.mp4` (**no** `.gif`): se migró de GIF a H.264 porque con
  `--hwdec=auto` mpv decodifica por el `v4l2m2m` de la Pi (zero-copy al
  framebuffer) — medido ~10-23% CPU por hardware vs ~100-137% CPU (más de un
  core) por software con GIF.
- DRM permite un solo "master" de pantalla a la vez: si el proceso anterior
  no cerró limpio (crash, `kill -9`), queda un **mpv huérfano** vivo que
  bloquea a cualquier mpv nuevo — el código detecta esto (conecta al socket
  IPC) y **reusa el huérfano** en vez de competir por la pantalla y quedar
  pegado en la última cara mostrada (bug real, ya resuelto).
- Backend alternativo (demo en PC/Windows, sin Pi real): Tkinter
  (`face_viewer.py`) — proceso separado que se lanza una vez y luego solo
  lee un "archivo de señal" (`.current_face`) cada 150ms para saber qué GIF
  mostrar; reafirma "always on top" cada 1s porque Windows se lo puede sacar
  solo.
- Selección de backend es automática: Linux sin `$DISPLAY` → DRM; cualquier
  otro caso (Windows, o Linux con sesión gráfica) → Tkinter.
- Código: `display.py` (interfaz pública `mostrar_cara(nombre)` / `detener()`,
  agnóstica de backend) + `face_viewer.py` (solo el backend Tkinter).

### 2.4 Voz de salida (lo que dice Lora)
- **3 motores** intercambiables por variable de entorno `VOZ_MOTOR`:
  - `telefono` (**default**): la síntesis la hace el NAVEGADOR del teléfono
    (Web Speech API, `speechSynthesis`) — cero costo de CPU/red en la Pi.
    Requiere que alguien tenga la página abierta y haya tocado "Activar voz"
    (los navegadores móviles exigen un gesto del usuario antes de la primera
    síntesis). Comunicación Pi→teléfono por **SSE** (Server-Sent Events).
  - `piper`: TTS neuronal 100% local en la Pi (ONNX, `onnxruntime`), voz
    default `es_MX-claude-high` (~60MB, mejor relación calidad/velocidad
    medida entre 7 voces probadas — ver la tabla comparativa en
    `Voice_Output_Client.py`). Limitado a 1 hilo (`OMP_NUM_THREADS=1`) para
    dejar 3 cores libres a Ollama — medido: sintetizar tarda IGUAL con 1, 2
    o 4 hilos.
  - `edge`: `edge-tts` (Microsoft, nube), voz `es-AR-ElenaNeural` — necesita
    internet, es el **motor de respaldo** si `telefono` no tiene nadie
    conectado a tiempo (20s timeout) o si `piper` no pudo cargar el `.onnx`.
- `hablar()` **bloquea** hasta terminar de reproducir (vía `mpv`) — el resto
  del turno espera a que Lora termine de hablar antes de seguir.
- Código: `Clients/Voice_Output_Client.py`.

### 2.5 Entrada de voz (lo que dice el usuario)
- **NO hay entrada de voz local en la Pi** (a propósito, por ahora): se
  probó `faster-whisper` (100% local, sin nube) y `WhisperModel(...)` se
  cuelga silenciosamente si se construye fuera del hilo principal — que es
  como corre `voz_server.py` hoy (hilo de fondo de Flask). No se resolvió
  ese bug de threading.
- En su lugar: transcripción en el **navegador del teléfono**
  (`SpeechRecognition`, Web Speech API) — botón de micrófono en la página
  web, manda el texto transcrito al mismo lugar que si se hubiera tipeado.
  Solo funciona bien en Chrome/Android (Safari/iOS nunca lo implementó).
  Exige "contexto seguro" (HTTPS o localhost) — como la Pi sirve HTTP plano
  en una IP de LAN, hay que habilitar un flag de Chrome a mano una vez por
  teléfono.

### 2.6 Página web (servidor de entrada/salida)
- `voz_server.py`: Flask + SSE, corre **en un hilo del mismo proceso**
  Python (no es un servicio aparte que haya que levantar), puerto 8081.
- 3 caminos de entrada de texto confluyen en **la misma cola** que lee
  `Orchestrator_Management.py`: teclado de la Pi, input de texto de la
  página, o voz transcrita en la página — todos hacen `queue.Queue().put()`.
- La página también recibe (SSE, endpoint `/eventos`) el texto que Lora
  tiene que decir, cuando `VOZ_MOTOR=telefono`.

## 3. LLM / Ollama

- Ollama corre local (`http://localhost:11434`), único servicio de red del
  stack. Compatible OpenAI (`/v1/chat/completions`), no la API nativa.
- 3 modelos, todos fine-tunes **LoRA** de `qwen2.5:0.5b` (~397MB cada uno,
  importados a mano con `ollama create`, NO con `ollama pull`):

| Modelo (env var) | Default | Rol |
|---|---|---|
| `CHAT_MODEL` | `lora-chat-libre-v4` | Genera las respuestas de Chat libre |
| `TRIVIA_MODEL` | `lora-trivia` | Reacciones del Juego de emociones (`comentar_resultado_emocion()`) |
| `SALIDA_TRIVIA_MODEL` | `lora-salida-trivia-v2` | Clasifica RESPUESTA vs SALIR a mitad de una pregunta de Trivia |

- `OLLAMA_KEEP_ALIVE=-1` (nunca se descargan de RAM) — se probó con timeout
  finito y la recarga desde disco costaba ~15-28s en el peor caso, muy
  notorio en uso real.
- Se eligió `qwen2.5:0.5b` sobre `llama3.2:3b` tras benchmarquear 7 modelos:
  484MB RAM / 1-3s de respuesta vs 2.5GB / 7-12s, misma calidad medida.
- **El router (Trivia/Chat libre) y el corrector de Trivia YA NO usan LLM**
  — se reemplazaron por clasificadores clásicos por velocidad y precisión:
  - `Agents/Agent_Router.py`: TF-IDF de n-gramas de caracteres + regresión
    logística (`scikit-learn`), modelo serializado en
    `Agents/router_modelo.joblib` (~185KB). ~95-96% held-out. Corre en
    microsegundos, sin tocar Ollama para nada.
  - `Agents/Agent_Corrector.py`: si `respuesta_esperada` es numérica,
    extrae número(s) de la respuesta del usuario y compara matemáticamente;
    si es texto, normaliza (sin tildes/mayúsculas/artículos) + substring +
    solapamiento de palabras clave (umbral 0.6) + fuzzy (`difflib`, umbral
    0.84 por palabra / 0.8 global). 98.4% accuracy medido contra 187
    preguntas reales, 100% recall en incorrectas (nunca acredita mal).
- Cliente: `Clients/Llama_Client.py` — `generar_respuesta()` (streaming,
  soporta `on_token` callback) y `clasificar_salida_trivia()` (sin
  streaming, `max_tokens=5`, `temperature=0`, devuelve `True`/`False`/`None`
  — `None` si Ollama no responde, y ahí el caller cae a listas de palabras
  clave como fallback).

## 4. Datos y memoria (todo en memoria del proceso, sin servidor aparte)

- **Preguntas de Trivia**: `base_datos/preguntas.jsonl` (248 filas), cargado
  entero a un dict en memoria al importar `preguntas.py`. Búsqueda por tema
  o aleatoria, sin BM25 (el BM25 viejo era para el RAG de Chat libre, que se
  eliminó). Esquema de cada fila:
  ```json
  {"id": 1, "pregunta": "...", "cara": "Neutral",
   "respuesta_esperada": "...", "tema": "Arte, música y cultura - Nivel 1",
   "cara_respuesta_buena": "Neutral", "cara_respuesta_mala": "Neutral",
   "musical": "", "desplazamiento": "Adelante"}
  ```
  51 sesiones/temas distintos (`TEMAS_CATALOGO` en `Orchestrator_Management.py`),
  ~5 preguntas por tanda. Conocido: 13 preguntas duplicadas por texto (con
  `id` distinto), sin resolver — ver TODO-mantenimiento.md.
- **Memoria episódica de Chat libre**: `memoria_episodica.py` — BM25 PURO
  (`rank_bm25`, sin embeddings/vector DB, por RAM: ya hay 3 modelos LoRA
  residentes) sobre los últimos 300 turnos de
  `chat_libre_training/conversaciones.jsonl` (el mismo log que ya escribe
  `registro_chat.py`, no un almacén nuevo). Gate de relevancia: **al menos 2
  palabras de CONTENIDO en común** (sin stopwords), no "score > 0" (ese
  umbral viejo enganchaba temas sin relación). Memoria compartida entre
  todos los usuarios, no por persona.
- **Registro de Chat libre**: `registro_chat.py` — cada turno (usuario +
  respuesta + qué modelo respondió) a JSONL, para un pipeline de curación
  posterior (`chat_libre_training/curar.py`) — nunca se entrena directo con
  esto (evita colapso por auto-refuerzo de errores).
- **Personalidad**: `personalidad.py` — Big Five (OCEAN) fijo, arma un
  system prompt (~200-350 tokens) que se cachea una vez por sesión. Se
  omite por completo si el modelo activo ya tiene la personalidad "horneada"
  vía fine-tuning (`lora-personalidad`, `lora-trivia`, `lora-chat`).

## 5. Métricas / observabilidad

- `perf_monitor.py`: decorador `@medir("componente")` (cronometra cada
  llamada) + `medir_bloque()` (context manager, para cuando el nombre del
  componente depende de un argumento, ej. `llama_generar:lora-chat` vs
  `llama_generar:lora-trivia`) → `logs/tiempos.csv`. Más un hilo daemon que
  muestrea CPU%/RAM RSS/threads del proceso cada 5s (vía `psutil`, opcional)
  → `logs/recursos.csv`.
- `perf_report.py`: reporte legible a partir de esos CSV.
- Decisiones reales de la Pi tomadas a partir de estas métricas (ejemplos
  documentados en TODO-mantenimiento.md): gobernador de CPU fijado en
  `performance` (baja latencia ~20%), `num_ctx` de `lora-trivia` bajado de
  4096→512, `OLLAMA_KEEP_ALIVE=-1`, decisión de sacar el LLM del comentario
  de Trivia por completo (frases fijas, `random.choice()`, 0ms).

## 6. Flujo completo de un turno (para entender qué debería ser un nodo)

### Chat libre
1. Llega texto (teclado/web/voz transcrita) → cola compartida.
2. `Agent_Router.enrutar()` clasifica CHAT_LIBRE (sin LLM).
3. `memoria_episodica.buscar_relevante()` busca un recuerdo BM25 relevante.
4. `Llama_Client.generar_respuesta()` con `CHAT_MODEL` (sin system prompt,
   a propósito — el comportamiento sale del fine-tuning, no de instrucciones
   repetidas).
5. `registro_chat.registrar()` guarda el turno.
6. `display.mostrar_cara("speaking")` → `Voice_Output_Client.hablar()`
   (bloquea) → `display.mostrar_cara("content")`.

### Trivia (pregunta con `respuesta_esperada`)
1. Router → TRIVIA. Si es la primera vez, se ofrecen 5 temas al azar del
   catálogo; el usuario elige (`resolver_tema()`, match exacto/difuso, sin
   LLM).
2. `preguntas.py` trae una tanda de 5 preguntas del tema.
3. Se hace la pregunta: `display` → "speaking", texto + voz (bloqueante),
   si es musical bloquea escuchando la canción primero.
4. Llega la respuesta del usuario. Si hay pregunta pendiente:
   - `Agent_Corrector.evaluar_respuesta()` → acierto/error (sin LLM).
   - `Agent_Behavior.elegir_cara_pregunta()` → qué cara mostrar.
   - `comentar_resultado()` → frase fija random (sin LLM desde 2026-08-31).
   - Orden estricto: **voz primero (bloquea)** → recién después cara +
     **`expresar_musica()`** + **`expresar_desplazamiento()` (motores)**
     juntos, en paralelo entre sí (fire-and-forget: música es `Popen`,
     motores es una escritura serial casi instantánea o un hilo aparte para
     "Girar 360°").
5. Se encadena la siguiente pregunta de la tanda, o se cierra con resumen de
   aciertos.

### Trivia (Juego de emociones/imitación — veredicto por cámara)
1. Se elige una cara al azar de las que pide la pregunta.
2. `display.mostrar_cara()` la muestra como referencia visual + se pide por
   voz ("¡Hazme una cara de feliz!").
3. `Camara_Client.detectar_emocion()` captura y clasifica (hasta 15
   intentos).
4. Veredicto por comparación directa (emoción pedida == emoción detectada),
   voz fija ("¡Correcto!"/"¡Incorrecto!") → cara + música + motores en
   paralelo, mismo patrón que arriba.
5. Corre la tanda ENTERA de un tirón (sin volver al loop principal entre
   pregunta y pregunta) — a diferencia de Trivia normal.

### Salir de Trivia a mitad de pregunta
- `_quiere_salir_trivia()`: si el mensaje es corto (≤4 palabras) y no
  matchea ninguna keyword, se asume respuesta directa sin llamar al LLM
  (optimización de latencia). Si no, consulta `SALIDA_TRIVIA_MODEL`
  (RESPUESTA vs SALIR, ve la pregunta pendiente + el mensaje juntos). Si
  Ollama no responde, cae a listas de palabras clave
  (`_SALIR_TRIVIA`/`_TEMA_PERSONAL`).

## 7. Restricciones/criterios de diseño que se repiten en TODO el código

1. **Ningún cliente de hardware/red lanza excepción por falla de
   conectividad** — motores, cámara, música, voz, LLM: si no responden, se
   loguea (`print`) y la conversación sigue. Esto debería mapear a nodos que
   degradan (publican estado de error, no matan el proceso) en vez de nodos
   que crashean y tiran abajo el sistema.
2. **Latencia real medida manda las decisiones de arquitectura** — router y
   corrector se sacaron del LLM por medición real en la Pi, no por
   preferencia. Cualquier capa nueva (tópicos/servicios/DDS) no debería
   introducir latencia perceptible en el loop conversacional.
3. **Orden estricto voz→cara→(música+motores en paralelo)** en cada
   reacción — no es negociable, está documentado como decisión a pedido del
   usuario en varios lados del código.
4. **Un solo "cerebro" de estado de sesión** hoy: el dict `estado{}` dentro
   de `_main_loop()` (pregunta pendiente, tema actual, en_trivia,
   ya_usados, aciertos/total, nombre). Es la máquina de estados central.
5. **3 entradas de texto, 1 sola cola** — cualquier nodo de "entrada" nuevo
   (voz, web, teclado) tiene que converger a un solo punto antes del
   enrutador.
6. Recursos MUY limitados: Pi4 sin GPU, CPU compartida entre Ollama (el
   proceso más pesado, ~3 de 4 cores en un turno de LLM) + síntesis de voz +
   decodificación de video de la cara + detección de cámara. Cualquier nodo
   ROS 2 nuevo (y su propio proceso/DDS overhead) suma a esa contención.
7. Import perezoso para hardware opcional (cámara) — el proceso principal
   tiene que poder arrancar igual sin ese hardware conectado.

## 8. Estructura de carpetas (para ubicar todo)

```
Arquitecture-Agentic-RAG/
├── ai-camera/                        ← cámara IMX500 (detección de objetos NPU + emociones CPU/ONNX)
│   ├── reconocer_emocion.py          ← detección de emoción (YuNet + FER+)
│   ├── snapshot_deteccion.py, live_stream.py, hdmi_live.py  ← detección de objetos (NPU real)
│   ├── modelos/                      ← .onnx (cara+emoción)
│   └── TODO-emociones-imx500.txt     ← historia/decisiones de este subsistema
├── carrito-mecanum-esp32/
│   └── 2-l298n-mecanum/
│       └── mecanum_car_esp32s3.ino   ← firmware de MOTORES (Arduino/C++)
├── base_datos/                       ← (nivel raíz, distinto del de deploy-raspberry-standalone)
└── deploy-raspberry-standalone/      ← TODO el software que corre en la Pi hoy
    ├── Orchestrator_Management.py    ← entrypoint, loop de conversación, máquina de estados
    ├── preguntas.py, personalidad.py, memoria_episodica.py, registro_chat.py
    ├── display.py, face_viewer.py, faces/*.mp4|*.gif
    ├── voz_server.py                 ← Flask+SSE, página web, puerto 8081
    ├── perf_monitor.py, perf_report.py
    ├── Agents/                       ← decisiones sin (o con) LLM
    │   ├── Agent_Router.py (+ router_modelo.joblib)
    │   ├── Agent_Corrector.py
    │   └── Agent_Behavior.py
    ├── Clients/                      ← wrappers hacia hardware/servicios externos
    │   ├── Llama_Client.py           ← HTTP a Ollama
    │   ├── Carrito_Client.py         ← Serial a ESP32 (MOTORES)
    │   ├── Camara_Client.py          ← puente a ai-camera/
    │   ├── Musica_Client.py          ← mpv
    │   └── Voice_Output_Client.py    ← edge-tts / Piper / navegador
    ├── base_datos/preguntas.jsonl    ← 248 preguntas de Trivia
    ├── musica/                       ← audio para la columna "musical"
    └── */training/                   ← pipelines de fine-tuning LoRA (router, corrector no, trivia, chat, salida-trivia, personalidad)
```

## 9. Lo que se le pide a ChatGPT

Diseñar la estructura de paquetes y **nodos ROS 2 Jazyy** para este robot,
de forma que el comportamiento (Trivia/Chat libre, reacciones, máquina de
estados) se modele con los mecanismos propios de ROS 2 (nodos, tópicos,
servicios, acciones, y opcionalmente un behavior tree o state machine para
la orquestación) en vez de un único script con imports directos. Puntos a
resolver explícitamente:

- Qué componentes de la lista de la sección 2-4 conviene que sean **nodos
  ROS 2 separados** (con su propio proceso) vs qué conviene dejar como
  **librería importada** dentro de un nodo más grande (dado que la Pi ya
  está muy ajustada de CPU/RAM con Ollama).
- Cómo modelar el control de **motores** (¿tópico tipo `geometry_msgs/Twist`
  transformado a comandos mecanum dentro del nodo del driver, o un mensaje
  custom con los mismos códigos F/B/SL/SR/RL/RR/S que ya entiende el
  firmware?) — el firmware YA espera ese protocolo de texto por Serial, así
  que el nodo "motores" en ROS 2 sería el que traduce de la interfaz ROS 2
  al mismo protocolo Serial que ya existe (no hace falta tocar el `.ino`).
- Cómo exponer las llamadas a Ollama (¿servicio síncrono, o acción por el
  streaming de `generar_respuesta()`?).
- Cómo mantener el orden estricto voz→cara→(música+motores en paralelo) con
  tópicos/servicios asíncronos sin perder esa garantía.
- Cómo mantener la degradación graciosa (sin cámara/motores/voz conectados,
  el robot sigue funcionando) con el patrón de nodos de ROS 2 (¿nodos
  opcionales, lifecycle nodes, timeouts en servicios?).
- Si conviene un paquete de mensajes/servicios custom (`lora_interfaces` o
  similar) para: comando de cara, comando de motor, resultado de
  cámara/emoción, evento de "reproducir música", veredicto de Trivia, etc.
