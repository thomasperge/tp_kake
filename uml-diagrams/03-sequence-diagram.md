# 🛒 Diagramme de Séquence - Parcours d'une Commande

Ce diagramme illustre le flux complet d'une commande depuis l'ajout au panier jusqu'à la notification, incluant les interactions entre tous les microservices.

## 🛍️ ÉTAPE 1 : Consultation du Catalogue et Ajout au Panier

```mermaid
sequenceDiagram
    actor Client as 👤 Client
    participant Gateway as 🚪 Passerelle API
    participant Auth as 🔐 Service Authentification
    participant Catalog as 🛍️ Service Catalogue
    participant Basket as 🛒 Service Panier
    participant Discount as 🎟️ Service Réductions
    participant Redis as ⚡ Cache Redis
    
    Note over Client,Redis: 📋 ÉTAPE 1A : Consultation du catalogue
    
    Client->>Gateway: 🔍 GET /catalog/products<br/>"Montre-moi les produits"
    Gateway->>Catalog: ➡️ Transfère la requête
    Catalog->>Catalog: 🔎 Recherche dans la base de données
    Catalog-->>Gateway: 📦 Liste des produits disponibles
    Gateway-->>Client: ✅ Affichage des produits
    
    Note over Client,Redis: 📋 ÉTAPE 1B : Ajout d'un produit au panier
    
    Client->>Gateway: 🛒 POST /basket/add-item<br/>"Ajoute ce produit à mon panier"
    Gateway->>Auth: 🔑 Vérification du token JWT
    Auth-->>Gateway: ✅ Token valide + ID utilisateur
    
    Gateway->>Basket: 📝 Ajouter au panier (ID utilisateur, ID produit, Quantité)
    Basket->>Catalog: 🔍 GET /products/{id}<br/>"Ce produit existe-t-il ? Quel est son prix ?"
    Catalog-->>Basket: 💰 Détails du produit (prix, nom, disponibilité)
    
    Basket->>Redis: 🔍 Récupérer le panier en cache
    alt Panier existe dans le cache
        Redis-->>Basket: 📦 Retourne le panier existant
    else Panier n'existe pas
        Basket->>Basket: 🆕 Créer un nouveau panier
    end
    
    Basket->>Basket: ➕ Ajouter/Mettre à jour l'article dans le panier
    Basket->>Redis: 💾 Mettre à jour le panier en cache
    Basket-->>Gateway: ✅ Panier mis à jour
    Gateway-->>Client: 🎉 "Produit ajouté avec succès !"
    
    Note over Client,Discount: 📋 ÉTAPE 1C : Application d'un code de réduction
    
    Client->>Gateway: 🎟️ POST /basket/apply-coupon {couponCode}<br/>"Applique le code NOEL2025"
    Gateway->>Auth: 🔑 Vérification du token
    Auth-->>Gateway: ✅ Token valide
    
    Gateway->>Basket: 🎫 Demande d'application du coupon
    Basket->>Discount: 🚀 gRPC GetDiscount(ProductId, CouponCode)<br/>"Ce code est-il valide ?"
    Discount->>Discount: 🔍 Vérification du coupon
    Discount-->>Basket: 💰 Montant de la réduction (ex: -20%)
    
    Basket->>Basket: 🧮 Recalculer les totaux avec la réduction
    Basket->>Redis: 💾 Mettre à jour le panier
    Basket-->>Gateway: ✅ Panier mis à jour avec réduction
    Gateway-->>Client: 🎉 "Réduction appliquée ! Nouveau total : XX€"
```

## 💳 ÉTAPE 2 : Passage de Commande (Checkout)

```mermaid
sequenceDiagram
    actor Client as 👤 Client
    participant Gateway as 🚪 Passerelle API
    participant Auth as 🔐 Service Authentification
    participant Basket as 🛒 Service Panier
    participant RabbitMQ as 📨 Bus de Messages
    participant Ordering as 📦 Service Commandes
    participant DB as 🗄️ Base SQL Server
    
    Note over Client,DB: 📋 ÉTAPE 2A : Validation de la commande
    
    Client->>Gateway: 💳 POST /basket/checkout<br/>"Je valide ma commande"
    Note over Client: 📝 Avec adresse de livraison,<br/>informations de paiement, etc.
    
    Gateway->>Auth: 🔑 Vérification du token JWT
    Auth-->>Gateway: ✅ Token valide + ID utilisateur
    
    Gateway->>Basket: 🛒 Validation du panier (ID utilisateur)
    Basket->>Basket: 🔍 Récupérer le panier depuis Redis
    Basket->>Basket: ✅ Vérifier le panier (articles existent, stock disponible)
    
    alt Panier valide
        Note over Client,DB: 📋 ÉTAPE 2B : Création de la commande
        
        Basket->>Basket: 📝 Créer l'événement BasketCheckoutEvent
        Note over Basket: 📦 L'événement contient :<br/>- ID utilisateur<br/>- Articles + Quantités + Prix<br/>- Prix total<br/>- Réduction appliquée<br/>- Adresse de livraison<br/>- Informations de paiement
        
        Basket->>RabbitMQ: 📤 Publier BasketCheckoutEvent<br/>"Nouveau panier à transformer en commande !"
        Basket-->>Gateway: ✅ Commande initiée
        Gateway-->>Client: ⏳ "Commande en cours de traitement..."
        
        RabbitMQ->>Ordering: 📨 BasketCheckoutEvent reçu
        
        Note over Ordering: 🔄 Gestionnaire d'événements<br/>(Traitement asynchrone)
        
        Ordering->>Ordering: 🏗️ Créer la commande (Agrégat DDD)
        Note over Ordering: 📋 Appliquer les règles métier :<br/>- Valider le total<br/>- Vérifier l'inventaire<br/>- Créer l'entité Commande
        
        Ordering->>DB: 🔄 DÉBUT TRANSACTION
        Ordering->>DB: 💾 INSERT Commande
        Ordering->>DB: 💾 INSERT Articles de la commande
        Ordering->>DB: ✅ COMMIT TRANSACTION
        
        Ordering->>Ordering: 📝 Créer OrderCreatedEvent
        Ordering->>RabbitMQ: 📤 Publier OrderCreatedEvent<br/>"Commande créée ! Envoyez les notifications"
        
    else Panier invalide
        Basket-->>Gateway: ❌ Erreur (article indisponible, etc.)
        Gateway-->>Client: 💥 "Échec de la commande"
    end
```

## 📧 ÉTAPE 3 : Notification et Suivi de Commande

```mermaid
sequenceDiagram
    actor Client as 👤 Client
    participant Gateway as 🚪 Passerelle API
    participant RabbitMQ as 📨 Bus de Messages
    participant Ordering as 📦 Service Commandes
    participant Notification as 📧 Service Notifications
    participant Email as 📮 Fournisseur Email
    
    Note over RabbitMQ,Notification: 🔮 Service optionnel (évolution future)
    
    Note over Client,Email: 📋 ÉTAPE 3A : Notification automatique
    
    RabbitMQ->>Notification: 📨 OrderCreatedEvent reçu<br/>"Une nouvelle commande a été créée !"
    Notification->>Notification: 📝 Formater le template d'email
    Notification->>Email: 📤 Envoyer l'email de confirmation
    Email-->>Client: 📧 Email reçu : "Votre commande #12345 a été confirmée !"
    
    Note over Client,Ordering: 📋 ÉTAPE 3B : Suivi de la commande par le client
    
    Client->>Gateway: 🔍 GET /orders/{orderId}<br/>"Montre-moi le statut de ma commande"
    Gateway->>Ordering: 📋 Récupérer les détails de la commande
    Ordering->>Ordering: 🔎 Requête sur la commande (Modèle de lecture CQRS)
    Ordering-->>Gateway: 📊 Détails de la commande + statut
    Gateway-->>Client: 📱 Affichage des informations de commande
    
    Note over Client,Ordering: 📋 ÉTAPE 3C : Mise à jour du statut (par admin ou système)
    
    Ordering->>Ordering: 🔄 Mettre à jour le statut de la commande
    Note over Ordering: 📈 Changements de statut :<br/>- ⏳ En attente<br/>- ✅ Confirmée<br/>- 🚚 Expédiée<br/>- 📦 Livrée<br/>- ❌ Annulée
    
    Ordering->>RabbitMQ: 📤 Publier OrderUpdatedEvent<br/>"Le statut de la commande a changé !"
    RabbitMQ->>Notification: 📨 OrderUpdatedEvent reçu
    Notification->>Email: 📤 Envoyer l'email de mise à jour
    Email-->>Client: 📧 Email reçu : "Votre commande #12345 a été expédiée !"
    
    Client->>Gateway: 📋 GET /orders/my-orders<br/>"Montre-moi toutes mes commandes"
    Gateway->>Ordering: 📊 Récupérer les commandes de l'utilisateur
    Ordering-->>Gateway: 📝 Liste des commandes
    Gateway-->>Client: 📱 Affichage de l'historique des commandes
```

## 📋 Description des Flux

### 🛍️ Flux 1 : Consultation du Catalogue et Ajout au Panier

**Étapes clés :**

1. **🔍 Consultation du catalogue** : Le client parcourt les produits via le Service Catalogue
2. **🔑 Authentification** : La Passerelle API valide le token JWT via le Service Authentification
3. **🛒 Ajout au panier** : 
   - Le Service Panier vérifie le produit auprès du Service Catalogue
   - Récupère ou crée le panier depuis Redis (cache super rapide)
   - Ajoute l'article et met à jour le cache
4. **🎟️ Application coupon** :
   - Communication gRPC synchrone avec le Service Réductions
   - Validation du coupon et récupération du montant de réduction
   - Recalcul des totaux avec la réduction appliquée

**Pattern utilisé :** Cache-Aside Pattern avec Redis ⚡

### 💳 Flux 2 : Passage de Commande (Checkout)

**Étapes clés :**

1. **✅ Validation du panier** : Vérification des articles, stock disponible, prix
2. **📤 Publication événement** : BasketCheckoutEvent publié vers RabbitMQ
3. **🔄 Traitement asynchrone** :
   - Le Service Commandes souscrit à l'événement
   - Crée la commande avec les règles métier (DDD)
   - Sauvegarde dans SQL Server avec transaction sécurisée
4. **📧 Notification** : OrderCreatedEvent publié pour les autres services

**Patterns utilisés :** 
- 🏗️ Event-Driven Architecture
- 📋 Saga Pattern (implicite)
- 🔄 CQRS (Command pour créer la commande)

### 📧 Flux 3 : Notification et Suivi

**Étapes clés :**

1. **📨 Notification automatique** : Service abonné aux événements envoie des emails
2. **🔍 Consultation commande** : CQRS Read Model pour requêtes optimisées
3. **🔄 Mise à jour statut** : Événements OrderUpdatedEvent pour communication asynchrone
4. **📋 Historique** : Le client peut consulter toutes ses commandes

**Patterns utilisés :**
- 👀 Observer Pattern (via RabbitMQ)
- 🔄 CQRS (Read Model optimisé pour les requêtes)

## 🏗️ Avantages de cette Architecture

### 🛡️ Résilience
- Si le Service Commandes est en panne, le BasketCheckoutEvent reste dans RabbitMQ et sera traité plus tard
- Les services sont découplés grâce à la communication asynchrone
- Un service en panne n'empêche pas les autres de fonctionner

### ⚡ Performance
- Redis cache pour accès ultra-rapide aux paniers
- gRPC pour communication haute performance (Service Réductions)
- CQRS pour séparer lecture/écriture et optimiser les performances

### 📈 Scalabilité
- Chaque service peut être scalé indépendamment selon ses besoins
- RabbitMQ gère la charge avec des queues intelligentes
- Possibilité d'ajouter des instances de services selon la demande

### 🔍 Observabilité
- Chaque interaction est traçable et loggée
- Logs centralisés permettent de suivre le parcours complet d'une commande
- Monitoring en temps réel de tous les services

## 🚨 Gestion des Erreurs

| Scénario | Gestion |
|----------|---------|
| **❌ Produit inexistant** | Le Service Panier vérifie avec le Service Catalogue avant ajout, retourne erreur 404 |
| **❌ Coupon invalide** | Le Service Réductions retourne réduction = 0, le panier n'est pas modifié |
| **❌ Service en panne** | La Passerelle API retourne erreur 503, le client peut réessayer plus tard |
| **❌ Événement perdu** | RabbitMQ garantit la livraison (messages persistants) |
| **❌ Transaction échouée** | Rollback automatique de la transaction SQL Server |

## 📡 Types de Communication

| Type | Usage | Avantages | Inconvénients |
|------|-------|-----------|---------------|
| **🌐 REST API** | Passerelle ↔ Services, Services ↔ Services | Standard, facile à déboguer | Plus lent que gRPC |
| **🚀 gRPC** | Panier → Réductions | Haute performance, typage fort | Plus complexe à implémenter |
| **📨 RabbitMQ** | Événements asynchrones | Découplage, résilience | Complexité, eventual consistency |

