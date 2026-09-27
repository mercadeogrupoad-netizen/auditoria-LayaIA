# Plan de implementación: cómo ejecutar y sacar provecho a Laya

Versión de referencia: Laya 0.3.20 (septiembre de 2026). Complementa el documento `01-auditoria-ciberseguridad-laya.md`.

---

## 1. Qué es y para qué sirve (y para qué no)

Laya es un motor de decisiones "Sistema 1": un codificador tipo BERT que responde preguntas cerradas sobre un texto en una sola pasada, sin generar texto. Recibe un **estado** (texto plano, JSON, correo, ticket) y un diccionario de **preguntas tipadas**, y devuelve por cada pregunta:

| Tipo | Devuelve | Ejemplo |
|---|---|---|
| `choice` | La opción elegida, probabilidad por opción y confianza | ¿Qué departamento atiende esto? billing / técnico / ventas |
| `score` | Un nivel esperado en una escala ordinal | ¿Qué tan urgente es? 0 = no urgente, 1 = pronto, 2 = bloqueante |
| `noul` | Probabilidad calibrada de que una afirmación sea cierta | ¿El cliente amenaza con cancelar? 0,89 |

Tres checkpoints y un `Router` que elige entre ellos según el idioma detectado:

| Checkpoint | Codificador | Parámetros | Contexto | Para |
|---|---|---|---|---|
| `laya` (english) | ModernBERT-large | 421 M | 512 tokens | Inglés |
| `laya-multilingual` | mmBERT-base | 322 M | 1.024 tokens (hasta 8.192) | 100+ idiomas, incluido español. 2 veces más rápido |
| `laya-typed-decisions` | ModernBERT-large | 421 M | 1.024 tokens | Flujos de trabajo afinados (facturación, incidentes, soporte, trazas de agentes) |

**Sirve para:** clasificar y priorizar tickets o correos, enrutar solicitudes al equipo o al modelo correcto, moderación de contenido, detección de jailbreaks antes de un LLM, extracción de campos estructurados (esquema JSON o pydantic), decisiones dentro de flujos LangGraph, juez local para agentes, cualquier decisión repetitiva con opciones cerradas.

**No sirve para:** chatear, resumir, redactar, traducir, responder preguntas abiertas, programar, razonar en varios pasos. No es un sustituto de ChatGPT/Claude/Gemini. Es el complemento rápido y barato que toma decisiones cerradas para que el LLM grande solo intervenga cuando haga falta.

**Expectativa realista de precisión:** los checkpoints base zero-shot son buenos en tareas cercanas a su entrenamiento (AG News 0,95, XNLI inglés 0,86, intención MASSIVE inglés 0,78) y flojos en tareas nuevas (typed-decisions 0,36, casi azar). El salto grande viene del fine-tuning con tus propios datos (0,766 en el mismo benchmark). Planifica tiempo para evaluar con datos propios antes de confiar en él.

---

## 2. Requisitos técnicos de tu computador

### 2.1 Mínimo para probar en local (CPU)

| Recurso | Requisito | Comentario |
|---|---|---|
| Sistema operativo | Linux, macOS o Windows 10/11 | El CI prueba Linux y Windows. En Windows conviene activar el "Modo desarrollador" o definir `HF_HUB_DISABLE_SYMLINKS=1` para la caché de Hugging Face |
| Python | 3.10, 3.11, 3.12 o 3.13 | `huggingface_hub` 1.x, `transformers` 5.x y `torch` 2.14 exigen 3.10 como mínimo. 3.11 es la opción más probada |
| RAM | 8 GB | Es lo que pide la guía Docker para el quickstart CPU. Cada checkpoint ocupa aproximadamente 1,3 a 1,7 GB en fp32 en memoria; el `Router` mantiene dos residentes por defecto |
| Disco | 10 GB libres | PyTorch CPU (~1 GB), transformers y dependencias (~1 GB), pesos (del orden de 1 a 3 GB según cuántos checkpoints descargues), caché |
| CPU | 4 núcleos físicos | Los benchmarks del proyecto en un AMD EPYC de 4 núcleos dan 193 ms por pregunta en `multilingual` y 580 ms en `english`. Con 2 núcleos funciona, pero más lento |
| Internet | Solo para instalar e hacer la primera descarga | Después puede correr sin conexión |

### 2.2 Recomendado para uso cómodo y para desarrollo

| Recurso | Recomendación |
|---|---|
| RAM | 16 GB (el pico medido con cinco checkpoints cargados fue 9,3 GiB) |
| CPU | 6 a 8 núcleos físicos. Fijar `LAYA_THREADS` al número de núcleos físicos y los hilos inter-op a 1: en un portátil Ryzen 9 pasó de 9.396 ms a 783 ms por llamada solo con eso |
| GPU (opcional) | NVIDIA con 6 a 8 GB de VRAM para un checkpoint, 12 GB para dos, 24 GB para los tres precargados con margen. Una T4 (16 GB) responde una pregunta en 33 ms. Apple Silicon (MPS) e Intel Arc (XPU) también están soportados; AMD ROCm no está validado por el proyecto |
| Disco SSD | 20 GB si instalas la variante CUDA de PyTorch (varios GB adicionales) |

### 2.3 Rendimiento esperado por tipo de máquina (datos medidos por el proyecto)

| Máquina | 1 pregunta | 10 preguntas en una llamada | Comentario |
|---|---|---|---|
| CPU servidor 4 núcleos, `multilingual` | 193 ms | 1,8 s | Escala casi lineal con el número de preguntas |
| CPU servidor 4 núcleos, `english` | 580 ms | 6,2 s | 3 veces más lento que multilingual |
| Portátil Ryzen 9 6900HX, 8 hilos | 329 ms | ~2,6 s | Con hilos bien configurados |
| Intel Arc B390 (XPU) | 30 ms | 97 ms | |
| NVIDIA T4 | 33 a 40 ms | 72 a 159 ms | 103 a 332 preguntas/s en lote |
| RTX 5060 Ti en lote | ~1 ms por decisión | | Con `predict_batch` |
| Apple GPU, documento de 4.000 tokens | 1,7 s | | Con `max_len=8192` |

Carga en frío de un checkpoint: 2,5 a 4,4 s en CPU, 7 a 10 s si el `Router` tiene que reconstruir uno expulsado. Precargar (`Router(preload=True)`) elimina ese costo en servidores.

### 2.4 Software adicional según lo que quieras hacer

| Objetivo | Instalar |
|---|---|
| Solo SDK y CLI | `pip install laya` |
| Interfaz web local y API JSON | `pip install "laya[serve]"` |
| Exponerlo como herramienta a Claude Desktop, Cursor u otro cliente MCP | `pip install "laya[mcp]"` |
| Usarlo dentro de LangChain/LangGraph | `pip install "laya[langchain]"` |
| Esquemas JSON / pydantic | `pip install "laya[structured]"` |
| Contenedores | Docker Engine o Docker Desktop con Compose v2 (y NVIDIA Container Toolkit para GPU) |
| Versión TypeScript/Node | Node.js 18+ y `onnxruntime-node` |
| Fine-tuning | GPU de 16 GB, o Kaggle gratuito (2x T4, 4 a 5 horas por corrida) |

---

## 3. Necesidades de servidor (si lo despliegas para un equipo)

### 3.1 Dimensionamiento

Regla práctica: Laya en CPU procesa aproximadamente 5 preguntas/s por cada 4 núcleos con `multilingual`, y 1,7 preguntas/s con `english`. En GPU T4 procesa 30 preguntas/s una a una y 100 a 300 en lote.

| Carga esperada | Opción de servidor | Notas |
|---|---|---|
| Hasta 10.000 decisiones al día, uso interno | VM de 4 vCPU físicos y 16 GB (equivalente AWS m7a.xlarge, ~0,23 USD/h) | Sin GPU. Latencia 0,2 a 0,6 s por pregunta |
| 50.000 a 500.000 decisiones al día, o latencia < 100 ms | VM con GPU T4 o L4 (AWS g4dn.xlarge / g6.xlarge), 16 GB RAM | Precargar los tres checkpoints cabe en 16 GB de VRAM |
| Millones de decisiones al día | GPU tipo L4/A10 con `predict_batch`, o varias réplicas detrás de un balanceador | Cada réplica es independiente; no hay estado compartido |
| Sin salida a internet (entorno cerrado) | Cualquiera de las anteriores más un espejo de los pesos (`LAYA_MODEL_PATH` o volumen con la caché) y `HF_HUB_OFFLINE=1` | Soportado nativamente |

### 3.2 Arquitectura recomendada

```
Clientes ──HTTPS──► Proxy inverso (nginx/Caddy/Traefik: TLS, rate limit)
                          │
                          ▼
                    laya-serve (contenedor oficial, usuario no root)
                    LAYA_API_KEY_FILE, LAYA_PRELOAD=1, LAYA_HOST=0.0.0.0 solo dentro de la red del contenedor
                          │
                          ▼
                    Volumen con caché de pesos (HF_HUB_OFFLINE=1 tras la primera descarga)
```

Elementos:

- Un contenedor por réplica, construido con el `Dockerfile` del repositorio (`compose.yaml` + `compose.http.yaml`, y `compose.cuda.yaml` para GPU).
- Secretos por archivo, nunca en variables de entorno de texto plano.
- Salud: `GET /health` responde cuando los checkpoints están cargados; el compose ya define `healthcheck`.
- Observabilidad: hooks `on_predict_end` para métricas (ejemplo `examples/hooks/otel.py`) y auditoría (`audit.py`), enviadas a tu propio sistema.
- Logs: uvicorn registra IP y ruta. Los cuerpos no se registran. Si necesitas trazabilidad, el hook de auditoría te da `run_id`, modelo, respuestas y latencia.

### 3.3 Alternativas de despliegue

- **NixOS:** módulo `services.laya-serve` con systemd endurecido, incluido en el repositorio.
- **Sin servidor propio:** la Space pública de Hugging Face es solo demo; no la uses con datos reales.
- **Ejecución en el navegador:** `laya-ts` con ONNX Runtime Web permite correrlo en el cliente sin servidor, útil para prototipos.

---

## 4. Plan de trabajo por fases

### Fase 0: Definir el caso de uso (1 a 2 días)

Entregable: una hoja con las preguntas exactas, sus opciones y 50 a 200 ejemplos etiquetados a mano.

Preguntas clave que debes responder antes de escribir código:

1. ¿Qué decisión repetitiva quiero automatizar? ¿Puedo expresarla como `choice`, `score` o `noul`?
2. ¿En qué idioma llegan los textos? Si es mayoritariamente español, usa `Router(default="multilingual")` o fija `model="multilingual"`.
3. ¿Cuántas opciones tiene cada `choice`? Si son más de 20, necesitas `predict_shortlist` o dividir en pregunta gruesa y fina.
4. ¿Qué longitud tienen los textos? Hasta ~4.000 tokens funciona bien en `multilingual` con `max_len=8192`; más allá conviene `predict_long`.
5. ¿Qué pasa si se equivoca? Define desde el inicio el umbral de confianza y qué ocurre por debajo (escalar a humano, enviar a un LLM grande).
6. ¿Tengo datos etiquetados? Sin al menos algunos cientos de ejemplos no podrás medir ni calibrar.
7. ¿Los textos contienen datos personales? Si sí, planifica el hook de redacción y el tratamiento del log.
8. ¿Volumen diario y latencia aceptable? Determina CPU o GPU (sección 3.1).
9. ¿Va a integrarse con un LLM en la nube vía MCP? Si sí, acepta que ese LLM verá los datos.

### Fase 1: Prueba de concepto local en CPU (medio día)

```bash
python3 -m venv .venv
source .venv/bin/activate            # Windows: .\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install torch --index-url https://download.pytorch.org/whl/cpu   # CPU; omitir para dejar que pip elija
python -m pip install "laya[serve]"
python -I -c "import laya; print(laya.__version__)"

# Sin descargar nada: solo la decisión de enrutado
laya "Me cobraron dos veces, por favor reembolsen"

# Descarga el checkpoint multilingüe (una sola vez) y responde
laya "Me cobraron dos veces, por favor reembolsen" --predict --model multilingual --json
laya "Mi pago falló dos veces" --preset triage --model multilingual
```

Después, en Python:

```python
from laya import Router

router = Router(default="multilingual")
estado = {"asunto": "Cobro duplicado", "cuerpo": "Nos cobraron dos veces en marzo. Reembolsen hoy o cancelamos."}
preguntas = {
    "departamento": {"type": "choice", "instructions": "¿Qué departamento debe atender esto?",
                     "criteria": {"facturacion": "facturas, pagos, reembolsos",
                                  "tecnico": "errores, caídas, fallos del sistema",
                                  "otro": "todo lo demás"}},
    "urgencia": {"type": "score", "instructions": "¿Qué tan urgente es?",
                 "criteria": ["no urgente", "pronto", "bloqueante"]},
    "riesgo_baja": {"type": "noul", "instructions": "¿El cliente amenaza con cancelar?"},
}
r = router.predict(estado, preguntas)
print(r["answers"], r["routing"])
```

Verificación de privacidad: tras la primera descarga, repite con `HF_HUB_OFFLINE=1` y sin red. Debe funcionar igual.

Interfaz web para experimentar: `python examples/server.py` y abrir `http://127.0.0.1:8000`.

### Fase 2: Evaluación con tus datos (1 semana)

1. Prepara un archivo JSONL con tus ejemplos etiquetados (formato en `research/evals/dataset.template.jsonl`).
2. Valida y corre el harness:

```bash
laya-evals validate mis_datos.jsonl
laya-evals run mis_datos.jsonl --model multilingual --slice language --json informe.json
```

3. Mide precisión por pregunta, ECE (calibración) y latencia. Compara `english` vs `multilingual` vs `typed-decisions` en tus datos, no en los del README.
4. Ajusta las temperaturas de calibración con datos propios (el notebook de fine-tuning incluye el bucle). Sin esto, la confianza reportada no es fiable para automatizar.
5. Decide el umbral: la confianza ordena decisiones, no garantiza que sean correctas. Elige el punto donde los errores que aceptas son tolerables.
6. Si la precisión zero-shot no basta (probable en tareas específicas), pasa a la Fase 5.

### Fase 3: Integración (1 a 2 semanas)

Elige una superficie:

| Superficie | Cuándo | Cómo |
|---|---|---|
| SDK Python en tu propio proceso | Aplicación Python, máximo control y mínima latencia | `Router(preload=True)` una vez al arrancar; `predict_batch` para lotes |
| `laya-serve` (HTTP) | Varios clientes o lenguajes | `LAYA_API_KEY`, `LAYA_PRELOAD=1`, `LAYA_MODELS=multilingual` si solo necesitas uno. Cliente con cualquier HTTP; compatible con clientes del protocolo Jev |
| MCP | Quieres que un asistente de IA llame a Laya como herramienta | `laya-mcp-server` en la configuración del cliente MCP |
| LangChain/LangGraph | Ya tienes flujos de agentes | `LayaRouter` como arista condicional, `LayaGuardrail` como filtro |
| Esquema JSON / pydantic | Quieres campos tipados directamente | `agent.decide(texto, schema=...)` |

Adopción por etapas (guía `docs/staged-adoption.md`): primero en sombra (Laya decide pero no actúa; registras la comparación con el proceso actual), luego comparación, luego automatización solo por encima del umbral con muestreo de revisión humana.

### Fase 4: Endurecimiento (2 a 3 días)

1. Fijar la revisión de los pesos y verificar hashes:

```python
from laya import Router
from laya.revisions import PINNED_REVISIONS
router = Router(revision=PINNED_REVISIONS["convaiinnovations/laya"])
```

Y calcula tú mismo los SHA-256 de `model.safetensors`, `rl_agent_config.json` y los archivos del tokenizador tras una descarga verificada, para pasarlos en `LAYA_SHA256_DIGESTS`.

2. `HF_HUB_OFFLINE=1` en producción; los pesos viajan en el volumen o la imagen.
3. `laya-serve` detrás de proxy con TLS y límite de tasa; `LAYA_API_KEY_FILE`; bind interno.
4. Hook de redacción de datos personales si aplica (`examples/hooks/redact.py`).
5. Hook de auditoría a tu propio sistema con `run_id` para trazabilidad.
6. `pip-audit` en tu pipeline; Dependabot o equivalente en tu repositorio.
7. Pruebas de carga con tus tamaños reales de texto; ajusta `LAYA_MAX_CONCURRENT` y `LAYA_THREADS`.

### Fase 5: Fine-tuning (opcional, 1 a 2 semanas)

Cuándo: si la Fase 2 muestra precisión insuficiente. El propio proyecto dice que "el fine-tuning es donde está la mayor parte del valor".

Qué necesitas:

- Entre 2.000 y 30.000 decisiones etiquetadas (el notebook usa ~30k preguntas, 4 épocas, 4 a 5 horas en 2x T4).
- GPU: Kaggle gratuito (2x T4) o una GPU propia de 16 GB (el ejemplo `docs/finetune_browser_agent.md` se hizo en una RTX 4070 Ti SUPER de 16 GB).
- Cuenta de Hugging Face solo si quieres publicar el resultado; puedes guardar el checkpoint en disco y cargarlo con `laya.load("/ruta/local")`.

Pasos: partir de `notebooks/laya_finetune_typed_decisions_2xT4_kaggle.ipynb`, sustituir el dataset por el tuyo, entrenar, ajustar temperaturas, evaluar en datos apartados, exportar. Revisar la celda de subida al Hub antes de ejecutarla (ver auditoría, sección 4.2).

### Fase 6: Operación

- Métricas a vigilar: latencia p50/p95, tasa de decisiones bajo el umbral (escaladas), acuerdo con revisión humana muestreada, uso de memoria.
- Reentrenar o recalibrar cuando la distribución de tus textos cambie.
- Seguir las releases del proyecto (versión 0.3.x, cambios frecuentes). Fijar la versión de `laya` en tu `requirements.txt`.

---

## 5. Lista de decisiones que tienes que tomar

| Decisión | Opciones | Recomendación inicial |
|---|---|---|
| Idioma principal | `english` / `multilingual` / router automático | `Router(default="multilingual")` para español |
| Hardware de arranque | CPU portátil / CPU servidor / GPU | Empezar en CPU; pasar a GPU si necesitas < 100 ms o más de 5 decisiones/s |
| Superficie de integración | SDK / HTTP / MCP / LangChain | SDK para prototipar, HTTP para compartir con un equipo |
| Privacidad de datos | Todo local / MCP con LLM en la nube | Todo local salvo que el flujo ya pase por un LLM externo |
| Fijación de pesos | `main` / revisión + hash | Revisión + hash en cualquier entorno que no sea tu portátil |
| Calibración | Usar la de fábrica / ajustar con datos propios | Ajustar siempre antes de automatizar por umbral |
| Fine-tuning | No / Kaggle / GPU propia | Decidir después de la Fase 2 con números en la mano |
| Contenedores | Instalación directa / Docker / NixOS | Docker con los compose del repositorio para servidor |

---

## 6. Presupuesto orientativo de tiempo

| Fase | Tiempo | Perfil |
|---|---|---|
| 0. Definir caso de uso y etiquetar 100 ejemplos | 1 a 2 días | Dueño del proceso + alguien que conozca los datos |
| 1. PoC local | Medio día | Desarrollador Python junior o superior |
| 2. Evaluación y calibración | 1 semana | Desarrollador con nociones de ML |
| 3. Integración | 1 a 2 semanas | Desarrollador backend |
| 4. Endurecimiento | 2 a 3 días | DevOps |
| 5. Fine-tuning (si hace falta) | 1 a 2 semanas | Desarrollador con experiencia en entrenamiento |
| Total sin fine-tuning | 3 a 4 semanas | |

---

## 7. Comandos de referencia rápida

```bash
# Instalación mínima
python -m pip install laya

# Servidor HTTP con clave y solo el checkpoint multilingüe, escuchando solo en tu equipo
LAYA_HOST=127.0.0.1 LAYA_API_KEY=cambia-esta-clave LAYA_PRELOAD=1 LAYA_MODELS=multilingual laya-serve

# Petición de prueba
curl -s localhost:8000/v1/systemone -H 'content-type: application/json' \
  -H 'Authorization: Bearer cambia-esta-clave' -d '{
  "state": {"cuerpo": "Nos cobraron dos veces, reembolsen por favor"},
  "questions": {"depto": {"type": "choice", "instructions": "¿Qué equipo?",
                "criteria": {"facturacion": "reembolsos", "tecnico": "errores"}}}}'

# Docker CPU (quickstart) y Docker HTTP
docker compose run --build --rm laya
docker compose -f compose.yaml -f compose.http.yaml up --build laya-serve

# Docker con GPU NVIDIA
docker compose -f compose.yaml -f compose.http.yaml -f compose.cuda.yaml up --build laya-serve

# Servidor MCP para Claude Desktop / Cursor
pip install "laya[mcp]"
laya-mcp-server

# Evaluación con datos propios
laya-evals validate mis_datos.jsonl
laya-evals run mis_datos.jsonl --model multilingual --min-accuracy 0.8 --max-ece 0.05

# Modo sin internet tras la primera descarga
HF_HUB_OFFLINE=1 laya "texto" --predict --model multilingual
```

Variables de entorno más útiles: `LAYA_DEVICE` (cpu, cuda, mps, xpu), `LAYA_PRELOAD`, `LAYA_MODELS`, `LAYA_THREADS`, `LAYA_HOST`, `LAYA_PORT`, `LAYA_API_KEY`, `LAYA_MAX_CONCURRENT`, `LAYA_SHA256_DIGESTS`, `HF_HUB_OFFLINE`, `HF_TOKEN` (solo para repositorios privados en Hugging Face; los de Laya son públicos).
