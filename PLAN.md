# Plan de desarrollo — Panel de indicadores hipotecarios

> **Fuente de verdad funcional/técnica:** `docs/project-info.md` (aprobado, con ajustes vigentes).  
> **Reglas de trabajo:** `AGENTS.md`. Una fase cada vez; no se inicia la siguiente sin revisión/cierre de la anterior.  
> **Alcance:** solo Euríbor 12M (`ECB_DATAFLOW=FM`, `ECB_SERIES_KEY=M.U2.EUR.RT.MM.EURIBOR1YD_.HSTA`), formato `csvdata`. React consume **solo** FastAPI. Sin Nginx/producción, sin jobs en segundo plano, sin funcionalidades fuera de `docs/project-info.md`.

## Normas generales (todas las fases)

- **Seguridad/secretos:** valores reales solo en `.env` (fuera de git). Mantener `.env.example` sin credenciales. Nunca poner contraseñas, `DATABASE_URL` real ni datos sensibles en código, commits o logs.
- **Código:** máximo 500 líneas por archivo; piezas pequeñas con una sola responsabilidad; lógica de negocio separada de UI/capas de transporte. No reescribir ni tocar lo no pedido.
- **Dependencias:** no añadir dependencias hasta que hagan falta. Antes de instalar cada una: indicar cuál, para qué, comprobar que existe y justificarla. Evitar dependencias innecesarias.
- **Pruebas:** toda función/cambio de comportamiento llega con pruebas. Bug → prueba que lo reproduce primero. Pruebas junto al código; E2E Playwright en `e2e/` si aplica. Antes de dar una fase por terminada: ejecutar pruebas relevantes y no aceptar errores/avisos/pruebas fallidas. Pruebas automatizadas de datos: validar **estructura y coherencia** (no valores numéricos históricos fijos).
- **Git:** commits locales, cortos y descriptivos, **solo cuando el usuario lo autorice expresamente**. Nunca `git push`. Nunca `git rebase` para traer cambios (usar `merge` si aplica). Actualizar `PLAN.md` en el commit que cierre una fase (cuando se autorice commit).
- **Producción:** nada a producción sin permiso explícito para esa acción concreta.

## Estado de fases

- [ ] Fase 0 — Estructura inicial y Docker Compose
- [ ] Fase 1 — PostgreSQL y backend
- [ ] Fase 2 — Integración con ECB y sistema de caché
- [ ] Fase 3 — Frontend React y visualización
- [ ] Fase 4 — Integración completa y pruebas

---

## Fase 0. Estructura inicial y Docker Compose

### Objetivo
Disponer de un esqueleto ejecutable con los tres servicios (`frontend`, `backend`, `db`) comunicándose en un único entorno de desarrollo Docker Compose, con configuración por variables de entorno y comprobaciones mínimas, **sin** implementar todavía la funcionalidad del Euríbor.

### Archivos que previsiblemente se crearán o modificarán
- `docker-compose.yml`
- `.env.example` (actualizar)
- `.gitignore` (verificar/ajustar: `.env`, artefactos Python/Node, build)
- `backend/Dockerfile`, `backend/requirements.txt` (o archivo de dependencias equivalente), `backend/app/main.py` (mínimo), `backend/app/__init__.py`, estructura base `backend/app/...`
- `frontend/Dockerfile`, `frontend/package.json` (generado por scaffold), `frontend/vite.config.*`, `frontend/src/App.*`, `frontend/src/main.*`, `frontend/index.html`, etc. (scaffold Vite + React)
- `docs/project-info.md` — **no modificar** salvo corrección autorizada.
- `PLAN.md` — marcar tareas/cierre solo cuando se autorice.

### Tareas concretas (ordenadas)
1. Definir estructura de carpetas: `backend/`, `frontend/`, (opcional según decisión) `backend/tests/`, `frontend/src/...`. Mantener separación de responsabilidades.
2. **Backend (mínimo):** crear app FastAPI con endpoint de salud/comprobación (p. ej. `GET /health` o `/api/health`) que responda JSON simple; sin lógica ECB/BD funcional del dominio.
3. **Frontend (mínimo):** scaffold React + Vite; pantalla inicial con nombre “Panel de indicadores hipotecarios” y comprobación visual mínima; configurar proxy de Vite hacia FastAPI (desarrollo) para rutas API acordadas.
4. **PostgreSQL:** servicio `db` en Compose con imagen oficial, volumen para datos, variables `POSTGRES_DB/POSTGRES_USER/POSTGRES_PASSWORD` desde `.env`.
5. **Dockerfiles:** uno para `backend` (Python, instala dependencias, ejecuta servidor dev) y otro para `frontend` (Node, instala dependencias, ejecuta Vite en modo dev accesible desde host). Mantener simples.
6. **docker-compose.yml:** servicios `frontend`, `backend`, `db`; red interna; `backend` accesible para proxy/frontend; `db` accesible para `backend`; puertos expuestos solo los necesarios para desarrollo; dependencias de arranque razonables (`depends_on`; healthcheck de `db` si se decide).
7. **Variables de entorno:** actualizar `.env.example` con nombres aprobados (`DATABASE_URL`, `POSTGRES_*`, `ECB_*`, `CACHE_TTL_SECONDS`, proxy Vite) sin valores reales. Documentar brevemente en comentarios si ayuda.
8. **Comprobación:** arrancar Compose; verificar que `db` acepta conexiones, `backend` responde al endpoint de salud, `frontend` carga y muestra el nombre; verificar comunicación frontend→backend vía proxy (petición de comprobación).
9. **Pruebas mínimas de fase:** añadir al menos una prueba automática simple (backend: endpoint de salud; y/o frontend: renderiza nombre) según decisiones de tooling, ejecutable en contenedor o localmente.

### Dependencias necesarias y justificación
> **No instalar ahora.** Antes de cada instalación: confirmar nombre, versión/compatibilidad y existencia.

| Área | Dependencia (prevista) | Para qué | Justificación |
|---|---|---|---|
| Backend | `fastapi` | Framework API | Requisito aprobado. |
| Backend | `uvicorn[standard]` o `uvicorn` | Servidor ASGI en desarrollo | Ejecutar FastAPI. |
| Backend | Cliente HTTP (`httpx` u alternativa) | Llamadas ECB en Fase 2 | Necesario para consumir API externa; se puede añadir en Fase 2 si se prefiere minimizar Fase 0. |
| Backend | Driver PostgreSQL (`psycopg`/`psycopg[binary]` o equivalente) | Conexión a PostgreSQL en Fase 1 | Acceso a BD; puede diferirse a Fase 1. |
| Backend | `pytest` (+ `pytest-cov` opcional) | Pruebas backend | Cumplir norma “todo va probado”. |
| Frontend | React, React DOM, Vite, plugin React (scaffold) | SPA y dev server | Requisitos aprobados. |
| Frontend | `recharts` | Gráfico (Fase 3) | Aprobado; puede diferirse a Fase 3. |
| Frontend | Framework de pruebas (Vitest + Testing Library, o decisión aprobada) | Pruebas frontend | “Todo va probado”; **decisión pendiente** (ver abajo). |
| Infra | Imágenes Docker oficiales `python`, `node`, `postgres` | Contenedores | Requisito Docker Compose. |

### Pruebas que deben ejecutarse
- Backend: prueba automática del endpoint de salud (respuesta 200 y estructura JSON esperada).
- Frontend: prueba automática que la pantalla muestra “Panel de indicadores hipotecarios” (o texto acordado).
- Integración manual guiada: servicios arriba, puertos accesibles, proxy funciona.
- Comandos previstos (ajustar a nombres reales de scripts/servicios):
  - `docker compose config`
  - `docker compose up --build -d`
  - `docker compose ps`
  - `docker compose logs --tail=50 backend frontend db`
  - `curl -fsS http://localhost:<puerto-backend>/health` (o ruta acordada)
  - `curl -fsS http://localhost:<puerto-frontend>/`
  - Pruebas backend/frontend según tooling elegido (p. ej. `docker compose exec backend pytest`, `docker compose exec frontend npm test -- --run`).

### Criterios de aceptación
- `docker compose up --build` levanta `frontend`, `backend` y `db` sin errores de arranque.
- El endpoint de salud del backend responde correctamente.
- El frontend carga y muestra el nombre de la app.
- React se comunica con FastAPI a través del proxy en desarrollo (comprobación mínima).
- `.env.example` existe, sin secretos; `.env` no está versionado.
- No hay funcionalidad ECB/Euríbor implementada más allá de placeholders no funcionales si fueran necesarios (evitar).

### Condiciones antes de comenzar la Fase 1
- Fase 0 aceptada/revisada por el usuario.
- Estructura y Compose estables.
- Decisiones técnicas marcadas como necesarias para Fase 1 resueltas (si aplica).

---

## Fase 1. PostgreSQL y backend

### Objetivo
Persistir observaciones mensuales y metadatos de sincronización en PostgreSQL, con acceso desde FastAPI usando `DATABASE_URL`, clave única por periodo mensual, operaciones de inserción/actualización/consulta sin duplicados y endpoint interno de consulta, todo probado.

### Archivos que previsiblemente se crearán o modificarán
- `backend/app/db/*` (conexión/engine/sesión; nombres concretos según enfoque aprobado)
- `backend/app/models/*` o esquema equivalente (observaciones, metadatos de sincronización)
- `backend/app/repositories/*` o `backend/app/services/storage*` (operaciones BD)
- `backend/app/api/*` o rutas (endpoint interno de consulta de datos almacenados)
- `backend/app/core/config*` (lectura `DATABASE_URL`, etc.)
- `backend/requirements.txt` (añadir driver/ORM si se aprueba)
- Scripts/migraciones de esquema (`backend/sql/*` o herramienta elegida) — **decisión pendiente**.
- `backend/tests/*` (conexión, persistencia, consulta, ausencia de duplicados)
- `docker-compose.yml` / `backend/Dockerfile` (si hace falta inicialización o dependencias)

### Tareas concretas (ordenadas)
1. Diseñar esquema mínimo aprobado por alcance:
   - **Observaciones mensuales:** periodo (`TIME_PERIOD`/`YYYY-MM` normalizado) como identificador único, valor numérico, estado/flags relevantes (p. ej. `OBS_STATUS`/`OBS_CONF` si se guardan), marcas temporales técnicas si se decide.
   - **Metadatos de sincronización:** última sincronización correcta con ECB (fecha/hora), y campos mínimos necesarios para TTL/fallback/visualización (evitar sobre-diseño).
2. Implementar inicialización de esquema (migración/SQL al arranque o script explícito) — enfoque a decidir.
3. Implementar conexión FastAPI ↔ PostgreSQL usando **`DATABASE_URL`** como configuración principal.
4. Implementar operaciones: insertar/actualizar observaciones con **upsert por periodo mensual** (sin duplicados), consultar histórico, consultar último valor almacenado, leer/escribir metadatos de sincronización.
5. Exponer endpoint interno (bajo prefijo API acordado) para consultar datos almacenados (p. ej. listado/último/estado), solo para verificación interna de fase; no es el contrato final de UI si se refina en Fase 3, pero debe ser coherente.
6. Añadir pruebas: conexión, inserción, upsert sin duplicados, consulta histórico/último, metadatos.

### Dependencias necesarias y justificación
> No instalar ahora. Confirmar existencia antes de instalar.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| Driver PostgreSQL (`psycopg` v3 recomendado habitualmente, o alternativa aprobada) | Ejecutar SQL/conexión | Requisito PostgreSQL. |
| ORM/migraciones (`SQLAlchemy`/`Alembic`) **opcional** | Modelado/migraciones | Puede simplificar, pero añade peso; **decidir** frente a SQL directo + scripts simples. |
| `pytest`, utilidades de test async/fixtures si hacen falta | Pruebas | Norma de pruebas. |

### Pruebas que deben ejecutarse
- Conexión a BD usando `DATABASE_URL` (en entorno Compose/test).
- Persistencia: guardar observaciones de prueba y leerlas.
- **Sin duplicados:** ejecutar upsert repetido para el mismo periodo; verificar un solo registro.
- Consulta: histórico ordenado y último valor.
- Metadatos: guardar/leer última sincronización correcta.
- Comandos previstos:
  - `docker compose up --build -d db backend`
  - `docker compose exec backend pytest -q` (ruta/flags según estructura)
  - Verificación manual del endpoint interno: `curl -fsS http://localhost:<backend>/api/...`

### Criterios de aceptación
- FastAPI conecta a PostgreSQL con `DATABASE_URL`.
- Existe clave/constraint única por periodo mensual; upserts repetidos no crean duplicados.
- Se pueden consultar observaciones y metadatos almacenados.
- Pruebas de conexión, persistencia, consulta y ausencia de duplicados pasan.

### Condiciones antes de comenzar la Fase 2
- Fase 1 aceptada.
- Esquema y operaciones BD estables.
- Decididos: enfoque de esquema/migraciones y cliente HTTP para ECB (si no se añadió antes).

---

## Fase 2. Integración con ECB y sistema de caché

### Objetivo
Implementar el cliente ECB (CSV) y la lógica de sincronización/caché con TTL 24h por defecto, carga inicial completa, upsert sin duplicados, fallback a datos guardados con aviso ante fallo del ECB, y error controlado cuando no hay datos; todo probado.

### Archivos que previsiblemente se crearán o modificarán
- `backend/app/clients/ecb_client.py` (o nombre equivalente)
- `backend/app/services/sync_service.py` / `cache_service.py` (lógica TTL/caché/sincronización)
- `backend/app/parsers/ecb_csv.py` (parseo/validación CSV) — separar de transporte
- `backend/app/api/*` (endpoint(s) que atienden carga de UI: disparan comprobación TTL/sync según reglas)
- `backend/app/core/config*` (`ECB_BASE_URL`, `ECB_DATAFLOW`, `ECB_SERIES_KEY`, `ECB_FORMAT`, `CACHE_TTL_SECONDS`)
- `backend/tests/test_ecb_parser.py`, `test_ecb_client.py` (con mocks), `test_sync_cache.py`, `test_fallback.py`
- `backend/requirements.txt` (cliente HTTP si falta)

### Tareas concretas (ordenadas)
1. **Antes de implementar:** revisar documentación oficial del ECB Data Portal disponible en ese momento y contrastar con la verificación previa de `docs/project-info.md` (endpoint `/service/data/{ECB_DATAFLOW}/{ECB_SERIES_KEY}`, `csvdata`, parámetros).
2. Implementar construcción de URL: `{ECB_BASE_URL}/service/data/{ECB_DATAFLOW}/{ECB_SERIES_KEY}` con `format=csvdata`.
3. Implementar cliente HTTP ECB (timeout razonable, manejo de errores HTTP/red). **No** llamado desde React.
4. Implementar parseo/validación CSV: extraer `TIME_PERIOD`, `OBS_VALUE`, estado (`OBS_STATUS`/`OBS_CONF` si se usan), normalizar tipos; ignorar/rechazar filas inválidas con criterio claro y trazable.
5. **Carga inicial:** si no hay datos/metadatos suficientes, sincronizar **todo el histórico disponible**.
6. Guardar/actualizar mediante **upsert por periodo mensual**; actualizar metadatos solo tras sincronización correcta.
7. Implementar TTL configurable (`CACHE_TTL_SECONDS`, defecto `86400`): al atender petición relevante, si caché vigente → leer BD sin llamar ECB; si caducada → intentar sincronizar. **Sin jobs en segundo plano.**
8. **Fallo ECB:** si hay datos guardados, devolver datos + indicador/aviso de fallback; si no hay datos, devolver error controlado (código/mensaje coherentes).
9. Asegurar que la respuesta exponga datos necesarios para UI: observaciones históricas, último valor, periodo del último dato, fecha/hora de última sincronización correcta, y estado/aviso.
10. Pruebas de todos los escenarios (con mocks/stubs del cliente ECB; no depender de red en tests unitarios).

### Dependencias necesarias y justificación
> No instalar ahora.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| `httpx` (o `requests`, decisión) | HTTP al ECB | Consumo API externa. `httpx` encaja bien con ecosistema moderno; elegir una y justificar. |
| `pytest`, `respx`/mocks (si `httpx`) o equivalente | Mockear HTTP | Pruebas deterministas sin llamar al ECB real. |
| Librería estándar `csv`/`io` | Parseo CSV | Preferir stdlib si basta (sin dependencia extra). |

### Pruebas que deben ejecutarse
- Parseo CSV válido (estructura ECB confirmada) → observaciones coherentes.
- Parseo/validación de filas inválidas o incompletas (comportamiento definido).
- Upsert repetido con mismos datos/períodos → sin duplicados.
- Caché vigente → no se invoca cliente ECB (mock verifica llamadas).
- Caché caducada → se invoca sincronización.
- Fallo ECB + datos en BD → respuesta con datos y aviso de fallback.
- Fallo ECB + BD vacía → error controlado.
- Sincronización correcta actualiza metadatos de última sincronización.
- Comandos previstos:
  - `docker compose exec backend pytest -q`
  - (Opcional, manual controlado) verificación contra ECB real solo en entorno de desarrollo, sin convertirla en prueba automática frágil.

### Criterios de aceptación
- URL ECB construida con `ECB_BASE_URL`/`ECB_DATAFLOW`/`ECB_SERIES_KEY` y `format=csvdata`.
- Primera sincronización carga histórico completo disponible.
- TTL 24h por defecto configurable; sin jobs en segundo plano.
- Upsert por periodo sin duplicados.
- Fallback funcional con aviso; error controlado sin datos.
- Pruebas automatizadas cubren escenarios clave y pasan.

### Condiciones antes de comenzar la Fase 3
- Fase 2 aceptada.
- Contrato de respuesta backend para la UI definido (campos para histórico, último valor, último periodo, última sincronización, aviso/estado) — **documentar/confirmar antes de implementar UI**.

---

## Fase 3. Frontend React y visualización

### Objetivo
Construir la interfaz React que consume exclusivamente FastAPI, muestra último valor, periodo del último dato, ambas fechas/etiquetas de actualización, gráfico Recharts con todo el histórico, y estados de carga/error/aviso; UI clara, sencilla y responsive.

### Archivos que previsiblemente se crearán o modificarán
- `frontend/src/App.*`, `frontend/src/main.*`
- `frontend/src/api/*` (cliente HTTP al backend propio)
- `frontend/src/components/*` (p. ej. valor actual, metadatos, gráfico, avisos/estados) — dividir si crece
- `frontend/src/hooks/*` (carga de datos) si aporta claridad
- `frontend/src/styles/*` o CSS módulos/simple
- `frontend/vite.config.*` (proxy a FastAPI)
- `frontend/package.json` (añadir `recharts` y tooling de test si aplica)
- `frontend/tests/*` o `frontend/src/**/*.test.*` (según decisión)
- `e2e/*` — solo si se introduce E2E en esta fase (normalmente Fase 4); **no mezclar innecesariamente**.

### Tareas concretas (ordenadas)
1. Configurar/ajustar proxy Vite para que el navegador llame al backend propio (rutas `/api/*` o prefijo acordado). Verificar que **no** hay llamadas a `data-api.ecb.europa.eu` desde el cliente.
2. Implementar cliente API frontend (solo backend propio, HTTP/JSON), manejo de errores y estados.
3. Construir UI principal:
   - Último valor disponible del Euríbor 12M.
   - Periodo del último dato publicado.
   - Fecha/hora de última sincronización correcta con ECB **y** periodo del último dato (ambos visibles/etiquetados).
   - Gráfico de evolución mensual con **Recharts**, **todo el histórico disponible**.
   - Estados: carga, error, aviso cuando se usan datos almacenados por fallo del ECB.
4. Mantener responsive y accesibilidad básica razonable (legibilidad, contraste suficiente, layout adaptable).
5. Separar componentes/lógica de presentación; evitar archivos > 500 líneas.
6. Pruebas de componentes/integración frontend (render, estados, aviso, datos básicos con mocks de API).

### Dependencias necesarias y justificación
> No instalar ahora.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| `recharts` | Gráfico | Aprobado en `docs/project-info.md`. |
| `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom` (o stack equivalente) | Pruebas UI | Norma de pruebas; **decisión pendiente** si no se fijó en Fase 0. |

### Pruebas que deben ejecutarse
- Renderiza nombre/estructura principal.
- Muestra último valor, periodo y ambas fechas cuando la API devuelve datos coherentes (mock).
- Muestra aviso de fallback cuando la API indica uso de datos almacenados por fallo ECB.
- Muestra estados de carga y error.
- Gráfico recibe/renderiza dataset (prueba razonable según tooling; evitar pruebas frágiles de píxeles).
- Comprobación manual/DevTools: ninguna petición del navegador al dominio ECB.
- Comandos previstos:
  - `docker compose up --build -d`
  - `docker compose exec frontend npm test -- --run` (o script acordado)
  - Navegación manual a `http://localhost:<frontend>`.

### Criterios de aceptación
- React consume exclusivamente FastAPI.
- Se muestran último valor, periodo último dato, última sincronización y gráfico histórico completo (Recharts).
- Aviso visible en fallback ECB; errores controlados visibles.
- UI clara, sencilla y responsive.
- Navegador sin llamadas directas al ECB.
- Pruebas frontend pasan.

### Condiciones antes de comenzar la Fase 4
- Fase 3 aceptada.
- Flujo completo usuario visible en entorno Compose.

---

## Fase 4. Integración completa y pruebas

### Objetivo
Validar el sistema completo en Docker Compose (carga inicial, persistencia, caché, fallback, no duplicados, UI) y asegurar pruebas automatizadas/documentación de ejecución.

### Archivos que previsiblemente se crearán o modificarán
- `README.md` (cómo ejecutar proyecto, tests, variables de entorno)
- `backend/tests/*`, `frontend/*` (ajustes de cobertura/estabilidad)
- `e2e/*` (solo si se aprueba/añade Playwright E2E real en esta fase)
- `docker-compose.yml`, Dockerfiles (ajustes menores si son necesarios para pruebas/ejecución; sin rediseño)
- `docs/project-info.md` / `PLAN.md` — solo actualizaciones coherentes si se detecta desviación autorizada (normalmente `PLAN.md` al cerrar).

### Tareas concretas (ordenadas)
1. Verificar arranque conjunto: `docker compose up --build` con `frontend`, `backend`, `db`.
2. Verificar **carga inicial**: primera ejecución sincroniza histórico completo (según BD vacía/metadatos).
3. Verificar **persistencia**: reiniciar contenedores (`down`/`up` conservando volumen) y confirmar datos siguen disponibles.
4. Verificar **caché vigente**: segunda carga dentro de TTL no dispara llamada innecesaria al ECB (observable por logs/control o mecanismo de diagnóstico interno acordado).
5. Verificar **caché caducada**: forzar expiración (`CACHE_TTL_SECONDS` bajo en `.env` de prueba local o manipulación controlada en entorno de test) y confirmar intento de sincronización.
6. Simular **fallo ECB** (mock en tests; y/o bloqueo controlado en entorno local si se diseña) y comprobar fallback a PostgreSQL con aviso; comprobar caso sin datos → error controlado.
7. Comprobar **ausencia de duplicados** tras sincronizaciones repetidas.
8. Verificar UI: datos, gráfico, aviso, carga/error.
9. Ejecutar todas las pruebas automatizadas backend y frontend (y E2E si aplica).
10. Corregir problemas encontrados (con prueba cuando corresponda) y documentar ejecución en `README.md`.
11. Preparar resumen final de verificación para el usuario.

### Dependencias necesarias y justificación
> No instalar ahora.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| Playwright (`@playwright/test`) | E2E real navegador | Útil en Fase 4; **decisión pendiente** (coste/configuración). Alternativa: E2E manual documentado + pruebas integración frontend/backend. |

### Pruebas que deben ejecutarse
- Suite backend completa.
- Suite frontend completa.
- E2E (si se adopta) o checklist manual reproducible documentado.
- Verificaciones de integración con Compose (arranque, reinicio, persistencia, caché, fallback).
- Comandos previstos:
  - `docker compose up --build -d`
  - `docker compose ps`
  - `docker compose exec backend pytest -q`
  - `docker compose exec frontend npm test -- --run`
  - `docker compose restart backend frontend` / `docker compose down && docker compose up -d` (según prueba)
  - `npx playwright test` (solo si se añade Playwright y se autoriza su configuración)

### Criterios de aceptación
- Los tres servicios funcionan juntos en Docker Compose.
- Carga inicial, persistencia, caché y fallback verificados.
- Sin duplicados en observaciones.
- UI muestra datos/gráfico/estados correctamente.
- Pruebas automatizadas pasan (y E2E si aplica).
- `README.md` documenta ejecución y pruebas.

### Condiciones antes de dar el proyecto por cerrado (a nivel de este plan)
- Fase 4 aceptada por el usuario.
- Cualquier commit/push realizado solo con autorización explícita.
- Sin acciones de producción.

---

## Decisiones técnicas pendientes de aprobación (antes de implementar la fase correspondiente)

1. **Stack de pruebas frontend (Fase 0/3):** ¿Vitest + React Testing Library (habitual con Vite) u otra opción? Definir scripts (`npm test`) y configuración.
2. **Cliente HTTP backend (Fase 2, o Fase 0 si se adelanta):** `httpx` vs `requests`. Recomendación técnica habitual: `httpx` (API moderna, buen soporte async/mocks), pero requiere aprobación.
3. **Acceso a BD (Fase 1):** SQL directo + scripts/SQL init vs SQLAlchemy (+ Alembic). La especificación pide “diseñar estructura/implementar conexión/modelo/operaciones”; no fija ORM. **Recomendación para simplicidad:** SQL claro con driver `psycopg` y esquema controlado, evitando ORM pesado salvo necesidad.
4. **Inicialización de esquema (Fase 1):** SQL en arranque controlado, script idempotente, o migraciones. Definir para no duplicar lógica.
5. **Contrato API backend↔frontend (antes de Fase 3):** rutas exactas, forma de respuesta (campos: `latest`, `observations`, `last_sync_at`, `last_data_period`, `warning/source`, etc.), códigos de error. Debe respetar requisitos funcionales sin añadir features.
6. **E2E con Playwright (Fase 4):** ¿se configura Playwright completo en este proyecto o se valida con pruebas de integración + checklist manual? `e2e/` existe pero está vacío.
7. **Modo sync/async en FastAPI para BD (Fase 1/2):** coherente con driver elegido (`psycopg` async o sync). Decisión de implementación a fijar al diseñar Fase 1.

## Resumen operativo
- **Trabajo:** fase por fase, probar y revisar antes de continuar.
- **Autorización actual:** solo creación/actualización de `PLAN.md` (cumplida). Siguiente paso requiere instrucción explícita del usuario (p. ej. aprobar plan e iniciar Fase 0).
- **Commits/push:** locales solo cuando se autorice; **nunca push**.