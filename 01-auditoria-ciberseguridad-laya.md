# Auditoría de ciberseguridad: repositorio Laya (NandhaKishorM/laya)

Fecha de la auditoría: 27 de septiembre de 2026
Versión auditada: Laya 0.3.20, commit `4066d5d` (merge del PR #464, 25 de septiembre de 2026)
Método: clonación del repositorio, lectura completa del paquete Python `laya/` (unas 12.000 líneas incluyendo ejemplos), del paquete TypeScript `laya-ts/`, de los archivos Docker, Nix, CI y notebooks, búsqueda de llamadas de red, deserialización insegura, secretos y variables de entorno, y ejecución de `pip-audit` sobre las dependencias declaradas.

Limitación: desde el entorno de auditoría no fue posible acceder a huggingface.co, por lo que el contenido exacto de los repositorios de pesos se describe a partir de lo que el código descarga y valida, no de una inspección directa. Tampoco se ejecutó el modelo.

---

## 1. Respuesta corta

**¿Qué es Laya?** No es un modelo de lenguaje tipo chat. Es un "motor de decisiones" basado en un codificador tipo BERT (ModernBERT-large o mmBERT-base) que responde preguntas tipadas sobre un texto: elegir una opción (`choice`), dar un puntaje ordinal (`score`) o una probabilidad sí/no (`noul`). No genera texto. Sirve para clasificar, priorizar, moderar, enrutar y extraer campos estructurados de correos, tickets o documentos.

**¿Es seguro correrlo en mi computador local?** Sí, con precauciones normales. El código del paquete principal:

- No contiene telemetría, analíticas ni llamadas a servidores del autor.
- La única conexión de red del núcleo es la descarga inicial de los pesos desde Hugging Face. Después puede funcionar 100 % sin internet (`HF_HUB_OFFLINE=1`).
- La inferencia ocurre íntegramente en tu CPU o GPU. El texto que le cargas no sale de tu máquina.
- No usa `pickle`, `torch.load`, `eval`, `exec`, `subprocess` ni `trust_remote_code`. Los pesos se cargan solo en formato `safetensors`, que no puede ejecutar código.

**¿Me roban la información?** No, si usas el SDK, la CLI, Docker o el servidor HTTP con clientes locales. Tu información sí puede salir de tu máquina únicamente en estos escenarios, todos opcionales y bajo tu control (detalle en la sección 5):

1. Si usas Laya como herramienta MCP de un asistente de IA en la nube (Claude Desktop, Cursor). Ahí el texto pasa por ese asistente, no por Laya.
2. Si apuntas la integración LangChain a un servidor remoto (`base_url`).
3. Si escribes tú mismo un hook que envíe los resultados a un servicio externo (los ejemplos lo muestran comentado).
4. Si ejecutas el notebook de fine-tuning, que sube el modelo entrenado a tu cuenta de Hugging Face.
5. Si ejecutas los scripts de investigación con `--backend jev`, que envían los casos a la API de TypeSafe.

**Riesgos reales que sí debes gestionar:**

- El servidor `laya-serve` escucha por defecto en todas las interfaces de red (`0.0.0.0`) y sin autenticación salvo que definas `LAYA_API_KEY`. En una red compartida, otros equipos podrían usarlo. (El `docker compose` sí lo limita a `127.0.0.1` por defecto).
- Por defecto descarga la rama `main` de los pesos en Hugging Face sin fijar versión ni verificar hash. Si el repositorio de pesos fuera comprometido, recibirías pesos alterados. El proyecto ofrece los mecanismos para fijarlos (`revision` y `expected_sha256`), pero hay que activarlos.
- La cadena de dependencias (PyTorch, transformers) es enorme. Laya en sí es pequeño, pero instalas cientos de MB de código de terceros. Es el mismo riesgo que cualquier proyecto de IA en Python.
- El proyecto no tiene `SECURITY.md` ni política de divulgación de vulnerabilidades.

---

## 2. Inventario del repositorio

Total: 295 archivos, 12 MB. Sin binarios de modelo dentro del repositorio (los pesos viven en Hugging Face).

| Carpeta / archivo | Qué contiene | Relevancia de seguridad |
|---|---|---|
| `laya/` (21 módulos Python) | El paquete instalable: `agent.py` (carga e inferencia), `router.py` (enrutado por idioma), `common.py` (arquitectura del modelo), `lang.py` (detección de idioma, Python puro), `serve.py` (servidor HTTP), `mcp/` (servidor MCP), `integrations/langchain.py`, `hooks.py`, `structured.py`, `email.py`, `presets.py`, `evals.py`, `fast.py` y `tl_kernels.py` (kernels GPU TileLang), `onnx_agent.py`, `shortlist.py`, `revisions.py`, `cli.py` | Auditado línea por línea en los módulos críticos. Sin llamadas de red salvo `huggingface_hub.snapshot_download` y `urllib` en la integración LangChain |
| `laya-ts/` | Port a TypeScript/ONNX Runtime (Node y navegador) con 24 archivos de prueba | Descarga artefactos ONNX desde huggingface.co. Dependencias opcionales `onnxruntime-node` y `onnxruntime-web` |
| `examples/` | `server.py` (3.436 líneas: API + interfaz web de pruebas), `langchain_quickstart.py`, `docker/`, `hooks/` (audit, cache, otel, redact) | El servidor de ejemplo no tiene autenticación, está pensado para uso local |
| `docker/`, `Dockerfile`, `compose*.yaml` | Imagen CPU/CUDA/DGX Spark, entrypoint que carga secretos desde archivo | Buenas prácticas: usuario no root, sin secretos en variables de entorno, puerto en loopback |
| `nix/`, `flake.nix`, `.envrc` | Empaquetado Nix y módulo NixOS con systemd endurecido | Solo relevante si usas NixOS |
| `.github/workflows/` | CI (Linux/Windows, Python 3.10 a 3.13), `security.yml` (gitleaks, pip-audit, CodeQL, comprobación anti-pickle), docker, docs, release a PyPI con Trusted Publishing, evals semanales | Pipeline de seguridad por encima del promedio de proyectos open source |
| `tests/` | 56 archivos, 161 funciones de prueba | Incluye pruebas de límites de cuerpo HTTP, autenticación, digests SHA-256, revisiones fijadas, rutas de escape, redirecciones |
| `research/`, `benchmarks/`, `notebooks/` | Scripts de benchmark, resultados JSON, notebook de fine-tuning en Kaggle | Algunos scripts contactan servicios externos (ver sección 5). No se instalan con `pip install laya` |
| `docs/` | Documentación (Docker, hooks, LangChain, evals, adopción por etapas) | Informativo |
| `AGENTS.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `LICENSE` | Reglas de contribución. Regla explícita: "No introducir dependencia de un servicio alojado o API externa" | Licencia Apache 2.0, uso comercial permitido |

Archivos de política de seguridad: no existe `SECURITY.md`. Sí existe un `.gitignore` extenso que excluye `.env`, tokens, credenciales y transcripciones de sesión.

---

## 3. Dependencias y cadena de suministro

### 3.1 Dependencias Python declaradas (`pyproject.toml`)

Núcleo obligatorio (5 paquetes):

| Paquete | Versión mínima | Para qué | Nota |
|---|---|---|---|
| `torch` | >= 2.0.0 | Inferencia | Dependencia más pesada (cientos de MB a varios GB con CUDA). Descarga de download.pytorch.org |
| `transformers` | >= 4.48.0 | Codificador y tokenizador | Nunca se usa `trust_remote_code`, así que no ejecuta código del repositorio del modelo |
| `safetensors` | >= 0.4.0 | Carga de pesos | Formato seguro, sin ejecución de código |
| `huggingface_hub` | >= 0.20.0 | Descarga de pesos | Única fuente de tráfico de red del núcleo |
| `numpy` | >= 1.20.0 | Cálculo | Estándar |

Extras opcionales (solo si los instalas):

| Extra | Paquetes | Uso |
|---|---|---|
| `laya[serve]` | fastapi, uvicorn, python-multipart | Servidor HTTP `laya-serve` y `examples/server.py` |
| `laya[mcp]` | mcp >= 2.2.0 | Servidor MCP por stdio |
| `laya[fast]` | tilelang >= 0.1.14 | Kernels GPU fusionados (compila código en tu máquina, solo CUDA) |
| `laya[structured]` | pydantic >= 2 | Esquemas JSON / pydantic |
| `laya[onnx]` | onnx, onnxruntime | Ruta ONNX |
| `laya[langchain]` / `[langgraph]` | langchain-core, langgraph | Integración LangChain |

Resultado de `pip-audit` (27 de septiembre de 2026) sobre las dependencias declaradas resueltas a sus versiones más recientes: **sin vulnerabilidades conocidas**. Advertencia: los rangos son abiertos (`>=`), así que si tu entorno ya tiene versiones viejas de `transformers` o `torch`, esas sí pueden tener CVE publicados. Mantén el entorno actualizado y ejecuta `pip-audit` en tu instalación real.

### 3.2 Dependencias TypeScript (`laya-ts/package.json`)

Sin dependencias obligatorias en producción. Opcionales: `onnxruntime-node`, `onnxruntime-web`. Desarrollo: `typescript`, `vitest`. El `package-lock.json` resuelve 126 paquetes, casi todos transitivos de vitest y onnxruntime.

### 3.3 Pesos del modelo (Hugging Face)

Repositorios: `convaiinnovations/laya` (agrupa los tres checkpoints en subcarpetas), y los independientes `convaiinnovations/laya-multilingual` y `convaiinnovations/laya-typed-decisions`. Públicos, sin cuenta necesaria.

Lo que el código descarga, y nada más (`allow_patterns` en `laya/agent.py:300`):

- `rl_agent_config.json` (configuración, se lee con `json.load`)
- `model.safetensors` (pesos, se lee con `safetensors.torch.load_file`)
- `tokenizer/*` (tokenizador, se lee con `AutoTokenizer`)
- `encoder/*` (configuración del codificador, `AutoConfig`)

Ningún archivo `.py`, `.bin`, `.pt` ni `.pkl` se descarga ni se ejecuta. Un checkpoint malicioso solo podría alterar los números de las predicciones, no ejecutar código en tu máquina. Además, `_verify_compatibility` y `load_state_dict(strict=True)` rechazan pesos cuya forma no coincida con la arquitectura esperada.

Controles de integridad disponibles pero **no activados por defecto** (`laya/revisions.py`):

- `PINNED_REVISIONS`: commits revisados de los tres checkpoints (`55cf4c4e...`, `e4e9ddf2...`, `1a793eb5...`). Se aplican solo si pasas `revision=` o `Router(revision=...)`.
- `expected_sha256` o variable `LAYA_SHA256_DIGESTS`: verificación SHA-256 de cada archivo antes de cargar nada. Rechaza rutas absolutas o con `..`.

Recomendación: en producción fija ambas cosas. Sin ello, cada primera descarga toma lo que haya en `main`.

Nota: si el checkpoint no trae carpeta `tokenizer/`, el código descarga el tokenizador del codificador base (`cfg["encoder"]`, por ejemplo `answerdotai/ModernBERT-large`) desde Hugging Face. Es un segundo origen de descarga, también sin ejecución de código.

### 3.4 CI y publicación

- Todas las acciones de GitHub de los flujos principales están fijadas por SHA de commit, no por etiqueta. Excepciones menores: el job `evals` de `ci.yml` y `evals.yml` usan `@v4`/`@v5`.
- `security.yml` corre en cada PR, en `main` y semanalmente: gitleaks (historial y árbol de trabajo), pip-audit, CodeQL con consultas extendidas, y un grep que falla la build si aparece `pickle`, `torch.load`, `eval(`, `exec(`, `subprocess` u `os.system` en `laya/`.
- Publicación a PyPI mediante Trusted Publishing (OIDC), sin token almacenado en secretos.
- Dependabot semanal para pip y GitHub Actions.
- Permisos de flujos en `read-all` por defecto.

Esto es un nivel de higiene de cadena de suministro superior al de la mayoría de proyectos de IA en GitHub.

---

## 4. Flujo de datos: a dónde va tu información

### 4.1 Uso normal (SDK Python, CLI, Docker, `laya-serve`)

```
Tu texto ──► tokenizador (local) ──► modelo en CPU/GPU (local) ──► JSON con respuestas (local)
```

Verificado en el código:

- `laya/agent.py`, `laya/router.py`, `laya/common.py`: sin `import requests`, `httpx`, `socket`, `urllib` ni `aiohttp`. La única función de red es `snapshot_download` para los pesos.
- `laya/lang.py` (detección de idioma): heurísticas en Python puro, sin modelo externo ni red.
- `laya/serve.py`: servidor FastAPI que recibe peticiones y llama a `router.predict` en un hilo local. No reenvía nada.
- `laya/mcp/`: servidor MCP por stdio (entrada/salida estándar). No abre puertos ni sockets.
- Variables de entorno leídas: `HF_TOKEN` (solo se envía a huggingface.co), `LAYA_*` (configuración local). Ninguna clave de servicios de terceros.
- No hay registro (log) del contenido de las peticiones. `laya-serve` registra excepciones con el nombre del modelo, no el texto. Los logs de acceso de uvicorn registran IP y ruta, no el cuerpo.
- La caché de Hugging Face (`~/.cache/huggingface`) guarda solo los pesos, no tus datos.

Cómo comprobarlo tú mismo: descarga los pesos una vez, luego ejecuta con `HF_HUB_OFFLINE=1` y sin conexión. Si funciona, no depende de internet. Adicionalmente puedes monitorear con `netstat`/`lsof` o un cortafuegos de aplicación mientras haces predicciones.

### 4.2 Escenarios en los que sí sale información (todos opcionales)

| Escenario | Archivo | A dónde va | Cómo evitarlo |
|---|---|---|---|
| Descarga de pesos | `laya/agent.py:307`, `laya/onnx_agent.py:111` | huggingface.co (solo petición de descarga; tu `HF_TOKEN` si lo defines) | Descargar una vez, luego `HF_HUB_OFFLINE=1`, o usar un directorio local con `LAYA_MODEL_PATH` |
| Servidor MCP usado por un asistente en la nube | `laya/mcp/server.py` | El asistente (Claude, Cursor...) envía a Laya el texto que él decide, y recibe la respuesta. Ese asistente ya tiene el texto | Usar el MCP solo con clientes locales, o aceptar que el LLM cliente ve los datos |
| LangChain con `base_url` | `laya/integrations/langchain.py:100-130` | Al servidor `laya-serve` que tú indiques, con `Authorization: Bearer`. Rechaza redirecciones a otro origen o de HTTPS a HTTP | No pasar `base_url`, o apuntarlo a tu propio servidor |
| Hooks que tú escribas | `examples/hooks/audit.py:22-23` | Donde tú programes (el ejemplo tiene el `requests.post` comentado a `example.invalid`) | No instalar hooks que envíen datos |
| Notebook de fine-tuning | `notebooks/laya_finetune_typed_decisions_2xT4_kaggle.ipynb` | Descarga el dataset `LocalLLaMA/typed-decisions` de Hugging Face y sube el modelo resultante a tu cuenta de Hugging Face (`create_repo`, `upload_folder`) con tu `HF_TOKEN` | Quitar la celda de subida, o subir a un repositorio privado |
| Benchmark comparativo con Jev | `research/benchmarks/feishu_zh/run.py:52-63` | `https://api.typesafe.ai/v1/systemone` con `TYPESAFE_API_KEY`, solo con `--backend jev` | No usar ese backend |
| Benchmarks de datasets | `research/scripts/bench_apps.py` | Descarga datasets públicos de Hugging Face (AG News, Banking77, GSM8K, etc.) | Solo si ejecutas esos scripts |
| `laya-ts` (Node/navegador) | `laya-ts/src/providers.ts:301`, `tokenizer.ts:375` | huggingface.co para descargar ONNX y tokenizador; en navegador usa `CacheStorage` | Igual que Python: caché local tras la primera descarga |
| Interfaz web de ejemplo | `examples/server.py` (botón "Edit in playground") | Codifica la petición en base64 en el fragmento de la URL (`/#r=...`). Se queda en el navegador, pero puede quedar en el historial | No compartir esas URL si contienen datos sensibles |

Conclusión: **ninguna ruta del paquete envía tus datos al autor del proyecto ni a un tercero sin que tú lo configures explícitamente.**

---

## 5. Análisis por superficie de ataque

### 5.1 Carga de modelos (riesgo: ejecución remota de código por pesos maliciosos)

Estado: **mitigado**.

- Solo `safetensors` (`laya/agent.py:283,373`). Sin `torch.load`, sin `pickle`. Un test de CI lo vigila (`security.yml`, job `supply_chain`).
- `AutoModel.from_config` y `AutoTokenizer.from_pretrained` sin `trust_remote_code`, así que transformers no importa código Python del repositorio del modelo.
- El código reescribe `tokenizer_config.json` en la caché (`_fix_tokenizer_config`) para compatibilidad. Lo hace con archivo temporal y `os.replace` atómico, y solo toca claves conocidas. No es una vulnerabilidad, pero por eso el directorio de checkpoint debe ser escribible.
- Rutas locales que empiezan por `/`, `./` o `../` nunca se descargan (`test_local_path_never_downloads`).

Pendiente: activar `revision` y `expected_sha256` para protegerte de un compromiso del repositorio de Hugging Face.

### 5.2 Servidor HTTP `laya-serve` (`laya/serve.py`)

Controles implementados:

- Autenticación por bearer token opcional (`LAYA_API_KEY`), comparación en tiempo constante con `hmac.compare_digest` en bytes (evita el 500 con cabeceras no ASCII).
- Límite de cuerpo 2 MB, aplicado por streaming (no confía en `Content-Length`).
- Máximo 64 preguntas, 50.000 caracteres de estado, 100 opciones por `choice`, 32 niveles por `score`, 512 opciones totales.
- Máximo 16 peticiones concurrentes admitidas; el exceso recibe 503 en vez de encolarse (evita agotamiento de memoria).
- Un solo hilo de inferencia, para no bloquear el bucle de eventos ni `/health`.
- Errores de inferencia responden un 500 genérico y registran la traza solo en el log del servidor (no filtra rutas ni memoria).
- Validación estricta del JSON: rechaza cuerpos malformados, `state` nulo, recursión profunda.

Debilidades:

- **Bind por defecto en `0.0.0.0`** y **sin autenticación por defecto**. Cualquier máquina de tu red puede enviar peticiones. Solución: `LAYA_HOST=127.0.0.1` o definir `LAYA_API_KEY`. El `compose.http.yaml` ya publica el puerto en `127.0.0.1` por defecto.
- Sin TLS. Ponerlo detrás de un proxy inverso (nginx, Caddy, Traefik) si se expone.
- Sin limitación de tasa por cliente. Un cliente autenticado puede saturar el servidor.
- `/health` expone qué checkpoints están cargados y sus revisiones, sin autenticación. Información de bajo riesgo.

### 5.3 Servidor de ejemplo `examples/server.py`

- Bind por defecto `127.0.0.1` (correcto).
- Sin autenticación en absoluto. No exponerlo fuera de tu equipo.
- Los errores 500 devuelven el tipo y mensaje de la excepción al cliente (`server.py:275-276`). Filtración menor de información en un servidor que solo debe ser local.
- Sin middleware CORS, lo que en la práctica protege contra páginas web de terceros que intenten usarlo desde tu navegador.
- Reutiliza los mismos límites de tamaño de `laya.serve`.

### 5.4 Servidor MCP (`laya/mcp/`)

- Transporte stdio únicamente. No escucha en red.
- Valida el estado y las preguntas antes de usarlas; los errores se serializan como JSON sin trazas.
- Riesgo real: es el cliente MCP (por ejemplo, un asistente en la nube) quien decide qué texto enviar. Laya no filtra nada, pero tampoco puede evitar que el asistente ya tenga tus datos.

### 5.5 Hooks (`laya/hooks.py`)

Los hooks son código tuyo que se ejecuta en cada predicción, con acceso al texto completo antes y después del modelo. No son un riesgo del proyecto, sino un punto donde tú podrías introducir fugas. Buen uso: el ejemplo `redact.py` elimina correos y teléfonos antes de que el modelo los vea. Existen `hooks_timeout` y `hooks_raise=False` para que un hook lento o fallido no tumbe el servicio.

### 5.6 Docker

- Imagen `python:3.11-slim-bookworm`, usuario no root (`uid 10001`).
- Secretos por archivo (`HF_TOKEN_FILE`, `LAYA_API_KEY_FILE`) que el entrypoint lee y borra de la variable `*_FILE`.
- `.dockerignore` excluye `.env` y todo lo que no sea el paquete.
- `TORCH_DISABLE_NATIVE_JIT=1` para evitar compilación de kernels Triton en tiempo de ejecución.
- La versión de torch se fija (`2.14.0`) y se verifica que la rueda instalada corresponde al índice pedido (`docker/check_torch.py`).
- Punto a revisar: `check_torch.py` reescribe metadatos de la rueda `nvidia-cusparselt-cu13` en ARM64 para corregir un defecto de etiqueta. Está acotado a dos versiones revisadas y verifica que la librería es ELF AArch64 antes de tocarla. No es riesgo para AMD64.

### 5.7 NixOS

Módulo systemd con `DynamicUser`, `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `RestrictNamespaces`, `LockPersonality`, `LoadCredential` para el token. Bind por defecto `127.0.0.1`. Bien endurecido.

### 5.8 Secretos en el repositorio

Búsqueda de patrones de tokens (Hugging Face, AWS, OpenAI, GitHub, claves privadas): **ninguno encontrado**. El CI corre gitleaks sobre el historial completo. El `.gitignore` menciona que en el pasado un token de PyPI apareció en una transcripción, lo que motivó el segundo pase de gitleaks. Es una señal de que el mantenedor reaccionó a un incidente real.

### 5.9 Entrada de usuario y prompt injection

Laya no genera texto, así que la "inyección de prompt" clásica no produce salidas peligrosas: lo peor que puede pasar es una clasificación equivocada. De hecho, el preset `guard_questions` está pensado para detectar jailbreaks e inyecciones como paso previo a un LLM. La precisión reportada en ese preset es 0,698 en un conjunto de 116 ejemplos, así que no debe ser tu única defensa.

---

## 6. Madurez del proyecto y confianza en el autor

- Autor: Convai Innovations (Nandha Kishor M, India). Licencia Apache 2.0.
- Actividad en GitHub al 27 de septiembre de 2026: 26.500 estrellas, 2.300 forks, 413 commits, 51 issues abiertos, 172 PR abiertos, versión 0.3.20 publicada en PyPI.
- 56 archivos de prueba, 161 funciones de prueba, CI en Linux y Windows para Python 3.10 a 3.13.
- Documentación honesta sobre las limitaciones del modelo (sección "Honest limits" del README).
- Regla escrita en `AGENTS.md`: prohibido añadir dependencias de servicios alojados. Coincide con lo observado en el código.
- Sin `SECURITY.md`: no hay canal formal para reportar vulnerabilidades. Sin auditoría externa conocida.
- Sin firma de commits o de paquetes verificada en esta auditoría.

Valoración: proyecto joven (versión 0.3.x) pero con prácticas de ingeniería y seguridad de cadena de suministro claramente por encima del promedio, y con un compromiso explícito y verificado de funcionar sin servicios externos.

---

## 7. Riesgos que no son "ciberseguridad" pero afectan usarlo como IA

Estos no comprometen tu equipo, pero sí la calidad de las decisiones:

- Los checkpoints base están cerca del azar en el benchmark typed-decisions sin fine-tuning (0,36 frente a 0,32 aleatorio). El 0,766 corresponde al checkpoint afinado en ese mismo dataset. Laya es una base rápida para especializar, no un oráculo zero-shot.
- Las probabilidades salen sobre-confiadas de fábrica y `laya-multilingual` no trae temperaturas calibradas. Hay que ajustar con datos propios antes de automatizar por umbral de confianza.
- Preguntas `noul` pueden seguir las etiquetas `true/false` en vez del texto (issue #156). Preguntas con negación fallan en ejemplos documentados (issue #377).
- `laya` (inglés) colapsa en idiomas no latinos (Khmer: 0,000 de precisión con 95 % de confianza). El `Router` existe para evitarlo, pero en textos cortos en español puede enrutar mal al checkpoint inglés; conviene `Router(default="multilingual")`.
- Moderación de contenido: 0,530 en datos no vistos. No sirve como único filtro.
- El campo `action.act_probability` no tiene señal útil (issue #185).

---

## 8. Recomendaciones concretas (lista de verificación)

Para uso local personal:

1. Crear un entorno virtual dedicado; no instalar en el Python del sistema.
2. Descargar los pesos una vez, luego trabajar con `HF_HUB_OFFLINE=1`.
3. No exponer `examples/server.py` fuera de `127.0.0.1`.
4. Si usas `laya-serve`, definir `LAYA_HOST=127.0.0.1` o `LAYA_API_KEY`.
5. Si usas el MCP con un asistente en la nube, asumir que ese asistente ve los datos.

Para uso en servidor o producción:

6. Fijar revisión y hashes: `Router(revision=...)` con los valores de `laya/revisions.py` y `LAYA_SHA256_DIGESTS` calculados por ti tras una descarga verificada.
7. Poner `laya-serve` detrás de un proxy inverso con TLS y limitación de tasa.
8. Usar el `Dockerfile` oficial (usuario no root, secretos por archivo) o el módulo NixOS.
9. Instalar un hook de redacción de datos personales antes de la inferencia si procesas información sensible (`examples/hooks/redact.py`).
10. Ejecutar `pip-audit` sobre tu entorno instalado periódicamente y actualizar torch/transformers.
11. No instalar `laya[fast]` (TileLang) salvo que lo necesites: compila kernels en tu máquina y añade superficie.
12. Monitorear salidas de red del proceso en la primera puesta en marcha para confirmar que solo contacta huggingface.co y download.pytorch.org.
13. Revisar el notebook antes de ejecutarlo en Kaggle: quitar o hacer privada la subida al Hub.

---

## 9. Veredicto

| Pregunta | Respuesta |
|---|---|
| ¿Contiene código malicioso, telemetría o puertas traseras? | No se encontró nada de eso |
| ¿Mis datos salen de mi computador al usarlo normalmente? | No. Solo salen en las integraciones opcionales listadas en la sección 4.2 |
| ¿Puede un modelo descargado ejecutar código en mi máquina? | No. Solo `safetensors` y JSON, sin `trust_remote_code` |
| ¿Es seguro correrlo en local? | Sí, con las precauciones de la sección 8 |
| ¿Es seguro exponerlo en red tal cual? | No sin `LAYA_API_KEY`, bind en loopback y un proxy con TLS |
| ¿Nivel de higiene del proyecto? | Alto para su tamaño: CI de seguridad, sin pickle, acciones fijadas por SHA, Trusted Publishing, Dependabot |
| ¿Qué falta? | `SECURITY.md`, fijación de pesos por defecto, límite de tasa en el servidor |
