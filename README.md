# Taller Práctico #1 — EcoMarket: Optimización de la Atención al Cliente con IA Generativa

Caso de estudio: EcoMarket, empresa de e-commerce de productos sostenibles, recibe
miles de consultas diarias de soporte (chat, correo, redes sociales). El 80% son
repetitivas (estado del pedido, devoluciones, características del producto) y el
20% restante requieren empatía y criterio humano (quejas, problemas técnicos,
sugerencias). El tiempo de respuesta promedio actual es de 24 horas.

Este repositorio contiene la respuesta a las tres fases del taller.

---

## Fase 1: Selección y Justificación del Modelo de IA

### ¿Qué tipo de modelo es el más adecuado?

Se propone una **solución híbrida**: un **LLM de propósito general** (no un
modelo pequeño afinado desde cero), usado mediante **prompting estructurado y
RAG (Retrieval-Augmented Generation)** sobre las bases de datos de EcoMarket
(pedidos, catálogo, envíos, políticas de devolución), combinado con **enrutamiento
a un agente humano** para el 20% de casos complejos.

No se recomienda un fine-tuning completo del modelo por tres razones:

- El catálogo, los pedidos y las políticas de EcoMarket cambian constantemente;
  reentrenar el modelo cada vez que cambian sería costoso y lento.
- El fine-tuning no elimina el riesgo de alucinación; seguiría siendo necesario
  anclar (grounding) las respuestas en datos reales para evitar que el modelo
  invente estados de pedido o políticas.
- Con prompting + RAG se obtiene la mayor parte del beneficio (respuestas
  ajustadas al negocio) a una fracción del costo y con actualización inmediata
  de la información.

### ¿Por qué este enfoque y no otro?

El caso de EcoMarket combina dos necesidades distintas:

- **Precisión** para temas de pedidos y devoluciones: el modelo no puede
  inventar un estado de envío o aprobar una devolución que no cumple las
  reglas. Esto se resuelve **restringiendo el modelo a la información
  proporcionada en el prompt** (RAG) y con reglas explícitas ("no inventes
  información", "si el dato no está disponible, indícalo"), tal como se
  implementó y validó en la Fase 3 de este taller.
- **Fluidez y empatía** para preguntas generales y para redactar respuestas
  claras y amables: esto lo resuelve mejor un LLM de propósito general
  (entrenado con grandes volúmenes de lenguaje natural) que un modelo pequeño
  muy especializado, que tiende a sonar rígido o repetitivo.

Un modelo de propósito general, bien *prompteado* y con acceso controlado a
los datos de EcoMarket, cubre ambas necesidades sin los costos y la rigidez
de un modelo afinado a medida.

### Arquitectura propuesta

```
Cliente (chat / correo / redes)
        │
        ▼
Clasificador de intención (¿consulta repetitiva o caso complejo?)
        │
   ┌────┴─────────────────────────────┐
   │                                   │
Repetitiva (80%)                  Compleja (20%)
   │                                   │
   ▼                                   ▼
RAG: consulta a la base de datos   Se enruta a un agente humano.
de EcoMarket (estado del pedido,   El LLM puede generar un borrador
catálogo, políticas de envío/      de respuesta como apoyo, pero el
devolución)                        agente decide y edita antes de
   │                                enviar (modelo "copiloto", no
   ▼                                autónomo).
LLM genera la respuesta usando
solo los datos recuperados +
reglas de negocio (prompt con
instrucciones estrictas)
   │
   ▼
Respuesta al cliente
```

- **Integración con la base de datos:** sí, es indispensable. El modelo nunca
  responde "de memoria"; siempre recibe en el prompt los datos reales del
  pedido/producto consultado (como se hizo en los Ejercicios 1 y 2 de la
  Fase 3), evitando que el LLM alucine información del negocio.
- **Propósito general vs. afinado:** propósito general, orientado con
  *prompt engineering* y RAG. Se reserva el fine-tuning para una fase futura,
  solo si se detectan patrones de error recurrentes que el prompting no logre
  corregir.

### Justificación (costo, escalabilidad, integración, calidad)

- **Costo:** un LLM de propósito general por API (pago por uso) evita el costo
  fijo y el mantenimiento de entrenar y alojar un modelo propio desde cero.
  Para el prototipo de este taller se usó además un modelo **open-source
  ejecutado localmente (Llama 3.2 vía Ollama)**, sin costo de API, lo que
  demuestra que la misma arquitectura puede escalar hacia abajo (autoalojada,
  costo casi nulo) o hacia arriba (API en la nube, mayor calidad) según el
  presupuesto, sin cambiar el diseño ni el código de integración (la librería
  cliente usada es compatible con OpenAI y con Ollama).
- **Escalabilidad:** una API en la nube escala automáticamente ante picos de
  miles de consultas diarias, algo que un equipo humano no puede hacer sin
  contratar más personal. El enfoque RAG tampoco requiere reentrenar el
  modelo cuando cambian los pedidos o el catálogo, solo actualizar la fuente
  de datos consultada.
- **Facilidad de integración:** los proveedores de LLM (OpenAI, y también
  soluciones locales como Ollama) exponen una API REST estándar, fácil de
  conectar a los canales existentes de EcoMarket (chat, correo, redes
  sociales) sin rediseñar la infraestructura.
- **Calidad de la respuesta esperada:** al anclar las respuestas en datos
  reales y reglas de negocio explícitas, se reduce la alucinación y se
  mantiene la coherencia con el estado real del pedido, como se validó en la
  Fase 3 (el modelo respetó el estado real del pedido en el 80% de los casos
  de prueba, con un modelo pequeño y gratuito; un modelo más grande de pago
  en producción debería acercarse a un porcentaje aún mayor).

---

## Fase 2: Evaluación de Fortalezas, Limitaciones y Riesgos Éticos

### Fortalezas

- **Reducción drástica del tiempo de respuesta:** de un promedio de 24 horas a
  segundos (con API en la nube) o pocos minutos (con modelo local, como se
  observó en las pruebas de la Fase 3).
- **Disponibilidad 24/7**, sin depender de turnos ni horarios de agentes.
- **Maneja el 80% de las consultas repetitivas** (estado de pedido,
  devoluciones, características de producto) sin intervención humana.
- **Consistencia:** aplica siempre las mismas reglas de negocio (por ejemplo,
  las reglas de devolución), a diferencia de agentes humanos que pueden
  interpretar una política de forma distinta según el día o la persona.
- **Libera tiempo de los agentes humanos** para dedicarlo a los casos
  complejos, de mayor valor y que sí requieren empatía real.

### Limitaciones

- **No maneja bien el 20% de casos complejos:** quejas, problemas técnicos o
  sugerencias que requieren negociación, juicio o contención emocional
  genuina. Forzar al modelo a resolver estos casos por sí solo degrada la
  experiencia del cliente.
- **Depende completamente de la calidad de los datos:** si la base de datos
  de EcoMarket tiene un error (por ejemplo, un estado de pedido desactualizado),
  el modelo repetirá ese error con total seguridad aparente.
- **Sin datos, no hay respuesta confiable:** si la información no está en el
  contexto proporcionado, el modelo debe decir explícitamente que no la tiene
  (así se diseñaron los prompts en la Fase 3); si esta regla no se refuerza,
  el riesgo de alucinación aumenta.
- **Calidad variable según el modelo usado:** un modelo pequeño y gratuito
  (como el usado en el prototipo local de este taller) puede ser menos fluido
  o cometer más errores de formato que un modelo grande de pago; esto es un
  trade-off directo entre costo y calidad que EcoMarket debe decidir según su
  presupuesto.
- **Latencia variable:** un modelo autoalojado en hardware modesto (CPU, sin
  GPU) responde notablemente más lento que una API en la nube, lo que puede
  no ser aceptable para un chat en tiempo real a gran escala.

### Riesgos éticos

- **Alucinaciones:** el modelo podría inventar un estado de pedido, una
  política de devolución o una característica de producto que no existe.
  *Mitigación:* anclar siempre las respuestas a datos reales (RAG), con
  reglas explícitas en el prompt ("no inventes información", "si el dato no
  está disponible, indícalo claramente"), y medir sistemáticamente (como se
  hizo en la Fase 3) qué porcentaje de respuestas respeta el estado real.
- **Sesgo:** el modelo podría dar un trato distinto según el nombre, la forma
  de escribir o el origen del cliente, reflejando sesgos presentes en los
  datos con los que fue entrenado. *Mitigación:* pruebas de equidad con
  perfiles de clientes diversos, reglas de negocio que no dependan de
  atributos personales, y monitoreo continuo de las respuestas generadas.
- **Privacidad de datos:** los prompts incluyen información sensible del
  cliente (dirección, historial de compras, contacto). *Riesgos:* que un
  proveedor externo almacene o use esos datos para entrenar sus modelos, o
  que se filtren en logs. *Mitigación:* enviar solo los datos mínimos
  necesarios para responder la consulta puntual, revisar la política de
  retención de datos del proveedor de IA, y considerar (como se probó en este
  taller con Ollama) un modelo **autoalojado** para los datos más sensibles,
  cumpliendo la normativa de protección de datos aplicable.
- **Impacto laboral:** existe el riesgo de que la automatización se use para
  reducir personal de soporte. *Postura recomendada:* usar la IA para
  **empoderar, no reemplazar** — absorber el volumen repetitivo y de bajo
  valor, liberando a los agentes humanos para los casos complejos, la mejora
  continua del sistema (revisar y corregir respuestas del modelo) y el trato
  con los clientes que más lo necesitan.

---

## Fase 3: Aplicación de la Ingeniería de Prompts

Implementación práctica en el notebook [`Script/Taller_1_EcoMarket_Olist.ipynb`](Script/Taller_1_EcoMarket_Olist.ipynb),
usando datos reales del dataset público de [Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
como base de pruebas.

### Qué contiene

- **Ejercicio 1 — Consulta de estado de pedido:** prompt que responde el
  estado de un pedido a partir de una base de datos de 10 pedidos reales,
  sin inventar información.
- **Ejercicio 2 — Devolución de producto:** prompt que determina si un
  producto puede devolverse según reglas de negocio explícitas (productos
  defectuosos/dañados sí, higiene personal abierta no, etc.), con una
  respuesta clara y empática.
- Evaluación cuantitativa de ambos ejercicios, con resultados guardados en
  `Salida/` (`Resultados_ModeloLocal_Ejercicio_1.xlsx` y
  `Resultados_ModeloLocal_Ejercicio_2.xlsx`).

### Modelo usado

Por indicación del taller ("para este ejercicio pueden usar un modelo
open-source sin ningún problema"), el notebook usa un modelo **open-source
ejecutado 100% en local con [Ollama](https://ollama.com/)** (`llama3.2`), sin
necesidad de clave de pago. La arquitectura es intercambiable con un modelo
en la nube (OpenAI u otro compatible) cambiando únicamente `base_url` y
`api_key` en la celda de configuración del cliente, ya que se usa la misma
librería cliente (`openai`) en ambos casos.

### Cómo ejecutarlo

1. Instala [Ollama](https://ollama.com/download) y descarga el modelo:
   ```
   ollama pull llama3.2
   ```
2. Instala las dependencias de Python: `pip install openai pandas openpyxl matplotlib`.
3. Descarga el dataset de Olist y ubícalo en `Entrada/archive/` (ver rutas
   configuradas al inicio del notebook).
4. Abre `Script/Taller_1_EcoMarket_Olist.ipynb` y ejecuta las celdas en orden.
