# Atelier 8 - Introduction aux Microservices avec ASP.NET Core

Solution complète du TP : deux microservices (`ProductsAPI`, `OrdersAPI`) exposés
derrière une passerelle API (`ApiGateway` avec Ocelot) et communication
asynchrone entre les microservices via RabbitMQ / MassTransit.

## Architecture

```
Client (Postman / navigateur)
        |
        v
   ApiGateway (Ocelot)  http://localhost:7067
        |                         |
        v                         v
 ProductsAPI                 OrdersAPI
 http://localhost:7066       http://localhost:7065
   |  (ProductsDb)              |  (OrdersDb)
   |                            ^
   |   ProductCreated (RabbitMQ)|
   +-------- MassTransit -------+
```

- `ProductsAPI` : CRUD des produits (émetteur). À chaque ajout d'un produit, un
  message `ProductCreated` est publié dans RabbitMQ.
- `OrdersAPI` : CRUD des commandes (récepteur). Le `ProductCreatedConsumer`
  écoute la file `event-listener` et enregistre le produit reçu dans la table
  `Products` de `OrdersDb`.
- `Shared` : bibliothèque de classes contenant le DTO `ProductCreated`.
- `ApiGateway` : passerelle Ocelot qui redirige `/apigateway/Products` et
  `/apigateway/Orders` vers les microservices.

## Structure du projet

```
TP8/
├── TP8.sln
├── .config/dotnet-tools.json
├── docker-compose.yml
├── Shared/
│   └── Models/ProductCreated.cs
├── ProductsAPI/
│   ├── Controllers/ProductsController.cs
│   ├── Data/ProductsAPIContext.cs
│   ├── Models/Product.cs
│   ├── Migrations/
│   └── Program.cs
├── OrdersAPI/
│   ├── Consumer/ProductCreatedConsumer.cs
│   ├── Controllers/OrdersController.cs
│   ├── Data/OrdersAPIContext.cs
│   ├── Models/Orders.cs
│   ├── Models/Product.cs
│   ├── Migrations/
│   └── Program.cs
└── ApiGateway/
    ├── Ocelot.json
    └── Program.cs
```

## Prérequis

- .NET 9 SDK
- SQL Server LocalDB (`MSSQLLocalDB`)
- Docker Desktop (pour RabbitMQ)
- Postman (ou Swagger) pour les tests

## 1. Démarrer RabbitMQ

Avec Docker Compose :

```bash
docker compose up -d
```

Ou avec Docker directement :

```bash
docker run -d --hostname my-rabbit --name ecomm -p 4001:5672 -p 15672:15672 rabbitmq:3-management
```

- AMQP (utilisé par MassTransit) : `localhost:4001`
- Interface d'administration : http://localhost:15672 (guest / guest)

## 2. Base de données

Les migrations sont déjà présentes dans le projet et les bases ont été créées.
Pour les recréer :

```bash
dotnet tool restore

dotnet tool run dotnet-ef database update --project ProductsAPI --startup-project ProductsAPI
dotnet tool run dotnet-ef database update --project OrdersAPI --startup-project OrdersAPI
```

Pour régénérer les migrations :

```bash
# ProductsAPI
dotnet tool run dotnet-ef migrations add InitialCreate --project ProductsAPI --startup-project ProductsAPI --output-dir Migrations

# OrdersAPI : table Orders
dotnet tool run dotnet-ef migrations add InitialCreate --project OrdersAPI --startup-project OrdersAPI --output-dir Migrations

# OrdersAPI : table Products (ajoutée ensuite)
dotnet tool run dotnet-ef migrations add AddProduct --project OrdersAPI --startup-project OrdersAPI --output-dir Migrations
```

Chaînes de connexion (dans chaque `appsettings.json`) :

- `ProductsDb` : `Server=(localdb)\mssqllocaldb;Database=ProductsDb;...`
- `OrdersDb` : `Server=(localdb)\mssqllocaldb;Database=OrdersDb;...`

## 3. Lancer les microservices

Ouvrir trois terminaux à la racine de la solution :

```bash
# Terminal 1
dotnet run --project ProductsAPI --launch-profile http

# Terminal 2
dotnet run --project OrdersAPI --launch-profile http

# Terminal 3
dotnet run --project ApiGateway --launch-profile http
```

| Service      | URL                                   | Swagger                                  |
|--------------|---------------------------------------|------------------------------------------|
| ProductsAPI  | http://localhost:7066                 | http://localhost:7066/swagger            |
| OrdersAPI    | http://localhost:7065                 | http://localhost:7065/swagger            |
| ApiGateway   | http://localhost:7067                 | http://localhost:7067/swagger            |

## 4. Tester

### Via la passerelle (recommandé)

```bash
# Ajouter un produit
curl -X POST http://localhost:7067/apigateway/Products ^
  -H "Content-Type: application/json" ^
  -d "{\"name\":\"prod12\",\"stockQte\":660,\"price\":1500}"

# Lister les produits
curl http://localhost:7067/apigateway/Products

# Lister les commandes
curl http://localhost:7067/apigateway/Orders
```

Le produit ajouté dans `ProductsAPI` est enregistré dans `ProductsDb`, puis le
message `ProductCreated` est publié dans RabbitMQ. `OrdersAPI` le consomme et
l'enregistre dans la table `Products` de `OrdersDb` : les deux bases sont
synchronisées de façon asynchrone.

## Versions utilisées

Le TP d'origine utilise Ocelot 18.0.0. Comme la solution cible .NET 9, les
versions compatibles suivantes ont été utilisées :

- `Ocelot` 25.0.2
- `MassTransit` / `MassTransit.RabbitMQ` 8.5.12
- `Microsoft.EntityFrameworkCore.SqlServer` / `.Design` 9.0.22
- `Swashbuckle.AspNetCore` 9.0.6

`MassTransit.AspNetCore` n'est plus nécessaire à partir de MassTransit 8 :
l'intégration ASP.NET Core est incluse dans le package `MassTransit`.
