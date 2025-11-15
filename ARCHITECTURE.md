# 🏗️ Architecture SmartVoice - Documentation Complète

## 📋 Vue d'Ensemble

SmartVoice est un système intelligent qui transforme des requêtes vocales en analyses de données via un pipeline complet d'IA et de Machine Learning. Le système est **100% dynamique** et s'adapte automatiquement à n'importe quelle base de données sans configuration préalable.

## 🔄 Flux Complet

```
┌─────────────────────────────────────────────────────────────────┐
│                         DEVICE (ESP32/Casque)                    │
│  ┌──────────────┐                                               │
│  │ Enregistre   │ ──Audio──>                                    │
│  │ Audio        │                                               │
│  └──────────────┘                                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                │ HTTP POST /api/audio/process
                                │ (multipart/form-data)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FASTAPI BACKEND (main.py)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Reçoit audio (en mémoire, pas de stockage)           │   │
│  │ 2. Lance tâche Celery (process_audio_task)              │   │
│  │ 3. Retourne task_id                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                │ Task Queue (Redis)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CELERY WORKER (tasks.py)                      │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 1: STT (stt_wrapper.py)                           │   │
│  │  • Transcription audio → texte (Whisper)                │   │
│  │  • Traitement in-memory (BytesIO)                       │   │
│  │  • Aucun stockage permanent                             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 2: NLP (llm_client.py)                            │   │
│  │  • Extraction intention                                  │   │
│  │  • Extraction entités (métrique, période, catégorie)    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 3: DATA MAPPER (data_mapper.py)                   │   │
│  │  • Détection tables pertinentes                         │   │
│  │  • Mapping vers schéma universel                        │   │
│  │  • Détection types sémantiques (date, amount, etc.)     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 4: SQL GENERATION (llm_client.py)                 │   │
│  │  • Génération SQL depuis requête naturelle              │   │
│  │  • Utilise contexte des tables                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 5: VALIDATION (validation.py)                     │   │
│  │  • Validation sécurité SQL (whitelist/blacklist)        │   │
│  │  • Validation qualité données                           │   │
│  │  • Validation cohérence métier                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 6: EXECUTION SQL (db.py)                          │   │
│  │  • Exécution requête                                     │   │
│  │  • Récupération résultats                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 7: ML (ml_service.py) - Si applicable            │   │
│  │  • EDA automatique                                       │   │
│  │  • Prédictions Prophet (séries temporelles)             │   │
│  │  • Prédictions XGBoost (classification/régression)      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ÉTAPE 8: RÉPONSE TEXTUELLE (llm_client.py)             │   │
│  │  • Génération réponse naturelle en français             │   │
│  │  • Intégration résultats SQL + ML                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                │ GET /api/task/{task_id}
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         DEVICE                                  │
│  ┌──────────────┐                                               │
│  │ Reçoit texte │ ──TTS Local──> Audio                         │
│  │              │                                               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

## 🧩 Modules Détaillés

### 1. **db.py - Universal Database Connector**

**Rôle**: Connexion universelle à n'importe quelle base de données

**Fonctionnalités**:
- Détection automatique du type de DB (PostgreSQL, MySQL, SQLite, MongoDB)
- Connexion via SQLAlchemy (SQL) ou PyMongo (MongoDB)
- Introspection automatique (tables, schémas)
- Exécution de requêtes SQL

**Pourquoi indépendant du domaine**:
- Utilise des URLs de connexion standard
- Détecte automatiquement la structure
- Pas de configuration préalable nécessaire

### 2. **data_mapper.py - Universal Data Mapper**

**Rôle**: Mappe automatiquement les structures DB vers un schéma universel

**Fonctionnalités**:
- Détection automatique des tables pertinentes (basée sur NLP)
- Détection des types sémantiques (date, amount, quantity, category, etc.)
- Mapping vers schéma universel
- Suggestion de colonnes pour séries temporelles

**Algorithme**:
1. Analyse les noms de colonnes avec patterns regex
2. Détecte les types sémantiques (date, amount, etc.)
3. Mappe vers un schéma universel
4. Construit un contexte pour le LLM

**Exemple**:
```python
# Table "sales" avec colonnes: id, date, amount, product_id
# → Détecte: date (type: date), amount (type: amount)
# → Suggère pour série temporelle: (date, amount)
```

### 3. **validation.py - Validation Engine**

**Rôle**: Valide la sécurité, qualité et cohérence des données

**Validations**:
1. **SQL Security**:
   - Whitelist de mots-clés autorisés
   - Blacklist de mots-clés bloqués (DROP, DELETE, etc.)
   - Détection d'injections SQL
   - Vérification que c'est une requête SELECT

2. **Time Series Validation**:
   - Vérification régularité des dates
   - Détection valeurs aberrantes
   - Statistiques descriptives

3. **Business Consistency**:
   - Vérification résultats vides
   - Vérification taille des résultats
   - Validations spécifiques par intention

**Décisions**:
- `allow`: Autoriser l'exécution
- `block`: Bloquer (sécurité)
- `fallback`: Utiliser une alternative
- `confirm`: Demander confirmation

### 4. **stt_wrapper.py - STT In-Memory**

**Rôle**: Transcription audio → texte sans stockage

**Fonctionnalités**:
- Utilise OpenAI Whisper
- Traitement 100% en mémoire (BytesIO)
- Conversion automatique de formats (MP3 → WAV)
- Aucun fichier sur disque

**Pourquoi in-memory**:
- Respecte la contrainte "pas de stockage audio"
- Plus rapide (pas d'I/O disque)
- Plus sécurisé (pas de fichiers temporaires)

### 5. **llm_client.py - LLM Client**

**Rôle**: Génération SQL, extraction intentions, résumés

**Fonctionnalités**:
1. **Extraction Intention**:
   - Analyse la requête utilisateur
   - Extrait intention et entités
   - Retourne confidence

2. **Génération SQL**:
   - Utilise contexte des tables (data_mapper)
   - Génère SQL adapté au type de DB
   - Respecte les règles de sécurité

3. **Résumé ML**:
   - Génère résumé des résultats ML
   - En français, naturel

4. **Réponse Textuelle**:
   - Génère réponse complète
   - Intègre résultats SQL + ML

**Providers supportés**:
- OpenAI GPT-4
- Anthropic Claude

### 6. **ml_service.py - ML Service**

**Rôle**: Machine Learning automatique (EDA, prédictions)

**Fonctionnalités**:
1. **EDA (Exploratory Data Analysis)**:
   - Statistiques descriptives
   - Détection tendance
   - Détection saisonnalité
   - Détection valeurs aberrantes

2. **Prophet (Séries Temporelles)**:
   - Prédictions futures
   - Intervalles de confiance
   - Détection saisonnalité automatique

3. **XGBoost**:
   - Classification ou régression
   - Détection automatique du type de tâche
   - Preprocessing automatique

**Décision d'application ML**:
- Minimum 10 points pour séries temporelles
- Minimum 20 points pour autres cas
- Vérification régularité des données

### 7. **tasks.py - Celery Tasks**

**Rôle**: Orchestration du pipeline complet

**Pipeline**:
1. STT → Transcription
2. NLP → Intention
3. Data Mapper → Tables
4. SQL Generation → Requête
5. Validation → Sécurité
6. Execution → Résultats
7. ML → Prédictions (si applicable)
8. Réponse → Texte final

**Gestion d'erreurs**:
- Try/catch à chaque étape
- Logging détaillé
- Retour d'erreur structuré

### 8. **main.py - FastAPI Backend**

**Rôle**: API REST pour recevoir requêtes

**Endpoints**:
- `POST /api/audio/process`: Traite audio
- `GET /api/task/{task_id}`: Récupère résultat
- `GET /api/health`: Health check
- `GET /api/database/info`: Info DB
- `GET /api/database/tables/{table_name}/schema`: Schéma table

## 🔒 Sécurité

### 1. **Whitelist SQL**
- Seules les requêtes SELECT autorisées
- Mots-clés bloqués: DROP, DELETE, TRUNCATE, etc.

### 2. **Sandbox LLM**
- Validation des prompts avant envoi
- Limitation des tokens
- Temperature basse (0.3) pour cohérence

### 3. **Audit/Logging**
- Toutes les requêtes loggées
- Logs structurés avec Loguru
- Tracking des erreurs

### 4. **Validation Multi-Niveaux**
- Validation SQL
- Validation données
- Validation métier

## 🚀 Déploiement

### Prérequis
- Python 3.10+
- Redis
- Base de données (PostgreSQL/MySQL/MongoDB)

### Installation
```bash
cd backend
pip install -r requirements.txt
```

### Configuration
Créer `.env` avec:
- `OPENAI_API_KEY`
- `DATABASE_URL`
- `REDIS_URL`

### Démarrage
1. Redis: `redis-server`
2. Celery: `celery -A tasks worker --loglevel=info`
3. FastAPI: `uvicorn main:app --reload`

## 📊 Exemple d'Utilisation

### Question utilisateur (audio)
> "Quelles sont les ventes du dernier mois ?"

### Traitement
1. **STT**: "Quelles sont les ventes du dernier mois ?"
2. **NLP**: Intent="ventes_mois_dernier", Entities={metric: "ventes", time_period: "dernier mois"}
3. **Data Mapper**: Détecte table "sales", colonnes "date", "amount"
4. **SQL**: `SELECT SUM(amount) FROM sales WHERE date >= CURRENT_DATE - INTERVAL '1 month'`
5. **Validation**: ✅ Sécurisé
6. **Execution**: Résultat = 15000€
7. **ML**: Prédiction tendance (si applicable)
8. **Réponse**: "Vos ventes du dernier mois s'élèvent à 15 000€. La tendance est à la hausse..."

### Device TTS
Le device reçoit le texte et fait la synthèse vocale localement.

## 🎯 Points Clés

1. **100% Dynamique**: S'adapte à n'importe quelle DB
2. **In-Memory**: Aucun stockage audio
3. **TTS Local**: Synthèse uniquement sur device
4. **Sécurisé**: Validation multi-niveaux
5. **Intelligent**: ML automatique si applicable
6. **Scalable**: Celery pour traitement asynchrone

