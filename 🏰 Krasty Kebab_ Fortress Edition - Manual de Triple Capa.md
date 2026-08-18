# 🏰 Krasty Kebab: Fortress Edition - Manual de Triple Capa

Bienvenido al sistema de máxima disponibilidad. Esta edición está diseñada para que tu asistente nunca esté fuera de línea, utilizando tres capas de infraestructura sincronizadas.

## 🏗️ La Arquitectura de Triple Capa

1.  **CAPA 1: Servidor Local (Primario)**
    - **Uso**: Tu máquina personal (PC/Mac/Linux).
    - **Ventaja**: Máxima privacidad, uso de GPUs locales (Ollama/LM Studio) y sin costes de hosting.
    - **Acceso**: `http://localhost:3000` (o vía IP local/VPN).

2.  **CAPA 2: Servidor Cloud (Respaldo Activo)**
    - **Uso**: Un hosting compartido, VPS o servidor en la nube.
    - **Ventaja**: Accesible desde cualquier lugar si tu PC local está apagado.
    - **Acceso**: `https://tu-dominio.com`.

3.  **CAPA 3: Bóveda Estática (Emergencia)**
    - **Uso**: GitHub Pages, S3 o una web estática simple.
    - **Ventaja**: Siempre disponible para consultar archivos y respaldos si los servidores de aplicación fallan.

## 📡 El Smart Gateway (Puerta de Enlace)

El archivo `public/gateway.html` es el punto de entrada inteligente. Debes alojarlo en un sitio siempre activo. Este archivo:
- Hace un "ping" a tu servidor local.
- Si no responde, hace un "ping" al servidor cloud.
- Te redirige automáticamente al nodo más rápido disponible.

## 🔄 Sincronización de Datos (Sync-Engine)

El archivo `scripts/sync-engine.js` es el corazón de la persistencia.
- Debe ejecutarse en tu **Servidor Local**.
- Cada 30 minutos, empaqueta tu base de datos y la envía de forma segura al **Servidor Cloud**.
- **Configuración**: Edita las URLs en el script y asegúrate de que el `SYNC_TOKEN` coincida en ambos extremos.

## 🚀 Pasos para el Despliegue

### En tu Servidor Local:
1. Inicia Ollama o LM Studio.
2. Ejecuta `node index.js` (Puerto 3000).
3. Ejecuta `node scripts/sync-engine.js` en una terminal separada.

### En tu Servidor Cloud:
1. Sube el proyecto y ejecuta `npm install`.
2. Configura la variable de entorno `NODE_TYPE=cloud`.
3. Inicia la app. Los datos se recibirán automáticamente desde tu local.

### En la Bóveda Estática:
1. Sube tus archivos de respaldo generados por el sistema de memoria de seguridad.

---
*¡La Fortaleza está lista. Tus datos están a salvo y tu IA siempre disponible!*
