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
- `docs/project-info.md` — **no modificar**.
- `PLAN.md` — marcar tareas/cierre solo cuando se autorice.

### Tareas concretas (ordenadas)
1. Definir estructura de carpetas: `backend/`, `frontend/`, `backend/tests/`, `frontend/src/...` (y estructura de tests frontend según Vitest). Mantener separación de responsabilidades.
2. **Backend (mínimo):** crear app FastAPI con endpoint de salud `GET /api/health` que responda JSON simple; sin lógica ECB/BD funcional del dominio.
3. **Frontend (mínimo):** scaffold React + Vite; pantalla inicial con nombre “Panel de indicadores hipotecarios” y comprobación visual mínima; configurar proxy de Vite hacia FastAPI para el prefijo `/api/`.
4. **PostgreSQL:** servicio `db` en Compose con imagen oficial, **volumen persistente** para datos, variables `POSTGRES_DB/POSTGRES_USER/POSTGRES_PASSWORD` desde `.env`.
5. **Dockerfiles:** uno para `backend` (Python, instala dependencias, ejecuta servidor dev) y otro para `frontend` (Node, instala dependencias, ejecuta Vite en modo dev accesible desde host). Mantener simples.
6. **docker-compose.yml:** servicios `frontend`, `backend`, `db`; red interna; `backend` accesible para proxy/frontend; `db` accesible para `backend`; puertos expuestos solo los necesarios para desarrollo; dependencias de arranque razonables (`depends_on`; healthcheck de `db` si se decide).
7. **Variables de entorno:** actualizar `.env.example` con nombres aprobados (`DATABASE_URL`, `POSTGRES_*`, `ECB_*`, `CACHE_TTL_SECONDS`, proxy Vite) sin valores reales. Documentar brevemente en comentarios si ayuda.
8. **Comprobación explícita:** arrancar Compose; verificar que `db` acepta conexiones, `backend` responde a `GET /api/health`, `frontend` carga y muestra el nombre; **comprobar explícitamente la comunicación React/Vite → proxy `/api/health` → FastAPI** (petición desde el navegador/pantalla o mecanismo equivalente que atraviese el proxy).
9. **Pruebas mínimas de fase:** añadir pruebas automáticas simples: backend (endpoint `/api/health`, respuesta 200 y JSON esperado) con `pytest`; frontend (renderiza “Panel de indicadores hipotecarios”) con **Vitest + React Testing Library**. Ejecutables en contenedor o localmente según configuración.

### Dependencias necesarias y justificación
> **No instalar ahora.** Antes de cada instalación: confirmar nombre, versión/compatibilidad y existencia. No introducir dependencias adicionales sin justificarlas.

| Área | Dependencia (prevista) | Para qué | Justificación |
|---|---|---|---|
| Backend | `fastapi` | Framework API | Requisito aprobado. |
| Backend | `uvicorn[standard]` o `uvicorn` | Servidor ASGI en desarrollo | Ejecutar FastAPI. |
| Backend | `httpx` | Cliente HTTP ECB | Aprobado; se incorpora en la **Fase 2** (no antes, salvo necesidad justificada). |
| Backend | `psycopg` v3 (`psycopg[binary]` o variante aprobada) | Conexión PostgreSQL | Aprobado; acceso **síncrono** inicial con SQL directo, sin SQLAlchemy. Se incorpora en la **Fase 1**. |
| Backend | `pytest` (+ plugins mínimos si hacen falta) | Pruebas backend | Norma “todo va probado”. |
| Frontend | React, React DOM, Vite, plugin React (scaffold) | SPA y dev server | Requisitos aprobados. |
| Frontend | `recharts` | Gráfico | Aprobado; se incorpora en la **Fase 3**. |
| Frontend | `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom` (o equivalentes mínimos) | Pruebas frontend | **Decisión cerrada:** Vitest + React Testing Library. |
| Infra | Imágenes Docker oficiales `python`, `node`, `postgres` | Contenedores | Requisito Docker Compose. PostgreSQL con **volumen persistente**. |

### Pruebas que deben ejecutarse
- Backend: prueba automática de `GET /api/health` (respuesta 200 y estructura JSON esperada).
- Frontend: prueba automática (Vitest + RTL) que la pantalla muestra “Panel de indicadores hipotecarios”.
- Integración manual guiada: servicios arriba, puertos accesibles.
- **Comprobación explícita:** comunicación **React/Vite → proxy `/api/health` → FastAPI** (la petición debe atravesar el proxy de Vite).
- Comandos previstos (ajustar a nombres reales de scripts/servicios):
  - `docker compose config`
  - `docker compose up --build -d`
  - `docker compose ps`
  - `docker compose logs --tail=50 backend frontend db`
  - `curl -fsS http://localhost:<puerto-backend>/api/health`
  - `curl -fsS http://localhost:<puerto-frontend>/`
  - `docker compose exec backend pytest -q` (o equivalente)
  - `docker compose exec frontend npm test -- --run` (o script Vitest equivalente)

### Criterios de aceptación
- `docker compose up --build` levanta `frontend`, `backend` y `db` sin errores de arranque.
- `GET /api/health` responde correctamente desde FastAPI.
- El frontend carga y muestra el nombre de la app.
- Se verifica explícitamente la comunicación React/Vite → proxy `/api/health` → FastAPI.
- PostgreSQL usa volumen persistente.
- `.env.example` existe, sin secretos; `.env` no está versionado.
- No hay funcionalidad ECB/Euríbor implementada.

### Condiciones antes de comenzar la Fase 1
- Fase 0 aceptada/revisada por el usuario.
- Estructura y Compose estables.
- Autorización expresa del usuario para iniciar la Fase 1.

---

## Fase 1. PostgreSQL y backend

### Objetivo
Persistir observaciones mensuales y metadatos de sincronización en PostgreSQL, con acceso desde FastAPI usando `DATABASE_URL`, clave única por periodo mensual, operaciones de inserción/actualización/consulta sin duplicados y endpoint interno de consulta, todo probado.

### Archivos que previsiblemente se crearán o modificarán
- `backend/app/db/*` (conexión con `psycopg` v3, ejecución SQL; nombres concretos al implementar)
- `backend/app/repositories/*` o `backend/app/services/storage*` (operaciones BD, SQL directo)
- `backend/app/api/*` o rutas (endpoint interno bajo `/api/` para consultar datos almacenados)
- `backend/app/core/config*` (lectura `DATABASE_URL`, etc.)
- `backend/sql/init.sql` (o script equivalente **idempotente**) para inicialización de esquema
- `backend/requirements.txt` (añadir `psycopg` v3 si no está)
- `backend/tests/*` (conexión, persistencia, consulta, ausencia de duplicados)
- `docker-compose.yml` / `backend/Dockerfile` (si hace falta inicialización o dependencias)

### Tareas concretas (ordenadas)
1. Diseñar esquema mínimo aprobado por alcance:
   - **Observaciones mensuales:** periodo (`TIME_PERIOD`/`YYYY-MM` normalizado) como identificador único, valor numérico, estado/flags relevantes (p. ej. `OBS_STATUS`/`OBS_CONF` si se guardan), marcas temporales técnicas mínimas si se decide.
   - **Metadatos de sincronización:** última sincronización correcta con ECB (fecha/hora), y campos mínimos necesarios para TTL/fallback/visualización (evitar sobre-diseño).
2. Implementar inicialización de esquema con **script SQL idempotente** (sin Alembic inicialmente). Definir cuándo/cómo se ejecuta (arranque controlado o mecanismo explícito) sin duplicar lógica.
3. Implementar conexión FastAPI ↔ PostgreSQL usando **`DATABASE_URL`** con **`psycopg` v3** y **acceso síncrono** inicial (implementación sencilla). SQL directo, **sin SQLAlchemy**.
4. Implementar operaciones: insertar/actualizar observaciones con **upsert por periodo mensual** (sin duplicados), consultar histórico, consultar último valor almacenado, leer/escribir metadatos de sincronización.
5. Exponer endpoint interno bajo prefijo `/api/` (respuesta JSON) para consultar datos almacenados (p. ej. listado/último/estado), solo para verificación interna de fase; no es el contrato final de UI si se refina antes de Fase 3.
6. Añadir pruebas: conexión, inserción, upsert sin duplicados, consulta histórico/último, metadatos.

### Dependencias necesarias y justificación
> No instalar ahora. Confirmar existencia antes de instalar. No añadir dependencias adicionales sin justificarlas.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| `psycopg` v3 (`psycopg[binary]` o variante aprobada) | Acceso PostgreSQL con SQL directo | **Decisión cerrada:** sin SQLAlchemy; acceso síncrono inicial para simplicidad. |
| `pytest` (+ plugins mínimos si hacen falta) | Pruebas | Norma de pruebas. |

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
- FastAPI conecta a PostgreSQL con `DATABASE_URL` usando `psycopg` v3 (acceso síncrono inicial).
- Esquema inicializado mediante script SQL idempotente; sin Alembic/SQLAlchemy.
- Existe clave/constraint única por periodo mensual; upserts repetidos no crean duplicados.
- Se pueden consultar observaciones y metadatos almacenados vía endpoint interno `/api/` (JSON).
- Pruebas de conexión, persistencia, consulta y ausencia de duplicados pasan.

### Condiciones antes de comenzar la Fase 2
- Fase 1 aceptada.
- Esquema y operaciones BD estables.
- Autorización expresa del usuario para iniciar la Fase 2.

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
> No instalar ahora. Confirmar existencia antes de instalar.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| `httpx` | Cliente HTTP al ECB | **Decisión cerrada:** se incorpora en la **Fase 2**. |
| `pytest`, utilidades de mock HTTP compatibles con `httpx` (p. ej. `respx`) si hacen falta | Mockear HTTP | Pruebas deterministas sin llamar al ECB real. Solo añadir si se justifica. |
| Librería estándar `csv`/`io` | Parseo CSV | Preferir stdlib (sin dependencia extra). |

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
- Contrato API definido y aprobado: prefijo `/api/`, respuestas JSON, rutas y campos concretos (histórico, último valor, periodo del último dato, última sincronización, aviso/fallback, errores) — **antes de implementar la Fase 3**.
- Autorización expresa del usuario para iniciar la Fase 3.

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
> No instalar ahora. Confirmar existencia antes de instalar.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| `recharts` | Gráfico | Aprobado en `docs/project-info.md`. |
| `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom` (o equivalentes mínimos) | Pruebas UI | **Decisión cerrada:** Vitest + React Testing Library. |

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
- Autorización expresa del usuario para iniciar la Fase 4.

---

## Fase 4. Integración completa y pruebas

### Objetivo
Validar el sistema completo en Docker Compose (carga inicial, persistencia, caché, fallback, no duplicados, UI) y asegurar pruebas automatizadas/documentación de ejecución.

### Archivos que previsiblemente se crearán o modificarán
- `README.md` (cómo ejecutar proyecto, tests, variables de entorno)
- `backend/tests/*`, `frontend/*` (ajustes de cobertura/estabilidad)
- `e2e/*` — solo si más adelante aparece una necesidad justificada y aprobada (Playwright **no** se incorpora inicialmente).
- `docker-compose.yml`, Dockerfiles (ajustes menores si son necesarios para pruebas/ejecución; sin rediseño)
- `docs/project-info.md` — **no modificar** salvo corrección autorizada.
- `PLAN.md` — actualizar al cerrar fases cuando se autorice.

### Tareas concretas (ordenadas)
1. Verificar arranque conjunto: `docker compose up --build` con `frontend`, `backend`, `db`.
2. Verificar **carga inicial**: primera ejecución sincroniza histórico completo (según BD vacía/metadatos).
3. Verificar **persistencia**: reiniciar contenedores (`down`/`up` conservando volumen) y confirmar datos siguen disponibles.
4. Verificar **caché vigente**: segunda carga dentro de TTL no dispara llamada innecesaria al ECB (observable por logs/control o mecanismo de diagnóstico interno acordado).
5. Verificar **caché caducada**: forzar expiración (`CACHE_TTL_SECONDS` bajo en `.env` de prueba local o manipulación controlada en entorno de test) y confirmar intento de sincronización.
6. Simular **fallo ECB** (mock en tests; y/o bloqueo controlado en entorno local si se diseña) y comprobar fallback a PostgreSQL con aviso; comprobar caso sin datos → error controlado.
7. Comprobar **ausencia de duplicados** tras sincronizaciones repetidas.
8. Verificar UI: datos, gráfico, aviso, carga/error.
9. Ejecutar todas las pruebas automatizadas backend y frontend, más comprobaciones manuales de integración.
10. Corregir problemas encontrados (con prueba cuando corresponda) y documentar ejecución en `README.md`.
11. Preparar resumen final de verificación para el usuario.

### Dependencias necesarias y justificación
> No instalar ahora. **Playwright no se incorpora inicialmente.** Solo se propondrá si aparece una necesidad justificada y se aprueba expresamente.

| Dependencia (prevista) | Para qué | Justificación |
|---|---|---|
| *(ninguna adicional prevista)* | — | Cobertura con `pytest` + Vitest/RTL + comprobaciones manuales de integración. |

### Pruebas que deben ejecutarse
- Suite backend completa (`pytest`).
- Suite frontend completa (Vitest + RTL).
- Checklist manual reproducible de integración (Compose, carga inicial, persistencia, caché, fallback, duplicados, UI).
- Comandos previstos:
  - `docker compose up --build -d`
  - `docker compose ps`
  - `docker compose exec backend pytest -q`
  - `docker compose exec frontend npm test -- --run`
  - `docker compose restart backend frontend` / `docker compose down && docker compose up -d` (según prueba)

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

## Decisiones técnicas cerradas

1. **Pruebas frontend:** Vitest + React Testing Library (Fase 0/3).
2. **Cliente HTTP para ECB:** `httpx`, incorporado en la **Fase 2**.
3. **Acceso a PostgreSQL:** SQL directo con `psycopg` v3, **sin SQLAlchemy**.
4. **Inicialización de esquema:** **script SQL idempotente**, evitando Alembic inicialmente.
5. **Contrato API:** prefijo `/api/`, respuestas JSON. Rutas y campos concretos se definen y aprueban **antes de implementar la Fase 3**.
6. **Playwright:** **no se incorpora inicialmente**. Se usan pruebas automatizadas de backend/frontend y comprobaciones manuales de integración. Solo se propondrá Playwright si aparece una necesidad justificada y aprobada.
7. **FastAPI + PostgreSQL:** acceso **síncrono** inicial con `psycopg` v3 para mantener una implementación sencilla.

## Condiciones transversales reflejadas en el plan
- En la Fase 0 se comprueba explícitamente **React/Vite → proxy `/api/health` → FastAPI**.
- PostgreSQL usa **volumen persistente**.
- Se mantienen las normas de `AGENTS.md`.
- No se introducen dependencias adicionales sin justificarlas.
- `docs/project-info.md` no se modifica.

## Resumen operativo
- **Trabajo:** fase por fase, probar y revisar antes de continuar.
- **Autorización actual:** solo actualización de `PLAN.md` (cumplida). **Todavía no hay autorización para iniciar la Fase 0.**
- **Siguiente paso:** esperar autorización expresa del usuario para comenzar la implementación de la Fase 0.
- **Commits/push:** locales solo cuando se autorice; **nunca push**.