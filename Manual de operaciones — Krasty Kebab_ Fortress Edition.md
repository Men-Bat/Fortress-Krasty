# Manual de operaciones — Krasty Kebab: Fortress Edition

**Versión:** `2.0.0-audited`  
**Propósito:** desplegar y operar un servicio privado con tres capas: servidor local primario, servidor cloud de contingencia y bóveda cifrada de recuperación.

## 1. Principios operativos

Krasty Kebab Fortress no es un conjunto de tres aplicaciones que escriben al mismo tiempo. Es un sistema con una **única fuente de escritura**: el nodo marcado como `NODE_ROLE=primary`. El nodo cloud comienza como `standby`, recibe snapshots verificados y solo se convierte en primario siguiendo el procedimiento de contingencia.

> **Nunca mantenga dos nodos en modo `primary`.** Antes de promover la contingencia, aísle o apague el nodo anterior. Esta regla evita que dos copias de SQLite acepten datos distintos y que luego no exista una forma segura de fusionarlos.

| Capa | Función | Contiene datos | Estado normal |
|---|---|---|---|
| Nodo local | Servicio principal, modelos locales y usuarios | Base SQLite activa y archivos de usuario | `primary` |
| Nodo cloud | Contingencia y recepción de snapshots | Último snapshot validado | `standby` |
| Bóveda HTTPS/WebDAV | Recuperación ante fallo doble o migración | Archivo cifrado de DB + `data/` | Solo archivo; sin ejecución |
| Gateway estático | Entrada que comprueba los nodos | No contiene secretos ni datos | Publicable |

La aplicación utiliza TLS, cookies seguras, cabeceras defensivas y límites de solicitudes. Estas medidas responden a prácticas de seguridad recomendadas para aplicaciones Express de producción [1].

## 2. Requisitos

| Componente | Requisito mínimo | Observación |
|---|---|---|
| Nodo local | Node.js 20 o superior, disco persistente, red estable | Se recomienda para modelos locales, Ollama, LM Studio o llama.cpp |
| Nodo cloud | Hosting que ejecute aplicaciones Node.js persistentes | Un hosting estático o un alojamiento PHP clásico no basta para `index.js` |
| Bóveda | Espacio HTTPS/WebDAV con usuario, contraseña y versionado | Debe permitir `PUT`; use una cuenta dedicada de privilegio mínimo |
| Gateway | Hosting estático HTTPS | Puede ser un subdominio o página estática independiente |
| IA | Ollama, LM Studio, llama.cpp y/o Chutes.ai | Las credenciales de Chutes viven en el servidor, no en el navegador |

La opción más ligera consiste en mantener el nodo local como servicio principal, usar un segundo hosting compatible con Node.js únicamente como contingencia y usar almacenamiento WebDAV como bóveda. Si necesita conmutación realmente automática, alta concurrencia, cuotas robustas de 50 GB o replicación de archivos de gran tamaño, el diseño debe evolucionar hacia una arquitectura de almacenamiento y coordinación dedicada; no basta con una copia periódica de SQLite.

## 3. Preparar secretos y configuración

En cada nodo, copie `.env.example` a `.env`. Este archivo **nunca** se sube a Git, a la web ni a la bóveda. En Linux, ejecute `chmod 600 .env` después de crearlo.

```bash
cp .env.example .env
chmod 600 .env
openssl rand -hex 32       # para SECRET_KEY
openssl rand -hex 32       # para SYNC_TOKEN
openssl rand -base64 32    # para VAULT_ENCRYPTION_KEY
```

| Variable | Nodo local primario | Nodo cloud standby | Gateway público |
|---|---|---|---|
| `SECRET_KEY` | Obligatoria | Igual que local para conservar sesiones tras promoción | No se usa |
| `SYNC_TOKEN` | Obligatoria | Igual que local; protege `/api/sync/upload` | No se usa |
| `VAULT_ENCRYPTION_KEY` | Obligatoria para crear y restaurar la bóveda | Solo si el cloud debe restaurar | **Nunca** se usa |
| `CHUTES_API_KEY` | Opcional; solo en servidor | Opcional; solo en servidor | No se usa |
| `CLOUD_NODE_URL` | URL HTTPS del nodo cloud | Vacía | No se usa |
| `VAULT_WEBDAV_*` | Credenciales de bóveda | Normalmente vacías | No se usa |
| `NODE_ROLE` | `primary` | `standby` | No se usa |

La gestión centralizada, mínima y rotada de secretos es esencial para limitar filtraciones y facilitar recuperación [2]. No comparta el contenido de `.env` por chat, correo sin cifrar, repositorios o capturas de pantalla.

## 4. Despliegue inicial del nodo local

1. Copie el directorio `krasty_fortress` a una ruta privada, por ejemplo `/opt/krasty-fortress`. No lo coloque dentro de una carpeta pública del servidor web.
2. Cree y complete `.env` a partir de la plantilla. Configure `NODE_TYPE=local` y `NODE_ROLE=primary`.
3. Instale exactamente las dependencias bloqueadas y audite el árbol:

```bash
npm ci --omit=dev
npm audit --omit=dev
```

4. Inicie el servicio:

```bash
npm start
```

5. Compruebe su estado desde el propio servidor:

```bash
HEALTHCHECK_URL=http://127.0.0.1:3000 npm run test:health
```

6. Sitúe un proxy HTTPS delante de la aplicación. Configure `TRUST_PROXY=1` únicamente si ese proxy es el que termina TLS. El acceso remoto al equipo local debe pasar por VPN o túnel autenticado; no exponga el puerto de Node directamente a Internet.

7. Cuando el chat use modelos locales, declare sus endpoints en `.env`, por ejemplo `OLLAMA_API_URL=http://127.0.0.1:11434/api/generate`. No permita que el cliente web decida la URL de destino.

## 5. Despliegue del nodo cloud de contingencia

El cloud debe aceptar Node.js y almacenamiento persistente. Para un hosting compartido, verifique específicamente que el proveedor permite procesos Node persistentes, variables de entorno, escritura fuera del directorio público y tareas en segundo plano. Si solo ofrece PHP/HTML, úselo como gateway o bóveda, no como nodo cloud.

1. Suba el mismo paquete al hosting cloud.
2. Cree `.env` con los mismos `SECRET_KEY` y `SYNC_TOKEN` del nodo local, pero con:

```dotenv
NODE_ENV=production
NODE_TYPE=cloud
NODE_ROLE=standby
COOKIE_SECURE=true
TRUST_PROXY=1
```

3. Cree `db/`, `data/` y `archives/` fuera de cualquier carpeta pública, con permisos de la cuenta de servicio.
4. Ejecute `npm ci --omit=dev`, después `npm audit --omit=dev`, y arranque `npm start` según el panel del proveedor o su gestor de procesos.
5. Añada el origen HTTPS del gateway a `ALLOWED_ORIGINS`, tanto en local como en cloud, para que pueda consultar `GET /api/health` desde el navegador.
6. Compruebe `https://tu-cloud.ejemplo/api/health`. Debe devolver `"role":"standby"` y `"writable":false`.

## 6. Configurar la bóveda cifrada

La bóveda se comunica mediante WebDAV HTTPS. Cree una cuenta exclusiva para el servicio; no use la cuenta principal del hosting. Esa cuenta necesita solamente crear y escribir archivos dentro de una carpeta específica, sin acceso a otros sitios o datos.

Añada al `.env` del primario:

```dotenv
VAULT_WEBDAV_URL=https://almacenamiento.ejemplo.com/webdav/krasty-vault
VAULT_WEBDAV_USERNAME=krasty_backup
VAULT_WEBDAV_PASSWORD=una_contraseña_larga_y_exclusiva
VAULT_ENCRYPTION_KEY=clave_base64_de_32_bytes
```

El comando siguiente crea un ZIP que contiene SQLite, archivos de `data/` y un manifiesto; después lo cifra con AES-256-GCM antes de enviarlo fuera del servidor. Node ofrece las primitivas de hash y cifrado necesarias mediante `node:crypto` [3].

```bash
npm run snapshot
```

Para ejecución recurrente, inicie el motor únicamente en el nodo local primario:

```bash
npm run start:sync
```

El proceso ejecuta primero una copia consistente de la base de datos hacia cloud y, después, un snapshot cifrado hacia WebDAV. Si la subida cloud o la bóveda fallan, conserva un mensaje de error local; configure supervisión del proceso y alertas externas si desea un aviso inmediato.

> Mantenga `db/`, `data/`, `archives/`, `.env` y los archivos de logs fuera de `public/`. OWASP advierte que los backups y archivos no referenciados dentro de una ruta web pueden filtrar código, credenciales y datos sensibles [4].

## 7. Configurar el gateway

1. Publique `public/gateway.html` y una copia de `public/gateway-config.example.js` renombrada como `gateway-config.js` en un hosting estático HTTPS.
2. Edite solo las tres URL públicas, sin añadir secretos:

```javascript
window.KRASTY_GATEWAY_CONFIG = {
  localUrl: 'https://krasty-local.tu-dominio.example',
  cloudUrl: 'https://krasty-cloud.tu-dominio.example',
  vaultUrl: 'https://archivos.tu-dominio.example/krasty-vault/'
};
```

3. Incluya el origen exacto del gateway, por ejemplo `https://acceso.tu-dominio.example`, en `ALLOWED_ORIGINS` de ambos nodos.
4. Compruebe que el gateway muestra al nodo local como primario. Si el local no es accesible desde la red externa, se debe a que está correctamente privado; publique un acceso solo por VPN o túnel autenticado si necesita detectarlo desde Internet.

El gateway **no** convierte automáticamente al cloud en primario. Si el local responde a usuarios distintos, pero el gateway no lo ve por un fallo de red, una promoción automática crearía dos fuentes de datos. La intervención manual es una protección deliberada.

## 8. Operación diaria y comprobaciones

| Frecuencia | Operación | Criterio de éxito |
|---|---|---|
| Diaria | `npm run test:health` en primario y cloud | HTTP 200 y roles esperados |
| Semanal | Revisar estado de snapshots, espacio y logs | Snapshot reciente, espacio libre suficiente, sin errores repetidos |
| Mensual | Ejecutar `npm audit --omit=dev` | Sin vulnerabilidades conocidas o plan documentado |
| Mensual | Restaurar un snapshot en entorno aislado | DB y archivos legibles, sin tocar producción |
| Trimestral | Rotar `SYNC_TOKEN`, contraseñas WebDAV y secretos si procede | Todos los nodos reinician correctamente |
| Antes de cada actualización | Snapshot manual y prueba en réplica | Recuperación disponible antes de modificar producción |

## 9. Cambio de hosting o migración de servidor

Este procedimiento evita una transición con pérdida de datos.

1. **Prepare el nuevo servidor sin tráfico.** Instale el paquete, cree `.env`, use las mismas claves de sesión y sincronización, pero marque `NODE_ROLE=standby`.
2. **Compruebe dependencias y salud.** Ejecute `npm ci --omit=dev`, `npm audit --omit=dev` y `npm run test:health`.
3. **Genere un último snapshot en el nodo actual.** Detenga temporalmente las nuevas tareas o active mantenimiento para evitar cambios durante la copia.
4. **Transfiera y restaure los datos.** Copie el último snapshot cloud o descargue el archivo `.enc` de la bóveda; descífrelo en el nuevo nodo usando el procedimiento de la sección 11.
5. **Aísle el servidor anterior.** Pare el proceso, retire su DNS del balanceo o bloquee la entrada para garantizar que ya no escribe.
6. **Promueva el nuevo servidor.** Cambie `NODE_ROLE=primary`, reinicie la aplicación y confirme `/api/health` con `writable:true`.
7. **Actualice Gateway/DNS.** Cambie `localUrl` o `cloudUrl` en `gateway-config.js` y, cuando sea oportuno, el registro DNS.
8. **Conserve el antiguo nodo apagado durante la ventana de observación.** No lo reactive como primario; conviértalo en `standby` después de validar la migración.

## 10. Fallo del nodo local: promoción controlada del cloud

1. Confirme que el local no responde y que no puede volver a escribir: apáguelo, desconéctelo de red o bloquee sus entradas con una regla de cortafuegos.
2. Identifique el snapshot más reciente y verificado dentro de `archives/incoming/` del cloud. Revise el archivo JSON asociado y su SHA-256.
3. Detenga el servicio cloud.
4. Haga una copia del archivo actual `db/krasty.db` por precaución y reemplace esa base por el snapshot seleccionado. Los archivos de usuarios posteriores al último snapshot deben restaurarse desde la bóveda cifrada.
5. Si es necesario, restaure el archivo cifrado siguiendo la sección 11.
6. Configure `NODE_ROLE=primary`, reinicie el servicio y ejecute la comprobación de salud.
7. Cambie el gateway para que el cloud sea el nodo elegido y comunique una ventana de posible pérdida de datos desde el último snapshot.
8. Cuando el local se recupere, no lo reactive de inmediato. Reinstálelo como `standby`, restaure desde el cloud primario y solo después reactívelo como contingencia.

## 11. Recuperación desde la bóveda

Descargue el archivo `.zip.enc` y su checksum desde WebDAV en un servidor seguro. Configure temporalmente `VAULT_ENCRYPTION_KEY` con la misma clave original y ejecute:

```bash
npm run restore:snapshot -- /ruta/krasty-fecha.zip.enc /ruta/restauracion.zip
unzip /ruta/restauracion.zip -d /ruta/restauracion
```

El contenido tendrá una estructura similar a:

```text
restauracion/
├── database/krasty.db
├── user-files/
└── manifest.json
```

Detenga la aplicación antes de copiar `database/krasty.db` sobre `db/krasty.db`. Después copie `user-files/` hacia la ruta configurada en `DATA_DIR`, valide permisos y arranque la aplicación. Realice siempre esta operación primero en un entorno aislado para comprobar que el snapshot es utilizable.

## 12. Actualización de Krasty Kebab

```bash
# 1. Crear una copia antes de modificar
npm run snapshot

# 2. Detener el servicio y actualizar código desde una fuente verificada
#    Ejemplo: git pull --ff-only (si administra el proyecto con Git)

# 3. Instalar exactamente el nuevo bloqueo y auditar
npm ci --omit=dev
npm audit --omit=dev

# 4. Revisar configuración y arrancar
npm start
npm run test:health
```

No edite directamente archivos de producción si puede evitarlo. Mantenga un historial de cambios y valide primero en una réplica. La documentación de Express recomienda mantener las dependencias actualizadas y ejecutar revisiones de seguridad del árbol de paquetes [1].

## 13. Limitaciones conocidas

El paquete no implementa actualmente cuotas efectivas de 50 GB, control de roles, MFA, carga de archivos, eliminación de cuentas, registro de auditoría estructurado ni replicación de archivos de gran volumen en tiempo real. El receptor cloud admite snapshots de tamaño limitado; no debe usarse para transferir archivos de decenas de gigabytes por JSON. Para ese volumen, use la bóveda WebDAV u otro almacenamiento de objetos compatible, con reanudación, versionado y restauraciones ensayadas.

## Referencias

[1]: https://expressjs.com/en/advanced/best-practice-security/ "Express — Production Best Practices: Security"
[2]: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html "OWASP — Secrets Management Cheat Sheet"
[3]: https://nodejs.org/api/crypto.html "Node.js — Crypto API"
[4]: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/04-Review_Old_Backup_and_Unreferenced_Files_for_Sensitive_Information "OWASP WSTG — Review Old Backup and Unreferenced Files"
