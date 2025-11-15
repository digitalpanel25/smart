# 🎤 SmartVoice - Système Intelligent de Requêtes Vocales

## 📋 Vue d'ensemble

SmartVoice est un système intelligent qui transforme des requêtes vocales en analyses de données via IA, Machine Learning et traitement automatique. Le système est **100% dynamique** et s'adapte automatiquement à n'importe quelle base de données sans configuration préalable.

### ✨ Fonctionnalités Principales

- 🎙️ **Reception audio** depuis devices (casque, ESP32, etc.)
- 🔄 **STT in-memory** : transformation audio → texte sans stockage
- 🧠 **NLP intelligent** : extraction d'intentions et génération SQL automatique
- 🗄️ **Universal Data Mapper** : détection automatique de structure DB
- ✅ **Validation Engine** : validation SQL, séries temporelles, cohérence métier
- 🤖 **ML automatique** : EDA, preprocessing, prédictions (Prophet/XGBoost)
- 📊 **Analyse automatique** : génération de réponses textuelles intelligentes
- 🔊 **TTS local** : synthèse vocale uniquement sur le device
- 🔒 **Sécurité** : whitelist SQL, sandbox LLM, audit complet

## 🏗️ Architecture

```
┌─────────────┐
│   Device    │ (ESP32 / Casque)
│  (Audio)    │
└──────┬──────┘
       │ HTTP POST /audio
       ▼
┌─────────────────────────────────────┐
│      FastAPI Backend (main.py)      │
│  ┌───────────────────────────────┐  │
│  │  STT Wrapper (in-memory)      │  │
│  └───────────┬───────────────────┘  │
│              │                       │
│  ┌───────────▼───────────────────┐  │
│  │  Celery Task Queue            │  │
│  └───────────┬───────────────────┘  │
└──────────────┼──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
┌─────────────┐  ┌──────────────┐
│   Redis     │  │ Celery Worker│
│   (Queue)   │  │   (tasks.py) │
└─────────────┘  └──────┬───────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ LLM Client  │ │ Data Mapper │ │ Validation  │
│             │ │  (Universal)│ │   Engine    │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │               │               │
       └───────┬───────┴───────┬───────┘
               │               │
               ▼               ▼
        ┌─────────────┐ ┌─────────────┐
        │   Database  │ │ ML Service  │
        │  (Any SQL/  │ │ (Prophet/   │
        │   NoSQL)    │ │  XGBoost)   │
        └─────────────┘ └─────────────┘
```

## 📦 Structure du Projet

```
smartvoice/
├── backend/
│   ├── main.py              # FastAPI backend (routes API)
│   ├── schemas.py           # Modèles Pydantic
│   ├── db.py                # Connexion DB universelle
│   ├── data_mapper.py       # Universal Data Mapper
│   ├── validation.py        # Validation Engine
│   ├── stt_wrapper.py       # STT in-memory
│   ├── llm_client.py        # Client LLM (OpenAI/Anthropic)
│   ├── tasks.py             # Tâches Celery
│   ├── ml_service.py        # Service ML (Prophet/XGBoost)
│   └── requirements.txt     # Dépendances Python
│
├── device/
│   ├── device_client.py     # Client Python de test
│   └── esp32_example.txt    # Pseudo-code ESP32
│
└── README.md                # Ce fichier
```

## 🚀 Installation

### Prérequis

- Python 3.10+
- Redis (pour Celery)
- PostgreSQL/MySQL/MongoDB (exemple de DB)

### Installation des dépendances

```bash
cd backend
pip install -r requirements.txt
```

### Configuration

Créez un fichier `.env` dans le dossier `backend/` :

```env
# OpenAI / LLM
OPENAI_API_KEY=your_openai_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key

# Database (exemple)
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
# ou
DATABASE_URL=mongodb://localhost:27017/dbname

# Redis
REDIS_URL=redis://localhost:6379/0

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0

# Security
ALLOWED_SQL_KEYWORDS=SELECT,FROM,WHERE,GROUP BY,ORDER BY,HAVING,JOIN,INNER,LEFT,RIGHT
BLOCKED_SQL_KEYWORDS=DROP,DELETE,TRUNCATE,ALTER,CREATE,INSERT,UPDATE
```

## 🏃 Démarrage

### 1. Démarrer Redis

```bash
redis-server
```

### 2. Démarrer le worker Celery

```bash
cd backend
celery -A tasks worker --loglevel=info
```

### 3. Démarrer le serveur FastAPI

```bash
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### 4. Tester avec le client device

```bash
cd device
python device_client.py
```

## 📡 API Endpoints

### POST `/api/audio/process`

Traite un fichier audio et retourne une réponse textuelle.

**Request:**
- `Content-Type: multipart/form-data`
- `file`: fichier audio (WAV, MP3, etc.)

**Response:**
```json
{
  "task_id": "abc123",
  "status": "processing"
}
```

### GET `/api/task/{task_id}`

Récupère le statut et le résultat d'une tâche.

**Response:**
```json
{
  "task_id": "abc123",
  "status": "completed",
  "result": {
    "text_response": "Vos ventes ont augmenté de 15% ce mois-ci...",
    "confidence": 0.95,
    "ml_used": true
  }
}
```

### GET `/api/health`

Vérifie l'état du système.

## 🔧 Modules Principaux

### 1. Universal Data Mapper (`data_mapper.py`)

Détecte automatiquement la structure de n'importe quelle base de données :
- Analyse des tables/collections
- Détection des types de données
- Mapping vers un schéma universel
- Support SQL (PostgreSQL, MySQL, SQLite) et NoSQL (MongoDB)

### 2. Validation Engine (`validation.py`)

Valide la qualité et la sécurité :
- Validation SQL (whitelist/blacklist)
- Validation séries temporelles
- Validation cohérence métier
- Décision : bloquer / fallback / demander confirmation

### 3. ML Service (`ml_service.py`)

Machine Learning automatique :
- EDA (Exploratory Data Analysis)
- Preprocessing automatique
- Prédictions Prophet (séries temporelles)
- Prédictions XGBoost (classification/régression)
- Résumé automatique avec LLM

### 4. STT Wrapper (`stt_wrapper.py`)

Traitement audio in-memory :
- Support OpenAI Whisper
- Traitement via BytesIO (pas de stockage)
- Support WAV, MP3, FLAC

### 5. LLM Client (`llm_client.py`)

Génération SQL et analyse :
- Extraction d'intentions
- Génération SQL dynamique
- Résumé de résultats ML
- Support OpenAI GPT-4 et Anthropic Claude

## 🔒 Sécurité

- **Whitelist SQL** : seules les requêtes SELECT autorisées
- **Sandbox LLM** : validation des prompts avant envoi
- **Audit logging** : toutes les requêtes sont loggées
- **Validation stricte** : validation multi-niveaux avant exécution

## 🧪 Exemples d'utilisation

### Question utilisateur (audio)
> "Quelles sont les ventes du dernier mois ?"

### Traitement automatique
1. STT : "Quelles sont les ventes du dernier mois ?"
2. NLP : Extraction intention → "ventes", "dernier mois"
3. Data Mapper : Détection table "sales", colonne "date", "amount"
4. SQL généré : `SELECT SUM(amount) FROM sales WHERE date >= ...`
5. Validation : ✅ Requête sécurisée
6. Exécution : Résultat = 15000€
7. ML (si applicable) : Prédiction tendance
8. Réponse texte : "Vos ventes du dernier mois s'élèvent à 15 000€..."

### Device TTS
Le device reçoit le texte et fait la synthèse vocale localement.

## 🐳 Docker (Optionnel)

```dockerfile
# Dockerfile exemple
FROM python:3.10-slim
WORKDIR /app
COPY backend/requirements.txt .
RUN pip install -r requirements.txt
COPY backend/ .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 📝 Notes Importantes

- ⚠️ **Aucun audio n'est stocké** : tout est traité en mémoire
- 🔊 **TTS uniquement sur device** : le backend renvoie uniquement du texte
- 🔄 **100% dynamique** : s'adapte à n'importe quelle DB sans configuration
- 🧠 **IA intégrée** : NLP, SQL generation, ML automatique

## 🤝 Contribution

Ce projet est un système complet et extensible. Vous pouvez :
- Ajouter de nouveaux adaptateurs DB
- Étendre le Validation Engine
- Ajouter de nouveaux modèles ML
- Améliorer la génération SQL

## 📄 Licence

MIT License

