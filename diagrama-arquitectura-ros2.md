# Diagrama de arquitectura — Lora sobre ROS 2 Jazzy

Ver el detalle completo de cada nodo, tópico y servicio en
[`ros2_ws/README.md`](ros2_ws/README.md) y el porqué de cada decisión en
[`contexto-ros2-lora.md`](contexto-ros2-lora.md). Este diagrama es el mapa
visual de esos mismos 4 paquetes.

```mermaid
flowchart TB
    subgraph EXT["Servicios / hardware externos"]
        OLLAMA[("Ollama (local)\n3 modelos LoRA qwen2.5:0.5b")]
        ESP32["ESP32-S3\ncarrito mecanum\n(firmware sin tocar)"]
        CAM["Cámara IMX500\n(picamera2)"]
        PANT["Pantalla LCD/HDMI\n(mpv --vo=drm)"]
        TEL["Navegador del teléfono\n(Web Speech API)"]
        EDGE[("edge-tts\n(nube, Microsoft)")]
    end

    subgraph BRAIN["lora_brain — el cerebro"]
        ORCH["orchestrator_node\nmáquina de estados\nTrivia / Chat libre"]
        ROUTER["agent_router.py\nTF-IDF + regresión log.\n(sin LLM)"]
        CORR["agent_corrector.py\nnormaliza+fuzzy/numérico\n(sin LLM)"]
        BEH["agent_behavior.py\ncara/música/motor\na partir del dataset"]
        LLM["llama_client.py\nHTTP directo"]
        MEM["memoria_episodica.py\nBM25 sobre chat libre"]
        PREG["preguntas.py\n248 preguntas en memoria"]
        BRIDGE["_ros_bridge.py\npublishers + service clients"]
        ORCH --- ROUTER
        ORCH --- CORR
        ORCH --- BEH
        ORCH --- MEM
        ORCH --- PREG
        ORCH --- BRIDGE
        ORCH -->|HTTP /v1/chat/completions| LLM
    end

    subgraph DRIVERS["lora_drivers — 3 LifecycleNode de hardware"]
        COMM["communication_node\ncomunicación verbal y no verbal"]
        VIS["visualization_faces_node\ncámara → emoción + cara animada"]
        MOV["moving_control_node\nSerial → protocolo del ESP32"]
    end

    LLM -->|HTTP| OLLAMA

    BRIDGE -->|topic /lora/face_command| VIS
    BRIDGE -->|topic /lora/motion_command| MOV
    BRIDGE -->|srv /lora/speak (bloquea)| COMM
    BRIDGE -->|srv /lora/play_music| COMM
    BRIDGE -->|srv /lora/detect_emotion (bloquea)| VIS
    BRIDGE -->|srv /lora/is_voice_client_connected| COMM
    COMM -->|topic /lora/user_input| ORCH

    COMM <-->|SSE + HTTP :8081| TEL
    COMM --> EDGE
    COMM -.->|stdin| ORCH

    VIS --> CAM
    VIS --> PANT

    MOV -->|Serial USB, 115200 baud\nF/B/SL/SR/RL/RR/S| ESP32

    classDef ext fill:#f5f5f5,stroke:#999,color:#333;
    classDef brain fill:#e8f0fe,stroke:#4285f4,color:#1a1a1a;
    classDef drivers fill:#fef3e0,stroke:#f9a825,color:#1a1a1a;
    class OLLAMA,ESP32,CAM,PANT,TEL,EDGE ext;
    class ORCH,ROUTER,CORR,BEH,LLM,MEM,PREG,BRIDGE brain;
    class COMM,VIS,MOV drivers;
```

## Lectura rápida

- **Azul (`lora_brain`)**: toda la lógica de decisión — sin conocimiento
  directo de hardware, solo tópicos/servicios hacia `lora_drivers` y HTTP
  directo hacia Ollama (no es hardware, así que no pasa por ROS 2).
- **Naranja (`lora_drivers`)**: los 3 `LifecycleNode` que sí tocan
  hardware/red externa — cada uno aislado en su propio proceso (ver la
  nota de "aislamiento de fallas" en
  [`enfoque-tdah-tea.md`](enfoque-tdah-tea.md)).
- **Gris**: todo lo que vive FUERA de este workspace ROS 2 — Ollama, el
  firmware del ESP32 (`carrito-mecanum-esp32/`, sin tocar), la cámara
  física, la pantalla, el navegador del teléfono y el servicio en la nube
  de síntesis de voz.
- La única flecha que **bloquea** el turno de conversación de punta a
  punta es `/lora/speak` — es la garantía de orden voz → cara →
  (música + motor) documentada en
  `orchestrator_node.py::_reaccionar_veredicto()`.
