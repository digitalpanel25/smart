# 🚀 Guide de Démarrage Rapide - SmartVoice

## Installation Rapide

### 1. Prérequis

```bash
# Python 3.10+
python --version

# Redis (pour Celery)
# Windows: Télécharger depuis https://redis.io/download
# Linux: sudo apt-get install redis-server
# macOS: brew install redis
```

### 2. Installation des Dépendances

```bash
cd backend
pip install -r requirements.txt
```

### 3. Configuration

Créez un fichier `.env` dans le dossier `backend/`:

```env
# OpenAI (requis pour STT et LLM)
OPENAI_API_KEY=sk-...

# Base de données (exemple PostgreSQL)
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# Redis
REDIS_URL=redis://localhost:6379/0
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
```

### 4. Préparer la Base de Données

Créez une base de données avec quelques tables d'exemple:

```sql
-- Exemple PostgreSQL
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    product_id INTEGER
);

INSERT INTO sales (date, amount, product_id) VALUES
    ('2024-01-01', 1000.00, 1),
    ('2024-01-02', 1500.00, 2),
    ('2024-01-03', 1200.00, 1);
```

### 5. Démarrer les Services

**Terminal 1 - Redis:**
```bash
redis-server
```

**Terminal 2 - Celery Worker:**
```bash
cd backend
celery -A tasks worker --loglevel=info
```

**Terminal 3 - FastAPI:**
```bash
cd backend
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### 6. Tester

**Option 1: Client Python**
```bash
cd device
python device_client.py path/to/audio.wav
```

**Option 2: API Directe**
```bash
curl -X POST "http://localhost:8000/api/audio/process" \
  -F "file=@audio.wav"

# Récupérer le résultat
curl "http://localhost:8000/api/task/{task_id}"
```

**Option 3: Interface Swagger**
Ouvrir: http://localhost:8000/docs

## Vérification

### Health Check
```bash
curl http://localhost:8000/api/health
```

### Info Base de Données
```bash
curl http://localhost:8000/api/database/info
```

## Exemple Complet

1. **Enregistrer un fichier audio** (WAV, 16kHz, mono) avec la question:
   > "Quelles sont les ventes du dernier mois ?"

2. **Envoyer au serveur:**
   ```bash
   python device_client.py audio.wav
   ```

3. **Le système va:**
   - Transcrire l'audio
   - Détecter l'intention
   - Générer SQL
   - Exécuter la requête
   - Faire du ML (si applicable)
   - Générer une réponse textuelle

4. **Le device reçoit le texte et fait TTS local**

## Dépannage

### Erreur: "Base de données non connectée"
- Vérifier `DATABASE_URL` dans `.env`
- Vérifier que la DB est accessible
- Vérifier les credentials

### Erreur: "Service STT non disponible"
- Vérifier `OPENAI_API_KEY` dans `.env`
- Vérifier que la clé est valide

### Erreur: "Celery worker ne démarre pas"
- Vérifier que Redis est démarré
- Vérifier `CELERY_BROKER_URL` dans `.env`

### Erreur: "Aucune table détectée"
- Vérifier que la DB contient des tables
- Vérifier les permissions de lecture

## Prochaines Étapes

1. Lire `ARCHITECTURE.md` pour comprendre le système
2. Lire `README.md` pour plus de détails
3. Personnaliser les prompts LLM dans `llm_client.py`
4. Ajouter des validations dans `validation.py`
5. Étendre le ML dans `ml_service.py`

