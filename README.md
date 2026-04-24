# Sistema RAG EPN — Orquestación de Servicios

Este repositorio contiene el archivo de composición Docker que levanta la totalidad del sistema de Asistente Académico RAG de la Escuela Politécnica Nacional. Actúa como punto de integración entre el repositorio del backend (`Proyecto-tesis-backend`) y el del frontend (`Proyecto-tesis-Frontend`).

---

## Tabla de contenidos

1. [Descripción del sistema](#1-descripción-del-sistema)
2. [Arquitectura de servicios](#2-arquitectura-de-servicios)
3. [Modelos de inteligencia artificial](#3-modelos-de-inteligencia-artificial)
4. [Caché semántico](#4-caché-semántico)
5. [Requisitos previos](#5-requisitos-previos)
6. [Estructura del repositorio](#6-estructura-del-repositorio)
7. [Configuración inicial](#7-configuración-inicial)
8. [Levantar el sistema](#8-levantar-el-sistema)
9. [Detener el sistema](#9-detener-el-sistema)
10. [Verificación del sistema](#10-verificación-del-sistema)
11. [Gestión de servicios individuales](#11-gestión-de-servicios-individuales)
12. [Gestión de volúmenes y datos](#12-gestión-de-volúmenes-y-datos)
13. [Notas de operación](#13-notas-de-operación)

---

## 1. Descripción del sistema

El sistema implementa un pipeline de Recuperación Aumentada por Generación (RAG) orientado a consultas académicas sobre documentación institucional de la EPN. El flujo general es:

1. El administrador sube documentos (PDF, DOCX, TXT, MD) a través del frontend.
2. El backend fragmenta cada documento, genera embeddings con un modelo de sentence-transformers y almacena los vectores en Qdrant.
3. Cuando un usuario realiza una consulta por WebSocket, el sistema genera el embedding de la pregunta, recupera los fragmentos más similares de Qdrant y construye un prompt contextualizado.
4. El prompt se envía al LLM activo (local vía vLLM, o en la nube vía Gemini) para generar la respuesta.
5. La respuesta se almacena en Redis para responder consultas semánticamente similares sin invocar nuevamente al LLM.

El sistema soporta dos modos de operación configurables desde el panel de administración, sin necesidad de reiniciar ningún servicio.

---

## 2. Arquitectura de servicios

| Servicio | Imagen | Puerto host | Función |
|---|---|---|---|
| `tesis-postgres` | postgres:16-alpine | 5433 | Base de datos relacional. Almacena usuarios, documentos, configuración del motor, parámetros RAG y configuración NLU |
| `tesis-qdrant` | qdrant/qdrant:latest | 6334 | Base de datos vectorial. Almacena los embeddings de los fragmentos de documentos para búsqueda por similitud coseno |
| `tesis-redis` | redis:7-alpine | 6380 | Caché semántico de respuestas RAG. Reduce latencia y consumo del LLM en consultas repetidas o similares |
| `tesis-vllm` | vllm/vllm-openai:latest | 8001 | Servidor de inferencia local. Sirve el modelo de lenguaje con una API compatible con OpenAI |
| `tesis-backend` | build local | 8000 | API REST y WebSocket. Implementa el pipeline RAG, autenticación JWT y toda la lógica de negocio |
| `tesis-frontend` | build local | 5173 | Interfaz web. Incluye el chat de consultas y el panel de administración |

Todos los servicios pertenecen a la red interna `tesis-net` (bridge). La comunicación entre contenedores usa los nombres de servicio como hostnames.

---

## 3. Modelos de inteligencia artificial

El sistema implementa dos modos de operación que se pueden cambiar en caliente desde el panel de administración del frontend sin reiniciar ningún servicio.

### Modo local:local (por defecto)

- **Embeddings**: `paraphrase-multilingual-MiniLM-L12-v2` (sentence-transformers, 384 dimensiones). Corre en CPU dentro del contenedor backend.
- **LLM**: `Qwen/Qwen2.5-3B-Instruct-AWQ`, servido por vLLM con cuantización AWQ bajo el nombre de servicio `llama3-local`. Requiere GPU NVIDIA.

Parámetros de vLLM relevantes para el hardware disponible:

| Parámetro | Valor | Descripción |
|---|---|---|
| `--max-model-len` | 2048 | Longitud máxima de contexto en tokens |
| `--gpu-memory-utilization` | 0.90 | Fracción de VRAM reservada para el modelo |
| `--max-num-seqs` | 4 | Máximo de secuencias en paralelo |
| `--quantization` | awq | Cuantización AWQ para reducir uso de VRAM |
| `--enable-prefix-caching` | — | Reutiliza KV-cache de prefijos de prompt repetidos |

El modelo se descarga automáticamente desde Hugging Face en el primer arranque y queda almacenado en el volumen `tesis_hf_models_cache`. La descarga es de aproximadamente 2 GB para el modelo cuantizado. Este volumen no debe eliminarse entre reinicios.

### Modo local:cloud (implementado, no activo por defecto)

- **Embeddings**: mismo modelo local (`paraphrase-multilingual-MiniLM-L12-v2`).
- **LLM**: Google Gemini 2.5 Flash, invocado mediante la API de Google AI Studio.

Este modo está completamente implementado y funcional. Para activarlo se requiere una `GOOGLE_API_KEY` válida en el `.env` del backend y cambiar el motor desde el panel de administración. El servicio `tesis-vllm` sigue corriendo pero no recibe solicitudes de generación en este modo.

---

## 4. Caché semántico

Redis implementa un caché de dos niveles para las respuestas RAG:

- **Caché exacto**: si la pregunta (hasheada) ya existe en Redis, retorna la respuesta almacenada sin invocar al LLM ni a Qdrant.
- **Caché semántico**: si no hay coincidencia exacta, compara el embedding de la nueva pregunta contra los embeddings almacenados. Si la similitud coseno supera el umbral configurado, retorna la respuesta semánticamente más cercana.

Las respuestas no se almacenan en caché si son respuestas de error, respuestas de rechazo (el LLM no encontró información), o si el texto es demasiado breve o incompleto.

Desde el frontend, el panel de administración permite listar todas las entradas del caché, buscar por texto de pregunta, ver la respuesta completa de cada entrada, corregir manualmente una respuesta almacenada y eliminar entradas individuales.

---

## 5. Requisitos previos

- Docker Engine 24 o superior
- Docker Compose Plugin v2
- GPU NVIDIA con soporte CUDA 12.x (requerido para `tesis-vllm`)
- NVIDIA Container Toolkit instalado y configurado
- Los repositorios `Proyecto-tesis-backend` y `Proyecto-tesis-Frontend` clonados como subdirectorios del mismo directorio donde se encuentra este `docker-compose.yml`

Verificar que el runtime NVIDIA está disponible para Docker:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

---

## 6. Estructura del repositorio

```
tesis-deploy/                        # Este repositorio
├── docker-compose.yml
├── .env                             # Variables del compose (no versionar)
├── .env.example                     # Plantilla de referencia
├── Proyecto-tesis-backend/          # Submódulo o clon del repo backend
└── Proyecto-tesis-Frontend/         # Submódulo o clon del repo frontend
```

---

## 7. Configuración inicial

### Paso 1 — Crear el archivo `.env`

Copiar la plantilla y completar los valores:

```bash
cp .env.example .env
```

Contenido del `.env`:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=contrasena_segura
POSTGRES_DB=db_tesis_cc
VITE_BACKEND_URL=http://IP_DE_LA_MAQUINA:8000
```

`VITE_BACKEND_URL` debe ser la dirección IP de la máquina donde corre el sistema, accesible desde el navegador del cliente. No usar `localhost` si el frontend se accede desde otra máquina.

### Paso 2 — Verificar el `.env.docker` del backend

El archivo `Proyecto-tesis-backend/.env.docker` debe existir con la configuración correcta para el entorno Docker. Consultar el README del backend para su contenido completo.

### Paso 3 — Construir las imágenes locales

Este paso solo es necesario la primera vez o cuando hay cambios en el código del backend o frontend:

```bash
docker compose build
```

El backend tarda varios minutos en construirse debido al tamaño de las dependencias Python.

---

## 8. Levantar el sistema

### Primer arranque

```bash
docker compose up -d
```

En el primer arranque, `tesis-vllm` descargará el modelo desde Hugging Face. El backend esperará a que PostgreSQL y Redis estén saludables antes de iniciar. El tiempo total hasta que todos los servicios estén operativos es de 5 a 15 minutos dependiendo de la conexión a internet y la GPU disponible.

Seguir el progreso de la carga del modelo:

```bash
docker logs tesis-vllm -f
```

El modelo está listo cuando los logs muestran:

```
Application startup complete.
```

### Arranques posteriores

```bash
docker compose up -d
```

El modelo ya está en el volumen `tesis_hf_models_cache` y no se vuelve a descargar. El sistema estará operativo en 1 a 3 minutos.

### Reconstruir solo el backend o el frontend

Usar únicamente cuando haya cambios en el código fuente o en `requirements.txt`:

```bash
docker compose up -d --build --no-deps backend
docker compose up -d --build --no-deps frontend
```

El flag `--no-deps` evita reconstruir servicios que no cambiaron.

---

## 9. Detener el sistema

### Detener y eliminar contenedores manteniendo los datos

```bash
docker compose down --remove-orphans
```

`--remove-orphans` elimina contenedores de servicios que ya no están definidos en el compose, evitando que queden procesos residuales ocupando puertos.

### Si algún contenedor queda activo después del down

```bash
docker rm -f tesis-postgres tesis-qdrant tesis-redis tesis-vllm tesis-backend tesis-frontend 2>/dev/null
```

### Verificar que no quedaron puertos ocupados

```bash
ss -tulnp | grep -E '8000|8001|5173|5433|6334|6380'
```

Si algún puerto sigue ocupado después de eliminar los contenedores, hay un proceso externo a Docker usando ese puerto. Identificarlo:

```bash
ss -tulnp | grep PUERTO
```

### Detener sin eliminar contenedores

```bash
docker compose stop
```

Los contenedores quedan en estado `Exited` y pueden reiniciarse con `docker compose start` sin necesidad de recrearlos.

---

## 10. Verificación del sistema

### Estado general

```bash
docker compose ps
```

Resultado esperado: los 6 servicios en estado `running`. Si alguno aparece en `restarting` o `exited`, revisar sus logs.

### Verificación de cada servicio

**PostgreSQL**

```bash
docker exec tesis-postgres pg_isready -U postgres
```

Respuesta esperada: `localhost:5432 - accepting connections`

**Redis**

```bash
docker exec tesis-redis redis-cli ping
```

Respuesta esperada: `PONG`

**Qdrant**

```bash
curl -s http://localhost:6334/healthz
```

Respuesta esperada: JSON con estado `ok` u `healthy`.

**vLLM — modelo cargado**

```bash
curl -s http://localhost:8001/v1/models | python3 -m json.tool
```

Debe aparecer el modelo `llama3-local` en la lista. Si vLLM aún está iniciando, este endpoint devolverá error hasta que termine de cargar.

**Backend**

```bash
curl -s http://localhost:8000/ | python3 -m json.tool
```

Respuesta esperada:

```json
{
    "status": "ok",
    "message": "Servidor Backend RAG (Arquitectura Dual) en línea.",
    "version": "4.0.0",
    "modos_activos": ["local:local", "local:cloud"]
}
```

**Frontend**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:5173
```

Respuesta esperada: `200`

### Consumo de recursos

```bash
docker stats --no-stream
```

Muestra CPU, RAM y red de cada contenedor en un snapshot. `tesis-vllm` tendrá el mayor consumo de memoria GPU. Los demás servicios en reposo deben mostrar consumo mínimo.

Para monitoreo continuo:

```bash
docker stats
```

### Logs de todos los servicios

```bash
docker compose logs --tail 50
```

Logs de un servicio específico en tiempo real:

```bash
docker logs tesis-backend -f --tail 100
docker logs tesis-vllm -f --tail 100
```

---

## 11. Gestión de servicios individuales

Todos los comandos se ejecutan desde el directorio que contiene el `docker-compose.yml`.

### Reiniciar un servicio específico

```bash
docker compose restart backend
docker compose restart frontend
docker compose restart redis
```

### Detener un servicio sin afectar los demás

```bash
docker compose stop vllm
```

### Iniciar un servicio detenido

```bash
docker compose start vllm
```

### Ciclo completo de un servicio (detener, eliminar contenedor y recrear)

```bash
docker compose up -d --force-recreate --no-deps backend
```

---

## 12. Gestión de volúmenes y datos

Los datos persistentes se almacenan en volúmenes Docker nombrados. Ningún volumen se elimina con `docker compose down` a menos que se use el flag `--volumes`.

| Volumen | Contenido | Impacto si se elimina |
|---|---|---|
| `tesis_hf_models_cache` | Modelo Qwen descargado (~2 GB cuantizado) | Se vuelve a descargar en el siguiente arranque |
| `tesis_postgres_data` | Base de datos completa (usuarios, documentos, configuración) | Pérdida total de datos. El sistema se reinicia en estado de primer arranque |
| `tesis_qdrant_storage` | Vectores indexados de todos los documentos | Todos los documentos deben reprocesarse |
| `tesis_redis_data` | Caché de respuestas | Se pierde el caché. Sin impacto funcional, el sistema regenera entradas al usarse |
| `tesis_documents_local` | Archivos físicos subidos (PDF, DOCX, etc.) | Los archivos físicos se pierden. Los registros de BD quedan inconsistentes |

### Listar volúmenes del proyecto

```bash
docker volume ls | grep tesis
```

### Inspeccionar el tamaño de un volumen

```bash
docker system df -v | grep tesis
```

### Eliminar un volumen específico (requiere que el servicio esté detenido)

```bash
docker compose stop postgres
docker volume rm tesis_postgres_data
```

---

## 13. Notas de operación

**El servicio vLLM requiere GPU.** Si la máquina no tiene GPU NVIDIA disponible o el NVIDIA Container Toolkit no está configurado, `tesis-vllm` fallará al iniciar. En ese caso, el modo `local:cloud` (Gemini) sigue siendo funcional siempre que `GOOGLE_API_KEY` esté configurada en el `.env.docker` del backend.

**El primer arranque de vLLM puede tardar varios minutos.** El backend realiza un precalentamiento al iniciar enviando una consulta interna. Si vLLM aún no está listo en ese momento, el precalentamiento falla silenciosamente y el sistema sigue operativo. La primera consulta real del usuario tendrá latencia adicional mientras el modelo termina de cargar.

**No usar `--build` en reinicios rutinarios.** Las imágenes del backend y frontend se reconstruyen completamente con ese flag, incluyendo la reinstalación de todos los paquetes Python. Solo es necesario cuando hay cambios en `requirements.txt` o en los `Dockerfile`.

**El volumen `tesis_hf_models_cache` no debe eliminarse.** Contiene el modelo descargado. Eliminarlo no causa pérdida de datos del sistema pero obliga a una nueva descarga al siguiente arranque.

**La configuración del motor activo persiste en la base de datos.** Cambiar entre `local:local` y `local:cloud` desde el panel de administración no requiere reiniciar ningún servicio y el cambio sobrevive reinicios del sistema.