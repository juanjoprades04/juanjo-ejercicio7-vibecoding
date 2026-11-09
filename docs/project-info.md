# Información del proyecto

## Qué es
«Panel de indicadores hipotecarios»: una aplicación web que consulta el Euríbor a 12 meses desde la API oficial del Banco Central Europeo (ECB Data Portal), muestra su último valor disponible y visualiza su evolución histórica.

## Para quién
Personas que quieren consultar de forma rápida y fiable el valor y la evolución del Euríbor a 12 meses (referencia habitual en hipotecas variables) sin consultar directamente fuentes técnicas.

## Funciones
- Consultar y mostrar el **último valor disponible** del Euríbor a 12 meses.
- Consultar y mostrar sus **observaciones históricas**.
- Mostrar la evolución mediante un **gráfico**.
- **Guardar las observaciones en PostgreSQL**.
- Usar los datos almacenados como **caché** para evitar llamadas innecesarias a la API del ECB.
- Si la API externa falla, mostrar un **aviso** y utilizar los datos guardados cuando estén disponibles.
- Mostrar la **fecha de la última actualización**.
- **React se comunica exclusivamente con FastAPI**; el navegador nunca llama al ECB.

## Fuera de alcance
- Otros indicadores además del Euríbor a 12 meses (salvo ampliación futura).
- Cuentas de usuario, autenticación o perfiles.
- Alertas, notificaciones o suscripciones.
- Cálculo de cuotas hipotecarias, simuladores u ofertas bancarias.
- Edición manual de datos por parte del usuario.
- Despliegue a producción sin permiso explícito (según `AGENTS.md`).

## Tecnología
- **Frontend:** React con Vite (entorno de desarrollo). Se comunica solo con el backend propio vía HTTP/JSON (proxy de Vite hacia FastAPI). Gráficos con **Recharts**.
- **Backend:** Python + FastAPI. Expone la API interna, consulta el ECB, aplica la lógica de caché y accede a PostgreSQL.
- **Base de datos:** PostgreSQL. Almacena observaciones y metadatos de sincronización.
- **Contenedores:** Docker y Docker Compose, **único entorno de desarrollo** (servicios: `frontend`, `backend`, `db`). Sin Nginx/producción por ahora.
- **Fuente externa:** ECB Data Portal (API SDMX), consumida solo desde FastAPI, formato **`csvdata`**.
- **Dependencias:** evitar dependencias innecesarias; solo se añadirán cuando hagan falta y tras comprobar que existen (según `AGENTS.md`).

## Fuente de datos (ECB) — confirmación previa a implementación
**Serie objetivo:** Euríbor a 1 año (12 meses), cierre histórico / media de observaciones del periodo.  
**Identificador propuesto (confirmado):** `FM.M.U2.EUR.RT.MM.EURIBOR1YD_.HSTA`  
- Dataflow: `FM` (Financial market data).  
- Dimensiones: frecuencia `M` (mensual), área `U2`, moneda `EUR`, proveedor `RT` (Refinitiv), instrumento `MM`, identificador `EURIBOR1YD_`, tipo `HSTA` (Historical close, average of observations through period).  
- Unidad: `%` anual (`PCPA`).

**Endpoint confirmado (API en vivo):**  
`https://data-api.ecb.europa.eu/service/data/FM/M.U2.EUR.RT.MM.EURIBOR1YD_.HSTA`

**Formato elegido para el cliente backend:** `format=csvdata` (CSV), por simplicidad de procesamiento en Python. El servicio también soporta `jsondata`; se usará CSV salvo necesidad justificada.

**Parámetros útiles confirmados empíricamente:**
- `format=csvdata` → CSV con columnas como `TIME_PERIOD`, `OBS_VALUE`, `OBS_STATUS`, `OBS_CONF`, etc.
- `lastNObservations=N` → últimas N observaciones.
- `startPeriod=YYYY-MM` / `endPeriod=YYYY-MM` → rango temporal (frecuencia mensual).

**Frecuencia de publicación:** la serie es **mensual** (`M`, `P1M`), media de observaciones del periodo. La disponibilidad del último mes cerrado depende del calendario de publicación del ECB/proveedor; la app la trata como mensual y no asume día/hora fija.

> Nota: antes de implementar el cliente ECB debe revisarse de nuevo la documentación oficial disponible. En esta fase previa, el endpoint, parámetros y formato se confirmaron además contra la respuesta real del servicio (`data-api.ecb.europa.eu`); las páginas de ayuda devolvían 503 en el momento de la verificación.

## Arquitectura (decidida, sencilla y separada por responsabilidades)
```text
Navegador (React + Vite, dev)
   │  HTTP/JSON vía proxy de Vite (solo a backend propio)
   ▼
FastAPI (backend)
   ├─ API interna (endpoints)
   ├─ servicio de sincronización/caché
   ├─ cliente ECB (solo servidor, CSV)
   └─ acceso a datos (PostgreSQL)
          │
          ▼
   ECB Data Portal SDMX API

PostgreSQL (db) ← observaciones + metadatos de sincronización
```

**Responsabilidades:**
- **React:** UI (último valor, gráfico con **Recharts**, aviso de fallo/uso de caché, fechas de actualización), estado de carga/error. Sin lógica de acceso al ECB.
- **FastAPI:** expone la API consumida por React; decide si la caché está vigente; sincroniza con el ECB cuando hace falta; parsea `csvdata`; guarda/lee en PostgreSQL; ante fallo del ECB usa datos almacenados si existen y devuelve aviso.
- **PostgreSQL:** persistencia de observaciones (una por periodo mensual, sin duplicados) y metadatos (última sincronización correcta, etc.).
- **Docker Compose (entorno único de desarrollo):** servicios `frontend`, `backend`, `db` en red interna. `frontend` usa Vite con proxy hacia FastAPI. No se configura Nginx/producción en esta etapa.

## Comportamiento de caché y sin conexión (decidido)
- **Almacenamiento:** cada observación mensual se guarda en PostgreSQL (`TIME_PERIOD` normalizado, valor numérico, estado). Se guarda la **fecha/hora de última sincronización correcta** con el ECB.
- **Carga inicial:** en la primera ejecución se sincroniza **todo el histórico disponible** desde el ECB.
- **Regla de caché:** **TTL configurable, por defecto 24 horas**. Al cargar la aplicación (petición atendida por FastAPI), el backend comprueba si la caché ha caducado. Si está vigente y hay datos, responde desde BD sin llamar al ECB. Si ha caducado, intenta actualizar desde el ECB. **No hay jobs en segundo plano.**
- **Sin duplicados:** la sincronización usa `upsert`/clave única por periodo mensual; nunca debe generar registros duplicados en PostgreSQL.
- **ECB disponible:** obtiene datos en `csvdata`, actualiza/mezcla observaciones por periodo, actualiza metadatos y responde con datos e información de actualización.
- **ECB falla** (timeout, error HTTP, red, respuesta no válida):
  - **Hay datos guardados:** la aplicación sigue funcionando. FastAPI responde con datos de BD y un **aviso claro** para el usuario indicando que no se pudo actualizar desde el ECB y que se muestran datos almacenados.
  - **No hay datos guardados:** responde error controlado con mensaje claro; la UI muestra el fallo sin romperse.
- **React:** nunca llama al ECB; solo muestra datos, aviso y fechas devueltos por FastAPI.

## Datos y claves
Datos: observaciones del Euríbor 12M (periodo mensual, valor, estado) y metadatos de sincronización. No hay datos personales.

Claves/configuración (solo nombres; valores reales en `.env`, nunca en git). Previsión inicial (ajustable al implementar, sin añadir secretos al código):
- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_HOST` (p. ej. `db` en Compose)
- `POSTGRES_PORT`
- `DATABASE_URL` (o construcción equivalente en backend)
- `ECB_BASE_URL` (por defecto `https://data-api.ecb.europa.eu`)
- `ECB_SERIES_KEY` (por defecto `FM/M.U2.EUR.RT.MM.EURIBOR1YD_.HSTA`)
- `ECB_FORMAT` (por defecto `csvdata`)
- `CACHE_TTL_SECONDS` (por defecto `86400`, 24 horas)
- `VITE_API_PROXY_TARGET` o equivalente para el proxy de Vite hacia FastAPI en desarrollo (nombre concreto a fijar al implementar)

Mantener `.env` fuera de git (`.gitignore`) y crear `.env.example` sin credenciales reales.

## Cómo se prueba
Caso concreto (dato real confirmado en la API ECB durante la verificación):
- Serie: `FM.M.U2.EUR.RT.MM.EURIBOR1YD_.HSTA`, `lastNObservations=1`, `format=csvdata` (o equivalente).
- Resultado esperado (verificado el 2026-11-09): periodo `2026-10`, valor `3.2385909` (% anual), estado normal.
- En la UI: se muestra ese valor como último disponible (si está en BD o tras sincronización), un gráfico de evolución mensual con **todo el histórico disponible** (Recharts), y **ambas** fechas/etiquetas: fecha/hora de la última sincronización con el ECB y periodo del último dato publicado.

Pruebas previstas (por fases en `PLAN.md`): parseo CSV ECB, `upsert` por periodo sin duplicados, caché vigente evita llamada ECB, caché caducada dispara actualización al cargar, fallo ECB + datos en BD devuelve datos y aviso, fallo ECB sin BD devuelve error controlado, React renderiza último valor/gráfico/aviso/ambas fechas.

## Decisiones cerradas
1. **ECB:** `csvdata`.
2. **Caché:** TTL configurable (`86400` s / 24 h por defecto). FastAPI comprueba caducidad al cargar; si caduca intenta actualizar; **sin jobs en segundo plano**. Si la API externa falla, usa datos guardados y avisa al usuario.
3. **Histórico UI:** todo el histórico disponible desde el principio.
4. **Gráfico:** Recharts.
5. **Actualización mostrada:** ambas: fecha/hora de última sincronización con el ECB y periodo del último dato publicado.
6. **Docker:** único entorno de desarrollo con Docker Compose (React + Vite con proxy hacia FastAPI). Sin Nginx/producción por ahora.
7. **Carga inicial:** sincronizar todo el histórico disponible en la primera ejecución.

**Condiciones adicionales acordadas:** arquitectura sencilla y separada por responsabilidades; evitar dependencias innecesarias; comprobar documentación oficial del ECB antes de implementar el cliente; la sincronización no debe generar duplicados en PostgreSQL; si la API externa falla, la app debe seguir funcionando con datos almacenados cuando existan.
