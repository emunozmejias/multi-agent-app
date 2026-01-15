# Multi-Agent Research Application

Aplicación web full-stack que utiliza inteligencia artificial multi-agente para realizar investigaciones automatizadas sobre tecnologías y áreas de negocio. El sistema busca y recopila automáticamente artículos de blog y videos de YouTube relacionados con combinaciones específicas de tecnologías y áreas de negocio.

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Stack Tecnológico](#stack-tecnológico)
- [Componentes del Sistema](#componentes-del-sistema)
- [Flujo de Interacción Frontend-Backend](#flujo-de-interacción-frontend-backend)
- [Instalación y Configuración](#instalación-y-configuración)
- [Iniciar la Aplicación](#iniciar-la-aplicación)
- [Estructura del Proyecto](#estructura-del-proyecto)

## 🎯 Descripción General

Esta aplicación permite a los usuarios especificar:
- **Technologies**: Lista de tecnologías a investigar (ej: "Generative AI", "Machine Learning")
- **Business Areas**: Lista de áreas de negocio (ej: "Customer Service", "Healthcare")

El sistema utiliza agentes de IA que trabajan en equipo (usando CrewAI) para:
1. Buscar en internet artículos de blog recientes relacionados con cada combinación tecnología/área de negocio
2. Buscar en YouTube videos relevantes sobre cada combinación
3. Compilar los resultados en un formato estructurado JSON

## 🏗️ Arquitectura del Sistema

### Arquitectura General

```
┌─────────────────┐         HTTP/REST API         ┌─────────────────┐
│                 │◄──────────────────────────────►│                 │
│    Frontend     │         (Polling cada 1s)      │    Backend      │
│   (Next.js)     │                                │   (Flask API)   │
│                 │                                │                 │
│  - React UI     │                                │  - CrewAI       │
│  - TypeScript   │                                │  - Multi-Agent  │
│  - TailwindCSS  │                                │  - LangChain    │
└─────────────────┘                                └─────────────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │  External APIs   │
                                                  │                 │
                                                  │  - Serper API    │
                                                  │  - YouTube API  │
                                                  │  - OpenAI API   │
                                                  └─────────────────┘
```

### Patrón de Comunicación

- **Frontend → Backend**: Peticiones HTTP REST (POST para iniciar, GET para consultar estado)
- **Backend → Frontend**: Respuestas JSON con estado y resultados
- **Comunicación Asíncrona**: El backend procesa las tareas en hilos separados, y el frontend consulta el estado periódicamente (polling)

## 💻 Stack Tecnológico

### Frontend

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Next.js** | 14.2.3 | Framework React con SSR y routing |
| **React** | 18.x | Biblioteca UI declarativa |
| **TypeScript** | 5.x | Tipado estático para JavaScript |
| **TailwindCSS** | 3.4.1 | Framework CSS utility-first |
| **Axios** | 1.6.8 | Cliente HTTP para peticiones API |
| **React Hot Toast** | 2.4.1 | Notificaciones toast elegantes |

### Backend

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Python** | 3.10-3.11 | Lenguaje de programación |
| **Flask** | 3.0.2 | Framework web ligero |
| **CrewAI** | 0.22.4 | Framework para sistemas multi-agente |
| **LangChain** | - | Framework para aplicaciones LLM |
| **LangChain OpenAI** | - | Integración con modelos OpenAI |
| **Pydantic** | 2.6.3 | Validación de datos y modelos |
| **Flask-CORS** | 4.0.0 | Manejo de CORS para desarrollo |
| **python-dotenv** | 1.0.0 | Carga de variables de entorno |

### APIs Externas

- **Serper API**: Búsquedas semánticas en internet
- **YouTube Data API v3**: Búsqueda de videos en YouTube
- **OpenAI API**: Modelo de lenguaje GPT-4 Turbo para los agentes

## 🧩 Componentes del Sistema

### Frontend Components

#### 1. **InputSection** (`components/InputSection.tsx`)
- Componente reutilizable para agregar/eliminar elementos de lista
- Usado para Technologies y Business Areas
- Incluye validación y UI interactiva

#### 2. **FinalOutput** (`components/FinalOutput.tsx`)
- Muestra los resultados finales de la investigación
- Presenta artículos de blog y videos de YouTube organizados por tecnología y área de negocio
- Enlaces clicables con formato estructurado

#### 3. **EventLog** (`components/EventLog.tsx`)
- Muestra el log de eventos en tiempo real
- Registra el progreso de las tareas de los agentes
- Scroll automático para eventos recientes

#### 4. **Header** (`components/Header.tsx`)
- Encabezado de la aplicación (si existe)

### Frontend Hooks

#### **useCrewOutput** (`hooks/useCrewOutput.tsx`)
Hook personalizado que gestiona:
- Estado de la aplicación (running, technologies, businessareas)
- Comunicación con el backend mediante polling
- Actualización automática de eventos y resultados
- Manejo de errores y notificaciones

### Backend Components

#### 1. **API Routes** (`api.py`)
- `POST /api/multiagent`: Inicia una nueva investigación
- `GET /api/multiagent/<input_id>`: Consulta el estado y resultados

#### 2. **Agents** (`agents.py`)
Define dos tipos de agentes de IA:

- **Research Manager Agent**:
  - Rol: Coordinar y agregar toda la información investigada
  - Herramientas: Búsqueda en internet, búsqueda en YouTube
  - Permite delegación de tareas

- **Research Agent**:
  - Rol: Investigar áreas de negocio específicas para una tecnología
  - Herramientas: Búsqueda en internet, búsqueda en YouTube
  - Ejecuta tareas de forma asíncrona

#### 3. **Tasks** (`tasks.py`)
Define las tareas que realizan los agentes:

- **technology_research**: Investigar cada tecnología en sus áreas de negocio
- **manage_research**: Agregar y organizar todos los resultados

#### 4. **Crew** (`crews.py`)
- **TechnologyResearchCrew**: Orquesta los agentes y tareas
- Gestiona el flujo de trabajo multi-agente
- Maneja la ejecución y resultados

#### 5. **Tools** (`tools/youtube_search_tools.py`)
- **YoutubeVideoSearchTool**: Herramienta personalizada para buscar videos en YouTube
- Integra con YouTube Data API v3
- Retorna resultados estructurados (título y URL)

#### 6. **Log Manager** (`log_manager.py`)
- Gestiona eventos y estado de las investigaciones
- Almacenamiento en memoria con thread-safety
- Tracking de progreso en tiempo real

#### 7. **Models** (`models.py`)
Modelos Pydantic para validación:
- `BusinessareaInfo`: Información de una combinación tecnología/área
- `BusinessareaInfoList`: Lista de todas las combinaciones
- `NamedUrl`: URL con nombre descriptivo

## 🔄 Flujo de Interacción Frontend-Backend

### 1. Inicio de Investigación

```
Usuario → Frontend → POST /api/multiagent
                      {technologies: [...], businessareas: [...]}
                      
Backend → Genera input_id único
       → Crea thread separado para procesamiento
       → Retorna {input_id: "uuid"}
       
Frontend → Recibe input_id
        → Inicia polling cada 1 segundo
```

### 2. Procesamiento Asíncrono (Backend)

```
Thread separado:
1. Crea TechnologyResearchCrew
2. Configura agentes (Research Manager, Research Agent)
3. Crea tareas para cada tecnología
4. Ejecuta crew.kickoff()
5. Los agentes buscan información usando:
   - SerperDevTool (búsquedas web)
   - YoutubeVideoSearchTool (búsquedas YouTube)
6. Agrega eventos al log_manager durante el proceso
7. Al finalizar, guarda resultados en outputs[input_id]
```

### 3. Polling y Actualización (Frontend)

```
Frontend (cada 1 segundo):
→ GET /api/multiagent/<input_id>

Backend:
→ Retorna {
    status: "STARTED" | "COMPLETE" | "ERROR",
    result: {...},
    events: [...]
  }

Frontend:
→ Actualiza UI con eventos nuevos
→ Muestra resultados cuando status === "COMPLETE"
→ Detiene polling cuando finaliza
```

### 4. Visualización de Resultados

```
Frontend:
→ Renderiza FinalOutput con businessareaInfoList
→ Cada item muestra:
   - Technology
   - Business Area
   - 3 URLs de artículos de blog
   - 3 videos de YouTube (título + URL)
```

## 🚀 Instalación y Configuración

### Prerrequisitos

- **Node.js** 18+ y npm
- **Python** 3.10 o 3.11
- **Poetry** (gestor de dependencias Python) o pip
- Cuentas y API keys para:
  - [Serper.dev](https://serper.dev) - API key gratuita disponible
  - [Google Cloud Console](https://console.cloud.google.com) - YouTube Data API v3
  - [OpenAI](https://platform.openai.com) - API key

### Configuración de Variables de Entorno

1. Copia el archivo de ejemplo:
```bash
cp .env.example .env
```

2. Edita `.env` en la raíz del proyecto y agrega tus API keys:

```env
# API Keys requeridas
SERPER_API_KEY=tu_serper_api_key_aqui
YOUTUBE_API_KEY=tu_youtube_api_key_aqui
OPENAI_API_KEY=tu_openai_api_key_aqui

# Configuración opcional de LangChain
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
LANGCHAIN_API_KEY=tu_langchain_api_key_aqui
LANGCHAIN_PROJECT=tu_nombre_proyecto_aqui
```

> **Nota**: El archivo `.env` está en `.gitignore` y no se subirá al repositorio.

### Instalación del Backend

```bash
# Navegar al directorio backend
cd backend

# Si usas Poetry (recomendado)
poetry install

# O si usas pip
pip install -r ../requirements.txt
```

### Instalación del Frontend

```bash
# Navegar al directorio frontend
cd frontend

# Instalar dependencias
npm install
```

## ▶️ Iniciar la Aplicación

### Opción 1: Iniciar por Separado (Recomendado para desarrollo)

#### Terminal 1 - Backend

```bash
cd backend
python api.py
```

El backend estará disponible en: `http://localhost:3001`

#### Terminal 2 - Frontend

```bash
cd frontend
npm run dev
```

El frontend estará disponible en: `http://localhost:3000`

### Opción 2: Usando Poetry (Backend)

```bash
cd backend
poetry run python api.py
```

### Verificar que Todo Funciona

1. Abre `http://localhost:3000` en tu navegador
2. Agrega algunas tecnologías (ej: "Generative AI", "Machine Learning")
3. Agrega algunas áreas de negocio (ej: "Customer Service", "Healthcare")
4. Haz clic en "Start"
5. Observa el log de eventos y los resultados cuando finalice

## 📁 Estructura del Proyecto

```
.
├── backend/
│   ├── agents.py              # Definición de agentes de IA
│   ├── api.py                 # API Flask con endpoints REST
│   ├── crews.py               # Configuración de CrewAI
│   ├── log_manager.py          # Gestión de eventos y estado
│   ├── models.py              # Modelos Pydantic para validación
│   ├── tasks.py               # Definición de tareas para agentes
│   ├── pyproject.toml         # Configuración Poetry
│   ├── tools/
│   │   └── youtube_search_tools.py  # Herramienta personalizada YouTube
│   └── .gitignore
│
├── frontend/
│   ├── app/
│   │   ├── page.tsx           # Página principal
│   │   ├── layout.tsx          # Layout de la aplicación
│   │   └── globals.css        # Estilos globales
│   ├── components/
│   │   ├── EventLog.tsx       # Componente de log de eventos
│   │   ├── FinalOutput.tsx    # Componente de resultados
│   │   ├── Header.tsx         # Encabezado
│   │   └── InputSection.tsx   # Componente de entrada de datos
│   ├── hooks/
│   │   └── useCrewOutput.tsx  # Hook personalizado para comunicación con API
│   ├── package.json
│   └── next.config.mjs
│
├── .env.example              # Plantilla de variables de entorno
├── .gitignore                # Archivos ignorados por Git
├── requirements.txt          # Dependencias Python (alternativa a Poetry)
└── README.md                 # Este archivo
```

## 🔍 Características Principales

- ✅ **Sistema Multi-Agente**: Utiliza CrewAI para coordinar múltiples agentes de IA
- ✅ **Búsqueda Automatizada**: Integra búsquedas web y de YouTube
- ✅ **Interfaz en Tiempo Real**: Actualización automática de eventos y resultados
- ✅ **Procesamiento Asíncrono**: Las tareas se ejecutan en background sin bloquear la UI
- ✅ **Validación de Datos**: Usa Pydantic para asegurar estructura de datos correcta
- ✅ **Type Safety**: TypeScript en frontend para mayor robustez

## 🛠️ Desarrollo

### Modo Desarrollo

- **Backend**: Flask en modo debug (`debug=True`)
- **Frontend**: Next.js con hot-reload automático

### Debugging

- Los logs del backend aparecen en la consola donde ejecutas `api.py`
- Los logs del frontend aparecen en la consola del navegador (F12)
- Los eventos de los agentes se muestran en tiempo real en el componente EventLog

## 📝 Notas Adicionales

- El backend usa threads para procesar múltiples investigaciones simultáneamente
- El frontend hace polling cada 1 segundo para actualizar el estado
- Los resultados se almacenan en memoria (se pierden al reiniciar el servidor)
- Para producción, considera usar una base de datos para persistencia

## 📄 Licencia

Este proyecto es parte de un curso de bootcamp de IA.

---

**Desarrollado con** ❤️ **usando CrewAI, Next.js y Flask**
