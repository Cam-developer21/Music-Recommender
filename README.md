# 🎵 Music Recommender System

Sistema de recomendación musical con RAG (Retrieval Augmented Generation) usando GCP.

## 📋 Requisitos

- Python 3.11+
- Cuenta en GCP
- API keys: Spotify, Genius, Last.fm

## 🚀 Setup

### 1. Clonar proyecto
```bash
git clone <tu-repo>
cd music-recommender
```

### 2. Crear ambiente virtual
```bash
python -m venv venv
source venv/bin/activate  # Mac/Linux
venv\Scripts\activate     # Windows
```

### 3. Instalar dependencias
```bash
cd ingestion
pip install -r requirements.txt
```

### 4. Configurar .env
Copia `.env.example` a `.env` y llena las credenciales.

### 5. Autenticar en GCP
```bash
gcloud auth login
gcloud config set project music-recommender-dev
gcloud auth application-default login
```

## 📊 Estructura del Proyecto
```
music-recommender/
├── ingestion/
│   ├── clients/
│   ├── processors/
│   ├── storage/
│   └── utils/
├── api/
├── dbt/
├── tests/
└── logs/
```

## 📊 Estructura del Proyecto
```
📁 MUSIC-RECOMMENDER
  📁 ingestion
    📁 clients
    📁 processors
    📁 storage
    📁 utils
  📁 api
  📁 dbt
  📁 tests
  📁 logs
  📄 .env
  📄 .gitignore
  📄 README.md


## 📊 Visión General del Pipeline

APIs Externas → Ingesta/ETL → Data Warehouse → Embeddings → Vector DB → RAG → API → Frontend → Feedback Loop
```

---

## 🔄 Pipeline Detallado Completo
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ FASE 1: INGESTA DE DATOS (Data Collection)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [Spotify API] ──┐                                                          │
│  [Genius API]  ──┼──→ [Cloud Function: ingestion] ←── [Cloud Scheduler]     │
│  [Last.fm API] ──┘         │                              (Cada 6 horas)    │
│                            │                                                │
│                            ▼                                                │
│                   [Cloud Storage: RAW]                                      │
│                   gs://bucket/raw/                                          │
│                   ├── spotify_YYYYMMDD.json                                 │
│                   ├── genius_YYYYMMDD.json                                  │
│                   └── lastfm_YYYYMMDD.json                                  │
│                            │                                                │
│                            ▼                                                │
│                   [BigQuery: Dataset RAW]                                   │
│                   ├── spotify_tracks                                        │
│                   ├── genius_lyrics                                         │
│                   └── lastfm_tags                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ FASE 2: TRANSFORMACIÓN (Data Cleaning & Feature Engineering)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [BigQuery: RAW] ──→ [dbt Core/Cloud] ──→ [BigQuery: CLEAN]                 │
│                           │                     │                           │
│                           │                     ├── tracks (unified)        │
│                           │                     ├── track_features          │
│                           │                     ├── lyrics_processed        │
│                           │                     └── tags_normalized         │
│                           │                                                 │
│                           └──→ [Transformaciones dbt]                       │
│                                - Deduplicación                              │
│                                - Normalización                              │
│                                - Feature engineering                        │
│                                - Agregaciones                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ FASE 3: EMBEDDINGS & VECTORIZACIÓN (Semantic Indexing)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [BigQuery: CLEAN] ──→ [Cloud Run: embedding-service]                       │
│        │                       │                                            │
│        │                       ├──→ [Vertex AI: text-embedding-004]         │
│        │                       │    Genera embeddings de:                   │
│        │                       │    - Letras de canciones                   │
│        │                       │    - Descripciones                         │
│        │                       │    - Tags combinados                       │
│        │                       │                                            │
│        │                       ▼                                            │
│        │              [Embeddings: Array[768]]                              │
│        │                       │                                            │
│        │                       ▼                                            │
│        └──────────────→ [Vertex AI Vector Search]                           │
│                         Index: music-embeddings                             │
│                         └── 100K+ vectores indexados                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ FASE 4: SISTEMA DE RECOMENDACIÓN (Recommendation Engine)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [Usuario Query] ──→ [Cloud Run: recommendation-service]                    │
│  "música triste          │                                                  │
│   pero energética"       │                                                  │
│                          ▼                                                  │
│                   ┌──────────────┐                                          │
│                   │ Query Embedding │                                       │
│                   └──────────────┘                                          │
│                          │                                                  │
│         ┌────────────────┼────────────────┐                                 │
│         │                │                │                                 │
│         ▼                ▼                ▼                                 │
│  [Vector Search]  [Collaborative]  [Content-Based]                          │
│  Top 100          Filtering        Filtering                                │
│  candidatos       (User-Item)      (Audio features)                         │
│         │                │                │                                 │
│         └────────────────┼────────────────┘                                 │
│                          ▼                                                  │
│                   [Hybrid Ranker]                                           │
│                   Combina señales:                                          │
│                   - Similitud semántica                                     │
│                   - Popularidad                                             │
│                   - Historial usuario                                       │
│                   - Contexto temporal                                       │
│                          │                                                  │
│                          ▼                                                  │
│                   [Top 20 Candidatos]                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ FASE 5: RAG (Retrieval Augmented Generation)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [Top 20 Candidatos] ──→ [RAG Pipeline]                                     │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │  RETRIEVAL   │                                     │
│                        └──────────────┘                                     │
│                               │                                             │
│              ┌────────────────┼────────────────┐                            │
│              │                │                │                            │
│              ▼                ▼                ▼                            │
│      [Vector Search]   [BigQuery]      [User Context]                       │
│      Canciones         Reviews         Historial                            │
│      similares         Comentarios     Preferencias                         │
│              │                │                │                            │
│              └────────────────┼────────────────┘                            │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │ AUGMENTATION │                                     │
│                        └──────────────┘                                     │
│                               │                                             │
│                     Contexto enriquecido:                                   │
│                     - Letras relevantes                                     │
│                     - Reviews positivos                                     │
│                     - Tags de usuarios                                      │
│                     - Canciones relacionadas                                │
│                               │                                             │
│                               ▼                                             │
│                        ┌──────────────┐                                     │
│                        │  GENERATION  │                                     │
│                        └──────────────┘                                     │
│                               │                                             │
│                               ▼                                             │
│                     [Vertex AI: Gemini]                                     │
│                     Prompt:                                                 │
│                     "Usuario busca: {query}                                 │
│                      Canciones encontradas: {tracks}                        │
│                      Evidencia: {context}                                   │
│                      Genera explicación personalizada"                      │
│                               │                                             │
│                               ▼                                             │
│                   [Recomendaciones + Explicaciones]                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                        
                         
                          

## 🎯 Uso

### Ejecutar ingesta local
```bash
cd ingestion
python main.py
```

## 📝 Licencia

MIT






