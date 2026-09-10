# Socially-Friendly-Robot-Lora-Astro

Migración a **ROS 2 Jazzy** del robot socialmente asistivo (SAR) **Lora**,
originalmente implementado como un proceso Python monolítico en
[`Arquitecture-Agentic-RAG/deploy-raspberry-standalone/`](https://github.com/Adr4563/Arquitecture-Agentic-RAG).

## Contenido de este repo

| Archivo/carpeta | Qué es |
|---|---|
| [`ros2_ws/`](ros2_ws/) | El workspace ROS 2 Jazzy: 4 paquetes (`lora_interfaces`, `lora_drivers`, `lora_brain`, `lora_bringup`). Ver su [README](ros2_ws/README.md) para arquitectura, tópicos/servicios, cómo compilar y correr. |
| [`diagrama-arquitectura-ros2.md`](diagrama-arquitectura-ros2.md) | Diagrama (Mermaid) de los 4 nodos, sus tópicos/servicios y el hardware/servicios externos (Ollama, ESP32, cámara, pantalla, teléfono). |
| [`contexto-ros2-lora.md`](contexto-ros2-lora.md) | Relevamiento técnico completo del proyecto original (hardware, modelos, flujos de turno) que sirvió de base para diseñar la migración. |
| [`enfoque-tdah-tea.md`](enfoque-tdah-tea.md) | Investigación de respaldo: diseño de robots socialmente asistivos (SAR) para jóvenes con TDAH/TEA, y cómo se conecta con las decisiones de diseño de Lora. |

## Resumen rápido

Lora conversa (Chat libre) y juega Trivia por voz/texto, con una carita
animada, música, movimiento (carrito mecanum) y un juego de reconocimiento
de emociones por cámara. La arquitectura ROS 2 separa esto en:

- **`lora_brain`** — el cerebro: `orchestrator_node` (máquina de estados
  de la conversación), router/corrector/comportamiento sin LLM, cliente a
  Ollama, memoria episódica.
- **`lora_drivers`** — 3 nodos de hardware (`LifecycleNode`):
  `communication_node` (comunicación verbal y no verbal), 
  `visualization_faces_node` (cámara → emoción + cara animada), 
  `moving_control_node` (motores vía Serial al ESP32-S3).
- **`lora_interfaces`** — mensajes/servicios custom que conectan los dos
  paquetes de arriba.
- **`lora_bringup`** — launch file + parámetros para levantar todo junto.

Ver [`ros2_ws/README.md`](ros2_ws/README.md) para el detalle completo,
incluyendo la tabla de mapeo módulo-por-módulo contra el proyecto original
y el checklist para compilar/correr en una Raspberry Pi (o WSL2) con
ROS 2 Jazzy real.
