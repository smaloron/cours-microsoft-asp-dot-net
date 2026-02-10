Parfait ! Entamons ce sixième module. Nous avons passé beaucoup de temps à construire une magnifique application web,
conçue pour être utilisée par des humains via un navigateur. Mais le monde moderne ne se limite pas à ça. Les
applications mobiles, les services d'autres entreprises, les objets connectés... tous ont besoin de communiquer avec nos
systèmes.

Aujourd'hui, nous allons apprendre à parler leur langue. Nous allons ouvrir notre application au monde en créant une API
RESTful. C'est le passage obligé pour construire des applications qui s'intègrent dans un écosystème plus large, la
compétence clé pour le développement backend moderne.

---

# Module 6 : Construire et Sécuriser une API RESTful - L'essentiel

### Objectifs Pédagogiques

À la fin de ce module, vous serez capable de :

* **Comprendre** les principes fondamentaux d'une API RESTful (ressources, verbes HTTP, codes de statut).
* **Créer** un projet d'API Web avec ASP.NET Core.
* **Développer** des contrôleurs d'API qui gèrent les opérations CRUD.
* **Utiliser** les `ActionResult<T>` pour retourner des réponses typées et des codes de statut HTTP appropriés.
* **Différencier** l'Authentification et l'Autorisation.
* **Mettre en place** un système d'authentification et d'autorisation simple pour sécuriser vos points de terminaison (
  endpoints).

### Introduction : Le service au volant de votre restaurant

Imaginez que votre application MVC est un restaurant traditionnel. Les clients (utilisateurs) entrent, s'assoient,
consultent un menu joliment présenté (les vues HTML), et un serveur (le contrôleur) leur apporte des plats complets.

Une **API RESTful** est le service au volant (drive-in) de ce même restaurant. Le client ne rentre pas. Il passe sa
commande (la requête HTTP) de manière très structurée ("Je veux un produit avec l'ID 5"). La cuisine (votre logique
métier) prépare la commande, mais au lieu de la servir dans une belle assiette, elle la met dans un emballage
standardisé et facile à transporter (le format **JSON**). Le client récupère les données brutes et fait ce qu'il veut
avec : les manger dans sa voiture (une application mobile), les ramener à la maison (un autre service web), etc.

Construire une API, c'est donc exposer la logique et les données de votre application de manière brute et universelle,
pour qu'elle puisse être consommée par d'autres programmes, et pas seulement par des navigateurs web.

---

### 1. Les Principes d'une API RESTful

REST (Representational State Transfer) n'est pas un protocole, mais un style d'architecture. Il repose sur quelques
idées simples pour rendre les APIs prévisibles et faciles à utiliser.

1. **Centrée sur les Ressources :** Tout est une "ressource". Un produit, un utilisateur, une commande... Chaque
   ressource est identifiée par une URL unique (un endpoint).
    * `/api/products` : La collection de tous les produits.
    * `/api/products/5` : La ressource "produit" avec l'ID 5.

2. **Utilisation des Verbes HTTP :** On utilise les verbes HTTP standards pour dire ce qu'on veut faire avec la
   ressource.
    * `GET` : Lire la ressource.
    * `POST` : Créer une nouvelle ressource.
    * `PUT` : Mettre à jour (remplacer) entièrement une ressource.
    * `DELETE` : Supprimer une ressource.

3. **Utilisation des Codes de Statut HTTP :** La réponse de l'API doit inclure un code de statut standard pour indiquer
   si l'opération a réussi et comment.
    * `200 OK` : Succès (pour un GET).
    * `201 Created` : La ressource a été créée avec succès (pour un POST).
    * `204 No Content` : Succès, mais il n'y a rien à renvoyer (pour un DELETE).
    * `400 Bad Request` : La requête du client était mal formée (ex: validation échouée).
    * `404 Not Found` : La ressource demandée n'existe pas.
    * `500 Internal Server Error` : Une erreur s'est produite sur le serveur.

4. **Communication sans état (Stateless) :** Chaque requête du client doit contenir toutes les informations nécessaires
   pour que le serveur la comprenne. Le serveur ne doit pas avoir besoin de se souvenir des requêtes précédentes.

---

### 2. Créer une API avec ASP.NET Core

Créer une API est très similaire à créer une application MVC. La principale différence réside dans le contrôleur et ce
qu'il retourne.

<procedure title="Création d'un projet API">
<step>
    <p>Utilisez le template CLI approprié :</p>

    ```bash
    dotnet new webapi -o MaSuperApi
    cd MaSuperApi
    ```

</step>
<step>
    <p>Observez les différences :</p>
    <ul>
        <li>Pas de dossiers <code>Views</code> ou <code>wwwroot</code>. Une API n'a pas d'interface utilisateur.</li>
        <li>Les contrôleurs héritent de <code>ControllerBase</code> au lieu de <code>Controller</code> (<code>Controller</code> ajoute juste le support pour les Vues).</li>
        <li>Les contrôleurs sont souvent décorés avec l'attribut <code>[ApiController]</code>.</li>
    </ul>
</step>
</procedure>

#### L'attribut `[ApiController]`

Cet attribut magique active plusieurs comportements qui facilitent la vie du développeur d'API :

* Il active le routage par attributs par défaut.
* Il déclenche automatiquement la validation du modèle (`ModelState.IsValid`) et renvoie une réponse `400 Bad Request`
  si la validation échoue, sans que vous ayez à écrire le `if`.
* Il infère la source des paramètres (ex: `[FromBody]`) pour plus de clarté.

#### Exemple : Un `ProductsApiController`

```c#
[ApiController]
[Route("api/[controller]")] // Route de base : /api/products
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;

    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }

    // GET: /api/products
    [HttpGet]
    public ActionResult<IEnumerable<Product>> GetProducts()
    {
        return Ok(_repository.GetAll()); // Retourne 200 OK + la liste des produits
    }

    // GET: /api/products/5
    [HttpGet("{id}")]
    public ActionResult<Product> GetProduct(int id)
    {
        var product = _repository.GetById(id);
        if (product == null)
        {
            return NotFound(); // Retourne 404 Not Found
        }
        return Ok(product); // Retourne 200 OK + le produit
    }

    // POST: /api/products
    [HttpPost]
    public ActionResult<Product> CreateProduct(ProductCreateModel model)
    {
        // La validation est automatique grâce à [ApiController]

        var newProduct = new Product { /* ... map from model ... */ };
        _repository.Add(newProduct);
        
        // Retourne 201 Created, l'URL de la nouvelle ressource,
        // et la ressource elle-même.
        return CreatedAtAction(nameof(GetProduct), 
                               new { id = newProduct.Id }, 
                               newProduct);
    }
}
```

---

### 3. Sécurité des APIs : Qui êtes-vous et que pouvez-vous faire ?

Une API publique est une porte ouverte sur vos données. Il est crucial de la sécuriser. La sécurité se divise en deux
concepts :

<tabs>
<tab title="Authentification (AuthN)">
    <strong>La question :</strong> "Qui êtes-vous ?"
    <br/>
    <strong>Le processus :</strong> Le client doit prouver son identité. La méthode la plus courante est de fournir un nom d'utilisateur et un mot de passe. Si c'est correct, le serveur lui donne une "carte d'identité" temporaire.
</tab>
<tab title="Autorisation (AuthZ)">
    <strong>La question :</strong> "Avez-vous le droit de faire ça ?"
    <br/>
    <strong>Le processus :</strong> Une fois le client authentifié, le serveur vérifie si sa carte d'identité lui donne les permissions nécessaires pour accéder à une ressource spécifique. Par exemple, "Seuls les administrateurs peuvent supprimer un produit".
</tab>
</tabs>

#### Comment ça marche pour une API ? Les Tokens JWT

Dans une application MVC, l'authentification est souvent gérée par des **cookies**. Le serveur envoie un cookie au
navigateur après la connexion, et le navigateur le renvoie à chaque requête.

Cela ne fonctionne pas bien pour les APIs, car les clients ne sont pas toujours des navigateurs. La solution standard
est le **JSON Web Token (JWT)**.

Pensez à un laissez-passer pour un festival :

1. **Authentification :** Vous allez au guichet (`/api/auth/login`) et montrez votre billet (login/mot de passe).
2. **Génération du Token :** Le guichetier vérifie votre billet et vous donne un bracelet (`JWT Token`). Ce bracelet
   contient des informations sur vous (votre ID, si vous êtes VIP/Admin, et une date d'expiration) et il est scellé avec
   une signature secrète que seul le festival connaît.
3. **Requêtes suivantes :** Pour accéder à n'importe quelle scène (endpoint), vous n'avez plus besoin de montrer votre
   billet, juste votre bracelet. Le garde à l'entrée (`Middleware d'authentification`) vérifie que le sceau n'est pas
   cassé et que le bracelet est valide.

Le client stocke ce token et l'envoie à chaque requête dans l'en-tête HTTP `Authorization` :
`Authorization: Bearer <votre_long_token_jwt>`.

#### Mise en place (simplifiée)

La mise en place complète est un peu longue, mais voici les grandes étapes :

1. **Installer les paquets NuGet :** `Microsoft.AspNetCore.Authentication.JwtBearer`.
2. **Configurer dans `Program.cs` :**
   ```c#
   builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
       .AddJwtBearer(options =>
       {
           options.TokenValidationParameters = new TokenValidationParameters
           {
               // Configurez ici comment valider le token
               // (qui l'a signé, pour qui il est, etc.)
           };
       });
   builder.Services.AddAuthorization();
   ```
3. **Ajouter les middlewares (dans le bon ordre !) :**
   ```c#
   app.UseAuthentication(); // 1. Qui êtes-vous ?
   app.UseAuthorization();  // 2. Que pouvez-vous faire ?
   ```
4. **Utiliser les attributs `[Authorize]` et `[AllowAnonymous]` :**

   ```c#
   [ApiController]
   [Route("api/[controller]")]
   public class ProductsController : ControllerBase
   {
       // Tout le monde peut voir les produits.
       [HttpGet]
       [AllowAnonymous]
       public ActionResult<IEnumerable<Product>> GetProducts() { /* ... */ }

       // Seuls les utilisateurs authentifiés peuvent voir les détails.
       [HttpGet("{id}")]
       [Authorize] 
       public ActionResult<Product> GetProduct(int id) { /* ... */ }

       // Seuls les utilisateurs authentifiés ET qui ont le rôle "Admin"
       // peuvent supprimer un produit.
       [HttpDelete("{id}")]
       [Authorize(Roles = "Admin")]
       public IActionResult DeleteProduct(int id) { /* ... */ }
   }
   ```

#### Exercice 3 : Sécuriser un endpoint

En supposant que l'authentification est configurée, comment modifieriez-vous l'action `CreateProduct` (POST) pour
qu'elle ne soit accessible qu'aux utilisateurs authentifiés ayant le rôle "Editor" ou "Admin" ?

##### Correction exercice 3 {collapsible='true'}

Il suffit d'ajouter l'attribut `[Authorize]` avec la propriété `Roles`. Vous pouvez spécifier plusieurs rôles en les
séparant par une virgule.

```c#
// POST: /api/products
[HttpPost]
[Authorize(Roles = "Admin,Editor")]
public ActionResult<Product> CreateProduct(ProductCreateModel model)
{
    // ...
}
```

---

### TP : Créer une API pour notre gestionnaire de produits

Nous allons exposer les fonctionnalités de notre gestionnaire de produits via une API RESTful.

<procedure>

<step title="Étape 1 : Créer un nouveau projet d'API Web">

<p>Créez un projet `dotnet new webapi` nommé `ProductApi`. Copiez-y les dossiers <code>Models</code>, <code>Interfaces</code>, <code>Services</code>, <code>Data</code> et le <code>appsettings.json</code> de votre projet MVC pour réutiliser toute la logique d'accès aux données avec EF Core.</p>

</step>
<step title="Étape 2 : Configurer les services">
<p>Dans le <code>Program.cs</code> de l'API, configurez le <code>DbContext</code> et l'injection de dépendances pour votre <code>IProductRepository</code>, exactement comme dans le projet MVC.</p>
</step>

<step title="Étape 3 : Créer le `ProductsController`">

<p>Créez un contrôleur d'API <code>ProductsController</code>. Injectez <code>IProductRepository</code>. Implémentez les cinq actions RESTful de base :</p>
<ul>
    <li><code>GET /api/products</code> : Récupère tous les produits.</li>
    <li><code>GET /api/products/{id}</code> : Récupère un produit par son ID.</li>
    <li><code>POST /api/products</code> : Crée un nouveau produit.</li>
    <li><code>PUT /api/products/{id}</code> : Met à jour un produit existant.</li>
    <li><code>DELETE /api/products/{id}</code> : Supprime un produit.</li>
</ul>
<p>Utilisez les verbes HTTP appropriés (`[HttpGet]`, `[HttpPost]`, etc.) et retournez les codes de statut corrects.</p>
</step>
<step title="Étape 4 : Tester l'API">
<p>Lancez votre API. Le template `webapi` inclut par défaut <strong>Swagger/OpenAPI</strong>, qui vous donne une interface web pour tester vos endpoints directement depuis votre navigateur. Utilisez cette interface pour créer, lister, mettre à jour et supprimer des produits.</p>
</step>
</procedure>

#### Correction TP : Créer une API pour notre gestionnaire de produits {collapsible='true'}

**`Controllers/ProductsController.cs`**

```c#
using Microsoft.AspNetCore.Mvc;
using MonAppMvc.Interfaces;
using MonAppMvc.Models;

[Route("api/[controller]")]
[ApiController]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;

    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }

    [HttpGet]
    public ActionResult<IEnumerable<Product>> GetProducts()
    {
        return Ok(_repository.GetAll());
    }

    [HttpGet("{id}")]
    public ActionResult<Product> GetProduct(int id)
    {
        var product = _repository.GetById(id);
        if (product == null) return NotFound();
        return Ok(product);
    }

    [HttpPost]
    public ActionResult<Product> PostProduct(Product product)
    {
        // Dans une vraie app, on utiliserait un DTO/ViewModel
        _repository.Add(product);
        return CreatedAtAction(nameof(GetProduct), 
                               new { id = product.Id }, product);
    }

    [HttpPut("{id}")]
    public IActionResult PutProduct(int id, Product product)
    {
        if (id != product.Id)
        {
            return BadRequest();
        }
        _repository.Update(product);
        return NoContent(); // Code 204
    }

    [HttpDelete("{id}")]
    public IActionResult DeleteProduct(int id)
    {
        var product = _repository.GetById(id);
        if (product == null)
        {
            return NotFound();
        }
        _repository.Delete(id);
        return NoContent(); // Code 204
    }
}
```

---

### Auto-évaluation

1. Quel verbe HTTP est utilisé pour mettre à jour une ressource existante ?

- a) `GET`
- b) `POST`
- c) `PUT`
- d) `UPDATE`

2. Quel code de statut HTTP indique qu'une ressource a été créée avec succès ?

- a) `200 OK`
- b) `201 Created`
- c) `204 No Content`
- d) `400 Bad Request`

3. Quelle est la différence entre l'Authentification et l'Autorisation ?

- a) Il n'y en a pas, c'est la même chose.
- b) L'authentification vérifie qui vous êtes, l'autorisation vérifie ce que vous pouvez faire.
- c) L'autorisation vérifie qui vous êtes, l'authentification vérifie ce que vous pouvez faire.
- d) L'authentification est pour les APIs, l'autorisation pour le MVC.

4. Quel attribut place-t-on sur une action pour exiger que l'utilisateur soit authentifié ?

- a) `[ApiController]`
- b) `[Authenticate]`
- c) `[AllowAnonymous]`
- d) `[Authorize]`

5. Qu'est-ce qu'une ressource dans le contexte de REST ? Donnez un exemple.
6. Expliquez le flux d'authentification basé sur les tokens JWT en 3 étapes.
7. Pourquoi l'attribut `[ApiController]` est-il si utile pour développer des APIs ? Citez deux de ses avantages.

---

### Conclusion

Félicitations ! Vous avez ajouté une corde essentielle à votre arc de développeur backend. Vous ne savez plus seulement
construire des applications pour les humains, mais aussi pour les machines. Vous comprenez les principes de
l'architecture **REST**, vous savez comment les implémenter avec ASP.NET Core pour créer des **contrôleurs d'API**
propres et efficaces, et vous avez fait vos premiers pas dans le monde crucial de la **sécurité des APIs**.

Cette compétence est fondamentale dans l'écosystème technologique actuel, où les applications sont de plus en plus
interconnectées. C'est la base pour construire des applications mobiles, des architectures microservices, et bien plus
encore.

Dans le dernier module, nous aborderons la dernière étape du cycle de vie d'une application : comment l'empaqueter et la
déployer pour qu'elle soit accessible au monde entier.