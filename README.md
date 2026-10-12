# ValidadorSAZ · CRUD de catálogos (v2)

Spring Boot 2.7 (Java 8) + JPA sobre Oracle 11g (esquema NSAZ) + interfaz en TypeScript sin framework.
Se empaqueta como WAR para WebLogic 12c.

## Estructura

```
sql/                      01_ddl, 02_seed (datos reales de los 6 catálogos), 99_rollback
src/main/java/.../common  CatalogEntity, NotFoundException, ApiExceptionHandler (traduce ORA-xxxxx)
.../catalogo/entity       6 entidades JPA con validaciones
.../catalogo/repository   6 repositorios JpaRepository
.../catalogo/service      CrudService genérico + EndpointService + NodoService
.../catalogo/controller   CrudController genérico + 6 controladores REST
src/main/resources/static index.html, css/, js/ (compilado desde frontend/)
src/main/webapp/WEB-INF   weblogic.xml (context-root y prefer-application-packages)
frontend/src              api.ts, config.ts, app.ts
```

## Pasos

1. Base de datos, en este orden (requiere CREATE TABLE y CREATE SEQUENCE en NSAZ):
   `01_ddl_catalogos.sql`, `02_seed_catalogos.sql`, `03_cat_endpoint_curls.sql`, `04_ecosistema_nodos.sql`.
   Si ya habías ejecutado 01 y 02, solo corre 03 y 04.
2. Conexión (local y WebLogic usan la misma base FreeSQL, definida en `application.properties`):
   - Variable obligatoria: `DB_PASSWORD`. Opcionales: `DB_URL`, `DB_USER`, `DB_SCHEMA`.
   - Local: variables de entorno en la configuración de ejecución del IDE.
   - WebLogic: ver la sección "Despliegue en WebLogic".
   Ninguna contraseña va en el código ni en `static/`.
3. Interfaz (solo si cambias el TypeScript): `cd frontend && npm install && npm run build`.
   El JS compilado ya viene incluido en `static/js`.
4. Backend: `mvn clean package` genera `target/ValidadorSAZCatalogos.war`.
5. Abre `http://servidor:puerto/ValidadorSAZCatalogos/`.

## API REST

| Catálogo            | Ruta base                      |
|---------------------|--------------------------------|
| Aplicaciones        | /api/aplicaciones              |
| Endpoints           | /api/endpoints                 |
| Nodos               | /api/nodos                     |
| Tipos de prueba     | /api/tipos-prueba              |
| Sistemas operativos | /api/sistemas-operativos       |
| Tipos de documento  | /api/tipos-documento           |
| Ecosistemas         | /api/ecosistemas               |
| Generador (lectura) | /api/generador/catalogos       |

Cada ruta acepta `GET` (lista), `GET /{id}`, `POST`, `PUT /{id}` y `DELETE /{id}`.
Extra: `GET /api/endpoints/por-aplicacion?idAplicacion=5`.

`GET /api/generador/catalogos` regresa en una sola respuesta (y una sola transacción de lectura)
ecosistemas, aplicaciones, endpoints, nodos y tipos de prueba activos. Es lo que usa el Generador.

## Reglas de negocio incluidas

- `CAT_ENDPOINT.ID_TEMPLATE` siempre se copia del aplicativo al guardar; no se captura.
- `CAT_NODO`: el cluster se limpia en BALANCEO y DEV y es obligatorio en PRINCIPAL y NUEVA
  (también lo valida el CHECK de Oracle). `CLUSTER` es palabra reservada en Oracle,
  por eso la columna se llama `NUM_CLUSTER`; en el JSON sigue siendo `cluster`.
- Endpoints únicos por aplicativo + ruta + método.
- Errores de Oracle (llave duplicada, registro en uso, CHECK) regresan HTTP 409 con mensaje en español.
- Las secuencias inician después de los datos semilla (aplicación 76, endpoint 14, nodo 61...).

## Novedades v2

- Menú lateral agrupado: Aplicaciones (aplicaciones, endpoints), Infraestructura (ecosistemas, nodos),
  Configuración (tipos de prueba, sistemas operativos, tipos de documento) y Herramientas
  (enlace al Generador de validaciones, incluido en `static/`).
- Nuevo catálogo `CAT_ECOSISTEMA`; el ID_IEECO lo captura el usuario (6235 = Granja de Seguros,
  -1 = Balanceo, -2 = Desarrollo).
- `CAT_NODO` con `ID_IEECO` (FK). BALANCEO usa ID y cluster -1; DEV usa ID y cluster -2.
- `CAT_ENDPOINT` con `HEADERS_JSON` y `BODY_JSON`; la API valida que sean JSON y los guarda compactos.
- Botón "Llenar desde un CURL" en Endpoints: pegas el curl y llena aplicativo, ruta, método,
  tipo, headers y body (omite el header Cookie).

## Novedades v2.1 — Generador conectado a Oracle

- El Generador de validaciones (`GeneradorValidacionesSAZ.html`) toma todo de Oracle vía
  `/api/generador/catalogos`: ecosistemas, aplicaciones, endpoints (con headers y body),
  granjas/clusters/nodos, balanceo, nodo DEV y el combo **Tipo de prueba** (`CAT_TIPO_PRUEBA`).
- Se quitaron los datos fijos que traía la página. Si la API no responde, se muestra el error
  y los botones de generar quedan deshabilitados (antes usaba datos viejos sin avisar claramente).
- Errores de acceso a Oracle (ORA-00942, ORA-01017…) regresan HTTP 503 con mensaje en español.
- Se eliminaron `static/application-local.properties` (se publicaba en el WAR con la contraseña)
  y `OracleConnection.kt`.

## Despliegue en WebLogic (conexión a FreeSQL)

1. Define `DB_PASSWORD` para el servidor administrado, de una de estas formas:
   - `$DOMAIN_HOME/bin/setUserOverrides.sh` (se crea si no existe):
     `export DB_PASSWORD='tu_contraseña'`
   - o en la consola: Servidores → tu servidor → Inicio del servidor → Argumentos:
     `-DDB_PASSWORD=tu_contraseña` (solo aplica si el servidor arranca con Node Manager).
2. Reinicia el servidor administrado y despliega `ValidadorSAZCatalogos.war`.
3. Verifica `http://servidor:puerto/ValidadorSAZCatalogos/api/generador/catalogos`.

Requisitos de red: el servidor de WebLogic necesita salida a `db.freesql.com:2484` (TCPS).
Errores frecuentes: `IO Error: The Network Adapter could not establish the connection`
(firewall/proxy), `PKIX path building failed` o `ORA-29024` (el JDK no confía en el certificado:
actualiza el JDK o importa la CA en su `cacerts`).

## Ejecutar en local desde PowerShell

La variable se define en la MISMA ventana donde corres Maven (no hereda la de IntelliJ):

```powershell
$env:DB_PASSWORD = 'tu_contraseña'        # comillas simples: el # y el $ no se interpretan
mvn -s C:\Proyectos\settings-central.xml clean spring-boot:run
```

Protección contra bloqueo: si Oracle rechaza la contraseña (ORA-01017) o la cuenta está bloqueada
(ORA-28000), la aplicación deja de intentar conectarse hasta que la reinicies. Así un error de
contraseña ya no bloquea la cuenta por reintentos de cada pantalla.

## Catálogo de CURLs en CAT_ENDPOINT (sql/05_sync_endpoints_curls.sql)

Carga o actualiza en `CAT_ENDPOINT` los endpoints de `CatalogoCURLS-EndPointsSeguros.txt` (15 al día de hoy).

- Se puede ejecutar cuantas veces sea necesario: inserta los nuevos y actualiza los existentes
  (llave: aplicativo + método + ruta). Busca el aplicativo por nombre, no por ID.
- No lleva prefijo de esquema. En FreeSQL corre tal cual; en el ambiente corporativo antes:
  `ALTER SESSION SET CURRENT_SCHEMA = NSAZ;`
- Agrega HEADERS_JSON/BODY_JSON si faltan y adelanta la secuencia si quedó atrás.
- Para consultar los CURL armados: `06_fn_endpoint_curls.sql` y luego `SELECT * FROM TABLE(FN_ENDPOINT_CURLS());` (no requiere CREATE VIEW).
- Reemplaza a `03_cat_endpoint_curls.sql`.

Para agregar endpoints: copia una línea `sincronizar(...)` al final del bloque y vuelve a ejecutar,
o captúralo en la pantalla de Endpoints con "Llenar desde un CURL".
