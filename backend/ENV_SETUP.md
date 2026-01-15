# Configuración de Variables de Entorno

Este proyecto requiere las siguientes variables de entorno para funcionar correctamente.

## Variables Requeridas

Crea un archivo `.env` en el directorio `backend/` con las siguientes variables:

```env
# Serper API Key - Para búsquedas en internet
# Obtén tu clave en: https://serper.dev
SERPER_API_KEY=tu_serper_api_key_aqui

# YouTube Data API v3 Key - Para búsquedas de videos en YouTube
# Obtén tu clave en: https://console.cloud.google.com/apis/credentials
# Necesitas habilitar la API de YouTube Data API v3 en Google Cloud Console
YOUTUBE_API_KEY=tu_youtube_api_key_aqui

# OpenAI API Key - Para el modelo de lenguaje
# Obtén tu clave en: https://platform.openai.com/api-keys
OPENAI_API_KEY=tu_openai_api_key_aqui
```

## Cómo Obtener las Claves de API

### 1. SERPER_API_KEY
1. Ve a https://serper.dev
2. Crea una cuenta o inicia sesión
3. Obtén tu API key desde el dashboard
4. Cópiala en tu archivo `.env`

### 2. YOUTUBE_API_KEY
1. Ve a https://console.cloud.google.com/
2. Crea un nuevo proyecto o selecciona uno existente
3. Habilita la "YouTube Data API v3" desde la biblioteca de APIs
4. Ve a "Credenciales" y crea una nueva clave de API
5. Restringe la clave a "YouTube Data API v3" para mayor seguridad
6. Cópiala en tu archivo `.env`

### 3. OPENAI_API_KEY
1. Ve a https://platform.openai.com/api-keys
2. Inicia sesión o crea una cuenta
3. Crea una nueva API key
4. Cópiala en tu archivo `.env`

## Nota Importante

- **NUNCA** subas el archivo `.env` a un repositorio Git (ya está en `.gitignore`)
- Mantén tus claves de API seguras y no las compartas
- Si recibes un error 403 en YouTube API, verifica que:
  - La API esté habilitada en Google Cloud Console
  - La clave tenga los permisos correctos
  - No hayas excedido la cuota de la API
