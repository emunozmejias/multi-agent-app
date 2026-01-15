# Arquitectura Multi-Agente del Backend

## 📖 Explicación Simple

### ¿Qué hace el sistema?

Imagina que tienes un equipo de investigadores trabajando juntos:

1. **El Jefe (Research Manager)**: Coordina todo el trabajo y asegura que se complete la tarea
2. **Los Investigadores (Research Agents)**: Cada uno investiga una tecnología específica

Cuando solicitas información sobre tecnologías y áreas de negocio:

1. El sistema crea un **equipo virtual** de agentes de IA
2. Cada agente recibe una **tarea específica** (investigar una tecnología)
3. Los agentes **trabajan en paralelo** buscando información en internet y YouTube
4. El jefe **reúne todos los resultados** y los organiza
5. Finalmente, recibes un **JSON estructurado** con todos los artículos y videos encontrados

### Flujo Simple

```
Usuario → API → Crea Equipo de Agentes → Agentes Trabajan → Resultados
```

**Ejemplo práctico:**
- Pides: Tecnologías ["Generative AI", "Machine Learning"] y Áreas ["Customer Service", "Healthcare"]
- El sistema crea 2 tareas de investigación (una por cada tecnología)
- Cada agente busca 3 artículos de blog + 3 videos de YouTube para cada área de negocio
- El manager junta todo y te devuelve un JSON organizado

---

## 🔍 Explicación Detallada

### Arquitectura General

El backend utiliza **CrewAI**, un framework diseñado para crear sistemas multi-agente donde varios agentes de IA colaboran para completar tareas complejas.

```
┌─────────────────────────────────────────────────────────────┐
│                      Flask API Layer                        │
│  - Recibe peticiones HTTP                                    │
│  - Crea threads para procesamiento asíncrono               │
│  - Gestiona estado y eventos                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  TechnologyResearchCrew                       │
│  - Orquesta el equipo de agentes                             │
│  - Configura tareas y flujo de trabajo                       │
│  - Ejecuta el crew y gestiona resultados                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
┌──────────────────┐         ┌──────────────────┐
│  Research Manager│         │  Research Agent  │
│     (Jefe)        │         │  (Investigador)   │
│                  │         │                  │
│  - Coordina      │         │  - Investiga     │
│  - Agrega datos  │         │  - Busca info    │
│  - Valida        │         │  - Ejecuta tareas│
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         └────────────┬───────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │      Tools/APIs        │
         │                        │
         │  - SerperDevTool       │
         │  - YoutubeSearchTool   │
         │  - OpenAI GPT-4        │
         └────────────────────────┘
```

### Componentes Principales

#### 1. **API Layer** (`api.py`)

**Responsabilidades:**
- Recibir peticiones HTTP del frontend
- Crear identificadores únicos para cada investigación
- Lanzar procesamiento asíncrono en threads separados
- Exponer endpoints REST para consultar estado y resultados

**Endpoints:**

```python
POST /api/multiagent
# Recibe: {technologies: [...], businessareas: [...]}
# Retorna: {input_id: "uuid"}

GET /api/multiagent/<input_id>
# Retorna: {status, result, events}
```

**Características clave:**
- **Procesamiento asíncrono**: Cada investigación corre en un thread separado
- **No bloqueante**: La API responde inmediatamente con un `input_id`
- **Thread-safe**: Usa locks para gestionar acceso concurrente a `outputs`

#### 2. **TechnologyResearchCrew** (`crews.py`)

**Responsabilidades:**
- Configurar y orquestar el equipo de agentes
- Definir el flujo de trabajo (qué agente hace qué)
- Ejecutar el crew y gestionar el ciclo de vida

**Flujo de setup:**

```python
1. Crear instancia de ResearchAgents
2. Instanciar agentes (Manager + Research Agent)
3. Crear instancia de ResearchTasks
4. Generar tareas para cada tecnología
5. Crear tarea de gestión final
6. Construir Crew con agentes y tareas
7. Ejecutar crew.kickoff()
```

**Estructura del Crew:**

```
Crew:
  Agents:
    - research_manager (1 instancia)
    - research_agent (1 instancia, reutilizada)
  
  Tasks:
    - technology_research (N tareas, una por tecnología)
    - manage_research (1 tarea final)
```

#### 3. **ResearchAgents** (`agents.py`)

**Define dos tipos de agentes:**

##### **Research Manager Agent**

```python
Role: "Research Manager"
Goal: Agregar toda la información investigada en un formato JSON estructurado
Backstory: Responsable de coordinar y consolidar resultados
Capabilities:
  - allow_delegation=True (puede delegar tareas a otros agentes)
  - tools: [SerperDevTool, YoutubeVideoSearchTool]
  - llm: GPT-4 Turbo
```

**Características:**
- **Coordinador**: No hace investigación directa, organiza resultados
- **Validador**: Asegura que todos los datos estén completos
- **Agregador**: Combina información de múltiples fuentes

##### **Research Agent**

```python
Role: "Research Agent"
Goal: Investigar áreas de negocio específicas para una tecnología
Backstory: Responsable de buscar información detallada
Capabilities:
  - tools: [SerperDevTool, YoutubeVideoSearchTool]
  - llm: GPT-4 Turbo
  - async_execution=True (puede ejecutar tareas en paralelo)
```

**Características:**
- **Especialista**: Se enfoca en una tecnología específica
- **Ejecutor**: Realiza búsquedas activas en internet y YouTube
- **Paralelizable**: Múltiples instancias pueden trabajar simultáneamente

**Herramientas disponibles para ambos agentes:**

1. **SerperDevTool**: Búsqueda semántica en internet
   - Busca artículos de blog relevantes
   - Retorna URLs y snippets de contenido

2. **YoutubeVideoSearchTool**: Búsqueda en YouTube
   - Busca videos por keywords
   - Retorna título y URL del video

#### 4. **ResearchTasks** (`tasks.py`)

**Define dos tipos de tareas:**

##### **technology_research Task**

```python
Description: "Investigar las áreas de negocio {businessareas} 
              para la tecnología {technology}"
Agent: research_agent
Expected Output: JSON con información para cada área de negocio
Execution: async_execution=True (ejecuta en paralelo)
Output Schema: BusinessareaInfo
```

**Comportamiento:**
- Se crea **una tarea por cada tecnología** solicitada
- Cada tarea se asigna al `research_agent`
- El agente busca información para **todas las áreas de negocio** de esa tecnología
- Ejecuta búsquedas en paralelo cuando es posible
- Retorna un JSON estructurado con los resultados

**Ejemplo:**
```
Tecnología: "Generative AI"
Áreas: ["Customer Service", "Healthcare"]

El agente busca:
- 3 artículos de blog sobre "Generative AI in Customer Service"
- 3 videos de YouTube sobre "Generative AI in Customer Service"
- 3 artículos de blog sobre "Generative AI in Healthcare"
- 3 videos de YouTube sobre "Generative AI in Healthcare"
```

##### **manage_research Task**

```python
Description: "Agregar toda la información investigada en un formato JSON"
Agent: research_manager
Expected Output: JSON con todas las combinaciones tecnología/área
Context: Recibe resultados de todas las tareas technology_research
Output Schema: BusinessareaInfoList
```

**Comportamiento:**
- Se ejecuta **después** de todas las tareas de investigación
- Recibe como `context` los resultados de todas las tareas previas
- El manager agrega, valida y estructura toda la información
- Asegura que no falte ninguna combinación
- Retorna el JSON final completo

#### 5. **Log Manager** (`log_manager.py`)

**Responsabilidades:**
- Almacenar eventos en tiempo real durante la ejecución
- Gestionar estado de cada investigación (STARTED, COMPLETE, ERROR)
- Proporcionar thread-safety para acceso concurrente

**Estructura de datos:**

```python
outputs: Dict[str, Output] = {
    "input_id_1": Output(
        status="COMPLETE",
        events=[Event(...), Event(...)],
        result="JSON string con resultados"
    ),
    ...
}
```

**Eventos capturados:**
- "CREW STARTED": Cuando inicia el procesamiento
- Salidas de cada tarea (exported_output)
- "CREW COMPLETED": Cuando finaliza exitosamente
- "CREW FAILED": Si ocurre un error

**Thread Safety:**
- Usa `Lock()` para prevenir condiciones de carrera
- Múltiples threads pueden agregar eventos simultáneamente
- El frontend puede consultar estado sin bloqueos

#### 6. **Models** (`models.py`)

**Modelos Pydantic para validación:**

```python
NamedUrl:
  - name: str (título del video)
  - url: str (enlace al video)

BusinessareaInfo:
  - technology: str
  - businessarea: str
  - blog_articles_urls: List[str] (3 URLs)
  - youtube_videos_urls: List[NamedUrl] (3 videos)

BusinessareaInfoList:
  - businessareas: List[BusinessareaInfo]
```

**Propósito:**
- Validar estructura de datos antes de retornar
- Asegurar que los agentes generen JSON válido
- Proporcionar esquema claro para el frontend

### Flujo de Ejecución Detallado

#### Fase 1: Inicialización

```
1. Frontend → POST /api/multiagent
   {
     technologies: ["Generative AI", "Machine Learning"],
     businessareas: ["Customer Service", "Healthcare"]
   }

2. API genera input_id único (UUID)

3. API crea thread separado:
   Thread(target=kickoff_crew, args=(input_id, technologies, businessareas))

4. API retorna inmediatamente:
   {input_id: "abc-123-def-456"}
```

#### Fase 2: Setup del Crew

```
En el thread separado:

1. Crear TechnologyResearchCrew(input_id)

2. setup_crew():
   a. Crear ResearchAgents()
      - Inicializar herramientas (SerperDevTool, YoutubeVideoSearchTool)
      - Crear instancia de ChatOpenAI (GPT-4 Turbo)
   
   b. Crear agentes:
      - research_manager = agents.research_manager(...)
      - research_agent = agents.research_agent()
   
   c. Crear ResearchTasks(input_id)
   
   d. Generar tareas:
      - technology_research("Generative AI", [...])
      - technology_research("Machine Learning", [...])
      - manage_research(..., [todas las tareas anteriores])
   
   e. Construir Crew:
      Crew(
        agents=[research_manager, research_agent],
        tasks=[task1, task2, manage_task]
      )
```

#### Fase 3: Ejecución del Crew

```
crew.kickoff() inicia el procesamiento:

1. CrewAI analiza las tareas y dependencias

2. Ejecuta tareas technology_research (pueden ser paralelas):
   
   Para "Generative AI":
     a. research_agent recibe la tarea
     b. Agente usa SerperDevTool para buscar artículos
        Query: "Generative AI Customer Service blog articles"
     c. Agente usa YoutubeVideoSearchTool para buscar videos
        Query: "Generative AI in Customer Service"
     d. Repite para "Healthcare"
     e. Genera JSON: BusinessareaInfo
     f. Callback: append_event(input_id, resultado)
   
   Para "Machine Learning":
     (Mismo proceso en paralelo si async_execution=True)

3. Ejecuta manage_research:
     a. research_manager recibe todas las tareas previas como context
     b. Agrega y valida toda la información
     c. Asegura que no falten combinaciones
     d. Genera JSON final: BusinessareaInfoList
     e. Callback: append_event(input_id, resultado final)

4. CrewAI retorna el resultado final
```

#### Fase 4: Finalización

```
1. kickoff_crew() guarda resultados:
   outputs[input_id].status = 'COMPLETE'
   outputs[input_id].result = results (JSON string)

2. Frontend hace polling:
   GET /api/multiagent/<input_id>
   
3. API retorna:
   {
     status: "COMPLETE",
     result: {
       businessareas: [
         {
           technology: "Generative AI",
           businessarea: "Customer Service",
           blog_articles_urls: [...],
           youtube_videos_urls: [...]
         },
         ...
       ]
     },
     events: [...]
   }
```

### Características Avanzadas

#### 1. **Delegación de Tareas**

El `research_manager` tiene `allow_delegation=True`, lo que significa:
- Puede decidir que otra tarea necesita más investigación
- Puede pedirle al `research_agent` que profundice en algo específico
- CrewAI gestiona automáticamente esta coordinación

#### 2. **Ejecución Asíncrona**

Las tareas `technology_research` tienen `async_execution=True`:
- Múltiples tecnologías se investigan en paralelo
- No hay que esperar a que termine una para empezar otra
- Reduce significativamente el tiempo total de ejecución

#### 3. **Callbacks y Eventos**

Cada tarea tiene un `callback`:
```python
callback=self.append_event_callback
```

Cuando una tarea completa:
1. Se ejecuta el callback
2. Se captura el `exported_output` de la tarea
3. Se agrega al log de eventos
4. El frontend puede ver el progreso en tiempo real

#### 4. **Validación de Esquema**

Las tareas especifican `output_json=BusinessareaInfo`:
- CrewAI valida que el output del agente coincida con el esquema
- Si no coincide, el agente debe corregir su respuesta
- Garantiza estructura de datos consistente

#### 5. **Thread Safety**

El `log_manager` usa locks:
```python
with outputs_lock:
    outputs[input_id].events.append(...)
```

Permite:
- Múltiples investigaciones simultáneas
- Múltiples threads agregando eventos
- Consultas del frontend sin bloqueos

### Integración con APIs Externas

#### SerperDevTool
- **API**: Serper.dev
- **Uso**: Búsqueda semántica en Google
- **Input**: Query de búsqueda
- **Output**: URLs de artículos, snippets, títulos

#### YoutubeVideoSearchTool
- **API**: YouTube Data API v3
- **Uso**: Búsqueda de videos
- **Input**: Keywords
- **Output**: Lista de videos con título y URL

#### ChatOpenAI (LangChain)
- **API**: OpenAI GPT-4 Turbo
- **Uso**: Razonamiento de los agentes
- **Input**: Prompts del agente
- **Output**: Decisiones y generación de texto

### Manejo de Errores

#### Niveles de Error

1. **Validación de API Keys**
   - Se valida `SERPER_API_KEY` al inicializar agentes
   - Se valida `YOUTUBE_API_KEY` al usar la herramienta
   - Errores se propagan como excepciones

2. **Errores de API Externa**
   - Error 403 en YouTube: Se captura y se retorna mensaje descriptivo
   - Errores de Serper: Se propagan al agente
   - El agente puede intentar búsquedas alternativas

3. **Errores de Ejecución**
   - Se capturan en `kickoff_crew()`
   - Se registran en `outputs[input_id].status = 'ERROR'`
   - Se agrega evento "CREW FAILED" con detalles

4. **Errores de Validación**
   - Si el output no coincide con el esquema, CrewAI lo detecta
   - El agente recibe feedback y puede corregir

### Optimizaciones y Consideraciones

#### Paralelización
- Tareas de diferentes tecnologías se ejecutan en paralelo
- Múltiples búsquedas pueden ocurrir simultáneamente
- El manager espera a que todas las tareas terminen

#### Memoria
- Los resultados se almacenan en memoria (`outputs` dict)
- Se pierden al reiniciar el servidor
- Para producción, considerar base de datos

#### Rate Limiting
- Las APIs externas tienen límites de rate
- CrewAI gestiona automáticamente los delays necesarios
- Considerar implementar rate limiting adicional si es necesario

#### Timeouts
- Las tareas pueden tardar varios minutos
- No hay timeout explícito configurado
- Considerar agregar timeouts para producción

### Extensibilidad

El sistema está diseñado para ser extensible:

1. **Nuevos Agentes**: Agregar más tipos de agentes en `ResearchAgents`
2. **Nuevas Tareas**: Crear nuevas tareas en `ResearchTasks`
3. **Nuevas Herramientas**: Agregar tools personalizados en `tools/`
4. **Nuevos Modelos**: Cambiar el LLM en `ChatOpenAI(model="...")`
5. **Nuevos Esquemas**: Agregar modelos Pydantic en `models.py`

### Resumen de la Arquitectura

```
┌──────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA MULTI-AGENTE                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  API Layer (Flask)                                            │
│    │                                                           │
│    ├─ Recibe peticiones                                      │
│    ├─ Crea threads                                            │
│    └─ Expone estado                                           │
│                                                               │
│  Crew Layer (CrewAI)                                          │
│    │                                                           │
│    ├─ Research Manager Agent                                  │
│    │   └─ Coordina y agrega resultados                       │
│    │                                                           │
│    └─ Research Agent                                          │
│        └─ Investiga tecnologías específicas                  │
│                                                               │
│  Task Layer                                                   │
│    │                                                           │
│    ├─ technology_research (N tareas)                           │
│    │   └─ Una por cada tecnología                            │
│    │                                                           │
│    └─ manage_research (1 tarea)                              │
│        └─ Agrega todos los resultados                        │
│                                                               │
│  Tool Layer                                                   │
│    │                                                           │
│    ├─ SerperDevTool (búsqueda web)                           │
│    ├─ YoutubeVideoSearchTool (búsqueda videos)                │
│    └─ ChatOpenAI (razonamiento)                               │
│                                                               │
│  State Management                                             │
│    │                                                           │
│    └─ Log Manager (eventos y estado)                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

Esta arquitectura permite que múltiples agentes de IA colaboren de manera coordinada para completar tareas complejas de investigación de manera eficiente y estructurada.
