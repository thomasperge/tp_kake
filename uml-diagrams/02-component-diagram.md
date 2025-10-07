# Diagramme de Composants - Architecture e-Commerce

Ce diagramme montre comment les différents services de la plateforme e-commerce communiquent entre eux.

## Vue Simplifiée

```mermaid
graph TB
    subgraph "Applications Client"
        WEB[WebApp<br/>Interface web clients]
        MOBILE[Mobile App<br/>Application mobile]
    end
    
    subgraph "API Gateway"
        GATEWAY[Ocelot API Gateway<br/>Point d'entrée unique<br/>Routage & Sécurité]
    end
    
    subgraph "Microservices"
        CATALOG[Catalog.API<br/>Gestion des produits<br/>Recherche & Filtrage]
        BASKET[Basket.API<br/>Gestion des paniers<br/>Calcul des totaux]
        DISCOUNT[Discount.API<br/>Gestion des coupons<br/>Validation codes promo]
        ORDERING[Ordering.API<br/>Gestion des commandes<br/>Suivi des statuts]
    end
    
    subgraph "Bases de Données"
        CATALOGDB[(MongoDB<br/>Produits<br/>Stockage produits)]
        BASKETDB[(PostgreSQL<br/>Paniers<br/>Sauvegarde paniers)]
        REDIS[(Redis<br/>Cache<br/>Sessions rapides)]
        DISCOUNTDB[(SQLite<br/>Coupons<br/>Codes de réduction)]
        ORDERINGDB[(SQL Server<br/>Commandes<br/>Transactions sécurisées)]
    end
    
    subgraph "Communication"
        RABBITMQ[RabbitMQ<br/>Message Broker<br/>Événements asynchrones]
        GRPC[gRPC<br/>Communication<br/>]
    end
    
    subgraph "Services Partagés"
        AUTH[Identity Service<br/>Authentification<br/>Gestion des utilisateurs]
    end
    
    WEB --> GATEWAY
    MOBILE --> GATEWAY
    
    GATEWAY --> CATALOG
    GATEWAY --> BASKET
    GATEWAY --> ORDERING
    GATEWAY --> AUTH
    
    CATALOG --> CATALOGDB
    BASKET --> BASKETDB
    BASKET --> REDIS
    BASKET -->|gRPC| DISCOUNT
    DISCOUNT --> DISCOUNTDB
    ORDERING --> ORDERINGDB
    
    BASKET --> RABBITMQ
    ORDERING --> RABBITMQ
    
    style GATEWAY fill:#4A90E2,color:#fff
    style CATALOG fill:#7ED321,color:#000
    style BASKET fill:#F5A623,color:#000
    style DISCOUNT fill:#BD10E0,color:#fff
    style ORDERING fill:#50E3C2,color:#000
    style RABBITMQ fill:#FF6B6B,color:#fff
    style GRPC fill:#9B59B6,color:#fff
    style AUTH fill:#4ECDC4,color:#000
```

## 📋 Architecture Microservices e-Commerce

### 🖥️ Applications Client

| Composant | Description |
|-----------|-------------|
| **WebApp** | Application web pour les clients |
| **Mobile App** | Application mobile pour smartphones |

### 🚪 API Gateway

| Composant | Description |
|-----------|-------------|
| **Ocelot API Gateway** | Point d'entrée unique, routage et sécurité |

### 🎯 Microservices

| Service | Rôle | Base de données |
|---------|------|----------------|
| **Catalog.API** | Gestion des produits | MongoDB |
| **Basket.API** | Gestion des paniers | PostgreSQL + Redis |
| **Discount.API** | Gestion des coupons | SQLite |
| **Ordering.API** | Gestion des commandes | SQL Server |

### 📨 Communication

| Composant | Rôle |
|-----------|------|
| **RabbitMQ** | Message Broker pour communication asynchrone |
| **gRPC** | Communication haute performance entre services |
| **Identity Service** | Authentification et autorisation |

## 🔄 Flux de Communication

### Communication Synchrone
- **WebApp/Mobile** → **Ocelot API Gateway** → **Microservices**
- **Basket.API** → **Discount.API** (gRPC)

### Communication Asynchrone
- **Basket.API** → **RabbitMQ** → **Ordering.API**
- Messages d'événements entre services

## 🏗️ Avantages de cette Architecture

### ✅ Indépendance des Services
- Chaque microservice a sa propre base de données
- Déploiement et scaling indépendants
- Technologie adaptée par service

### ✅ Résilience
- Si un service tombe, les autres continuent
- Messages asynchrones avec RabbitMQ
- Pas de point de défaillance unique

### ✅ Performance
- Redis pour cache rapide des paniers
- Bases de données optimisées par usage
- Communication gRPC haute performance

## 🐳 Containerisation

Tous les services sont containerisés avec Docker :

```yaml
services:
  - catalog.api + postgresql
  - basket.api + postgresql + redis  
  - discount.api + sqlite
  - ordering.api + sqlserver
  - ocelot.api.gateway
  - rabbitmq
```

## 📊 Technologies Utilisées

| Composant | Technologie |
|-----------|-------------|
| **API Gateway** | Ocelot |
| **Microservices** | ASP.NET Core |
| **Message Broker** | RabbitMQ |
| **Bases de données** | MongoDB, PostgreSQL, Redis, SQLite, SQL Server |
| **Containerisation** | Docker Compose |

