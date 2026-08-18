# Manual completo de Krasty Kebab para XAMPP en Windows

**Versión del paquete:** 1.0 · **Autor:** Manus AI · **Audiencia:** administradores sin experiencia previa en PHP, Apache o copias de seguridad.

## 1. Propósito, alcance y advertencia importante

Krasty Kebab es una aplicación PHP de registro, chat con modelo local opcional y almacenamiento de archivos por cuenta. Su dirección de uso es `https://web.com/usuario/Krasty/`: por ejemplo, `https://web.com/ana/Krasty/`. La dirección no es una carpeta pública de Ana; es una ruta que Apache entrega a la aplicación, que verifica la sesión antes de permitir acceder a cualquier archivo.

Los datos se guardan **fuera** de la carpeta web y se separan por identificador de usuario. La aplicación conserva las contraseñas mediante hash, utiliza sesiones con cookies HTTP-only, tokens CSRF en formularios, consultas SQLite preparadas y descargas controladas. Cada cuenta dispone de una cuota de **50 GiB** por defecto; no obstante, el máximo tamaño de un archivo individual es 2 GiB para mantener las cargas de navegador razonables. La cuota puede modificarse en `config.php`.

> **Aviso de publicación:** Apache Friends advierte expresamente que XAMPP está concebido para desarrollo y viene configurado de forma abierta; no basta con abrir un puerto del router para considerarlo seguro [1]. Este paquete es operativo para pruebas reales, registros y un despliegue doméstico o de pequeña organización **solo después** de completar la lista de seguridad, activar HTTPS y mantener actualizados Windows, XAMPP y el router. Si se va a ofrecer servicio público permanente a muchas personas, la recomendación responsable es migrar después a un Windows Server o a un Apache mantenido y endurecido específicamente para producción.

| Componente | Función | Ubicación recomendada | Debe quedar expuesto a Internet |
|---|---|---|---|
| Apache de XAMPP | Sirve el portal PHP | `C:\xampp` | Solo HTTPS 443; HTTP 80 únicamente para redirigir a HTTPS o emitir certificado |
| Aplicación Krasty | Código PHP, hojas de estilo y scripts | `C:\KrastyApp` | Solo la subcarpeta `web` mediante Apache |
| Datos privados | SQLite, usuarios, archivos y copias | `C:\KrastyData` o una unidad local amplia | **No** |
| Ollama opcional | Modelo de IA en el mismo PC | `127.0.0.1:11434` | **No** |
| Segundo PC | Copia de seguridad por red local SMB | `\\ORDENADOR-RESPALDO\KrastyBackups` | **No** |
| Bóveda | Archivo cifrado cargado manualmente | Hosting o almacenamiento externo privado | Solo el archivo cifrado; nunca la clave |

## 2. Antes de empezar: material, espacio y decisiones

Necesitará un ordenador principal con Windows 10/11, conexión estable, permisos de administrador, XAMPP con PHP 8 o superior y un segundo ordenador encendido regularmente en la misma red local. Instale XAMPP desde su sitio oficial y compruebe la suma de verificación que ofrece la descarga [2]. XAMPP incluye Apache y PHP, por lo que **no se instala Node.js ni una base de datos de servidor** para esta versión; SQLite es un único archivo privado.

Planifique la capacidad antes de aceptar registros. Si ofrece 50 GiB a diez usuarios, la capacidad lógica anunciada es 500 GiB. Para que las tres capas funcionen, el PC principal, el segundo PC y la bóveda deben disponer de espacio suficiente para los datos que realmente se suban. El tercer respaldo crea temporalmente un ZIP antes de cifrarlo; reserve como mínimo el tamaño de la copia que pretende hacer además del espacio ocupado por los datos.

| Decisión | Valor de ejemplo | Dónde se configura |
|---|---|---|
| Dominio | `web.com` | DNS del registrador y virtual host de Apache |
| Carpeta de aplicación | `C:\KrastyApp` | Copia inicial del paquete y scripts `.bat` |
| Datos principales | `C:\KrastyData` | `config\config.php` y `backup_to_second_pc.bat` |
| Cuota por cuenta | `50 * 1024 * 1024 * 1024` | `config\config.php` |
| Destino secundario | `\\ORDENADOR-RESPALDO\KrastyBackups` | `backup_to_second_pc.bat` |
| Modelo local | `llama3.2` u otro instalado | `config\config.php` |

## 3. Instalar XAMPP y comprobar Apache

Descargue el instalador de XAMPP para Windows desde [Apache Friends][2]. Ejecútelo como administrador y acepte la ubicación habitual `C:\xampp`; evitar `Program Files` simplifica los permisos de escritura. En el panel de control de XAMPP, pulse **Start** en Apache. Visite `http://localhost/` en el navegador. La FAQ de XAMPP indica que `http://localhost/` o `http://127.0.0.1/` permiten comprobar la instalación [1].

No es necesario iniciar MariaDB, FileZilla, Mercury ni Tomcat para Krasty Kebab. Dejar servicios no utilizados detenidos reduce la superficie expuesta. Si Apache no inicia porque el puerto 80 ya está ocupado, compruebe primero IIS, World Wide Web Publishing Service, Skype u otro servidor; XAMPP describe este conflicto de puerto en su guía para Windows [1].

> **No desactive el firewall de Windows.** Microsoft indica que el firewall bloquea por defecto el tráfico entrante que no coincide con una regla y recomienda no deshabilitarlo [9]. Más adelante creará reglas concretas para Apache, no una excepción general.

## 4. Copiar el paquete y crear el almacenamiento privado

Descomprima `Krasty_XAMPP_Completo.zip` en `C:\KrastyApp`. Debe obtener, como mínimo, las carpetas `app`, `config`, `docs`, `scripts` y `web`. **No copie `C:\KrastyData` dentro de `htdocs` ni dentro de `web`**: los datos privados se deben almacenar fuera de cualquier carpeta publicada por Apache.

Abra el Explorador de archivos, cree `C:\KrastyData` y verifique que la cuenta que ejecuta Apache puede escribir en ella. En una instalación XAMPP doméstica suele ser el usuario de Windows que arrancó Apache. No comparta esta carpeta mediante SMB ni la sincronice con una carpeta pública.

La carpeta `web` es el único directorio que Apache debe publicar. El código de la aplicación y su configuración pueden permanecer fuera de `htdocs`, algo preferible porque impide que un error de configuración publique archivos internos. La FAQ de XAMPP sitúa el contenido web por defecto en `C:\xampp\htdocs` [1], pero Apache permite definir otro `DocumentRoot` mediante un virtual host.

## 5. Configurar Apache, `mod_rewrite` y rutas `/usuario/Krasty/`

Abra `C:\xampp\apache\conf\httpd.conf` con un editor de texto ejecutado como administrador. Busque y asegúrese de que estas líneas **no** empiezan por `#`:

```apache
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule headers_module modules/mod_headers.so
Include conf/extra/httpd-vhosts.conf
```

`mod_rewrite` es el módulo de Apache que aplica reglas para reescribir URLs [3]. El archivo incluido `web\.htaccess` usa este módulo para enviar rutas como `/ana/Krasty/` a `index.php` sin mostrar nombres de archivos internos.

Ahora abra `C:\xampp\apache\conf\extra\httpd-vhosts.conf` y añada al final esta configuración **para la prueba local**. Sustituya `krasty.local` por el nombre que vaya a usar; mientras prueba, mantenga el ejemplo tal cual.

```apache
<VirtualHost *:80>
    ServerName krasty.local
    DocumentRoot "C:/KrastyApp/web"

    <Directory "C:/KrastyApp/web">
        Options -Indexes
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog "logs/krasty-error.log"
    CustomLog "logs/krasty-access.log" combined
</VirtualHost>
```

El valor `AllowOverride All` hace que Apache lea el `.htaccess` incluido. Apache confirma que `AllowOverride` determina qué directivas pueden emplearse en `.htaccess`; no lo active globalmente para todo el disco, sino únicamente para el directorio web de Krasty [4]. Pulse **Stop** y después **Start** en Apache para aplicar los cambios.

Para probar `krasty.local` sin Internet, abra el Bloc de notas como administrador y edite `C:\Windows\System32\drivers\etc\hosts`. Añada esta línea y guarde:

```text
127.0.0.1 krasty.local
```

Abra `http://krasty.local/`. Si aparece error 500, vuelva a comprobar que `rewrite_module` y `headers_module` están habilitados y consulte `C:\xampp\apache\logs\error.log`.

## 6. Configurar extensiones y límites de PHP

Abra `C:\xampp\php\php.ini` como administrador. PHP en Windows carga extensiones declaradas en `php.ini`; tras modificarlo debe reiniciar Apache [1] [5]. Busque cada directiva siguiente. Si aparece con un punto y coma inicial, quítelo; si no existe, añádala en la sección de extensiones.

```ini
extension=curl
extension=fileinfo
extension=openssl
extension=pdo_sqlite
extension=sqlite3
extension=zip
```

Estas extensiones permiten, respectivamente, hablar con Ollama, identificar ficheros cargados, cifrar la bóveda, utilizar SQLite y crear/extraer ZIP. Para una primera instalación con archivos grandes, cambie también los límites; el valor de 2 GiB coincide con el límite de archivo individual aplicado por la aplicación.

```ini
upload_max_filesize = 2048M
post_max_size = 2050M
max_execution_time = 3600
max_input_time = 3600
memory_limit = 256M
```

Reinicie Apache después de cada modificación de `php.ini`. Si la modificación no tiene efecto, XAMPP recomienda verificar la ruta del archivo realmente cargado con su página `phpinfo` local [1]. No deje una página `phpinfo()` propia accesible desde Internet.

## 7. Crear `config.php` de forma segura

Abra una ventana de **Símbolo del sistema** y ejecute lo siguiente para copiar la plantilla:

```bat
copy C:\KrastyApp\config\config.example.php C:\KrastyApp\config\config.php
C:\xampp\php\php.exe -r "echo bin2hex(random_bytes(32)), PHP_EOL;"
```

El segundo comando imprime un secreto largo. Ejecútelo tres veces y guarde los resultados en un gestor de contraseñas o, preferiblemente, en una copia física segura. Abra `C:\KrastyApp\config\config.php` y reemplace todos los textos que empiezan por `REEMPLACE_`.

```php
return [
    'app_name' => 'Krasty Kebab Local',
    'data_root' => 'C:/KrastyData',
    'app_secret' => 'PEGUE_AQUI_EL_PRIMER_SECRETO_DE_64_CARACTERES',
    'first_admin_token' => 'PEGUE_AQUI_EL_SEGUNDO_SECRETO',
    'quota_bytes' => 50 * 1024 * 1024 * 1024,
    'cookie_secure' => false,
    'base_path' => '',
    'model' => [
        'enabled' => false,
        'provider' => 'ollama',
        'ollama_url' => 'http://127.0.0.1:11434/api/generate',
        'default_model' => 'llama3.2',
        'timeout_seconds' => 120,
    ],
    'secondary_backup_share' => '\\\\ORDENADOR-RESPALDO\\KrastyBackups',
    'vault_passphrase' => 'PEGUE_AQUI_EL_TERCER_SECRETO_LARGO',
];
```

Durante esta prueba por `http://krasty.local`, mantenga `cookie_secure => false`; una cookie marcada como segura no se envía por HTTP. **Antes de abrir el portal a Internet, active HTTPS y cambie obligatoriamente a `cookie_secure => true`.** No publique `config.php`, no lo comparta por correo y no lo incluya en un repositorio. El archivo `.gitignore` del paquete evita que se incluya por accidente en un repositorio Git.

## 8. Registrar el primer administrador y los usuarios

Visite `http://krasty.local/register`. Introduzca un nombre de usuario de 3 a 32 caracteres, una contraseña de al menos 12 caracteres y el valor exacto de `first_admin_token`. La primera cuenta exige ese token y se convierte en administrador. Después de crearla, cambie `first_admin_token` por otro valor largo almacenado fuera de línea; ya no es necesario para los siguientes registros.

Cada usuario posterior se registra desde la misma página, sin token. Al entrar, la aplicación redirige a su dirección personal:

```text
http://krasty.local/usuario/Krasty/
```

Por ejemplo, una cuenta llamada `ana` usa `http://krasty.local/ana/Krasty/`. Cada descarga se verifica en PHP contra el propietario del archivo. Un usuario normal que pruebe a cambiar la URL por otro nombre recibirá acceso denegado. El administrador puede ver el resumen de cuentas desde **Administración**; no comparta su contraseña ni use la cuenta administradora para trabajo diario.

| Acción | Quién puede hacerla | Resultado |
|---|---|---|
| Registrarse | Cualquier visitante | Crea cuenta normal, salvo la primera con token |
| Subir/descargar archivo | Solo su propietario; administrador | El archivo no se expone como URL estática |
| Ver chat | Solo su propietario; administrador | Historial SQLite separado por usuario |
| Ver panel de resumen | Administrador | Lista de cuentas y consumo declarado |
| Crear copia secundaria o bóveda | Administrador del PC | Incluye los datos privados completos |

## 9. Activar Ollama como IA local opcional

Ollama es opcional: si no se instala, el resto de la plataforma sigue funcionando y el chat muestra una indicación de configuración. Descárguelo desde [Ollama para Windows][6], instálelo y abra una consola nueva. Su documentación indica que, tras instalarlo, Ollama sirve la API local en `http://localhost:11434` [6].

Ejecute un modelo y espere a que termine la descarga:

```bat
ollama pull llama3.2
ollama run llama3.2
```

Cierre la conversación con `Ctrl+C` si lo desea. En `config.php`, establezca `enabled => true` y confirme que `default_model` tiene el nombre exacto descargado. Reinicie Apache y envíe un mensaje desde el chat de Krasty. La aplicación envía una petición JSON no transmitida al navegador directamente a `127.0.0.1:11434/api/generate`, que es el endpoint de generación documentado por Ollama [7].

> Mantenga Ollama en `127.0.0.1`. **No abra ni redirija el puerto 11434 en el firewall o en el router.** Los usuarios deben usar el portal autenticado; no la API local directamente.

Si necesita usar otro modelo, ejecute `ollama pull NOMBRE_DEL_MODELO` y sustituya `default_model`. El tamaño de los modelos puede ser de decenas de gigabytes, por lo que Ollama documenta la variable `OLLAMA_MODELS` para mover su almacenamiento a otra unidad [6].

## 10. Preparar el segundo ordenador para la copia automática

La segunda capa no es una web ni requiere VPN: es una carpeta compartida SMB dentro de la red privada. El programa `robocopy` de Windows copia carpetas, permite reanudar transferencias con `/Z` y devuelve códigos de error de 8 o más cuando una copia falla [8]. El script incluido trata esos códigos como fallo y escribe un registro.

### 10.1 Crear una cuenta dedicada en ambos PCs

En el **PC principal** y en el **segundo PC**, cree una cuenta local llamada `krastybackup` con la **misma contraseña larga**. No la use para navegar ni administrar. En Windows 11 puede hacerlo desde **Configuración > Cuentas > Otros usuarios > Agregar cuenta > Agregar un usuario sin cuenta Microsoft**. La cuenta debe poder iniciar sesión localmente en el PC principal y modificar la carpeta compartida en el segundo PC.

### 10.2 Crear la carpeta compartida en el segundo PC

En el segundo PC cree, por ejemplo, `D:\KrastyBackups`. Haga clic derecho sobre ella, elija **Propiedades > Uso compartido > Uso compartido avanzado**, marque **Compartir esta carpeta** y asigne el nombre `KrastyBackups`. En **Permisos**, elimine `Todos` si estuviera presente y otorgue **Cambiar** y **Leer** a `krastybackup`. En la pestaña **Seguridad**, otorgue también a esa cuenta **Modificar**. Anote el nombre real del PC, por ejemplo `ORDENADOR-RESPALDO`.

Desde el PC principal, abra el Explorador y compruebe que puede abrir:

```text
\\ORDENADOR-RESPALDO\KrastyBackups
```

Inicie sesión como `krastybackup` cuando Windows lo solicite. Si funciona, la autenticación de paso con ambas cuentas del mismo nombre y contraseña permitirá que la tarea automática escriba en el recurso. No conceda acceso público ni comparta `C:\KrastyData` desde el PC principal.

### 10.3 Ajustar y probar el script

Abra `C:\KrastyApp\scripts\backup_to_second_pc.bat` y revise estas líneas:

```bat
set "PHP_EXE=C:\xampp\php\php.exe"
set "PROJECT_DIR=C:\KrastyApp"
set "BACKUP_SHARE=\\ORDENADOR-RESPALDO\KrastyBackups"
set "DATA_ROOT=C:\KrastyData"
```

Sustituya únicamente el nombre del segundo ordenador o las rutas que difieran. Guarde. Abra una consola como administrador y ejecute el `.bat` una primera vez. Deben aparecer `users` y `database` dentro de la carpeta compartida. Revise siempre el archivo:

```text
C:\KrastyApp\logs\backup_to_second_pc.log
```

El script crea primero una copia coherente de SQLite con `SQLite3::backup()`, luego copia las carpetas de usuarios y el snapshot. No usa `/MIR` ni `/PURGE`: por diseño no borra contenido ya existente en el segundo PC. Esta decisión es conservadora para recuperación; borre versiones antiguas manualmente solo tras confirmar que hay una copia reciente y válida.

### 10.4 Programar cada 30 minutos

Haga clic derecho en `C:\KrastyApp\scripts\setup_task_scheduler.bat` y elija **Ejecutar como administrador**. Escriba la cuenta de copia cuando se solicite, por ejemplo `NOMBRE-PC-PRINCIPAL\krastybackup`, y escriba su contraseña cuando `schtasks` la pida. La tarea se guarda en el Programador de tareas y se ejecuta cada treinta minutos aun con la sesión cerrada.

El comando oficial `schtasks /create` admite el plan `MINUTE`, el modificador `/MO` para definir el intervalo y una cuenta `/RU` para ejecutar la tarea [10]. Puede forzar una prueba sin esperar 30 minutos:

```bat
schtasks /Run /TN "Krasty Kebab - Copia al segundo ordenador"
```

Abra **Programador de tareas > Biblioteca del Programador de tareas**, seleccione `Krasty Kebab - Copia al segundo ordenador` y compruebe `Último resultado de ejecución`. Si falla, abra el log y consulte la tabla de solución de problemas. No cambie la tarea a `SYSTEM` salvo que sepa conceder permisos SMB a cuentas de equipo: la cuenta `krastybackup` evita esa complejidad.

## 11. Crear y custodiar la bóveda cifrada (tercera copia)

La tercera capa genera un ZIP con `users/` y un snapshot consistente de `krasty.sqlite`, lo cifra en streaming con **AES-256-CBC**, protege la integridad con **HMAC-SHA-256** y deriva claves desde `vault_passphrase` con PBKDF2-SHA256. El archivo no se puede recuperar si pierde la frase de bóveda. Guárdela fuera del servidor y fuera del hosting, al menos en dos ubicaciones físicas protegidas.

Antes de usarla, confirme que `extension=zip` y `extension=openssl` están activas. Desde una consola administrativa ejecute:

```bat
C:\xampp\php\php.exe C:\KrastyApp\scripts\create_vault_backup.php
```

El resultado aparece en:

```text
C:\KrastyData\backups\vault\krasty-vault-AAAA-MM-DD_HH-MM-SS.zip.enc
C:\KrastyData\backups\vault\krasty-vault-AAAA-MM-DD_HH-MM-SS.zip.enc.manifest.json
C:\KrastyData\backups\vault\krasty-vault-AAAA-MM-DD_HH-MM-SS.zip.enc.sha256
```

Suba **los tres archivos** a la tercera ubicación manualmente. Un hosting estático solo es aceptable si ofrece capacidad suficiente, no indexa el fichero y permite una cuenta privada; para una bóveda que puede contener decenas o cientos de GiB suele ser más adecuado un almacenamiento de archivos privado del proveedor. El contenido está cifrado, pero el control de acceso al hosting sigue siendo importante. No suba `config.php`, la frase de bóveda ni un ZIP sin cifrar.

Antes de borrar cualquier copia local, compare el hash local con el subido. En PowerShell:

```powershell
Get-FileHash "C:\KrastyData\backups\vault\krasty-vault-AAAA-MM-DD_HH-MM-SS.zip.enc" -Algorithm SHA256
```

El valor debe coincidir con el contenido del archivo `.sha256`. Programe esta bóveda de forma manual semanal o mensual al principio, y después de cualquier cambio importante. Es preferible realizarla cuando hay poca actividad, porque prepara un ZIP temporal y puede consumir disco y tiempo.

## 12. Publicar en Internet sin túnel ni VPN

Esta sección publica Apache directamente desde su ordenador. Requiere un dominio que controle, una IP pública o servicio DNS que apunte a ella, un certificado HTTPS válido y acceso a la configuración de su router. **No publique hasta haber probado registro, carga, copia al segundo PC y recuperación local.**

### 12.1 Activar HTTPS primero

Obtenga un certificado TLS válido para `web.com` de una autoridad certificadora. Apache documenta que un virtual host HTTPS mínimo necesita `Listen 443`, `SSLEngine on`, `SSLCertificateFile` y `SSLCertificateKeyFile` [11]. Añada un virtual host equivalente al siguiente; cambie las rutas de certificado por las entregadas por su autoridad.

```apache
Listen 443
<VirtualHost *:443>
    ServerName web.com
    DocumentRoot "C:/KrastyApp/web"

    SSLEngine on
    SSLCertificateFile "C:/ruta/segura/web.com.crt"
    SSLCertificateKeyFile "C:/ruta/segura/web.com.key"

    <Directory "C:/KrastyApp/web">
        Options -Indexes
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Cuando `https://web.com/` funcione sin advertencia del navegador, cambie en `config.php`:

```php
'cookie_secure' => true,
```

Reinicie Apache y pruebe iniciar y cerrar sesión. No deje el portal público por HTTP: las credenciales y sesiones necesitan el cifrado de transporte.

### 12.2 Firewall, DNS y router

En **Firewall de Microsoft Defender con seguridad avanzada**, cree una regla de entrada de tipo **Puerto**, protocolo **TCP**, puerto local **443**, perfil **Público** y nombre `Krasty HTTPS`. Si usa temporalmente HTTP para una redirección a HTTPS, cree una regla TCP 80 separada; si no, manténgala cerrada. Microsoft explica que las reglas pueden restringir tráfico por aplicación, dirección, protocolo y puerto, y que no se debe desactivar el firewall completo [9].

En el router, cree un reenvío de puertos con esta única regla:

| Puerto externo | Protocolo | IP privada de destino | Puerto interno | Propósito |
|---|---|---|---|---|
| 443 | TCP | IP fija del PC principal, por ejemplo `192.168.1.50` | 443 | Portal HTTPS de Krasty |

La pantalla y los nombres cambian según la marca del router; consulte su manual con el término **Port Forwarding** o **NAT**. No cree reenvíos a 3306 (MariaDB), 11434 (Ollama), 21 (FTP), 22 (SSH) ni a las carpetas SMB. Cree un registro DNS `A` para `web.com` hacia la IP pública de la conexión. Si su proveedor de Internet usa CGNAT o bloquea 443, el reenvío no funcionará y deberá solicitar una IP pública o usar un servidor dedicado; esto no es un fallo de Krasty.

Por último, desde una red móvil distinta al Wi‑Fi local, visite `https://web.com/`. Compruebe el candado HTTPS, cree una cuenta de prueba, suba un archivo pequeño y elimínela si no la necesita.

## 13. Procedimientos de recuperación

No restaure sobre una aplicación en uso. Informe a los usuarios, detenga Apache desde el panel de XAMPP y haga una copia de la carpeta dañada con fecha antes de reemplazar nada. Cuando finalice, reinicie Apache, entre con una cuenta de prueba y descargue un archivo para validar.

### 13.1 Recuperar desde el segundo ordenador

En el PC principal, detenga Apache y abra una consola administrativa. Si el directorio principal está dañado, conserve una copia para investigación y recupere los datos:

```bat
ren C:\KrastyData C:\KrastyData_fallida_2026-08-18
mkdir C:\KrastyData
robocopy "\\ORDENADOR-RESPALDO\KrastyBackups\users" "C:\KrastyData\users" /E /COPY:DAT /R:2 /W:5
copy "\\ORDENADOR-RESPALDO\KrastyBackups\database\krasty.sqlite" "C:\KrastyData\krasty.sqlite"
```

Compruebe que existe `C:\KrastyData\krasty.sqlite`, que hay carpetas bajo `C:\KrastyData\users` y que la fecha del archivo corresponde con la última copia válida. Arranque Apache, abra el panel de administración y pruebe las cuentas. Si el segundo PC no tiene una copia reciente, no sobrescriba nada: use la bóveda.

### 13.2 Recuperar desde la bóveda cifrada

Descargue localmente los archivos `.zip.enc`, `.manifest.json` y `.sha256`. Compare primero el SHA-256 del archivo descargado con el `.sha256`. Cree una carpeta **vacía** de recuperación y ejecute:

```bat
mkdir C:\KrastyRecuperado
C:\xampp\php\php.exe C:\KrastyApp\scripts\decrypt_vault_backup.php "C:\Descargas\krasty-vault-AAAA-MM-DD_HH-MM-SS.zip.enc" "C:\KrastyRecuperado"
```

El recuperador verifica el HMAC antes de extraer el ZIP. Si la contraseña es incorrecta o el archivo fue alterado, no extrae datos. Tras una verificación correcta, encontrará:

```text
C:\KrastyRecuperado\users\
C:\KrastyRecuperado\database\krasty.sqlite
```

Con Apache detenido, renombre `C:\KrastyData` como copia fallida, cree de nuevo `C:\KrastyData`, copie `users` y copie `database\krasty.sqlite` como `C:\KrastyData\krasty.sqlite`. Inicie Apache y compruebe registro, inicio de sesión, archivos y panel antes de anunciar la recuperación.

### 13.3 Recuperar una cuenta o archivo concreto

Para restaurar solo un usuario, localice la carpeta `ID-nombre` correspondiente dentro del respaldo. **No copie la carpeta sin actualizar SQLite**, porque la base de datos contiene el listado y la relación de los archivos. La recuperación más segura es restaurar en un PC de pruebas y extraer desde la interfaz el archivo necesario. Para operaciones frecuentes de restauración granular, solicite una revisión técnica antes de manipular SQLite manualmente.

## 14. Migrar a otro ordenador o servidor

La migración se realiza en una ventana de mantenimiento para que la base de datos y los archivos coincidan.

1. Instale XAMPP y copie el paquete en el nuevo equipo siguiendo los capítulos 3 a 7.
2. Cree `C:\KrastyData` en el nuevo equipo y configure las rutas nuevas en `config.php`.
3. En el servidor antiguo, anuncie la pausa y detenga Apache. Ejecute una última vez `backup_to_second_pc.bat` o cree una bóveda cifrada.
4. Copie desde el segundo PC o desde la bóveda las carpetas `users` y `krasty.sqlite` al nuevo `C:\KrastyData`.
5. Copie `config.php` de forma segura o cree uno nuevo. Para descifrar una bóveda se necesita la **misma** `vault_passphrase`; cambiar `app_secret` invalida sesiones existentes, lo cual es conveniente tras una migración.
6. Configure el virtual host y HTTPS en el nuevo PC, pruebe localmente y solo entonces cambie el destino del reenvío 443 del router y el DNS si fuera necesario.
7. Mantenga el servidor antiguo apagado, sin publicar, durante unos días como copia de contingencia. No permita escrituras simultáneas en ambos equipos.

## 15. Mantenimiento y comprobaciones periódicas

Una copia no está demostrada hasta que se ha probado una restauración. Cada mes, cree una cuenta de prueba, suba un archivo de prueba, ejecute el respaldo al segundo PC y confirme que se encuentra allí. Cada trimestre, realice una restauración de la bóveda en un directorio alternativo, sin reemplazar producción.

| Frecuencia | Comprobación | Resultado esperado |
|---|---|---|
| Diaria | Revisar `backup_to_second_pc.log` | Una ejecución correcta reciente y sin código Robocopy ≥ 8 |
| Semanal | Crear bóveda cifrada | Tres archivos `.enc`, `.manifest.json` y `.sha256` completos |
| Mensual | Aplicar actualizaciones de Windows/XAMPP y revisar espacio | Apache funciona; hay espacio antes de que la cuota se agote |
| Trimestral | Simular recuperación en carpeta alternativa | Se abre SQLite y se recupera un archivo de prueba |
| Tras cambiar router/dominio | Probar HTTPS desde red externa | Certificado válido y solo 443 publicado |

## 16. Solución de problemas

| Síntoma | Causa probable | Solución segura |
|---|---|---|
| Apache no inicia | Puerto 80/443 ocupado por IIS u otro servicio | Detenga el servicio que ocupa el puerto o cambie la configuración; consulte `error.log`. No desactive el firewall. |
| Error 500 al abrir `/ana/Krasty/` | `mod_rewrite` o `mod_headers` desactivado; `.htaccess` ignorado | Revise `LoadModule`, `AllowOverride All` solo en `C:/KrastyApp/web` y reinicie Apache. |
| “La configuración no es válida” | `config.php` no existe o mantiene un texto `REEMPLACE_` | Copie de nuevo la plantilla y complete `app_secret` con un secreto real. |
| El inicio de sesión vuelve al formulario | `cookie_secure` está activo con URL HTTP | Pruebe localmente con `false`; active `true` solamente cuando HTTPS ya funcione. |
| Archivo demasiado grande | Límite de PHP o límite de 2 GiB por archivo | Revise `upload_max_filesize` y `post_max_size`; divida archivos mayores de 2 GiB. |
| El chat informa que Ollama no responde | Ollama o el modelo no están iniciados/nombre incorrecto | Ejecute `ollama run llama3.2`, confirme el modelo en `config.php` y mantenga URL `127.0.0.1`. |
| La copia secundaria no abre el recurso | Nombre SMB, permisos o contraseña de `krastybackup` incorrectos | Pruebe `\\ORDENADOR-RESPALDO\KrastyBackups` iniciando sesión con esa cuenta y corrija permisos de compartir y Seguridad. |
| El log muestra Robocopy 8 o superior | Uno o más archivos no se copiaron | Lea las líneas previas del log, compruebe disco, red y permisos; vuelva a ejecutar manualmente. |
| La tarea no se ejecuta | Cuenta de tarea incorrecta o contraseña cambiada | Abra Programador de tareas, actualice la cuenta/contraseña o ejecute de nuevo `setup_task_scheduler.bat`. |
| La bóveda no descifra | Frase incorrecta, archivo alterado o extensión PHP ausente | Verifique SHA-256, use la frase original y habilite `openssl` y `zip`; no borre otros respaldos. |
| La web no abre desde Internet | Falta DNS, certificado, regla de firewall, NAT o IP pública | Pruebe primero HTTPS local; después 443 en firewall/router y la IP pública. Consulte al proveedor si usa CGNAT. |

## 17. Lista final antes de aceptar usuarios públicos

Antes de publicar, confirme todos los puntos siguientes en una revisión pausada. Si alguno es negativo, mantenga el servicio privado hasta resolverlo.

- [ ] `config.php` está fuera de `web`, tiene secretos reales y no está en Git.
- [ ] `C:\KrastyData` está fuera de la raíz web y no está compartida públicamente.
- [ ] La primera cuenta administradora se creó con token; este token fue sustituido después.
- [ ] Cada usuario de prueba solo puede abrir su propio `/usuario/Krasty/`.
- [ ] Apache solo publica `C:\KrastyApp\web` y no hay listado de directorios.
- [ ] HTTPS funciona con un certificado válido y `cookie_secure` es `true`.
- [ ] El firewall no está desactivado; solo existe la regla mínima para 443.
- [ ] El router solo reenvía 443 al PC principal; no se publican SMB, Ollama, MariaDB, phpMyAdmin, FTP ni la bóveda.
- [ ] El segundo PC recibe una copia y se revisó el log tras una ejecución programada.
- [ ] La bóveda se creó, se verificó con SHA-256, se cargó manualmente y la frase está custodiada fuera de línea.
- [ ] Se ha realizado una restauración de prueba desde al menos una capa de respaldo.

## Referencias

[1]: https://www.apachefriends.org/faq_windows.html "XAMPP FAQs para Windows — Apache Friends"
[2]: https://www.apachefriends.org/download.html "Descargas oficiales de XAMPP — Apache Friends"
[3]: https://httpd.apache.org/docs/current/mod/mod_rewrite.html "Módulo mod_rewrite — Apache HTTP Server 2.4"
[4]: https://httpd.apache.org/docs/current/mod/core.html#allowoverride "Directiva AllowOverride — Apache HTTP Server 2.4"
[5]: https://www.php.net/manual/en/install.windows.extensions.php "Instalación de extensiones de PHP en Windows — Manual de PHP"
[6]: https://docs.ollama.com/windows "Ollama para Windows — Documentación oficial"
[7]: https://docs.ollama.com/api/generate "API Generate de Ollama — Documentación oficial"
[8]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy "robocopy — Microsoft Learn"
[9]: https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/ "Descripción general de Windows Firewall — Microsoft Learn"
[10]: https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks-create "schtasks create — Microsoft Learn"
[11]: https://httpd.apache.org/docs/current/ssl/ssl_howto.html "Guía de SSL/TLS para Apache HTTP Server"
