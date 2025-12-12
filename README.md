# Projet d'automatisation du traitement des tickets GLPI avec n8n

---

## 1. Présentation générale

Ce projet consiste à automatiser le traitement des tickets GLPI à l'aide de **n8n** et de l'**API REST GLPI**.  

L'objectif est d'analyser automatiquement les tickets entrants, de détecter les informations manquantes, de contacter l'utilisateur par email si nécessaire, puis de mettre à jour le ticket après réception de la réponse.

L'ensemble du projet est conçu comme un **POC / projet scolaire**, avec une architecture simple, reproductible et documentée.

---

## 2. Technologies utilisées

- **GLPI** – Outil de gestion des tickets

- **n8n** – Orchestrateur de workflows

- **API REST GLPI**

- **GreenMail** – Serveur mail de test (SMTP / IMAP)

- **Docker & Docker Compose**

- **JavaScript**

---

## 3. Architecture globale

### Flux fonctionnel

1. Un ticket est créé dans GLPI

2. n8n analyse le ticket via l'API GLPI

3. Les informations manquantes sont détectées

4. Un email est envoyé à l'utilisateur si nécessaire

5. L'utilisateur répond par email

6. n8n traite la réponse

7. Le ticket est mis à jour (suivi + statut)

---

## 4. Infrastructure

### 4.1 Docker Compose

```yaml
version: "3.8"

services:
  glpi:
    image: diouxx/glpi
    ports:
      - "8080:80"
    volumes:
      - glpi_data:/var/www/html/glpi

  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_BASIC_AUTH_ACTIVE=false

  greenmail:
    image: greenmail/standalone
    ports:
      - "3025:3025"
      - "3110:3110"
      - "3143:3143"
      - "8081:8080"

volumes:
  glpi_data:
  n8n_data:
```

### 4.2 Lancement de l'infrastructure

```bash
docker-compose up -d
```

### Accès aux services

- **GLPI** : http://localhost:8080
- **n8n** : http://localhost:5678
- **GreenMail** : http://localhost:8083

---

## 5. Configuration GLPI

### Donnée de connexion de la BDD :**
  SQL SERVER : mariadb**
  SQL USER : glpi**
  SQL PASSWORD : glpipassword**
  
### 5.1 Activation de l'API REST

1. Aller dans **Configuration > Général > API**
2. Activer l'**API REST**
3. Activer les **App-Tokens**
4. Autoriser les méthodes **PUT / POST**

### 5.2 Création des tokens

**App-Token**
- Créé depuis l'interface GLPI (Configuration > API)

**Session-Token**
- Obtenu via l'API :

```
POST /apirest.php/initSession
```

**Headers:**
```
App-Token: APP_TOKEN
```

---

## 6. Structure des workflows n8n

### Workflow 1 – Analyse du ticket

**Rôle :**
- Récupérer le ticket GLPI
- Analyser le contenu
- Déterminer si des informations sont manquantes
- Déclencher la suite du traitement

**Nœuds principaux :**
- Manual Trigger
- HTTP Request (GET Ticket)
- Code (normalisation des données)
- Message to Model (analyse IA)
- Code (extraction des champs)
- If (email requis ?)
- Webhook (appel du workflow 2)

### Workflow 2 – Demande d'informations complémentaires

**Rôle :**
- Générer un email automatique
- Envoyer la demande d'informations à l'utilisateur

**Nœuds principaux :**
- Webhook (entrée depuis le workflow 1)
- Code (préparation de l'email)
- SMTP Send Email (GreenMail)

### Workflow 3 – Traitement de la réponse utilisateur

**Rôle :**
- Détecter une réponse par email
- Extraire le contenu
- Mettre à jour le ticket GLPI
- Changer le statut du ticket

**Nœuds principaux :**
- Email Trigger (IMAP)
- Code (extraction du contenu)
- If (ticket concerné détecté)
- HTTP Request (PUT Ticket)
- HTTP Request (ajout de suivi si nécessaire)

---

## 7. Envoi et réception d'emails (GreenMail)

### Envoi d'un email de test (PowerShell)

```powershell
Send-MailMessage `
  -To "support@test.local" `
  -From "user@test.local" `
  -Subject "PC ne démarre plus" `
  -Body "Je suis bloqué en magasin" `
  -SmtpServer "localhost" `
  -Port 3025
```

---

## 8. Mise à jour d'un ticket via l'API GLPI

### 8.1 Changement de statut

```
PUT /apirest.php/Ticket/{id}
```

**Headers:**
```
App-Token: APP_TOKEN
Session-Token: SESSION_TOKEN
Content-Type: application/json
```

**Body:**
```json
{
  "input": {
    "status": 2
  }
}
```

### Codes de statut GLPI courants

| Code | Statut |
|------|--------|
| 1 | Nouveau |
| 2 | En cours |
| 3 | En attente |
| 4 | Résolu |
| 6 | Fermé |

---

## 9. Ajout d'un suivi (TicketFollowup)

```
POST /apirest.php/TicketFollowup
```

**Headers:**
```
App-Token: APP_TOKEN
Session-Token: SESSION_TOKEN
Content-Type: application/json
```

**Body:**
```json
{
  "input": {
    "itemtype": "Ticket",
    "items_id": 2,
    "content": "Réponse utilisateur reçue par email",
    "is_private": 0
  }
}
```

---

## 10. Tests réalisés

- ✅ Création de tickets GLPI
- ✅ Analyse automatique via n8n
- ✅ Envoi d'emails conditionnels
- ✅ Réception des réponses utilisateur
- ✅ Mise à jour automatique du ticket
- ✅ Changement de statut validé dans GLPI

---

## 11. Livrables fournis

- `docker-compose.yml`
- Workflows n8n exportés :
  - `workflow_1_analyse_ticket.json`
  - `workflow_2_envoi_email.json`
  - `workflow_3_traitement_reponse.json`
- Ce document `README.md`

---

## 12. Conclusion

Ce projet démontre la faisabilité d'une automatisation complète du traitement des tickets GLPI à l'aide de n8n.

La solution permet de réduire les interventions manuelles, d'améliorer la qualité des tickets et d'accélérer les échanges entre les utilisateurs et le support technique.

