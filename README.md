# Krasty Kebab para XAMPP

Este paquete permite desplegar Krasty Kebab sin Node.js, usando Apache y PHP de XAMPP con SQLite. Sus funciones principales son el registro de usuarios, rutas privadas `web.com/usuario/Krasty/`, archivos aislados por cuenta, cuota predeterminada de 50 GiB, chat local opcional mediante Ollama y tres capas de respaldo.

> **Empiece por [`docs/MANUAL_COMPLETO.md`](https://github.com/Men-Bat/Fortress-Krasty/blob/local/Manual%20%20Krasty%20Kebab%20para%20XAMPP%20en%20Windows.md).** No copie este paquete directamente a una web pública sin completar la configuración HTTPS y las comprobaciones de seguridad explicadas allí.

| Primer paso | Archivo o carpeta |
|---|---|
| Configurar secretos y rutas | `config/config.example.php` → copia privada `config/config.php` |
| Publicar con Apache | `web/` como `DocumentRoot` del virtual host |
| Lógica PHP | `app/bootstrap.php` y `web/index.php` |
| Copia al segundo PC | `scripts/backup_to_second_pc.bat` |
| Crear bóveda cifrada | `scripts/create_vault_backup.php` |
| Recuperar bóveda | `scripts/decrypt_vault_backup.php` |
| Instrucciones completas | `docs/MANUAL_COMPLETO.md` |

No incluya `config/config.php`, datos de usuarios, registros ni bóvedas en copias de código o repositorios.
