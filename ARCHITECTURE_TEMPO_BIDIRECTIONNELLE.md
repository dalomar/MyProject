# 📋 Document de Présentation - Architecture Messaging Bidirectionnelle

## Vue d'ensemble de l'architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    BOUCLE COMPLÈTE                          │
└─────────────────────────────────────────────────────────────┘

TempoLims (FTP)
    ↓ (FTP Upload)
┌──────────────────────────────────────────────────────────────┐
│         BCI Connect 2 (Publisher/Transformer)                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  COMMON Queue                                          │ │
│  │  • Reçoit données FTP depuis TempoLims                 │ │
│  │  • Transforme en format Tempo                          │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
    ↓ (HTTP POST - Transformed Data)
    │
    ↓
┌──────────────────────────────────────────────────────────────┐
│         Tempo (Subscriber/Processor)                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  API Endpoint: POST /api/v1/receive                    │ │
│  │  ├─ Reçoit messages                                    │ │
│  │  ├─ Traite les données                                │ │
│  │  └─ Enregistre en PostgreSQL                           │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Validation & Processing                              │ │
│  │  ├─ Valide les résultats                              │ │
│  │  └─ Prépare réponse                                   │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
    ↓ (HTTP POST - Results)
    │
    ↓
┌──────────────────────────────────────────────────────────────┐
│         BCI Connect 2 (Subscriber/Transformer)               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  COMMON Queue (Return)                                 │ │
│  │  • Reçoit résultats de Tempo                           │ │
│  │  • Transforme en format TempoLims                      │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
    ↓ (FTP/Autre protocole)
    │
    ↓
TempoLims (Récupère résultats)
```

## Flux détaillé en 9 étapes

| Étape | Composant | Action | Format | Direction |
|-------|-----------|--------|--------|-----------|
| **1** | TempoLims | Envoie données | FTP | → |
| **2** | BCI Connect | Récupère FTP | Brut | ← |
| **3** | COMMON Queue | Transforme → Tempo | JSON | → |
| **4** | Tempo API | POST `/receive` | JSON | ← |
| **5** | Tempo DB | Enregistre en PostgreSQL | SQL | (local) |
| **6** | Tempo Validation | Valide et prépare réponse | JSON | → |
| **7** | BCI Connect API | POST `/api/v1/results` | JSON | ← |
| **8** | COMMON Queue | Transforme → TempoLims | Format Lims | → |
| **9** | TempoLims | Récupère résultats | FTP/HTTP | ← |

## Architecture détaillée - Flux Aller-Retour

### Phase 1 : TempoLims → Tempo (via BCI Connect)

```
TempoLims
  │
  ├─→ [FTP] TempoLims_RawData.txt
  │
  ↓
BCI Connect 2
  │
  ├─→ FTP Listener détecte nouveau fichier
  ├─→ Lit et parse le fichier
  ├─→ Stocke en COMMON queue
  │
  ├─→ Transformer:
  │   INPUT:  TempoLims format (CSV/XML/Binary)
  │   OUTPUT: Tempo JSON format
  │
  ├─→ Publish HTTP POST
  │
  ↓
Tempo API (POST /api/v1/receive)
  │
  ├─→ Reçoit payload JSON
  ├─→ Valide schéma
  ├─→ Crée transaction PostgreSQL
  ├─→ Enregistre:
  │   ├─ messageId
  │   ├─ source (TempoLims)
  │   ├─ data payload
  │   └─ timestamp
  ├─→ Traite les données
  │   ├─ Calculs/analyses
  │   ├─ Validation métier
  │   └─ Enrichissement
  ├─→ Enregistre résultats en DB
  │
  └─→ Return 200 OK + resultsId
```

### Phase 2 : Tempo → TempoLims (via BCI Connect)

```
Tempo (Post-Processing)
  │
  ├─→ Validation des résultats
  │   ├─ Vérifie intégrité
  │   ├─ Valide règles métier
  │   └─ Approuve traitement
  │
  ├─→ Prépare payload résultat:
  │   {
  │     "messageId": "original_uuid",
  │     "resultsId": "tempo_12345",
  │     "status": "completed",
  │     "processedAt": "2026-03-25T12:00:00Z",
  │     "data": {...results...}
  │   }
  │
  ├─→ HTTP POST vers BCI Connect
  │   POST /api/v1/results
  │   Authorization: Bearer <BCI_TOKEN>
  │
  ↓
BCI Connect 2 (API Endpoint)
  │
  ├─→ Reçoit résultats de Tempo
  ├─→ Stocke en COMMON queue (Return)
  │
  ├─→ Transformer:
  │   INPUT:  Tempo JSON format
  │   OUTPUT: TempoLims format (FTP compatible)
  │
  ├─→ Publish vers TempoLims
  │   Protocole: FTP (ou HTTP POST)
  │   File: TempoLims_Results_${timestamp}.txt
  │
  ↓
TempoLims
  │
  └─→ Récupère résultats via FTP
      ├─ Parse fichier
      ├─ Met à jour statut job
      └─ Affiche résultats
```

## Configuration BCI Connect 2 - Bidirectionnelle

```yaml
BCI_CONNECT_CONFIG:
  
  # DIRECTION 1: TempoLims → Tempo
  inbound:
    source:
      type: FTP
      host: tempolims.server.com
      path: /incoming/
      polling: 60s
    
    queue: COMMON
    
    transformer:
      name: "TempoLims_to_Tempo"
      rules:
        - source_field: "jobId"
          target_field: "jobId"
        - source_field: "sampleData"
          target_field: "data.samples"
    
    subscriber:
      - name: "Tempo"
        type: "HTTP_POST"
        endpoint: "https://tempo-api.com/api/v1/receive"
        auth: "Bearer ${TEMPO_API_TOKEN}"
        timeout: 30s
        retry: 3
        backoff: exponential
  
  # DIRECTION 2: Tempo → TempoLims  
  outbound:
    endpoint: "POST /api/v1/results"
    auth_required: true
    
    queue: COMMON_RETURN
    
    transformer:
      name: "Tempo_to_TempoLims"
      rules:
        - source_field: "resultsId"
          target_field: "resultId"
        - source_field: "data"
          target_field: "resultData"
    
    publisher:
      - name: "TempoLims_FTP"
        type: "FTP"
        host: tempolims.server.com
        path: /results/
        filename_pattern: "Result_${messageId}_${timestamp}.txt"
        retry: 5
```

## APIs à implémenter

### API 1 : Tempo reçoit données (Inbound)

```
POST /api/v1/receive

Headers:
  Authorization: Bearer <TOKEN>
  Content-Type: application/json

Body:
{
  "messageId": "msg-uuid-123",
  "timestamp": "2026-03-25T10:30:00Z",
  "source": "TempoLims",
  "data": {
    "jobId": "JOB-001",
    "sampleId": "SAMPLE-123",
    "sampleData": {...}
  }
}

Response 200 OK:
{
  "success": true,
  "messageId": "msg-uuid-123",
  "processingStarted": "2026-03-25T10:30:05Z"
}
```

### API 2 : Tempo envoie résultats (Outbound)

```
POST /api/v1/results

Headers:
  Authorization: Bearer <BCI_TOKEN>
  Content-Type: application/json

Body:
{
  "messageId": "msg-uuid-123",
  "resultsId": "tempo-result-456",
  "status": "completed",
  "processedAt": "2026-03-25T11:00:00Z",
  "validatedBy": "system",
  "data": {
    "jobId": "JOB-001",
    "results": {...},
    "metrics": {...}
  }
}

Response 200 OK:
{
  "success": true,
  "resultsId": "tempo-result-456",
  "deliveredTo": "TempoLims"
}
```

## Architecture PostgreSQL - Tempo

```sql
-- Table pour stocker les messages reçus
CREATE TABLE tempo_messages (
  id UUID PRIMARY KEY,
  message_id VARCHAR(255) UNIQUE,
  source VARCHAR(50),
  payload JSONB,
  status VARCHAR(50),
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  INDEX ON source, status
);

-- Table pour stocker les résultats
CREATE TABLE tempo_results (
  id UUID PRIMARY KEY,
  message_id VARCHAR(255) REFERENCES tempo_messages(message_id),
  results_data JSONB,
  validation_status VARCHAR(50),
  validated_at TIMESTAMP,
  validated_by VARCHAR(255),
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  sent_back_at TIMESTAMP,
  INDEX ON message_id, validation_status
);

-- Table pour audit/logging
CREATE TABLE tempo_audit_log (
  id UUID PRIMARY KEY,
  message_id VARCHAR(255),
  event VARCHAR(100),
  details JSONB,
  created_at TIMESTAMP,
  INDEX ON message_id, created_at
);
```

## Flux de validation Tempo

```
Message Reçu
    ↓
[PROCESSING]
├─ Valide schéma JSON
├─ Valide données métier
├─ Enregistre en DB
├─ Lance traitement
    ↓
[COMPLETED]
├─ Calculs terminés
├─ Résultats générés
├─ Enregistre résultats en DB
    ↓
[VALIDATION]
├─ Inspection manuelle (optionnel)
├─ Validation automatisée
├─ Approbation
    ↓
[APPROVED] → POST vers BCI Connect
    ou
[REJECTED] → Notifier TempoLims (erreur)
```

## Points de faisabilité

### ✅ Forces

- **Architecture découplée et scalable** : Chaque composant indépendant
- **Transformation centralisée** : BCI Connect gère tous les mappages
- **Audit complet** : PostgreSQL + logs détaillés
- **Gestion d'erreurs bidirectionnelle** : Retry automatique
- **Scalabilité** : Facile d'ajouter nouvelles sources

### ⚠️ Points à valider

- [ ] Authentification mutuelle (Tempo ↔ BCI Connect) ?
- [ ] Timeout idéal pour traitement Tempo ?
- [ ] Gestion des doublons (idempotence) ?
- [ ] Monitoring des erreurs et alertes ?
- [ ] Plan de rollback si validation échoue ?
- [ ] Chiffrement des données sensibles ?
- [ ] SLA et time-to-response requis ?
- [ ] Capacité de charge estimée ?

## Plan d'implémentation

| Phase | Responsable | Tâches |
|-------|-------------|--------|
| **Design** | Équipe | ✓ Schéma JSON (inbound/outbound) |
| | | ✓ Authentification mutuelle |
| | | ✓ Schéma DB PostgreSQL |
| **Dev BCI** | BCI Team | ✓ Transformer bidirectionnel |
| | | ✓ Endpoint POST `/api/v1/results` |
| | | ✓ Queue Return management |
| **Dev Tempo** | Tempo Team | ✓ Endpoint POST `/api/v1/receive` |
| | | ✓ PostgreSQL integration |
| | | ✓ Validation & processing logic |
| | | ✓ HTTP client (POST vers BCI) |
| **Test** | QA | ✓ Tests E2E complets |
| | | ✓ Tests de charge |
| | | ✓ Failover scenarios |
| **Prod** | DevOps | ✓ Deploy BCI + Tempo |
| | | ✓ Monitoring/Alerting complet |

## Avantages de cette approche complète

✅ **Découplage total** : Chaque composant autonome  
✅ **Traçabilité** : Audit complet du cycle  
✅ **Robustesse** : Gestion d'erreurs bidirectionnelle  
✅ **Scalabilité** : Facile d'ajouter nouvelles sources  
✅ **Maintenabilité** : Responsabilités claires  
✅ **Validation** : Contrôle qualité intégré  

---

**Document généré le:** 2026-03-25 14:35:47  
**Version:** 1.0