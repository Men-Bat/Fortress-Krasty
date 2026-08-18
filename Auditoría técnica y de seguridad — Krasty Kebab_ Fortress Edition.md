# Auditoría técnica y de seguridad — Krasty Kebab: Fortress Edition

**Autor:** Manus AI  
**Fecha:** 18 de agosto de 2026  
**Versión auditada:** `2.0.0-audited`

## Dictamen ejecutivo

La edición anterior de **Krasty Kebab Fortress** era un prototipo de arquitectura y **no debía publicarse como servicio multiusuario de producción**: aceptaba sesiones anónimas, contenía secretos por defecto, aceptaba endpoints de modelos proporcionados por el navegador y describía una conmutación automática que no era segura ni funcionalmente completa. También carecía de un manifiesto de dependencias, por lo que no podía reproducirse ni auditarse de manera fiable.

La revisión incorpora un servidor endurecido, un paquete reproducible, límites de solicitudes, autenticación obligatoria, gestión de secretos por variables de entorno, copias verificadas y cifradas, y una conmutación explícitamente **manual** entre el nodo local y el de contingencia. Tras actualizar `sqlite3` a la rama 6 y regenerar el bloqueo de dependencias, el análisis de producción con `npm audit --omit=dev` terminó con **0 vulnerabilidades conocidas** en el momento de la comprobación.

> **Conclusión:** el paquete está preparado como una **base endurecida para pruebas controladas y despliegue privado**. No se debe anunciar como plataforma pública multiusuario terminada hasta completar las acciones de la sección “Riesgos residuales y bloqueadores”. En particular, la actual réplica cloud conserva snapshots verificados, pero la promoción sigue siendo manual, las cuotas de 50 GB no se imponen aún y no existe control de roles administrativos.

## Alcance y metodología

Se revisaron el backend Node.js, el gateway de conmutación, el motor de sincronización, el proceso de snapshots, la configuración de despliegue y las dependencias de producción. Se realizaron comprobaciones sintácticas de los scripts, una instalación limpia de dependencias, un análisis de dependencias, una prueba de salud y pruebas negativas contra los endpoints de chat y registro.

| Área revisada | Evidencia | Resultado |
|---|---|---|
| Arranque y sintaxis | `node --check` para servidor y scripts | Correcto |
| Dependencias | `npm install --omit=dev` y `npm audit --omit=dev` | **0 vulnerabilidades conocidas** al cierre de la revisión |
| Salud del nodo | `GET /api/health` | Correcto; declara nodo, rol y capacidad de escritura |
| Acceso al chat | `POST /api/chat` sin cookie válida | Correctamente rechazado con HTTP 401 |
| Validación de cuentas | Registro con usuario/contraseña no válidos | Correctamente rechazado con HTTP 400 |
| Snapshots | Archivo de prueba + snapshot cifrado | Correcto; genera `.zip.enc` y SHA-256 |
| Cabeceras HTTP | Respuesta de salud | CSP, HSTS y `X-Content-Type-Options` presentes |

## Hallazgos iniciales y correcciones aplicadas

| ID | Hallazgo original | Riesgo | Corrección aplicada | Estado |
|---|---|---|---|---|
| F-01 | `SECRET_KEY` y `SYNC_TOKEN` tenían valores por defecto | Suplantación de sesión y falsificación de sincronización | El proceso no inicia sin secretos de al menos 32 caracteres | Corregido |
| F-02 | El chat permitía modo invitado | Acceso no autorizado a modelos y coste/abuso del servidor | `requireAuth` es obligatorio para chat y catálogo de modelos | Corregido |
| F-03 | El navegador podía enviar un `endpoint` arbitrario | SSRF: el servidor podría llamar a servicios internos elegidos por un atacante | Lista cerrada de proveedores y URLs configuradas solo en `.env` | Corregido |
| F-04 | La clave de Chutes.ai se enviaba desde el frontend | Exposición de una clave de proveedor a usuarios y registros del navegador | La clave se mantiene exclusivamente en `CHUTES_API_KEY` del servidor | Corregido |
| F-05 | No había límites contra fuerza bruta | Abuso de login y consumo de recursos | Rate limiting general y específico de autenticación | Corregido |
| F-06 | Cookies sin configuración de producción ni expiración explícita | Robo o persistencia indebida de sesión | Cookie `httpOnly`, `sameSite`, `secure` en producción y expiración de 8 h | Corregido |
| F-07 | Gateway con `no-cors` y selección por latencia | Falsos positivos; podía redirigir a un nodo no preparado para escritura | Health JSON con rol; prioridad del primario; promoción manual documentada | Corregido |
| F-08 | Copia de SQLite cruda y sin comprobación de integridad | Corrupción o aceptación de datos alterados | Snapshot de SQLite, SHA-256, token comparado en tiempo constante y recepción atómica | Corregido |
| F-09 | La tercera copia no estaba implementada | Falsa expectativa de respaldo externo | Snapshot de DB + archivos de usuario, cifrado AES-256-GCM y subida WebDAV configurable | Corregido, sujeto a configuración |
| F-10 | Sin `package.json` ni bloqueo de dependencias | Despliegue no reproducible | Manifiesto, `package-lock.json` y scripts de operación | Corregido |
| F-11 | Dependencia SQLite transitiva con alertas críticas | Riesgo en cadena de suministro | Actualización a `sqlite3 ^6.0.1`; auditoría final sin vulnerabilidades conocidas | Corregido al cierre |

Las medidas aplicadas se alinean con recomendaciones de Express para TLS, validación de entradas, cookies seguras, mitigación de fuerza bruta y revisión continuada de dependencias [1]. La decisión de no exponer secretos en el código o en el navegador sigue la necesidad de centralizar, rotar y auditar credenciales [2].

## Arquitectura auditada

```text
Usuario
  │ HTTPS
  ▼
Gateway estático ──consulta salud──► Nodo local (primario; escritura)
  │                                      │
  │                                      ├─ Modelos locales / Chutes.ai
  │                                      ├─ SQLite + data/ fuera del directorio público
  │                                      └─ Sync Engine (solo primario)
  │                                               │ snapshot SQLite verificado
  │                                               ▼
  └──────────────────────────────► Nodo cloud (standby; recepción de snapshots)
                                                 
Primario ──snapshot cifrado AES-256-GCM + SHA-256──► Bóveda WebDAV HTTPS
```

La base de datos contiene metadatos de usuarios, agentes y conversaciones; `data/` está reservado para archivos de usuario. La bóveda recibe una copia cifrada que contiene ambos componentes. La criptografía se implementa con el módulo estable `node:crypto`, que ofrece primitivas de hash y cifrado basadas en OpenSSL [3].

## Riesgos residuales y bloqueadores de una publicación abierta

| Prioridad | Riesgo residual | Impacto | Acción necesaria antes de abrir al público |
|---|---|---|---|
| Crítica | No existe control de roles, panel de administración ni MFA | Cualquier cuenta autenticada comparte el mismo nivel funcional | Implementar RBAC, cuentas administradoras, MFA y recuperación de contraseña segura |
| Crítica | La cuota de 50 GB existe solo como campo de base de datos | Un usuario puede sobrepasar almacenamiento disponible | Implementar cálculo real de uso, límites de carga, alertas y rechazo al superar cuota |
| Alta | La réplica cloud recibe snapshots pero no se promueve automáticamente | Riesgo de pérdida de cambios desde el último snapshot y posibilidad de conflicto si se promueve mal | Mantener promoción manual; para failover automático usar almacenamiento transaccional/replicación con quorum |
| Alta | El motor WebDAV no se ha probado contra un proveedor concreto | Diferencias entre configuraciones WebDAV pueden impedir subida | Validar con el proveedor elegido en un entorno de ensayo y restaurar al menos un snapshot |
| Alta | No se conservan historiales de revocación de JWT | Una sesión válida no se puede invalidar selectivamente antes de expirar | Incorporar tabla de sesiones/revocación o tokens de corta duración con refresh seguro |
| Media | No hay auditoría de eventos de usuarios, administración ni sincronización persistente | Dificulta investigar abusos o incidentes | Implementar logs estructurados, rotación, protección contra manipulación y alertas |
| Media | Los iframes externos pueden estar bloqueados por CSP o `X-Frame-Options` de su origen | El avatar/proyecto no se visualizará | Validar cada origen y ofrecer apertura en pestaña nueva como alternativa |
| Media | El gateway estático solo puede alcanzar el nodo local si la URL es accesible desde el navegador | Fuera de la red local puede no detectar el servidor casero | Usar VPN o túnel autenticado; no abrir el puerto local sin capa de acceso |
| Media | La protección de fuerza bruta es local a un proceso | En varios nodos puede ser insuficiente | Usar un limitador compartido cuando se escale horizontalmente |

> **Regla de operación:** nunca marque dos nodos como `NODE_ROLE=primary` al mismo tiempo. Una partición de red podría crear dos fuentes de escritura distintas (*split brain*) y causar pérdida o divergencia de datos.

## Observaciones de respaldo y privacidad

La arquitectura separa el código servido en `public/` de `db/`, `data/` y `archives/`. Esta separación es obligatoria: OWASP advierte que copias, archivos de configuración, registros y snapshots dentro de rutas publicables pueden divulgar credenciales, código o datos sensibles [4]. Los directorios de datos y archivos cifrados deben quedar fuera del *document root*, con permisos mínimos y sin índices de directorio.

El snapshot externo está cifrado **antes** de subirlo. La clave `VAULT_ENCRYPTION_KEY` no debe almacenarse en el mismo hosting, repositorio, WebDAV o ticket de soporte. Guarde una copia fuera de línea en un gestor de contraseñas o soporte físico protegido. Perder esa clave hace imposible descifrar la bóveda; filtrarla equivale a perder la confidencialidad de sus copias.

## Decisión de disponibilidad: automática frente a segura

Se eliminó la promesa de “conmutación automática total”. Un navegador puede detectar que un endpoint no responde, pero no puede determinar de manera fiable que el primario está definitivamente apagado y no únicamente aislado por una red rota. Promover automáticamente el secundario en ese caso puede permitir escrituras simultáneas en dos lugares. Por esta razón, el gateway solo ayuda a elegir el nodo primario declarado como saludable; la promoción de la réplica exige un procedimiento humano de aislamiento, restauración y cambio de rol.

| Objetivo | Diseño actual | Diseño requerido para automatización real |
|---|---|---|
| Acceso prioritario local | Sí; gateway prioriza `role=primary` local | Mantener VPN/túnel autenticado |
| Respaldo cloud de SQLite | Sí; snapshots verificados | Replicación transaccional o base de datos gestionada con consenso |
| Tercera copia de archivos | Sí; archivo cifrado WebDAV configurable | Pruebas periódicas de restauración y almacenamiento versionado/inmutable |
| Failover sin operador | No; intencionalmente | Quorum, *fencing*, elección de líder y almacenamiento compartido preparado para múltiples nodos |

## Lista de aceptación antes del despliegue

| Verificación | Debe cumplirse |
|---|---|
| Secretos | `.env` solo en los servidores, permisos restringidos, sin valores de ejemplo |
| Transporte | HTTPS válido en gateway, local accesible de forma autenticada y cloud; redirección HTTP→HTTPS |
| Datos | `db/`, `data/` y `archives/` fuera de rutas públicas y excluidos del repositorio |
| Dependencias | `npm ci --omit=dev` y `npm audit --omit=dev` sin vulnerabilidades conocidas o con excepciones aprobadas |
| Modelos | URLs locales configuradas exclusivamente en variables de entorno; Chutes.ai por clave del servidor |
| Bóveda | Snapshot cifrado subido, checksum comprobado y restauración ensayada |
| Contingencia | Nodo cloud en `standby`, procedimiento de promoción probado en ensayo |
| Usuarios | No publicar hasta implementar cuotas reales, roles y el resto de bloqueadores críticos |

## Referencias

[1]: https://expressjs.com/en/advanced/best-practice-security/ "Express — Production Best Practices: Security"
[2]: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html "OWASP — Secrets Management Cheat Sheet"
[3]: https://nodejs.org/api/crypto.html "Node.js — Crypto API"
[4]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/04-Review_Old_Backup_and_Unreferenced_Files_for_Sensitive_Information "OWASP WSTG — Review Old Backup and Unreferenced Files"
