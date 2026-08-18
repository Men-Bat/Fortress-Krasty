# 🏰🥙 Krasty Kebab: Fortress Edition

**Fortress Edition** combina un servidor local primario, una réplica cloud de contingencia y una bóveda cifrada de recuperación. Está orientado a despliegues privados y administrados, no a una publicación abierta inmediata.

> **Lea `AUDITORIA_TECNICA.md` antes de desplegar.** El paquete incorpora mejoras críticas, pero aún requiere cuotas reales, roles administrativos y MFA antes de ofrecer cuentas públicas.

## Inicio rápido seguro

```bash
cp .env.example .env
# Complete todos los secretos y URLs reales en .env; nunca lo suba a Git.
npm ci --omit=dev
npm audit --omit=dev
npm start
```

La documentación operativa, la migración de hosting, la promoción de contingencia y la restauración desde la bóveda se encuentran en **[`MANUAL_OPERACIONES_FORTRESS.md`](https://github.com/Men-Bat/Fortress-Krasty/blob/Pruebas/Manual%20de%20operaciones%20%E2%80%94%20Krasty%20Kebab_%20Fortress%20Edition.md).**.
## Archivos clave

| Archivo o carpeta | Finalidad |
|---|---|
| `index.js` | Aplicación HTTP, autenticación, chat y endpoint de sincronización |
| `public/gateway.html` | Punto de entrada que prioriza el primario disponible |
| `public/gateway-config.example.js` | Plantilla pública de URLs, sin secretos |
| `scripts/sync-engine.js` | Sincronización de base de datos al cloud y archivo cifrado a la bóveda |
| `scripts/create-snapshot.js` | Genera un backup cifrado de SQLite y `data/` |
| `scripts/decrypt-snapshot.js` | Descifra una copia de la bóveda para recuperación |
| `.env.example` | Inventario de configuración; debe copiarse como `.env` |

## Advertencias fundamentales

La aplicación permite una sola fuente de escritura. No ejecute dos nodos con `NODE_ROLE=primary`. No exponga `data/`, `db/`, `archives/` o `.env` a la web. No coloque claves de Chutes.ai ni de la bóveda en el navegador.
