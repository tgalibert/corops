# Architecture Système & Décisions Techniques — corOps

Ce document résume les choix d'architecture, l'organisation des dépôts de code et le modèle de licence retenus pour le projet **corOps**.

---

## 1. Vue d'ensemble & Blocs Applicatifs

Le système est structuré autour de 4 blocs applicatifs principaux, complétés par un service SaaS de licence indépendant :

```
                  ┌──────────────────────────────────────────────┐
                  │                 HÉBERGÉ CLOUD                │
                  │                                              │
                  │  ┌──────────────────┐  ┌──────────────────┐  │
                  │  │  Site Vitrine    │  │ Serveur Licence  │  │
                  │  │  & Docs (Next.js)│  │ (Django + Stripe)│  │
                  │  └──────────────────┘  └─────────┬────────┘  │
                  └──────────────────────────────────┼───────────┘
                                                     │ Clé signée (JWT / RSA)
                                                     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        ENVIRONNEMENT CLIENT (ON-PREMISE)               │
│                                                                        │
│   ┌───────────────────────────┐         ┌──────────────────────────┐   │
│   │     Dashboard Web         │ ◄─────► │      Backend API         │   │
│   │     (React / Vite)        │  HTTP/WS│      (Django / DRF)      │   │
│   └───────────────────────────┘         └────────────▲─────────────┘   │
│                                                      │                 │
│                                                      │ mTLS / Tokens   │
│                                                      ▼                 │
│                                         ┌──────────────────────────┐   │
│                                         │      Agent Système       │   │
│                                         │      (Binaire Go)        │   │
│                                         └──────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### Détail des composants :
1. **Agent Machine (Go)** :
   - *Rôle* : Binaire autonome, léger et performant déployé sur chaque serveur surveillé.
   - *Fonctions* : Collecte des métriques système (CPU, RAM, Disque), streaming de logs, exécution des actions locales (Docker, `systemctl`, `.htaccess`).
   - *Déploiement* : On-premise (installé en service systemd ou conteneur).
2. **Backend Serveur (Django REST Framework)** :
   - *Rôle* : Cœur de l'application, centralise les données des agents, applique les règles de gestion et expose l'API.
   - *Fonctions* : Gestion de flotte de serveurs, alertes, vérification locale des quotas de licence.
   - *Déploiement* : On-premise (conteneur Docker).
3. **Dashboard Utilisateur (React / Vite + Tailwind)** :
   - *Rôle* : Interface web pour administrer les serveurs, visualiser les métriques en temps réel et piloter les actions.
   - *Déploiement* : On-premise (servi via Nginx ou packagé avec Django).
4. **Site Vitrine & Documentation (Next.js)** :
   - *Rôle* : Présentation commerciale du projet, landing page, documentation technique publique.
   - *Déploiement* : Cloud / Hébergé par corOps (Vercel / Cloudflare).
5. **Serveur de Licensing (Django)** :
   - *Rôle* : Gestion des abonnements payants (Stripe) et génération des clés cryptographiques signées.
   - *Déploiement* : Cloud privé corOps (totalement isolé de l'installation client).

---

## 2. Organisation des Dépôts Git

Pour allier simplicité de développement et sécurité des données sensibles, nous retenons un découpage en **deux dépôts distincts** :

### Dépôt 1 : Le Monorepo Produit (`corops`)
* **Périmètre** : Contient l'intégralité du produit distribué aux clients (On-Premise).
* **Visibilité** : Démarrage en privé le temps du MVP, avec ouverture progressive vers un modèle Open-Source / Source-Available (style Sentry/GitLab).
* **Arborescence cible** :
  ```text
  corops/
  ├── apps/
  │   ├── agent/       # Agent Go
  │   ├── server/      # Backend Django DRF
  │   └── dashboard/   # Frontend React
  ├── deploy/          # Configurations Docker Compose pour le client
  ├── ARCHITECTURE.md
  └── PROJECT.md
  ```

### Dépôt 2 : Le Dépôt Cloud & Business (`corops-cloud`)
* **Périmètre** : Hébergement de la logique commerciale.
* **Visibilité** : 100% Privé (jamais distribué aux utilisateurs finaux).
* **Contenu** : Serveur de licence (gestion Stripe, base abonnés, clé privée de signature) et site vitrine Next.js.

---

## 3. Modèle de Licence : Offline-First & Gestion des Quotas

### Les Paliers :
- **Palier Gratuit (Community / Sans clé)** :
  - Limite stricte : **1 serveur max** et **3 projets max**.
  - Fonctionnalités de base : Métriques CPU/RAM/Disque, logs standards.
- **Palier Pro (Payant - 39 €/mois)** :
  - Déblocage multi-serveurs et multi-projets.
  - Déblocage des modules avancés : Snapshots BDD automatisés, planificateur Cron centralisé, streaming de logs avancés, terminal SSH web.

### Mécanisme de validation asymétrique (Sans dépendance Internet) :
1. **Signature (Côté Cloud corOps)** :
   - Le serveur de licence génère un jeton cryptographique contenant le profil client (`tier: "pro"`, `max_servers: 50`, `expires_at: "..."`, `features: [...]`).
   - Le jeton est signé avec la **clé privée** de corOps.
2. **Vérification (Côté Client On-Premise)** :
   - Le client colle sa clé dans les paramètres de son Dashboard.
   - Django valide la signature mathématique à l'aide de la **clé publique** embarquée.
   - Une fois validé, Django met à jour dynamiquement ses quotas et ses drapeaux de fonctionnalités (*feature flags*).

---

## 4. Stratégie de Distribution

- **Format prioritaire** : Distribution sous forme de conteneurs **Docker** via un `docker-compose.yml`.
- **Bénéfices** :
  - Déploiement chez le client en une commande.
  - Isolation totale de l'environnement (versions de Python, PostgreSQL, dépendances).
  - Gestion simplifiée des mises à jour.
