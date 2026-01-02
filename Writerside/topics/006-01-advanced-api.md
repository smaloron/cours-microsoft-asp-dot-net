Parfaitement ! Maintenant que vous savez construire et sécuriser une API, il est temps d'apprendre les techniques
avancées qui feront la différence entre une API fonctionnelle et une API de qualité professionnelle.

Dans la partie essentielle, nous avons mis en place le service au volant de notre restaurant. Dans cette partie, nous
allons optimiser le processus de commande, créer un menu détaillé pour nos clients programmeurs, et apprendre à gérer
les commandes qui évoluent dans le temps. Ce sont les détails qui rendent une API agréable à utiliser, performante et
pérenne.

---

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

### 1. Les DTOs (Data Transfer Objects) : L'emballage sur mesure

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

<tip>
Le mapping manuel peut devenir fastidieux. Des librairies comme **AutoMapper** peuvent automatiser ce processus. C'est un outil indispensable dans les projets de grande envergure.
</tip>

---

### 2. Le Versioning d'API : Ne cassez pas Internet

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

### 3. Pagination : Gérez les grandes collections

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

---

### 4. Swagger et Postman : Les outils indispensables du développeur d'API

* **Swagger (OpenAPI) :** C'est une spécification pour décrire des APIs REST. Les outils Swashbuckle (inclus dans le
  template `webapi`) lisent votre code (contrôleurs, attributs, DTOs) et génèrent automatiquement :
    1. Un fichier `swagger.json` qui décrit votre API de manière standardisée.
    2. Une magnifique **interface utilisateur HTML interactive** qui vous permet de voir tous vos endpoints et de les
       tester directement depuis le navigateur. C'est un outil de documentation et de test inestimable.

* **Postman :** C'est un client HTTP avancé. C'est comme un navigateur surpuissant pour les développeurs d'API. Il vous
  permet de :
    * Forger n'importe quel type de requête HTTP (`GET`, `POST`, `PUT`...).
    * Ajouter des en-têtes personnalisés (comme l'en-tête `Authorization` pour les tokens JWT).
    * Envoyer des données dans le corps de la requête (JSON, form-data...).
    * Sauvegarder et organiser vos requêtes dans des collections.
    * Écrire des scripts de test.

<tip>
Apprendre à utiliser Postman est une compétence non négociable pour un développeur backend. C'est votre principal outil pour interagir avec et déboguer vos APIs.
</tip>

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