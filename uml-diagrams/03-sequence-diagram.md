# Diagramme de Séquence - Parcours d'une Commande

Ce diagramme illustre le flux complet d'une commande depuis l'ajout au panier jusqu'à la notification, incluant les interactions entre tous les microservices.

## 1. Ajout au Panier et Application de Réduction

```mermaid
sequenceDiagram
    actor Client
    participant Gateway as API Gateway
    participant Auth as Identity Service
    participant Catalog as Catalog.API
    participant Basket as Basket.API
    participant Discount as Discount.API
    participant Redis as Redis Cache
    
    Client->>Gateway: GET /catalog/products
    Gateway->>Catalog: Forward request
    Catalog->>Catalog: Query products
    Catalog-->>Gateway: Return product list
    Gateway-->>Client: Display products
    
    Client->>Gateway: POST /basket/add-item
    Gateway->>Auth: Validate JWT Token
    Auth-->>Gateway: Token valid + UserId
    
    Gateway->>Basket: Add item to basket (UserId, ProductId, Quantity)
    Basket->>Catalog: GET /products/{id} (verify product exists & price)
    Catalog-->>Basket: Product details
    
    Basket->>Redis: Get cached basket
    alt Basket exists in cache
        Redis-->>Basket: Return basket
    else Basket not in cache
        Basket->>Basket: Create new basket
    end
    
    Basket->>Basket: Add/Update item in basket
    Basket->>Redis: Update cached basket
    Basket-->>Gateway: Basket updated
    Gateway-->>Client: Item added successfully
    
    Note over Client,Discount: Application d'un code de réduction
    
    Client->>Gateway: POST /basket/apply-coupon {couponCode}
    Gateway->>Auth: Validate token
    Auth-->>Gateway: Valid
    
    Gateway->>Basket: Apply coupon request
    Basket->>Discount: gRPC GetDiscount(ProductId, CouponCode)
    Discount->>Discount: Validate coupon
    Discount-->>Basket: Discount amount/percentage
    
    Basket->>Basket: Recalculate totals with discount
    Basket->>Redis: Update basket
    Basket-->>Gateway: Updated basket with discount
    Gateway-->>Client: Discount applied
```

## 2. Passage de Commande (Checkout)

```mermaid
sequenceDiagram
    actor Client
    participant Gateway as API Gateway
    participant Auth as Identity Service
    participant Basket as Basket.API
    participant RabbitMQ as RabbitMQ
    participant Ordering as Ordering.API
    participant DB as SQL Server
    
    Client->>Gateway: POST /basket/checkout
    Note over Client: Avec adresse livraison,<br/>infos paiement, etc.
    
    Gateway->>Auth: Validate JWT Token
    Auth-->>Gateway: Valid + UserId
    
    Gateway->>Basket: Checkout basket (UserId)
    Basket->>Basket: Get basket from Redis
    Basket->>Basket: Validate basket (items exist, stock available)
    
    alt Basket valid
        Basket->>Basket: Create BasketCheckoutEvent
        Note over Basket: Event contains:<br/>- UserId<br/>- Items + Quantities + Prices<br/>- TotalPrice<br/>- Applied discount<br/>- Delivery address<br/>- Payment info
        
        Basket->>RabbitMQ: Publish BasketCheckoutEvent
        Basket-->>Gateway: Checkout initiated
        Gateway-->>Client: Order processing...
        
        RabbitMQ->>Ordering: BasketCheckoutEvent received
        
        Note over Ordering: Event Handler<br/>(Asynchronous)
        
        Ordering->>Ordering: Create Order (DDD Aggregate)
        Note over Ordering: Apply business rules:<br/>- Validate total<br/>- Check inventory<br/>- Create Order entity
        
        Ordering->>DB: BEGIN TRANSACTION
        Ordering->>DB: INSERT Order
        Ordering->>DB: INSERT OrderItems
        Ordering->>DB: COMMIT TRANSACTION
        
        Ordering->>Ordering: Create OrderCreatedEvent
        Ordering->>RabbitMQ: Publish OrderCreatedEvent
        
    else Basket invalid
        Basket-->>Gateway: Error (item unavailable, etc.)
        Gateway-->>Client: Checkout failed
    end
```

## 3. Notification et Suivi de Commande

```mermaid
sequenceDiagram
    actor Client
    participant Gateway as API Gateway
    participant RabbitMQ as RabbitMQ
    participant Ordering as Ordering.API
    participant Notification as Notification Service
    participant Email as Email Provider
    
    Note over RabbitMQ,Notification: Service optionnel (évolution future)
    
    RabbitMQ->>Notification: OrderCreatedEvent received
    Notification->>Notification: Format email template
    Notification->>Email: Send confirmation email
    Email-->>Client: Email received
    
    Note over Client,Ordering: Suivi de la commande
    
    Client->>Gateway: GET /orders/{orderId}
    Gateway->>Ordering: Get order details
    Ordering->>Ordering: Query order (CQRS Read Model)
    Ordering-->>Gateway: Order details + status
    Gateway-->>Client: Display order info
    
    Note over Client,Ordering: Mise à jour du statut (par admin ou système)
    
    Ordering->>Ordering: Update order status
    Note over Ordering: Status changes:<br/>- Pending<br/>- Confirmed<br/>- Shipped<br/>- Delivered<br/>- Cancelled
    
    Ordering->>RabbitMQ: Publish OrderUpdatedEvent
    RabbitMQ->>Notification: OrderUpdatedEvent received
    Notification->>Email: Send status update email
    Email-->>Client: Email received
    
    Client->>Gateway: GET /orders/my-orders
    Gateway->>Ordering: Get user orders
    Ordering-->>Gateway: List of orders
    Gateway-->>Client: Display order history
```

## Description des Flux

### Flux 1 : Ajout au Panier et Application de Réduction

**Étapes clés :**

1. **Consultation du catalogue** : Le client parcourt les produits via Catalog.API
2. **Authentification** : L'API Gateway valide le JWT token via Identity Service
3. **Ajout au panier** : 
   - Basket.API vérifie le produit auprès de Catalog.API
   - Récupère ou crée le panier depuis Redis (cache)
   - Ajoute l'item et met à jour le cache
4. **Application coupon** :
   - Communication gRPC synchrone avec Discount.API
   - Validation du coupon et récupération du montant
   - Recalcul des totaux avec réduction

**Pattern utilisé :** Cache-Aside Pattern avec Redis

### Flux 2 : Passage de Commande (Checkout)

**Étapes clés :**

1. **Validation du panier** : Vérification des items, stock, prix
2. **Publication événement** : BasketCheckoutEvent publié vers RabbitMQ
3. **Traitement asynchrone** :
   - Ordering.API souscrit à l'événement
   - Crée la commande avec les règles métier (DDD)
   - Persiste dans SQL Server avec transaction
4. **Notification** : OrderCreatedEvent publié pour autres services

**Patterns utilisés :** 
- Event-Driven Architecture
- Saga Pattern (implicite)
- CQRS (Command pour créer la commande)

### Flux 3 : Notification et Suivi

**Étapes clés :**

1. **Notification automatique** : Service abonné aux événements envoie des emails
2. **Consultation commande** : CQRS Read Model pour requêtes optimisées
3. **Mise à jour statut** : Événements OrderUpdatedEvent pour communication asynchrone
4. **Historique** : Client peut consulter toutes ses commandes

**Patterns utilisés :**
- Observer Pattern (via RabbitMQ)
- CQRS (Read Model optimisé pour les requêtes)

## Avantages de cette Architecture

### Résilience
- Si Ordering.API est down, le BasketCheckoutEvent reste dans RabbitMQ et sera traité plus tard
- Les services sont découplés grâce à la communication asynchrone

### Performance
- Redis cache pour accès rapide aux paniers
- gRPC pour communication haute performance (Discount)
- CQRS pour séparer lecture/écriture

### Scalabilité
- Chaque service peut être scalé indépendamment
- RabbitMQ gère la charge avec des queues

### Observabilité
- Chaque interaction est traçable
- Logs centralisés permettent de suivre le parcours complet

## Gestion des Erreurs

| Scénario | Gestion |
|----------|---------|
| **Produit inexistant** | Basket.API vérifie avec Catalog avant ajout, retourne erreur 404 |
| **Coupon invalide** | Discount.API retourne discount = 0, le panier n'est pas modifié |
| **Service down** | API Gateway retourne erreur 503, client réessaie plus tard |
| **Événement perdu** | RabbitMQ garantit la livraison (persistent messages) |
| **Transaction échouée** | Rollback automatique de la transaction SQL Server |

## Communication

| Type | Usage | Avantages | Inconvénients |
|------|-------|-----------|---------------|
| **REST API** | Gateway ↔ Services, Services ↔ Services | Standard, facile à déboguer | Plus lent que gRPC |
| **gRPC** | Basket → Discount | Haute performance, typage fort | Plus complexe |
| **RabbitMQ** | Événements asynchrones | Découplage, résilience | Complexité, eventual consistency |

