# Enfoque de Lora para jóvenes con TDAH / TEA

> Investigación de respaldo para el diseño de interacción de Lora, un
> robot socialmente asistivo (**SAR — Socially Assistive Robot**)
> implementado sobre una **arquitectura ROS 2 Jazzy** (ver
> [`ros2_ws/`](ros2_ws/) y su [README](ros2_ws/README.md)). Este documento
> conecta la evidencia de la literatura de robótica social con las
> decisiones de diseño que YA tiene el proyecto (algunas por accidente,
# otras a propósito) y señala dónde la arquitectura ROS 2 ayuda o dónde
> todavía falta trabajo.

## 1. Qué dice la investigación

### TEA (Trastorno del Espectro Autista)

- No existen todavía guías de diseño formales y universales para robots
  de intervención con niños con TEA, pero sí criterios de diseño y
  aplicación propuestos por varios grupos de investigación
  ([Criteria for the Design and Application of SAR in Interventions for
  Children with Autism](https://link.springer.com/chapter/10.1007/978-3-030-48989-2_18)).
- Los niños con TEA suelen sentirse atraídos por los robots, pero si el
  objetivo es enseñar habilidades sociales (un déficit central de la
  población), la aceptabilidad social del robot importa tanto como su
  funcionalidad
  ([Designing a SAR for Long-Term In-Home Use for Children with ASD](https://arxiv.org/pdf/2001.09981)).
- La **personalización computacional** es necesaria para que el sistema se
  adapte a las necesidades únicas y cambiantes de cada chico
  ([Long-Term Personalization of an In-Home SAR](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7805891/)).
- Los robots ofrecen un entorno de interacción **consistente y
  predecible**, crucial para chicos con TEA que suelen prosperar con
  rutina y predictibilidad; hay una necesidad específica de
  predictibilidad, apoyo visual y presentación secuencial de la
  información
  ([Utilizing Human–Robot Interaction in Autism Therapy](https://dl.acm.org/doi/10.1145/3776539)).
- El refuerzo sensorial que da el robot (música, movimiento, luces)
  genera reacciones más positivas que el elogio verbal humano solo
  ([Adherence and acceptability of a robot-assisted PRT protocol](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7229010/)).
- Ojo con el otro extremo: los chicos con TEA tienden a rechazar el juego
  cuando notan que el robot NO puede actuar de forma autónoma, o cuando
  el escenario se vuelve DEMASIADO predecible/repetitivo -- la
  predictibilidad ayuda, pero un robot sin ninguna variación ni
  sofisticación pierde el enganche con el tiempo (misma fuente de
  arriba).
- La comunicación afectiva (reconocer y expresar emociones) es un área
  activa de investigación en SAR para TEA
  ([Affective Communication for SARs for Children with ASD: revisión sistemática](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8347754/)),
  y hay trabajo específico en si el aspecto/apariencia del robot cambia
  el reconocimiento de emociones en chicos con TEA
  ([Do different robot appearances change emotion recognition in children with ASD?](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10017775/)).

### TDAH (Trastorno por Déficit de Atención e Hiperactividad)

- Estudiantes con TDAH responden bien a SARs simples que monitorean la
  atención y dan feedback NO verbal
  ([Design and Evaluation of a SAR Schoolwork Companion for College Students with ADHD](https://arxiv.org/html/2401.06289v1)).
- Sistemas concretos ya probados: **Atent@** (asistente robótico + hogar
  inteligente con sensores IoT en silla/escritorio para tareas escolares)
  y **Kip3** (dispositivo con test de rendimiento continuo basado en
  tablet para medir inatención/impulsividad)
  ([Socially Assistive Robotics combined with AI for ADHD](https://www.researchgate.net/publication/349993680_Socially_Assistive_Robotics_combined_with_Artificial_Intelligence_for_ADHD)).
- Las intervenciones tradicionales para TDAH suelen fallar en escalar
  (necesitan mucho recurso humano) y en adaptabilidad en tiempo real; la
  mayoría de las herramientas actuales tiene **contenido fijo que no se
  ajusta** al comportamiento fluctuante del usuario, y les falta
  **interacción multimodal** (voz + texto + visual a la vez)
  ([mismo estudio de arriba](https://arxiv.org/html/2401.06289v1)).

## 2. Cómo se traduce esto a lo que Lora YA tiene

| Hallazgo de la investigación | Dónde ya aparece en Lora |
|---|---|
| Predictibilidad / interacción consistente | `comentar_resultado()` usa frases FIJAS (`random.choice()`, sin LLM) para el veredicto de Trivia -- el robot siempre reacciona con una idea reconocible, no con texto generado distinto cada vez. |
| Refuerzo sensorial > elogio verbal solo | `Agent_Behavior.expresar_musica()`/`expresar_desplazamiento()` -- cada acierto puede disparar música y movimiento del carrito, no solo una frase hablada. |
| Comunicación afectiva / reconocimiento de emociones | El "Juego de emociones"/"Juego de imitación" (`visualization_faces_node`, `DetectEmotion.srv`) -- Lora pide una cara, la cámara verifica, y reacciona -- exactamente el área de "affective communication for SAR" de la literatura. |
| Interacción multimodal (voz + texto + visual) | `communication_node` (texto tipeado, voz transcrita del navegador, voz de salida) + `visualization_faces_node` (cara animada) ya cubren varios canales a la vez. |
| Feedback no verbal / monitoreo simple | La cara en pantalla (`FaceCommand`: happy/sad/angry/content/speaking/countdown) da una señal visual constante del estado del robot, sin depender solo de texto/voz. |
| Rechazo a la sobre-predictibilidad | Hay 4 variantes de saludo/despedida (`SALUDOS_APERTURA`/`DESPEDIDAS`) elegidas al azar -- ya rompe la repetición exacta sesión a sesión. |

## 3. Dónde la arquitectura ROS 2 ayuda (y por qué importa para esta población)

- **Aislamiento de fallas por nodo** (`LifecycleNode` en `lora_drivers`):
  si la cámara falla a mitad del Juego de emociones, solo se cae
  `visualization_faces_node` -- Lora sigue hablando y reaccionando con
  normalidad. Para un chico con TEA, un robot que se "traba" o se apaga
  entero por un error de hardware puntual es exactamente el tipo de
  quiebre de predictibilidad que la literatura marca como negativo. Con
  nodos separados, el quiebre es parcial y contenido.
- **Degradación graciosa end-to-end**: todos los `Client`/driver originales
  (y ahora los servicios ROS2 `Speak`/`PlayMusic`/`DetectEmotion`) ya
  devuelven `False`/`detected=False` en vez de excepción -- el turno de
  conversación nunca se corta por un problema de hardware. Eso es
  consistencia de la interacción, un requisito explícito de la
  investigación.
- **Separación entre la lógica de decisión (`lora_brain`) y el hardware
  (`lora_drivers`)** facilita agregar señales de personalización a futuro
  (ej. ajustar ritmo/duración de Trivia según el chico) sin tocar código
  de cámara/motores/voz -- la personalización computacional es justo lo
  que la literatura marca como necesario y hoy es la parte MENOS
  desarrollada de Lora (la personalidad es fija por sesión, Big Five
  estático, no por chico).

## 4. Brechas abiertas (para donde seguir, no implementado todavía)

1. **Personalización por chico**, no solo por sesión -- hoy
   `personalidad.py` es un único perfil fijo (Big Five estático) y la
   memoria episódica (`memoria_episodica.py`) es compartida entre todos
   los que usaron el robot, no por persona (ya documentado como
   limitación conocida en el proyecto original).
2. **Variación controlada** para no caer en el rechazo por
   sobre-predictibilidad: hoy las reacciones de Trivia son 4 frases fijas
   por resultado -- suficiente variación para el estudio de referencia,
   pero vale la pena medir en uso real si hace falta más.
3. **Feedback de atención tipo TDAH** (Atent@/Kip3): Lora no mide
   señales de atención/impulsividad hoy -- sería un servicio/nodo nuevo
   candidato si se quiere ese caso de uso específico, no algo que ya
   exista.

## Fuentes

- [Designing a Socially Assistive Robot for Long-Term In-Home Use for Children with Autism Spectrum Disorders](https://arxiv.org/pdf/2001.09981)
- [An Approach to the Design of Socially Acceptable Robots for Children with ASD](https://link.springer.com/content/pdf/10.1007/s12369-010-0063-x.pdf)
- [Do different robot appearances change emotion recognition in children with ASD?](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10017775/)
- [Criteria for the Design and Application of Socially Assistive Robots in Interventions for Children with Autism](https://link.springer.com/chapter/10.1007/978-3-030-48989-2_18)
- [Design Path for a Social Robot for Emotional Communication for Children with ASD](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10255993/)
- [Affective Communication for SARs for Children with ASD: A Systematic Review](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8347754/)
- [Long-Term Personalization of an In-Home Socially Assistive Robot for Children With ASD](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7805891/)
- [Toward Socially Assistive Robotics for Augmenting Interventions for Children with ASD](https://link.springer.com/content/pdf/10.1007/978-3-642-00196-3_24.pdf)
- [Design and Evaluation of a Socially Assistive Robot Schoolwork Companion for College Students with ADHD](https://arxiv.org/html/2401.06289v1)
- [Sequencing Matters: Investigating Suitable Action Sequences in Robot-Assisted Autism Therapy](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8959535/)
- [Socially Assistive Robotics combined with Artificial Intelligence for ADHD](https://www.researchgate.net/publication/349993680_Socially_Assistive_Robotics_combined_with_Artificial_Intelligence_for_ADHD)
- [Adherence and acceptability of a robot-assisted Pivotal Response Treatment protocol for children with ASD](https://www.nature.com/articles/s41598-020-65048-3)
- [Utilizing Human–Robot Interaction in Autism Therapy to Enhance Children's Social Skills: Literature Review](https://dl.acm.org/doi/10.1145/3776539)
