# 🚀 DetectaRiesgo (KinesiApp)

<p align="center">
  <b>Proyecto de grado — Ingeniería de Software (7.° semestre)</b><br>
  Universidad Cooperativa de Colombia 🏫
</p>

---

## 📖 1. ¿Qué es este proyecto?

**DetectaRiesgo** es una aplicación móvil diseñada para deportistas, entrenadores y administradores de clubes o equipos deportivos. Su objetivo principal es **prevenir lesiones deportivas** analizando la biomecánica y técnica de ejecución en ejercicios de alto riesgo.

### 🎯 El problema que resuelve
Muchas lesiones comunes (rodilla, tobillo, columna) no ocurren por un golpe traumático, sino por patrones de movimiento defectuosos repetidos entrenamiento tras entrenamiento sin supervisión — por ejemplo, un aterrizaje con valgo de rodilla o pérdida de la curvatura lumbar en una sentadilla profunda.

La app permite al usuario grabar un breve video ejecutando un movimiento (salto o sentadilla), subirlo y recibir un informe automatizado:
* 📉 Articulaciones con movimiento de riesgo.
* 📊 Puntaje de riesgo técnico.
* 💡 Recomendaciones en lenguaje sencillo antes de que el mal hábito cause una lesión.

### 👥 Roles del sistema
* 🏃 **Deportista:** Graba/sube videos, consulta su historial de análisis y estadísticas personales.
* 📋 **Entrenador (Coach):** Gestiona deportistas a su cargo y revisa el desempeño colectivo del equipo.
* 🛡️ **Administrador:** Controla altas de usuarios, asignación de roles y distribución de entrenadores.

> **📌 Estado actual:** Es un MVP funcional con flujo completo de punta a punta (autenticación JWT, base de datos real, roles y gestión de archivos). El análisis de movimiento actual es una **simulación con datos de demostración** para validar la arquitectura; la integración del motor de IA real es el siguiente hito.

---

## 🛠️ 2. Tecnologías utilizadas

* **🟢 Backend:** Python con FastAPI (API REST), SQLAlchemy (ORM), Alembic (migraciones) sobre PostgreSQL y autenticación con JWT.
* **🌐 Frontend:** aplicación web responsive con Next.js (App Router) y TypeScript, desplegada en Vercel. Reemplaza al cliente Flutter anterior. Diseño en [`docs/design/web-frontend-architecture.md`](docs/design/web-frontend-architecture.md).
* **🐳 Contenedores:** Docker y Docker Compose para aislar servicios en desarrollo y producción.
* **🤖 Inteligencia artificial (En desarrollo):** MediaPipe, OpenCV (visión por computador) y API de Gemini (generación de recomendaciones en lenguaje natural).

---

## 🚢 3. Despliegue planeado y arquitectura de contenedores

* **API:** Google Cloud Run (`europe-west3`), con la misma imagen de `backend/Dockerfile` (`--port 8000`).
* **Base de datos:** Neon (Postgres administrado, `eu-central-1`), tanto en producción como en desarrollo local. Ya no se levanta Postgres en Docker. Alembic sigue siendo la fuente de verdad del esquema.
* **Frontend:** Vercel (región `fra1`).
* **Videos:** almacenamiento de objetos compatible con S3 de Neon (bucket `videos`, lectura pública). El backend no guarda archivos en disco.

El detalle está en `docs/design/video-analysis-pipeline.md` §9 y `docs/design/web-frontend-architecture.md` §2.

---

## 🧠 4. Inteligencia artificial y capas de procesamiento

### 👁️ Capa 1 — Visión por Computador (MediaPipe + OpenCV)
Extrae el esqueleto cinemático fotograma a fotograma (hombros, caderas, rodillas, tobillos) para calcular ángulos articulares y asimetrías de impacto sin necesidad de hardware gráfico dedicado (GPU).

### 💬 Capa 2 — Lenguaje Natural (API de Gemini)
Traduce los datos numéricos de la Capa 1 a explicaciones comprensibles para el deportista y alimenta un asistente interactivo de consultas. *(Nota: Gemini nunca recibe datos biométricos crudos ni videos, solo métricas calculadas).*

---

## ⚡ 5. Concurrencia: Event Loop, Hilos y Microtareas (Event Loop del Evaluador)

> Esta arquitectura está implementada las tareas del event loop, hilos y microtareas en el proyecto tanto en el backend como en el frontend para evitar bloqueos en las interfaces y en las peticiones concurrentes:

### 🐍 Backend (Python / FastAPI)
* **Event Loop (`asyncio`):** Gestiona las peticiones asíncronas concurrentes. A diferencia de Node.js, Python no maneja microtareas independientes para promesas, pero separa claramente el flujo con corrutinas (`async/await`).
* **Threadpool automático para I/O:** Las operaciones de lectura/escritura de archivos pesados (como la subida inicial de videos) se configuran deliberadamente en funciones síncronas (`def` en lugar de `async def`), permitiendo que FastAPI las derive a un **hilo secundario (Threadpool)** para liberar el event loop principal y evitar congelar peticiones HTTP concurrentes (como el inicio de sesión).
* **Background Tasks y Workers:** Las tareas de procesamiento intensivo se delegan fuera del ciclo de respuesta web utilizando colas de fondo para garantizar una experiencia fluida al usuario.

### 🌐 Frontend (Next.js, en construcción)
* **Rendering por ruta:** CSR, SSR, SSG, ISR y streaming SSR, según la pantalla.
* **Event loop:** tareas frente a microtareas, tanto en el navegador (progreso de la subida de video) como en Node (render en streaming).
* **Web Worker:** compresión del avatar fuera del hilo principal; en el cliente Flutter se hacía con un Isolate.

El detalle está en [`docs/design/web-frontend-architecture.md`](docs/design/web-frontend-architecture.md) §3, §7 y §10.

---

## 📐 6. Patrones de diseño aplicados

### 🧩 Backend
* **Arquitectura por capas:** Separación estricta en `Router` (Controlador) ➔ `Service` (Lógica de negocio) ➔ `Repository` (Acceso a datos).
* **Repository Genérico:** `BaseRepository` para centralizar operaciones CRUD comunes.
* **Inyección de dependencias:** Uso nativo de `Depends` en FastAPI para sesiones y servicios.

### 🎨 Frontend
* **Backend for Frontend (BFF):** Next.js guarda los tokens en cookies httpOnly y llama a la API desde el servidor.
* **Server Components por defecto:** los Client Components solo se usan donde hacen falta APIs del navegador.

---

## 🚀 7. Cómo correr el proyecto localmente

### 🐘 Backend (contra Neon)

La base de datos es Neon, también en local. En el `.env` de la raíz (copia de `.env.example`), `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB` salen del connection string **directo** de Neon (el host **sin** `-pooler`), el mismo que usa Alembic. El `-pooler` es sólo para la API en Cloud Run.

> ⚠️ Si el `.env` apunta a la misma base que producción, `alembic upgrade head` y todo lo que hagas en local escribe en datos reales. Para desarrollar con datos de prueba, crea una rama en Neon y usa su host.

```bash
cd backend
.venv\Scripts\activate
$env:PGSSLMODE="require"     # bash: export PGSSLMODE=require (libpq la lee del entorno, no del .env)

alembic upgrade head          # aplica migraciones en Neon
uvicorn app.main:app --reload # http://localhost:8000
```

Opcional, para probar la imagen Docker que va a Cloud Run: `docker compose up -d --build api` (http://localhost:8001). Usa los mismos datos de Neon del `.env`. El servicio `db` de Postgres local quedó comentado en `docker-compose.yml`.

### 🌐 Frontend (Next.js)

Requiere Node.js 20 o superior.

```bash
cd frontend
cp .env.example .env.local   # API_BASE_URL: puerto 8001 (docker compose) u 8000 (uvicorn)
npm install
npm run dev                  # http://localhost:3000
```

Desde el celular: la cámara del navegador solo funciona con HTTPS, así que en la red local hay que usar `npm run dev -- --experimental-https` (ver diseño §5.2).

### 🧪 Ejecutar pruebas unitarias

```bash
# Pruebas de Backend
cd backend
pytest

# Frontend: lint y build de producción
cd frontend
npm run lint
npm run build

```

---

## ✍️ 8. Autor

* **Daniers Alexander Solarte Lima** — Ingeniería de Software, 7.° semestre, Universidad Cooperativa de Colombia.
