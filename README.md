# ⚖️ Asistente Legal RAG — Sync Google Drive + PGVector

Sistema de consulta legal con IA sobre documentos propios, con sincronización automática desde Google Drive, deduplicación inteligente por checksum MD5 y persistencia en PostgreSQL con PGVector. Diseñado para equipos legales que necesitan consultar su base documental completa desde un chat.

---

## 🧩 El problema que resuelve

Un despacho o departamento legal acumula cientos de documentos: contratos, reglamentos, jurisprudencia, acuerdos. Encontrar un dato puntual requiere abrir varios archivos y leer manualmente. Y cada vez que se agrega un documento nuevo, hay que volver a indexar todo el sistema.

**Este asistente resuelve ambos problemas:**
- Sincroniza automáticamente los documentos desde una carpeta de Google Drive (sin subida manual)
- Solo re-indexa los archivos que cambiaron (deduplicación MD5)
- Permite consultar toda la base documental desde un chat en lenguaje natural

---

## 🏗️ Arquitectura

```
┌──────────────────────────────────────────────────────────────────┐
│         ASISTENTE LEGAL RAG — DRIVE SYNC + PGVECTOR             │
│                                                                  │
│  FLUJO 1: SINCRONIZACIÓN Y VECTORIZACIÓN (Cron diario)          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Cron Trigger (programado)                               │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  Listar archivos en Google Drive                         │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  Para cada archivo:                                      │   │
│  │    Calcular MD5 del archivo                              │   │
│  │           │                                              │   │
│  │    ¿MD5 existe en PostgreSQL (ingested_files)?          │   │
│  │    SÍ ──► Saltar (sin cambios)                          │   │
│  │    NO ──► Continuar                                      │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  Descargar PDF de Drive                                  │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  Recursive Character Text Splitter (overlap: 250)       │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  Generar embeddings (text-embedding-3-large)             │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  INSERT en n8n_vectors (PGVector)                        │   │
│  │  INSERT en ingested_files (MD5 + metadata)               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  FLUJO 2: CONSULTA AL AGENTE                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Chat Trigger (pregunta del usuario)                     │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  AI Agent (GPT-4.1-mini)                                │   │
│  │  + Herramienta: PGVector retrieval (similitud semántica) │   │
│  │  + Memory: Window Buffer Memory                          │   │
│  │  + System Prompt: abogado especialista                   │   │
│  │           │                                              │   │
│  │           ▼                                              │   │
│  │  Respuesta fundamentada en los documentos del Drive      │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Stack tecnológico

| Componente | Tecnología |
|---|---|
| Orquestación | n8n |
| Modelo de lenguaje | OpenAI GPT-4.1-mini |
| Embeddings | OpenAI text-embedding-3-large |
| Vector Store | PostgreSQL + PGVector (NeonTech) |
| Sincronización de documentos | Google Drive API |
| Deduplicación | Checksum MD5 |
| Tabla de control | PostgreSQL (`ingested_files`) |
| Text Splitter | Recursive Character Text Splitter |
| Programación de sync | Cron Trigger n8n |
| Memoria de conversación | Window Buffer Memory |

---

## 🔄 Flujo detallado

### 1. Sincronización automática con Google Drive

El cron ejecuta la sincronización en el intervalo configurado. Para cada archivo en la carpeta de Drive:

**Verificación de cambios (MD5):**
```
Archivo en Drive → Calcular MD5 → Consultar tabla ingested_files
    │
    ├── MD5 ya existe → No ha cambiado → Saltar archivo
    │
    └── MD5 nuevo o diferente → Archivo nuevo o modificado → Procesar
```

Esto garantiza que solo se vectorizan archivos nuevos o con cambios, sin re-indexar todo el corpus en cada ejecución.

**Vectorización:**
1. Descarga el PDF desde Google Drive
2. Extrae el texto completo del documento
3. Divide el contenido con Recursive Character Text Splitter (overlap de 250 tokens para mantener contexto entre fragmentos)
4. Genera embeddings con `text-embedding-3-large` (3072 dimensiones)
5. Inserta los vectores en la tabla `n8n_vectors` de PostgreSQL con PGVector
6. Registra el MD5 y metadata del archivo en `ingested_files` para futuras comparaciones

### 2. Consulta al agente legal

El usuario escribe su pregunta en el chat. El agente:

1. **Busca semánticamente** en `n8n_vectors` los fragmentos más relevantes para la pregunta
2. **Construye el contexto** combinando esos fragmentos con el historial de conversación
3. **Genera la respuesta** con GPT-4.1-mini, fundamentada exclusivamente en los documentos indexados
4. **Indica cuando no sabe** si la información no está en el corpus documental

---

## 🗄️ Estructura de base de datos

### Tabla de control de ingestión

```sql
CREATE TABLE ingested_files (
  id          SERIAL PRIMARY KEY,
  file_id     VARCHAR(255) UNIQUE NOT NULL,  -- ID del archivo en Google Drive
  file_name   VARCHAR(500) NOT NULL,
  md5_hash    VARCHAR(32) NOT NULL,           -- Checksum MD5 para deduplicación
  ingested_at TIMESTAMP DEFAULT NOW()
);
```

### Tabla de vectores (PGVector — generada por n8n)

```sql
-- n8n crea y gestiona esta tabla automáticamente
-- Estructura aproximada:
CREATE TABLE n8n_vectors (
  id        SERIAL PRIMARY KEY,
  content   TEXT,          -- Fragmento de texto original
  metadata  JSONB,         -- Nombre del archivo, posición, etc.
  embedding VECTOR(3072)   -- text-embedding-3-large (3072 dims)
);

CREATE INDEX ON n8n_vectors USING ivfflat (embedding vector_cosine_ops);
```

---

## ⚙️ Configuración

### Credenciales requeridas en n8n

| Credencial | Descripción |
|---|---|
| `OpenAI API` | API Key para embeddings y GPT-4.1-mini |
| `Google Drive OAuth2` | Cuenta con acceso a la carpeta de documentos |
| `PostgreSQL` | Conexión a tu base de datos (NeonTech recomendado) |

### Habilitar extensión PGVector en PostgreSQL

```sql
-- Ejecutar una sola vez en tu base de datos
CREATE EXTENSION IF NOT EXISTS vector;
```

### Configurar la carpeta de Google Drive

En el nodo de **Listar archivos de Drive**, especifica el ID de la carpeta que contiene los documentos legales. Puedes obtenerlo desde la URL de la carpeta en Google Drive:

```
https://drive.google.com/drive/folders/[FOLDER_ID_AQUI]
```

### Parámetros configurables

| Parámetro | Valor por defecto | Descripción |
|---|---|---|
| Cron interval | Configurable | Frecuencia de sincronización con Drive |
| Chunk Overlap | 250 tokens | Superposición entre fragmentos |
| Embedding Model | text-embedding-3-large | Modelo para vectores (3072 dims) |
| Chat Model | GPT-4.1-mini | Modelo para generar respuestas |

---

## 🚀 Instalación

1. **Clona el repositorio**
   ```bash
   git clone https://github.com/Jairh1294/Asistente-Legal-RAG-Sync-Google-Drive-PGVector.git
   ```

2. **Crea la base de datos en PostgreSQL**
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;

   CREATE TABLE ingested_files (
     id          SERIAL PRIMARY KEY,
     file_id     VARCHAR(255) UNIQUE NOT NULL,
     file_name   VARCHAR(500) NOT NULL,
     md5_hash    VARCHAR(32) NOT NULL,
     ingested_at TIMESTAMP DEFAULT NOW()
   );
   ```

3. **Importa el workflow en n8n**
   - Ve a **Workflows → Import from File**
   - Selecciona el archivo `.json` del repositorio

4. **Configura las credenciales**
   - En n8n: **Settings → Credentials → New**
   - Agrega: OpenAI API, Google Drive OAuth2, PostgreSQL

5. **Configura la carpeta de Drive**
   - En el nodo de listado de archivos, reemplaza el ID de carpeta con el tuyo

6. **Activa el workflow**
   - Haz clic en **Activate**
   - Ejecuta la primera sincronización manualmente para indexar los documentos existentes
   - El cron se encargará de las actualizaciones automáticas

---

## 💬 Ejemplo de uso

```
[Primera ejecución: sincroniza 47 documentos de la carpeta Drive]
✅ Sincronización completada: 47 nuevos documentos indexados.

👤 Abogado: ¿En qué contratos tenemos una cláusula de arbitraje?

🤖 Agente: Encontré cláusulas de arbitraje en los siguientes documentos:

          1. **Contrato_Distribucion_Acme_2024.pdf** — Cláusula 15:
             Disputa resuelta mediante arbitraje ante la Cámara de
             Comercio Internacional, sede Ciudad de México.

          2. **Acuerdo_Servicios_TechCorp_2025.pdf** — Cláusula 22:
             Arbitraje vinculante conforme al Reglamento de la CAM.

          Los demás contratos indexados no contienen esta cláusula.

👤 Abogado: ¿Cuál es el plazo de prescripción en el contrato de Acme?

🤖 Agente: Según la Cláusula 18 del Contrato_Distribucion_Acme_2024.pdf,
           las acciones derivadas de este contrato prescriben a los
           2 años contados desde que ocurra el incumplimiento.
```

---

## 🔄 Ventajas vs. versión in-memory

| Característica | Vector Store en Memoria | PGVector + Drive Sync |
|---|---|---|
| Persistencia | ❌ Se pierde al reiniciar | ✅ Permanente en PostgreSQL |
| Sincronización | ❌ Subida manual | ✅ Automática desde Drive |
| Deduplicación | ❌ Re-indexa todo | ✅ Solo archivos nuevos/modificados |
| Escalabilidad | ❌ Limitada por RAM | ✅ Cientos de documentos |
| Múltiples usuarios | ❌ Complejo | ✅ Base de datos compartida |
| Producción | ❌ Solo demo | ✅ Listo para equipos |

---

## 🔐 Seguridad

- Las credenciales de Google Drive y OpenAI se gestionan de forma segura a través del sistema de credenciales de n8n
- Los documentos nunca se exponen directamente; solo se consultan los fragmentos relevantes
- La conexión a PostgreSQL usa las credenciales almacenadas en n8n, no variables de entorno expuestas

---

## 📄 Licencia

MIT License — libre para uso comercial y personal.

---

*Construido con n8n · OpenAI GPT-4.1-mini · text-embedding-3-large · PostgreSQL PGVector · Google Drive API*
