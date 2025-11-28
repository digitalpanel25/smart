# Vue d'Ensemble du Système

## 🎯 Principe Fondamental

**Transformer une question en langage naturel en analyse de données avec détection d'anomalies et alertes automatiques.**

## 🔄 Flux Simplifié

```
QUESTION UTILISATEUR
    ↓
"Analyse le retard du projet X"
    ↓
┌─────────────────────┐
│   LLM PLANNER       │ ← Génère un plan structuré
│   (GPT-4)           │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│   ORCHESTRATION     │ ← Exécute les étapes
│   (Temporal)         │
└──────────┬──────────┘
           ↓
    ┌──────┴──────┐
    ↓            ↓
┌─────────┐  ┌──────────┐
│   RAG   │  │  CODE    │ ← Recherche contexte
│ Service │  │ Generator│   Génère SQL/Pandas
└────┬────┘  └────┬─────┘
     └──────┬─────┘
            ↓
    ┌───────────────┐
    │   VALIDATOR   │ ← Vérifie sécurité
    │   & SANDBOX   │   Exécute isolé
    └───────┬───────┘
            ↓
    ┌───────────────┐
    │  EXECUTION    │ ← Exécute sur ClickHouse
    │    ENGINE     │   Retourne KPI
    └───────┬───────┘
            ↓
    ┌───────────────┐
    │    ANOMALY    │ ← Détecte anomalies
    │   DETECTION   │   Score + explication
    └───────┬───────┘
            ↓
    ┌───────────────┐
    │    ALERTS     │ ← Envoie alertes
    │    SERVICE    │   Slack/Email/SMS
    └───────┬───────┘
            ↓
      RÉSULTAT FINAL
```

## 🧩 Composants Principaux

### 1. Frontend (React - Port 3000)
- **Rôle** : Interface utilisateur style Grok.ai
- **Fonctions** :
  - Chat interface
  - Authentification
  - Affichage des résultats
  - Visualisation des plans

### 2. LLM Planner (Port 8001)
- **Rôle** : Génère des plans d'exécution
- **Input** : Requête utilisateur en langage naturel
- **Output** : Plan JSON structuré avec étapes
- **Technologie** : OpenAI GPT-4

### 3. RAG Service (Port 8002)
- **Rôle** : Recherche sémantique
- **Stockage** : Milvus (vector DB)
- **Usage** : Trouve la documentation/schémas pertinents

### 4. Code Generator (Port 8003)
- **Rôle** : Génère du code SQL/Pandas
- **Input** : Intent + contexte RAG
- **Output** : Code optimisé et commenté

### 5. Validator & Sandbox (Port 8004)
- **Rôle** : Sécurité et exécution isolée
- **Vérifications** :
  - Patterns interdits (DROP, DELETE, etc.)
  - Analyse statique (Bandit)
  - Syntaxe SQL
- **Exécution** : Container Kubernetes isolé

### 6. Execution Engine (Port 8007)
- **Rôle** : Exécute les requêtes
- **Backends** :
  - ClickHouse (SQL)
  - Pandas (Python)
- **Output** : KPI JSON

### 7. Anomaly Detection (Port 8005)
- **Rôle** : Détecte les anomalies
- **Méthodes** :
  - Z-score (statistique)
  - Prophet (saisonalité)
  - Isolation Forest (multivarié)
- **Output** : Score [0..1] + explication

### 8. Alerts Service (Port 8006)
- **Rôle** : Gère les alertes
- **Fonctions** :
  - Règles d'alerte
  - Déduplication
  - Dispatchers (Slack, Email, SMS, Voice)

### 9. Repair Service (Port 8008)
- **Rôle** : Auto-correction
- **Fonction** : Répare automatiquement les erreurs avec LLM

### 10. Auth Service (Port 8009)
- **Rôle** : Authentification
- **Fonctions** :
  - Inscription/Connexion
  - JWT tokens
  - Gestion clés API

## 🔐 Sécurité Multi-Niveaux

### Niveau 1 : Authentification
- JWT tokens avec expiration
- Hashage bcrypt des mots de passe
- Clés API hashées (SHA-256)

### Niveau 2 : Validation
- Patterns interdits détectés
- Analyse statique du code
- Validation Pydantic des inputs

### Niveau 3 : Sandbox
- Container isolé par exécution
- Limites CPU/Memory
- Pas d'accès réseau/fichier

### Niveau 4 : Base de Données
- Paramètres préparés (anti-injection)
- Pool de connexions sécurisé
- Audit logs

## 📊 Exemple Complet

### Input Utilisateur
```
"Analyse le retard du projet X entre 2025-01-01 et 2025-10-31, 
par phase, et détecte anomalies"
```

### Traitement

1. **LLM Planner** génère :
```json
{
  "intent": "analyze_project_delay",
  "steps": [
    {"type": "rag_lookup", "params": {"query": "schema project"}},
    {"type": "generate_sql", "params": {"kpi": "delay_by_phase"}},
    {"type": "validate_and_run"},
    {"type": "anomaly_detection", "params": {"kpi": "delay_by_phase"}},
    {"type": "alert_if", "params": {"threshold": 0.2}}
  ]
}
```

2. **RAG Service** trouve : Schéma de la table `projects`

3. **Code Generator** génère :
```sql
SELECT phase, AVG(delay_days) as avg_delay
FROM projects
WHERE start_date BETWEEN '2025-01-01' AND '2025-10-31'
GROUP BY phase;
```

4. **Validator** vérifie : ✅ Pas de DROP, syntaxe OK

5. **Execution Engine** exécute et retourne :
```json
{
  "data": [
    {"phase": "design", "value": 42},
    {"phase": "testing", "value": 85}  // ← Anomalie !
  ]
}
```

6. **Anomaly Detection** détecte :
```json
{
  "anomalies": [{
    "timestamp": "2025-10-15",
    "value": 85,
    "score": 0.87,
    "explanation": "Pic anormal détecté dans la phase testing"
  }]
}
```

7. **Alerts Service** envoie :
```
🚨 Alerte Slack : Anomalie détectée dans delay_by_phase
Score: 0.87 (seuil: 0.2)
Phase: testing
Action recommandée: Vérifier les ressources de la phase testing
```

### Output Final
- ✅ KPI calculé : Retard moyen par phase
- ⚠️ Anomalie détectée : Pic à 85% dans "testing"
- 📢 Alerte envoyée : Slack notification

## 🚀 Avantages

1. **Langage Naturel** : Pas besoin de connaître SQL
2. **Intelligent** : L'IA comprend le contexte
3. **Sécurisé** : Multiples niveaux de protection
4. **Automatique** : Détection et alertes automatiques
5. **Résilient** : Auto-correction des erreurs
6. **Scalable** : Architecture microservices
7. **Observable** : Monitoring complet

## 📚 Documentation Complète

- **docs/HOW_IT_WORKS.md** : Architecture détaillée
- **docs/QUICK_EXPLANATION.md** : Explication rapide
- **docs/SECURITY.md** : Détails sécurité
- **docs/architecture.md** : Diagrammes complets

