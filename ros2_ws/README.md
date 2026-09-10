# Lora en ROS 2 Jazzy

Migración del robot Lora desde la arquitectura monolítica original
(`Arquitecture-Agentic-RAG/deploy-raspberry-standalone/`, un solo proceso
Python) a 4 paquetes ROS 2 Jazzy. Ver el plan completo de la migración en
`C:\Users\user\.claude\plans\giggly-wandering-moth.md` y el contexto de
hardware/software en
`Socially-Friendly-Robot-Lora-Astro/contexto-ros2-lora.md`.

## Estado de este workspace

Escrito y verificado en una máquina Windows **sin ROS 2 instalado** — se
compiló la sintaxis de cada `.py` con `python -m py_compile` (pasa) y se
revisó a mano que cada paquete tiene la estructura que exige `colcon`
(`package.xml`, `setup.py`/`CMakeLists.txt`, `resource/<paquete>`,
`entry_points`), pero **nadie corrió `colcon build` ni `ros2 launch`
todavía**. Antes de darlo por andando de verdad hay que hacer el checklist
de la sección "Cómo probarlo" en la Raspberry Pi (o WSL2 con ROS 2 Jazzy).

## Paquetes

```
src/
├── lora_interfaces/    mensajes y servicios (ament_cmake)
├── lora_drivers/       3 LifecycleNode de hardware (ament_python)
│   ├── communication_node          comunicación verbal y no verbal
│   ├── visualization_faces_node    cámara -> emoción + display de la cara
│   └── moving_control_node          motores (Serial -> ESP32-S3)
├── lora_brain/          orchestrator_node, el "cerebro" (ament_python)
└── lora_bringup/         launch + parámetros (ament_cmake -- ver nota abajo)
```

> Nota: el plan original decía `ament_python` para `lora_bringup`; se
> implementó como `ament_cmake` porque el paquete no tiene ningún nodo
> Python propio, solo `launch/` y `config/` — es el patrón más común en
> paquetes ROS 2 de solo-bringup. No cambia nada del comportamiento ni de
> cómo se invoca (`ros2 launch lora_bringup lora_bringup.launch.py` es
> idéntico en los dos casos).

## Tópicos y servicios (`lora_interfaces`)

| Nombre | Tipo | De -> a |
|---|---|---|
| `/lora/user_input` | topic `UserInput` | communication_node -> orchestrator_node |
| `/lora/face_command` | topic `FaceCommand` | orchestrator_node -> visualization_faces_node |
| `/lora/motion_command` | topic `MotionCommand` | orchestrator_node -> moving_control_node |
| `/lora/speak` | service `Speak` | orchestrator_node -> communication_node (bloquea) |
| `/lora/play_music` | service `PlayMusic` | orchestrator_node -> communication_node |
| `/lora/detect_emotion` | service `DetectEmotion` | orchestrator_node -> visualization_faces_node (bloquea, ~8-9s peor caso) |
| `/lora/is_voice_client_connected` | service `IsVoiceClientConnected` | orchestrator_node -> communication_node |

`clasificar_salida_trivia()` y todo lo de Ollama (`llama_client.py`) se
quedan como HTTP directo DENTRO de `lora_brain` -- no hay ningún tópico ni
servicio para eso, mismo criterio que el proyecto original (no es
hardware).

## Mapeo: módulo original -> archivo nuevo

| Original (`deploy-raspberry-standalone/`) | Nuevo | Paquete |
|---|---|---|
| `Orchestrator_Management.py` | `orchestrator_node.py` | `lora_brain` |
| `Agents/Agent_Router.py` | `agent_router.py` | `lora_brain` |
| `Agents/Agent_Corrector.py` | `agent_corrector.py` | `lora_brain` |
| `Agents/Agent_Behavior.py` | `agent_behavior.py` | `lora_brain` (ya no llama hardware directo, recibe un `bridge`) |
| `Clients/Llama_Client.py` | `llama_client.py` | `lora_brain` |
| `personalidad.py`, `memoria_episodica.py`, `registro_chat.py`, `preguntas.py`, `perf_monitor.py` | igual nombre | `lora_brain` |
| — (nuevo) | `_ros_bridge.py` | `lora_brain` (publishers/service-clients hacia `lora_drivers`) |
| `voz_server.py` | `_web_bridge.py` | `lora_drivers` (`communication_node`) |
| `Clients/Voice_Output_Client.py` | `_voice_backends.py` | `lora_drivers` (`communication_node`) |
| `Clients/Musica_Client.py` | `_music_player.py` | `lora_drivers` (`communication_node`) |
| `display.py` + `face_viewer.py` | `_face_display.py` + `face_viewer.py` | `lora_drivers` (`visualization_faces_node`) |
| `Clients/Camara_Client.py` + `ai-camera/reconocer_emocion.py` | `_emotion_detector.py` | `lora_drivers` (`visualization_faces_node`) |
| `Clients/Carrito_Client.py` | `_cart_serial.py` | `lora_drivers` (`moving_control_node`) |
| `carrito-mecanum-esp32/.../mecanum_car_esp32s3.ino` | **sin cambios** | (no es parte de este workspace) |

**No se portaron** (siguen siendo herramientas de entrenamiento, no forman
parte del runtime del robot): `router_training/`, `chat_training/`,
`trivia_training/`, `salida_trivia_training/`, `personalidad_training/`,
`chat_libre_training/`, `excel_a_jsonl.py`, `perf_report.py`. Se siguen
corriendo desde `deploy-raspberry-standalone/` como hasta ahora.

## Decisiones de diseño importantes

- **`Speak` es un servicio síncrono, no una `action`**: se acepta perder
  cancelación/streaming a cambio de simplicidad -- mismo trade-off que ya
  tenía `voz_output.hablar()` en el original (bloqueaba, sin streaming
  real desde que se sacó el print token a token).
- **Garantía de orden voz -> cara -> (música + motores en paralelo)**:
  documentada explícitamente en `orchestrator_node.py::_reaccionar_veredicto()`
  -- depende 100% de que ese método llame a los servicios/tópicos en ese
  orden y espere la respuesta de `Speak` antes de disparar los otros dos.
- **Llamar un servicio ROS2 desde dentro de un callback sin deadlockear**:
  el patrón completo (por qué `client.call()` deadlockea con un
  `SingleThreadedExecutor`, por qué la solución es `call_async()` + espera
  activa corta + `MultiThreadedExecutor`) está documentado en
  `lora_brain/_ros_bridge.py::_llamar_servicio_sync()` -- es el punto más
  fácil de romper si se toca este código sin conocer el porqué.
- **"salir" ya no apaga el proceso**: en el original, decir "salir"
  terminaba el script (era una herramienta de prueba en terminal). Acá
  termina la SESIÓN (despedida) y arranca una nueva -- un robot desplegado
  tiene que seguir listo para el próximo usuario. Ver la nota en
  `orchestrator_node.py::_finalizar_sesion()`.
- **`LifecycleNode` en los 3 drivers**: `on_configure` prepara objetos sin
  tocar hardware; `on_activate` recién ahí abre el Serial / arranca Flask /
  muestra la primera cara. Igual criterio de "falla gracioso" del proyecto
  original (sin cámara/carrito/mpv conectado, el nodo sigue vivo y
  responde con `detected=False`/`success=False`, nunca una excepción sin
  atrapar).

## Cómo probarlo (checklist para la Pi / WSL2 con ROS 2 Jazzy)

```bash
# 1. Dependencias de sistema (además de lo que resuelva rosdep):
cd ros2_ws
pip install -r src/lora_drivers/requirements.txt
pip install -r src/lora_brain/requirements.txt
# mpv tiene que estar instalado aparte: sudo apt install mpv

# 2. Compilar
colcon build --symlink-install

# 3. Source
source install/setup.bash

# 4. Ollama tiene que estar arriba y los 3 modelos importados -- igual
#    que el README del proyecto original (ollama create lora-chat-libre-v4,
#    lora-trivia, lora-salida-trivia-v2).

# 5. Levantar todo
ros2 launch lora_bringup lora_bringup.launch.py
# variantes: ros2 launch lora_bringup lora_bringup.launch.py voz_motor:=edge
#            ros2 launch lora_bringup lora_bringup.launch.py carrito_port:=/dev/ttyUSB0
```

Verificación básica una vez arriba:

```bash
ros2 node list
# communication_node, visualization_faces_node, moving_control_node, orchestrator_node

ros2 lifecycle get /communication_node
# debería decir "active" a los pocos segundos de arrancar

ros2 topic echo /lora/user_input
# escribir en la terminal donde corre communication_node (stdin) o abrir
# http://<ip>:8081/ y mandar texto -- tiene que aparecer acá

ros2 service call /lora/is_voice_client_connected lora_interfaces/srv/IsVoiceClientConnected
```

Y un smoke test end-to-end: escribir el nombre cuando Lora saluda, pedir
Trivia, elegir un tema, contestar una pregunta con `respuesta_esperada`
numérica (ej. una de multiplicación) y confirmar que dice el veredicto,
cambia la cara y (si hay carrito/música configurados en esa pregunta) se
mueve/suena.

## Limitaciones conocidas (heredadas o nuevas de la migración)

- `ROTATE360` sigue sin calibrar contra el hardware real (6 pulsos de
  `RR` con 0.4s de pausa, aproximado) -- igual que en el original.
- No hay entrada de voz local en la Pi (sigue siendo Web Speech API del
  navegador) -- mismo motivo que el original (bug de threading con
  `faster-whisper` nunca resuelto).
- El streaming token-a-token de Ollama existe en `llama_client.py`
  (`on_token`) pero no se expone por ROS2 -- `Speak.srv` es
  request/response completo. Mismo estado que el original desde que se
  sacó el streaming a consola.
- `voz_motor` está declarado como parámetro en DOS nodos
  (`communication_node` y `orchestrator_node`) y tienen que coincidir a
  mano -- mismo trade-off ya documentado en `personalidad.py` del proyecto
  original sobre duplicar un default entre dos módulos en vez de acoplarlos.
- Nada de esto se probó contra hardware real ni contra un ROS 2 Jazzy real
  todavía (ver "Estado de este workspace" arriba) -- la sintaxis está
  verificada, el comportamiento no.
