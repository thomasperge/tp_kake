# Diagrammes UML - Plateforme e-Commerce Microservices

## 📋 Vue d'ensemble

Ce dossier contient les diagrammes UML pour le projet de plateforme e-commerce basée sur une architecture microservices avec .NET 9, conformément aux livrables attendus du TP.

## 📁 Structure des Diagrammes

### 1. [Diagramme de Cas d'Utilisation](./01-use-case-diagram.md)
**Objectif :** Représenter les interactions entre les acteurs (Client et Administrateur) et le système.

**Contenu :**
- Parcours client complet (navigation, panier, commande, suivi)
- Parcours administrateur (gestion produits, stocks, statistiques)
- Relations include/extend entre cas d'utilisation
- Tableau descriptif de chaque cas d'utilisation

**Acteurs :**
- 👤 Client (utilisateur final)
- 👨‍💼 Administrateur (gestionnaire de la plateforme)

---

### 2. [Diagramme de Composants](./02-component-diagram.md)
**Objectif :** Illustrer l'architecture microservices complète avec tous les services, bases de données et communications.

**Contenu :**
- **4 Microservices principaux :**
  - 🛍️ **Catalog.API** (PostgreSQL) - Gestion des produits
  - 🛒 **Basket.API** (PostgreSQL + Redis) - Gestion des paniers
  - 🎟️ **Discount.API** (SQLite) - Gestion des coupons
  - 📦 **Ordering.API** (SQL Server) - Gestion des commandes

- **Infrastructure :**
  - 🚪 API Gateway (Ocelot/YARP)
  - 🐰 RabbitMQ (Message Bus)
  - 🔐 Identity Service
  - 📊 Logging & Monitoring

- **Architectures par service :**
  - Catalog & Basket : Vertical Slice + CQRS
  - Discount : Multi-Layer + gRPC
  - Ordering : DDD + CQRS + Clean Architecture

**Bases de données polyglottes :**
- PostgreSQL (Catalog, Basket)
- Redis (Cache)
- SQLite (Discount)
- SQL Server (Ordering)

---

### 3. [Diagramme de Séquence](./03-sequence-diagram.md)
**Objectif :** Détailler le parcours complet d'une commande avec toutes les interactions entre services.

**Flux couverts :**

#### 🔹 Flux 1 : Ajout au Panier et Application de Réduction
- Consultation du catalogue
- Authentification via JWT
- Ajout d'items au panier (avec cache Redis)
- Application d'un coupon de réduction (via gRPC)
- Recalcul des totaux

#### 🔹 Flux 2 : Passage de Commande (Checkout)
- Validation du panier
- Publication de `BasketCheckoutEvent` vers RabbitMQ
- Traitement asynchrone par Ordering.API
- Création de la commande avec règles métier DDD
- Persistance transactionnelle
- Publication de `OrderCreatedEvent`

#### 🔹 Flux 3 : Notification et Suivi
- Réception événement par Notification Service
- Envoi d'email de confirmation
- Consultation du statut de commande (CQRS Read Model)
- Mise à jour de statut (Pending → Confirmed → Shipped → Delivered)
- Notifications automatiques sur changements

**Patterns illustrés :**
- Cache-Aside Pattern
- Event-Driven Architecture
- CQRS (Command Query Responsibility Segregation)
- Saga Pattern
- Observer Pattern

---

## 🛠️ Technologies Utilisées

### Format des Diagrammes
- **Mermaid.js** : Diagrammes as Code, faciles à versionner et à maintenir

### Comment Visualiser

#### Option 1 : GitHub/GitLab
Les fichiers `.md` s'affichent automatiquement avec les diagrammes Mermaid rendus.

#### Option 2 : Visual Studio Code
Installer l'extension **Markdown Preview Mermaid Support**
```bash
code --install-extension bierner.markdown-mermaid
```

#### Option 3 : Mermaid Live Editor
Copier/coller le code Mermaid dans : https://mermaid.live/

#### Option 4 : Outils de documentation
- **Draw.io** : Importer les diagrammes Mermaid
- **Notion, Confluence** : Support natif de Mermaid
- **Docusaurus, MkDocs** : Plugins Mermaid disponibles

---

## 📐 Principes d'Architecture Démontrés

### Microservices
✅ Services indépendants avec leurs propres bases de données  
✅ Déploiement et scaling indépendants  
✅ Technologies adaptées par service (PostgreSQL, Redis, SQLite, SQL Server)

### Communication
✅ **Synchrone** : REST API (HTTP/JSON) et gRPC (haute performance)  
✅ **Asynchrone** : RabbitMQ pour event-driven architecture  
✅ API Gateway comme point d'entrée unique

### Patterns Modernes
✅ **CQRS** : Séparation lecture/écriture (Catalog, Basket, Ordering)  
✅ **DDD** : Domain-Driven Design (Ordering avec aggregates)  
✅ **Repository Pattern** : Abstraction d'accès aux données  
✅ **Vertical Slice Architecture** : Organisation par feature  
✅ **Event Sourcing** : Communication par événements

### Résilience
✅ Découplage via RabbitMQ (eventual consistency)  
✅ Cache Redis pour performance  
✅ Circuit Breaker (à implémenter dans API Gateway)  
✅ Retry policies et timeouts

### Observabilité
✅ Logs centralisés (Serilog + ELK)  
✅ Métriques et monitoring (Prometheus + Grafana)  
✅ Distributed tracing pour debugging

---

## 🐳 Containerisation

Tous les composants sont containerisés avec Docker :

```yaml
services:
  - catalog.api + postgresql
  - basket.api + postgresql + redis
  - discount.api (SQLite embarqué)
  - ordering.api + sqlserver
  - apigateway
  - rabbitmq
```

**Orchestration :** Docker Compose pour développement local

---

## 🚀 Évolutions Futures

Les diagrammes sont conçus pour faciliter l'ajout de nouveaux services :

- 📧 **Notification Service** : Emails, SMS, Push notifications
- 📊 **Inventory Service** : Gestion centralisée des stocks
- 🔍 **Search Service** : Elasticsearch pour recherche avancée
- 🤖 **Recommendation Service** : ML pour suggestions personnalisées
- 💳 **Payment Service** : Intégration Stripe/PayPal
- ☸️ **Kubernetes** : Orchestration pour production

---

## 📚 Références

### Architecture
- [Microservices Pattern - Chris Richardson](https://microservices.io/)
- [.NET Microservices Architecture Guide](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)
- [Domain-Driven Design - Eric Evans](https://www.domainlanguage.com/ddd/)

### Documentation
- [Mermaid Documentation](https://mermaid.js.org/)
- [C4 Model](https://c4model.com/)
- [UML Diagrams](https://www.uml-diagrams.org/)

### Patterns
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)

---

## 📝 Notes de Conformité

Ces diagrammes répondent aux **Livrables Attendus** du TP :

✅ **Diagramme de cas d'utilisation** : Parcours client et admin  
✅ **Diagramme de composants** : Microservices + DB + communication  
✅ **Diagramme de séquence** : Parcours complet d'une commande (ajout panier → commande → notification)

**Format :** Mermaid (recommandé dans les outils de documentation du TP)  
**Documentation :** README explicatif avec descriptions détaillées  
**Standards :** UML 2.5 + adaptations pour microservices

---

## 👨‍💻 Utilisation dans le Projet

1. **Phase de conception** : Référence pour l'équipe de développement
2. **Documentation technique** : Intégration dans la doc projet
3. **Onboarding** : Aide les nouveaux développeurs à comprendre l'architecture
4. **Revues d'architecture** : Support pour ADR (Architecture Decision Records)
5. **Communication** : Présentation aux stakeholders

---

## 📄 Licence

Ces diagrammes font partie du projet e-Commerce Microservices  
Tous droits réservés © 2025

