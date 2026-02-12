# Module 6 : Conception d'API Avancée et Bonnes Pratiques - Pour aller plus loin

### Objectifs Pédagogiques

À la fin de ce module complémentaire, vous serez capable de :

* **Utiliser** les DTOs (Data Transfer Objects) pour découpler votre API de votre modèle de données interne.
* **Implémenter** le versioning d'API pour gérer l'évolution de votre contrat de service sans casser les clients
  existants.
* **Mettre en place** la pagination pour gérer efficacement les grandes collections de données.
* **Générer** une documentation d'API interactive et professionnelle avec Swagger/OpenAPI.
* **Utiliser** des outils comme Postman pour tester et interagir avec votre API.

### Introduction : Devenir un fournisseur de services cinq étoiles

Vous avez ouvert votre service au volant. Les clients viennent et repartent. C'est bien. Mais comment passer au niveau
supérieur ? Un service cinq étoiles ne se contente pas de donner la nourriture. Il propose un emballage optimisé pour le
transport (DTOs), un menu clair et détaillé (Swagger), et il sait gérer les très grosses commandes sans bloquer la
circulation (pagination). Plus important encore, si vous décidez de changer une recette, vous ne rendez pas l'ancien
menu obsolète du jour au lendemain (versioning).

Ces concepts sont la clé pour créer des APIs que les autres développeurs **aimeront** utiliser. Une bonne API n'est pas
seulement fonctionnelle, elle est aussi stable, bien documentée, performante et prévisible. Ce sont ces qualités qui
feront de vous un concepteur d'API recherché.

---

## 1. Les DTOs (Data Transfer Objects) : L'emballage sur mesure

**Le problème :** Dans notre TP, nous avons accepté et retourné notre entité EF Core `Product` directement. C'est
simple, mais dangereux.

1. **Sur-exposition :** Votre entité peut contenir des données que vous ne voulez pas exposer (ex: le coût d'achat, des
   métadonnées internes).
2. **Sous-exposition :** Votre API a peut-être besoin de retourner des données calculées qui n'existent pas dans l'
   entité.
3. **Couplage fort :** Si vous renommez une propriété dans votre entité de base de données, vous cassez le contrat de
   votre API et tous vos clients.
4. **Sécurité :** Un client pourrait essayer de poster des données pour des champs que vous ne voulez pas qu'il
   modifie (ex: `IsAdmin`). C'est une vulnérabilité de "mass assignment".

**La solution :** Les DTOs. Un DTO est une classe simple, conçue **spécifiquement** pour le transfert de données via
votre API. C'est l'emballage. Vous ne donnez pas au client le plat qui sort du four, vous le mettez dans un emballage
adapté.

On crée généralement des DTOs différents pour la lecture et pour l'écriture.

```c#
// DTO pour lire un produit
public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string CategoryName { get; set; } // Donnée agrégée
}

// DTO pour créer/mettre à jour un produit
public class ProductUpsertDto
{
    [Required]
    public string Name { get; set; }
    
    [Range(0, 10000)]
    public decimal Price { get; set; }
    
    [Required]
    public int CategoryId { get; set; }
}
```

Votre contrôleur doit maintenant faire le **mapping** entre l'entité et le DTO.

```c#
[HttpGet("{id}")]
public ActionResult<ProductDto> GetProduct(int id)
{
    var product = _repository.GetById(id);
    if (product == null) return NotFound();
    
    // Mapping manuel
    var productDto = new ProductDto
    {
        Id = product.Id,
        Name = product.Name,
        Price = product.Price,
        CategoryName = product.Category?.Name
    };
    
    return Ok(productDto);
}
```

#### Mapping automatique
Le mapping manuel peut devenir fastidieux. Des librairies comme **AutoMapper** peuvent automatiser ce processus. 
C'est un outil indispensable dans les projets de grande envergure.

##### Installation
```
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

##### Configuration

```c#
// Mappings/MappingProfile.cs

using AutoMapper;


/// PROFIL DE MAPPING AUTOMAPPER
/// 
/// AutoMapper est une bibliothèque qui permet de convertir automatiquement
/// un objet d'un type vers un autre type (par exemple, Product vers ProductDto).
/// 
/// Un "Profile" est une classe qui définit COMMENT ces conversions doivent se faire.
/// C'est comme un plan de correspondance entre différents objets.

public class MappingProfile : Profile
{

    /// CONSTRUCTEUR - Appelé une seule fois au démarrage de l'application
    /// C'est ici qu'on configure toutes les règles de conversion
    public MappingProfile()
    {
        // ═══════════════════════════════════════════════════════════════
        // MAPPING 1 : Product → ProductDto
        // ═══════════════════════════════════════════════════════════════
        // Convertit un objet "Product" (entité de base de données)
        // vers un "ProductDto" (objet pour envoyer les données au client)
        
        CreateMap<Product, ProductDto>()
            
            // RÈGLE PERSONNALISÉE #1 : Calcul de TotalValue
            // -----------------------------------------------
            // .ForMember() = "Pour la propriété TotalValue de ProductDto..."
            .ForMember(
                dest => dest.TotalValue,  // dest = destination (ProductDto)
                                           // On cible la propriété "TotalValue"
                
                opt => opt.MapFrom(       // opt.MapFrom = "prends la valeur depuis..."
                    src => src.Price * src.Stock  // src = source (Product)
                                                   // Calcul : Prix × Stock
                )
                // Résultat : TotalValue sera automatiquement calculé lors du mapping
                // Exemple : Si Price=10 et Stock=5, alors TotalValue=50
            )
            
            // RÈGLE PERSONNALISÉE #2 : Récupération du nom de catégorie
            // ----------------------------------------------------------
            .ForMember(
                dest => dest.CategoryName,  // On cible "CategoryName" dans ProductDto
                
                opt => opt.MapFrom(
                    src => src.Category.Name  // On va chercher le nom dans l'objet lié
                                               // Product.Category.Name
                )
                // Résultat : Au lieu de renvoyer tout l'objet Category,
                // on envoie juste son nom (string)
                // Cela évite d'envoyer trop de données au client
            );
        
        // NOTE : Les autres propriétés (Id, Name, Price, Stock) seront mappées
        // automatiquement car elles ont le même nom dans Product et ProductDto
        
        
        // ═══════════════════════════════════════════════════════════════
        // MAPPING 2 : ProductCreateDto → Product
        // ═══════════════════════════════════════════════════════════════
        // Convertit les données de création (venant du client)
        // vers un objet Product (pour l'enregistrer en base de données)
        
        CreateMap<ProductCreateDto, Product>();
        
        // Pas de règles personnalisées ici !
        // AutoMapper fera automatiquement la correspondance entre
        // les propriétés qui ont le même nom
        // Exemple : ProductCreateDto.Name → Product.Name
        
        
        // ═══════════════════════════════════════════════════════════════
        // MAPPING 3 : ProductUpdateDto → Product
        // ═══════════════════════════════════════════════════════════════
        // Convertit les données de mise à jour (venant du client)
        // vers un objet Product existant
        
        CreateMap<ProductUpdateDto, Product>()
            
            // RÈGLE SPÉCIALE : Ne mapper que les valeurs NON-NULL
            // ----------------------------------------------------
            // .ForAllMembers() = "Pour TOUTES les propriétés..."
            .ForAllMembers(
                opts => opts.Condition(  // .Condition() = "seulement si..."
                    (src, dest, srcMember) => srcMember != null
                    // src = ProductUpdateDto (source)
                    // dest = Product (destination)
                    // srcMember = la valeur de la propriété à mapper
                    
                    // Condition : srcMember != null
                    // = "Ne mappe que si la valeur n'est pas null"
                )
            );
        
        /*
          POURQUOI CETTE RÈGLE ?
          ----------------------
          Lors d'une mise à jour, l'utilisateur envoie parfois seulement
          certains champs à modifier (par exemple, juste le prix).
          
          Sans cette règle :
          - ProductUpdateDto { Price = 15, Name = null, Stock = null }
          - Résultat : Product.Name et Product.Stock seraient écrasés avec null
          
          Avec cette règle :
          - Seul Product.Price est mis à jour avec 15
          - Product.Name et Product.Stock gardent leurs valeurs actuelles
          
          C'est ce qu'on appelle un "PATCH partiel"
        */
    }
}
```

##### Résumé des concets clefs

1. Profile
   → Classe de configuration pour AutoMapper
   → Définit les règles de mapping

2. CreateMap<Source, Destination>()
   → Crée une règle de conversion de Source vers Destination
   → Par défaut, mappe automatiquement les propriétés de même nom

3. ForMember(dest => dest.Property, ...)
   → Définit une règle personnalisée pour UNE propriété spécifique
   → Utile pour calculs, conversions, ou navigation dans objets liés

4. MapFrom(src => ...)
   → Indique d'où vient la valeur à mapper
   → Peut contenir des calculs ou accès à des propriétés imbriquées

5. ForAllMembers(...)
   → Applique une règle à TOUTES les propriétés
   
6. Condition(...)
   → Ajoute une condition : "ne mapper que si..."
   → Exemple : seulement si la valeur n'est pas null

**Avantages**

- Moins de code répétitif
- Conversions automatiques
- Code plus propre et maintenable
- Séparation entre entités (base de données) et DTOs (API)



##### Déclaration du service

```c#
// Program.cs
builder.Services.AddAutoMapper(typeof(Program));
```

##### Utilisation

```c#
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;
    private readonly IMapper _mapper;  // Injecter AutoMapper

    public ProductsController(IProductRepository repository, IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }

    [HttpGet]
    public ActionResult<IEnumerable<ProductDto>> GetAll()
    {
        var products = _repository.GetAll();
        
        // Mapping automatique
        var dtos = _mapper.Map<IEnumerable<ProductDto>>(products);
        
        return Ok(dtos);
    }

    [HttpGet("{id}")]
    public ActionResult<ProductDto> GetById(int id)
    {
        var product = _repository.GetById(id);
        
        if (product == null)
            return NotFound();
        
        // Mapping automatique
        var dto = _mapper.Map<ProductDto>(product);
        
        return Ok(dto);
    }

    [HttpPost]
    public ActionResult<ProductDto> Create(ProductCreateDto createDto)
    {
        // Mapping automatique DTO → Entité
        var product = _mapper.Map<Product>(createDto);
        
        _repository.Add(product);
        
        // Mapping automatique Entité → DTO
        var dto = _mapper.Map<ProductDto>(product);
        
        return CreatedAtAction(nameof(GetById), new { id = dto.Id }, dto);
    }

    [HttpPut("{id}")]
    public IActionResult Update(int id, ProductUpdateDto updateDto)
    {
        var product = _repository.GetById(id);
        
        if (product == null)
            return NotFound();
        
        // Mapping automatique avec mise à jour de l'existant
        _mapper.Map(updateDto, product);
        
        _repository.Update(product);
        
        return NoContent();
    }
}
```



---

## 2. Le Versioning d'API : Ne cassez pas Internet

**Le problème :** Votre API V1 est utilisée par 50 applications mobiles. Vous devez maintenant ajouter un champ
obligatoire ou changer la structure d'un objet. Si vous déployez la modification, les 50 applications cessent de
fonctionner. C'est inacceptable.

**La solution :** Le Versioning. Vous devez permettre aux anciens clients de continuer à utiliser l'ancienne version de
l'API, tout en permettant aux nouveaux clients d'utiliser la nouvelle.

Il existe plusieurs stratégies de versioning :

* **Dans l'URL (le plus courant) :** `/api/v1/products` vs `/api/v2/products`.
* **Dans la Query String :** `/api/products?api-version=2.0`.
* **Dans un en-tête HTTP :** `Accept: application/json;v=2.0`.

La stratégie par URL est la plus simple à mettre en place et à comprendre.

**Comment faire avec ASP.NET Core :**

1. **Installer le paquet NuGet :** `Microsoft.AspNetCore.Mvc.Versioning.ApiExplorer`.
2. **Configurer dans `Program.cs` :**
   ```c#
   builder.Services.AddApiVersioning(options =>
   {
       options.AssumeDefaultVersionWhenUnspecified = true;
       options.DefaultApiVersion = new ApiVersion(1, 0);
       options.ReportApiVersions = true;
   });
   ```
3. **Appliquer les versions aux contrôleurs :**

   ```c#
   // Controllers/v1/ProductsController.cs
   [ApiVersion("1.0")]
   [Route("api/v{version:apiVersion}/[controller]")]
   [ApiController]
   public class ProductsControllerV1 : ControllerBase { /* ... */ }

   // Controllers/v2/ProductsController.cs
   [ApiVersion("2.0")]
   [Route("api/v{version:apiVersion}/[controller]")]
   [ApiController]
   public class ProductsControllerV2 : ControllerBase { /* Logique de la v2 ... */ }
   ```

Vous pouvez maintenant faire évoluer votre API en toute sécurité.

---

## 3. Pagination : Gérez les grandes collections

**Le problème :** Votre endpoint `GET /api/products` retourne 10 000 produits. C'est énorme. La requête est lente, la
charge sur le serveur et la base de données est immense, et le client reçoit une quantité de données qu'il ne peut
probablement pas traiter.

**La solution :** La Pagination. Au lieu de tout retourner, vous retournez les résultats par "pages". Le client demande
une page spécifique et vous ne lui envoyez que cette tranche de données.

Une requête paginée ressemble à ça : `GET /api/products?pageNumber=2&pageSize=50`. (Donne-moi la deuxième page, avec 50
produits par page).

**Comment l'implémenter :**

```c#
[HttpGet]
public ActionResult<IEnumerable<ProductDto>> GetProducts(
    [FromQuery] int pageNumber = 1, [FromQuery] int pageSize = 10)
{
    var products = _repository.GetAll()
        .Skip((pageNumber - 1) * pageSize) // Saute les produits des pages précédentes
        .Take(pageSize) // Prend le bon nombre de produits pour la page actuelle
        .ToList();
    
    // Mapper en DTOs...
    
    return Ok(productDtos);
}
```

<warning>

**Important :** Il faut également retourner des informations sur la pagination dans la réponse (nombre total d'éléments, nombre total de pages, etc.) pour que le client puisse construire son interface de pagination. On le fait souvent via un en-tête HTTP personnalisé (`X-Pagination`) ou en enveloppant la réponse dans un objet plus complexe.

</warning>

#### Exercice 4 : Pagination simple

Modifiez votre action `GetProducts` pour qu'elle accepte des paramètres de pagination `pageNumber` et `pageSize` et
retourne uniquement la tranche de données correspondante.

##### Correction exercice 4 {collapsible='true'}

L'implémentation est exactement celle montrée dans l'exemple ci-dessus. Le point clé est d'appliquer `.Skip()` **avant**
`.Take()`. Il est aussi crucial de noter que ces opérations doivent être appliquées sur un `IQueryable` (avant le
`.ToList()`) pour que la pagination soit faite en SQL par la base de données, et non en mémoire dans votre application,
ce qui serait très inefficace.

```c#
public class EfProductRepository : IProductRepository
{
    // ...
    // La méthode GetAll doit retourner IQueryable pour permettre
    // le chaînage des opérations LINQ.
    public IQueryable<Product> GetAllQueryable()
    {
        return _context.Products.AsNoTracking();
    }
    // ...
}

// Dans le contrôleur
[HttpGet]
public ActionResult<IEnumerable<ProductDto>> GetProducts(
    [FromQuery] int pageNumber = 1, [FromQuery] int pageSize = 10)
{
    // On récupère le IQueryable, on n'a pas encore touché à la BDD.
    var query = _repository.GetAllQueryable();
    
    // On construit la requête de pagination.
    var pagedProducts = query
        .OrderBy(p => p.Name) // La pagination nécessite un ordre !
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToList(); // La requête SQL est exécutée ici.
    
    // ... mapper et retourner ...
    return Ok(mappedProducts);
}
```


#### Retour des informations de pagination : Deux approches

Voici deux méthodes pour retourner les métadonnées de pagination dans une API REST ASP.NET Core.

---

##### Approche avec un DTO de réponse (Wrapper)

Cette approche encapsule les données et les métadonnées dans un objet unique.

```c#
// DTO pour encapsuler les données paginées
public class PagedResult<T>
{
    public List<T> Data { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalRecords { get; set; }
    public int TotalPages { get; set; }
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;
}

// Dans votre controller
[HttpGet]
public ActionResult<PagedResult<ProductDto>> GetProducts(
    [FromQuery] int pageNumber = 1, 
    [FromQuery] int pageSize = 10)
{
    var totalRecords = _repository.GetAll().Count();
    
    var products = _repository.GetAll()
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToList();
    
    var productDtos = _mapper.Map<List<ProductDto>>(products);
    
    var result = new PagedResult<ProductDto>
    {
        Data = productDtos,
        PageNumber = pageNumber,
        PageSize = pageSize,
        TotalRecords = totalRecords,
        TotalPages = (int)Math.Ceiling(totalRecords / (double)pageSize)
    };
    
    return Ok(result);
}
```

**Exemple de réponse JSON**

```json
{
  "data": [
    { "id": 11, "name": "Product 11", "price": 99.99 },
    { "id": 12, "name": "Product 12", "price": 149.99 }
  ],
  "pageNumber": 2,
  "pageSize": 10,
  "totalRecords": 10000,
  "totalPages": 1000,
  "hasPreviousPage": true,
  "hasNextPage": true
}
```

**Avantages**
- Simplicité pour le client (tout est dans le body JSON)
- Facile à documenter dans Swagger
- Pas de problèmes CORS
- Parsing immédiat côté JavaScript

**Inconvénients**
- Moins conforme aux standards REST purs
- Structure de réponse plus complexe

---

##### Approche avec un en-tête HTTP personnalisé

Cette approche garde les données dans le body et place les métadonnées dans un header HTTP.



```c#
// Classe pour les métadonnées de pagination
public class PaginationMetadata
{
    public int CurrentPage { get; set; }
    public int PageSize { get; set; }
    public int TotalRecords { get; set; }
    public int TotalPages { get; set; }
}

// Dans votre controller
[HttpGet]
public ActionResult<IEnumerable<ProductDto>> GetProducts(
    [FromQuery] int pageNumber = 1, 
    [FromQuery] int pageSize = 10)
{
    var totalRecords = _repository.GetAll().Count();
    
    var products = _repository.GetAll()
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToList();
    
    var productDtos = _mapper.Map<List<ProductDto>>(products);
    
    // Créer les métadonnées de pagination
    var metadata = new PaginationMetadata
    {
        CurrentPage = pageNumber,
        PageSize = pageSize,
        TotalRecords = totalRecords,
        TotalPages = (int)Math.Ceiling(totalRecords / (double)pageSize)
    };
    
    // Ajouter l'en-tête personnalisé
    Response.Headers.Append("X-Pagination", 
        JsonSerializer.Serialize(metadata));
    
    // Permettre au client d'accéder à cet en-tête (CORS)
    Response.Headers.Append("Access-Control-Expose-Headers", "X-Pagination");
    
    return Ok(productDtos);
}
```

**Exemple de réponse**

**Headers:**
```
X-Pagination: {"currentPage":2,"pageSize":10,"totalRecords":10000,"totalPages":1000}
Access-Control-Expose-Headers: X-Pagination
```

**Body (JSON):**
```json
[
  { "id": 11, "name": "Product 11", "price": 99.99 },
  { "id": 12, "name": "Product 12", "price": 149.99 }
]
```

###### Exemple de consommation côté client (JavaScript)

```javascript
fetch('/api/products?pageNumber=2&pageSize=10')
  .then(response => {
    // Récupérer les métadonnées depuis le header
    const paginationHeader = response.headers.get('X-Pagination');
    const pagination = JSON.parse(paginationHeader);
    
    console.log(`Page ${pagination.currentPage} sur ${pagination.totalPages}`);
    
    return response.json();
  })
  .then(products => {
    console.log(products);
  });
```

**Avantages**
- Plus conforme aux bonnes pratiques REST
- Séparation claire entre données et métadonnées
- Body JSON plus simple et direct

**Inconvénients**
- Nécessite configuration CORS (`Access-Control-Expose-Headers`)
- Moins visible dans la documentation API
- Parsing un peu plus complexe côté client

---

##### Tableau comparatif

| Critère                | DTO Wrapper            | En-tête HTTP                                           |
|------------------------|------------------------|--------------------------------------------------------|
| **Simplicité client**  | ✅ Plus facile à parser | ⚠️ Nécessite d'accéder aux headers                     |
| **Standards REST**     | ⚠️ Moins standard      | ✅ Plus proche des bonnes pratiques REST                |
| **Support JavaScript** | ✅ Immédiat             | ⚠️ Nécessite `Access-Control-Expose-Headers` pour CORS |
| **Documentation API**  | ✅ Visible dans Swagger | ⚠️ Moins visible                                       |
| **Clarté du code**     | ⚠️ Structure imbriquée | ✅ Séparation nette                                     |

---


##### Bonus : Approche hybride

Vous pouvez combiner les deux approches :
- Retourner les métadonnées dans le body (pour la simplicité)
- Ajouter également un header `Link` pour les URLs de navigation (standard RFC 5988)

```c#
// Ajouter des liens de navigation
Response.Headers.Append("Link", 
    $"</api/products?pageNumber=1&pageSize=10>; rel=\"first\", " +
    $"</api/products?pageNumber={pageNumber - 1}&pageSize=10>; rel=\"prev\", " +
    $"</api/products?pageNumber={pageNumber + 1}&pageSize=10>; rel=\"next\", " +
    $"</api/products?pageNumber={totalPages}&pageSize=10>; rel=\"last\"");
```

Cette approche offre le meilleur des deux mondes !


---

## 4. Swagger/OpenAPI : La documentation automatique de l'API

**Swagger (OpenAPI)** est une spécification pour décrire des APIs REST de manière standardisée. Les outils Swashbuckle (inclus dans le template `webapi` d'ASP.NET Core) lisent votre code (contrôleurs, attributs, DTOs) et génèrent automatiquement :

1. Un fichier `swagger.json` qui décrit votre API de manière standardisée
2. Une **interface utilisateur HTML interactive** (Swagger UI) qui vous permet de :
    - Voir tous vos endpoints documentés
    - Comprendre les paramètres requis
    - Tester directement les endpoints depuis le navigateur
    - Voir les modèles de données (DTOs)

C'est un outil de documentation et de test inestimable qui transforme votre API en un produit professionnel.

### Configuration de base

#### Étape 1 : Installation (déjà inclus dans le template webapi)

Si vous n'avez pas créé votre projet avec le template webapi, installez le package :

```bash
dotnet add package Swashbuckle.AspNetCore
```

#### Étape 2 : Configuration dans Program.cs

```c#
// Program.cs

var builder = WebApplication.CreateBuilder(args);

// ══════════════════════════════════════════════════════════
// CONFIGURATION DES SERVICES
// ══════════════════════════════════════════════════════════

builder.Services.AddControllers();

// Configuration de Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    // Informations générales sur l'API
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Version = "v1",
        Title = "Products API",
        Description = "Une API ASP.NET Core pour gérer des produits",
        Contact = new OpenApiContact
        {
            Name = "Votre Nom",
            Email = "votre.email@example.com",
            Url = new Uri("https://votresite.com")
        },
        License = new OpenApiLicense
        {
            Name = "MIT License",
            Url = new Uri("https://opensource.org/licenses/MIT")
        }
    });

    // Activer les commentaires XML pour enrichir la documentation
    var xmlFilename = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFilename);
    options.IncludeXmlComments(xmlPath);
});

var app = builder.Build();

// ══════════════════════════════════════════════════════════
// CONFIGURATION DU PIPELINE HTTP
// ══════════════════════════════════════════════════════════

// Activer Swagger en développement ET en production
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "Products API v1");
        options.RoutePrefix = string.Empty; // Swagger UI à la racine (http://localhost:5000/)
    });
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### Étape 3 : Activer la génération des commentaires XML

Modifiez votre fichier `.csproj` pour générer automatiquement le fichier XML de documentation :

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    
    <!-- Génération du fichier XML pour Swagger -->
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn> <!-- Ignorer les warnings pour les membres non documentés -->
  </PropertyGroup>

</Project>
```

### Documenter vos endpoints avec des commentaires XML

Ajoutez des commentaires XML au-dessus de vos actions pour enrichir la documentation Swagger :

```c#
/// <summary>
/// Récupère tous les produits avec pagination
/// </summary>
/// <param name="pageNumber">Numéro de la page (défaut: 1)</param>
/// <param name="pageSize">Taille de la page (défaut: 10)</param>
/// <returns>Une liste paginée de produits</returns>
/// <response code="200">Retourne la liste des produits</response>
[HttpGet]
[ProducesResponseType(typeof(PagedResult<ProductDto>), StatusCodes.Status200OK)]
public ActionResult<PagedResult<ProductDto>> GetProducts(
    [FromQuery] int pageNumber = 1, 
    [FromQuery] int pageSize = 10)
{
    // ... implémentation
}

/// <summary>
/// Récupère un produit spécifique par son ID
/// </summary>
/// <param name="id">L'identifiant unique du produit</param>
/// <returns>Le produit demandé</returns>
/// <response code="200">Retourne le produit</response>
/// <response code="404">Si le produit n'existe pas</response>
[HttpGet("{id}")]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public ActionResult<ProductDto> GetById(int id)
{
    // ... implémentation
}

/// <summary>
/// Crée un nouveau produit
/// </summary>
/// <param name="createDto">Les données du produit à créer</param>
/// <returns>Le produit créé</returns>
/// <response code="201">Retourne le produit nouvellement créé</response>
/// <response code="400">Si les données sont invalides</response>
[HttpPost]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public ActionResult<ProductDto> Create([FromBody] ProductCreateDto createDto)
{
    // ... implémentation
}

/// <summary>
/// Met à jour un produit existant
/// </summary>
/// <param name="id">L'identifiant du produit à modifier</param>
/// <param name="updateDto">Les nouvelles données du produit</param>
/// <response code="204">Mise à jour réussie</response>
/// <response code="404">Si le produit n'existe pas</response>
/// <response code="400">Si les données sont invalides</response>
[HttpPut("{id}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public IActionResult Update(int id, [FromBody] ProductUpdateDto updateDto)
{
    // ... implémentation
}

/// <summary>
/// Supprime un produit
/// </summary>
/// <param name="id">L'identifiant du produit à supprimer</param>
/// <response code="204">Suppression réussie</response>
/// <response code="404">Si le produit n'existe pas</response>
[HttpDelete("{id}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public IActionResult Delete(int id)
{
    // ... implémentation
}
```

### Documenter vos DTOs

N'oubliez pas de documenter également vos classes DTO :

```c#
/// <summary>
/// Représente un produit retourné par l'API
/// </summary>
public class ProductDto
{
    /// <summary>
    /// Identifiant unique du produit
    /// </summary>
    /// <example>1</example>
    public int Id { get; set; }
    
    /// <summary>
    /// Nom du produit
    /// </summary>
    /// <example>Laptop Dell XPS 15</example>
    public string Name { get; set; }
    
    /// <summary>
    /// Prix du produit en euros
    /// </summary>
    /// <example>1299.99</example>
    public decimal Price { get; set; }
    
    /// <summary>
    /// Nom de la catégorie du produit
    /// </summary>
    /// <example>Électronique</example>
    public string CategoryName { get; set; }
}

/// <summary>
/// Données requises pour créer un nouveau produit
/// </summary>
public class ProductCreateDto
{
    /// <summary>
    /// Nom du produit
    /// </summary>
    /// <example>MacBook Pro 16"</example>
    [Required(ErrorMessage = "Le nom est obligatoire")]
    [StringLength(100, ErrorMessage = "Le nom ne peut pas dépasser 100 caractères")]
    public string Name { get; set; }
    
    /// <summary>
    /// Prix du produit (doit être positif)
    /// </summary>
    /// <example>2499.99</example>
    [Required]
    [Range(0, 999999.99, ErrorMessage = "Le prix doit être entre 0 et 999999.99")]
    public decimal Price { get; set; }
    
    /// <summary>
    /// ID de la catégorie du produit
    /// </summary>
    /// <example>3</example>
    [Required]
    public int CategoryId { get; set; }
}
```

### Utiliser Swagger UI

Une fois votre application lancée, accédez à Swagger UI :

```
https://localhost:5001/
```

Dans Swagger UI, vous pouvez :

1. **Explorer tous les endpoints** : Voir la liste complète de vos routes
2. **Voir les détails** : Paramètres requis, types de retour, codes de statut
3. **Tester en direct** : Cliquer sur "Try it out" pour exécuter des requêtes
4. **Voir les schémas** : Examiner la structure des DTOs
5. **Copier les requêtes cURL** : Pour les utiliser en ligne de commande

### Exemple de test dans Swagger UI

**Étapes pour tester un endpoint POST :**

1. Cliquez sur `POST /api/products`
2. Cliquez sur "Try it out"
3. Modifiez le JSON dans la zone de texte :
   ```json
   {
     "name": "iPhone 15 Pro",
     "price": 1199.99,
     "categoryId": 1
   }
   ```
4. Cliquez sur "Execute"
5. Voyez la réponse avec le code de statut, les headers, et le body

### Configuration avancée : Authentification JWT dans Swagger

Si votre API utilise l'authentification JWT, configurez Swagger pour l'accepter :

```c#
builder.Services.AddSwaggerGen(options =>
{
    // ... configuration de base ...
    
    // Définir le schéma de sécurité
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header en utilisant le schéma Bearer. " +
                      "Entrez 'Bearer' [espace] puis votre token. " +
                      "Exemple: 'Bearer eyJhbGc...'",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

Cela ajoute un bouton "Authorize" dans Swagger UI où vous pouvez entrer votre JWT token.

### Bonnes pratiques

1. **Documentez tout** : Chaque endpoint, paramètre et DTO
2. **Utilisez des exemples** : La balise `<example>` aide les utilisateurs
3. **Spécifiez les codes de retour** : Avec `[ProducesResponseType]`
4. **Versionnez votre documentation** : Un SwaggerDoc par version d'API
5. **Gardez Swagger à jour** : La doc doit refléter le code actuel

### Pourquoi Swagger est indispensable

- **Documentation vivante** : Se met à jour automatiquement avec le code
- **Environnement de test** : Testez sans Postman
- **Communication d'équipe** : Les frontend devs savent exactement quoi appeler
- **Standard universel** : OpenAPI est compris partout (compatible avec de nombreux outils)
- **Professionnel** : Montre que vous prenez votre API au sérieux


---

### Conclusion

Excellent travail ! Vous avez exploré les concepts qui élèvent une API du statut de "prototype" à celui de "produit
professionnel". Vous savez maintenant comment la structurer proprement avec des **DTOs**, la faire évoluer sans douleur
avec le **versioning**, la rendre performante avec la **pagination**, et fournir une documentation de qualité avec *
*Swagger**.

Ces pratiques sont essentielles pour travailler en équipe et pour fournir un service fiable à vos consommateurs. Une API
bien conçue réduit les coûts de maintenance, facilite l'intégration pour vos partenaires et renforce votre réputation en
tant que développeur compétent.

Nous arrivons au terme de notre parcours de construction. Dans le dernier module, il sera temps de prendre notre
application, qu'il s'agisse du site MVC ou de l'API, et de la mettre au monde en la déployant sur un serveur.