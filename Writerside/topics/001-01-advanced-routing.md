# Module 2 : Routage et Liaison de Données Avancés - Pour aller plus loin

### Objectifs Pédagogiques

À la fin de ce module complémentaire, vous serez capable de :

* **Appliquer** des contraintes de route pour des URLs plus robustes.
* **Générer** des URLs de manière dynamique et sécurisée dans votre code.
* **Utiliser** des types de retour d'action fortement typés avec `ActionResult<T>`.
* **Contrôler** précisément la source des données avec les attributs de liaison de modèle (Model Binding).
* **Mettre en place** la validation côté client pour une meilleure expérience utilisateur.
* **Créer** vos propres règles de validation personnalisées.

### Introduction : Devenir l'architecte du flux d'informations

Dans la partie précédente, vous avez appris à guider les requêtes des utilisateurs vers les bons services de votre
ville-application. C'était le travail du postier : suivre une adresse standard pour livrer le courrier.

Maintenant, vous allez devenir l'architecte de la circulation. Vous allez définir des règles de circulation
spécifiques ("cette rue est réservée aux entiers !"), installer des panneaux indicateurs intelligents qui s'adaptent au
trafic (génération d'URL), et mettre en place des postes de contrôle plus sophistiqués pour inspecter les "cargaisons" (
liaison de modèle avancée) avant même qu'elles n'arrivent à destination. Ces techniques sont essentielles pour
construire des applications complexes, et surtout des APIs RESTful robustes.

---

### 1. Le Routage de Précision

#### Les Contraintes de Route : Le Videur à l'entrée

**Le problème :** Votre route par défaut est `/{controller}/{action}/{id?}`. Si un utilisateur navigue vers
`/Products/Details/hello`, votre action `Details(int id)` va planter car "hello" ne peut pas être converti en `int`.
Comment refuser cette requête avant même qu'elle n'atteigne l'action ?

**La solution :** Les contraintes de route. Pensez-y comme un videur à l'entrée d'une rue. Il vérifie l'identité du
véhicule avant de le laisser passer.

Vous pouvez ajouter ces contraintes directement dans le pattern de la route.

* **Avec le routage par convention (`Program.cs`) :**

  ```c#
  app.MapControllerRoute(
      name: "default",
      pattern: "{controller=Home}/{action=Index}/{id:int?}"); // id doit être un entier
  ```

* **Avec le routage par attributs (sur le contrôleur/action) :** C'est encore plus précis.

  ```c#
  [HttpGet("details/{id:int}")] // L'id DOIT être un entier
  public IActionResult Details(int id) { /* ... */ }

  [HttpGet("user/{username:alpha:minlength(3)}")] // Le username ne doit contenir
                                                  // que des lettres et faire
                                                  // au moins 3 caractères
  public IActionResult Profile(string username) { /* ... */ }
  ```

Avec ces contraintes, une URL comme `/Products/Details/hello` ne correspondra plus à la route et générera directement
une erreur 404. C'est plus propre et plus sécurisé.

#### Génération d'URLs : Ne jamais coder une URL en dur

**Le problème :** Dans votre code, vous devez créer un lien vers une autre page. Tenteriez-vous d'écrire
`string url = "/Products/Details/" + product.Id;` ? C'est une très mauvaise idée ! Si vous changez votre structure de
routage plus tard, tous ces liens codés en dur seront cassés.

**La solution :** Les helpers de génération d'URL. Demandez à ASP.NET Core de construire l'URL pour vous en lui donnant
le *nom* de la destination (le contrôleur et l'action). C'est comme demander à votre GPS "Génère-moi l'itinéraire vers
la maison" au lieu de mémoriser l'adresse.

Dans un contrôleur, vous pouvez utiliser `Url.Action()` ou `RedirectToAction()`. Dans une vue Razor, vous avez déjà
utilisé les Tag Helpers comme `asp-action` qui font exactement cela.

```c#
// Dans une action de contrôleur
public IActionResult ProcessSomething()
{
    int newProductId = 42;
    
    // Mauvais : coder l'URL en dur
    // return Redirect("/Products/Details/42");

    // Bon : laisser le système de routage générer l'URL
    return RedirectToAction("Details", "Products", new { id = newProductId });
}
```

---

### 2. Des Résultats d'Actions Spécifiques avec `ActionResult<T>`

**Le problème :** `IActionResult` est flexible, mais il ne dit rien sur le type de données que votre action est censée
retourner en cas de succès. Pour une application MVC, ce n'est pas très grave, mais pour une API, c'est un problème. Les
outils de documentation automatique (comme Swagger/OpenAPI) ne peuvent pas deviner ce que vous retournez.

**La solution :** `ActionResult<T>`. C'est un type de retour qui dit : "Je suis un résultat d'action, et si tout se
passe bien, je retournerai un objet de type `T`".

Pensez à `IActionResult` comme une commande "Apportez-moi quelque chose à manger". `ActionResult<Product>` est une
commande "Apportez-moi le plat du jour, qui est un `Product`". C'est plus précis pour le cuisinier et pour vous.

```c#
// Ancien code (fonctionnel mais moins descriptif)
public IActionResult GetProduct(int id)
{
    var product = _products.FirstOrDefault(p => p.Id == id);
    if (product == null)
    {
        return NotFound();
    }
    return Ok(product); // Ok() retourne un IActionResult
}

// Nouveau code avec ActionResult<T> (pratique moderne)
public ActionResult<Product> GetProduct(int id)
{
    var product = _products.FirstOrDefault(p => p.Id == id);
    if (product == null)
    {
        return NotFound(); // Le framework gère la conversion
    }
    return product; // On peut retourner l'objet directement !
                    // Le framework le convertira en Ok(product)
}
```

<tip>

L'utilisation de `ActionResult<T>` est devenue la norme pour le développement d'API avec ASP.NET Core car elle améliore la lisibilité, la maintenabilité et l'intégration avec les outils de génération de documentation.

</tip>

#### Exercice 3 : Moderniser une action

Reprenez l'action `Details(int id)` du TP sur les produits et modifiez sa signature et son code pour utiliser
`ActionResult<Product>`.

##### Correction exercice 3 {collapsible='true'}

```c#
// Controllers/ProductsController.cs

// La signature de la méthode change
public ActionResult<Product> Details(int id)
{
    var product = _products.FirstOrDefault(p => p.Id == id);
    if (product == null)
    {
        return NotFound(); // NotFound() est toujours un type de résultat valide
    }

    // Ici, au lieu de 'return View(product)', qui est spécifique à MVC,
    // on retourne le produit.
    // Pour une API, ce serait parfait. Pour une vue, il faut
    // toujours retourner View().
    // L'avantage ici est purement pour la clarté de la signature.
    return View(product); 
}
```

<warning>

Dans une application MVC pure qui retourne toujours des vues, l'intérêt de `ActionResult<T>` est moindre. `IActionResult` reste parfaitement valable. Cependant, `ActionResult<T>` devient indispensable dans le contexte des APIs, où l'on retourne directement des données (JSON).

</warning>

---

### 3. Le Model Binding Démystifié : Les Sources de Liaison

**Le problème :** Parfois, le Model Binding automatique ne trouve pas les données. Par exemple, vous envoyez des données
JSON dans le corps (body) d'une requête, mais votre paramètre d'action reste `null`. Comment dire explicitement au Model
Binder *où* chercher ?

**La solution :** Les attributs de source de liaison. Vous les placez devant les paramètres de votre action pour
dire : "Toi, `id`, tu viens de la route. Toi, `searchQuery`, tu viens de la query string. Et toi, `product`, tu viens du
corps de la requête."

C'est comme donner des instructions précises à un coursier : "Le premier colis est sur le palier (`FromRoute`), le
deuxième est dans la boîte aux lettres (`FromQuery`), et pour le troisième, il faut sonner et je vous le donnerai en
main propre (`FromBody`)."

Les attributs les plus courants :

* `[FromRoute]` : Lie à partir des données de la route (ex: `{id}`).
* `[FromQuery]` : Lie à partir de la query string (ex: `?name=...`).
* `[FromForm]` : Lie à partir des champs d'un formulaire envoyé.
* `[FromBody]` : Lie à partir du corps de la requête (souvent du JSON).
* `[FromHeader]` : Lie à partir d'un en-tête HTTP.

```c#
[HttpPost("api/products/{id}/update")]
public IActionResult UpdateProduct(
    [FromRoute] int id, 
    [FromQuery] bool notifyUsers, 
    [FromBody] ProductUpdateModel model)
{
    // id viendra de l'URL (ex: /api/products/5/update)
    // notifyUsers viendra de la query string (ex: ?notifyUsers=true)
    // model sera rempli à partir du JSON envoyé dans le corps de la requête
    
    // ...
    return Ok();
}
```

<warning>
Attention : une action ne peut avoir **qu'un seul** paramètre décoré avec `[FromBody]`. En effet, le corps de la requête ne peut être lu qu'une seule fois.
</warning>

---

### 4. Aller plus loin avec la Validation

#### Validation Côté Client

**Le problème :** Actuellement, pour voir les erreurs de validation, l'utilisateur doit soumettre le formulaire,
attendre la réponse du serveur, puis voir la page se recharger avec les erreurs. C'est lent et frustrant.

**La solution :** La validation côté client. ASP.NET Core facilite grandement cela. Les Data Annotations que vous avez
mises sur votre modèle (`[Required]`, `[StringLength]`, etc.) peuvent être automatiquement transformées en attributs
HTML `data-*`.

```html
<input asp-for="Name" class="form-control"/>
```

... sera rendu en HTML comme :

```html
<input class="form-control" type="text" data-val="true"
       data-val-required="Le nom du produit est obligatoire."
       id="Name" name="Name" value="">
```

Il suffit ensuite d'inclure les scripts de validation jQuery Unobtrusive dans votre page (le template par défaut le fait
déjà via `_ValidationScriptsPartial.cshtml`). Ces scripts lisent les attributs `data-val` et effectuent la validation
directement dans le navigateur, donnant un retour immédiat à l'utilisateur.

<tip>
La validation côté client est une **amélioration de l'expérience utilisateur**, pas une mesure de sécurité. Un utilisateur malveillant peut la contourner. Vous devez **TOUJOURS** re-valider les données côté serveur avec `ModelState.IsValid`.
</tip>

#### Attributs de Validation Personnalisés

**Le problème :** Les validateurs intégrés sont excellents, mais que faire si vous avez une règle métier complexe ? Par
exemple, "La date de début de l'événement doit être dans le futur". Il n'y a pas d'attribut pour ça.

**La solution :** Créez le vôtre ! C'est comme écrire une nouvelle loi dans le code de la route de votre application. Il
suffit de créer une classe qui hérite de `ValidationAttribute` et de surcharger la méthode `IsValid`.

**Exemple : Un validateur pour s'assurer qu'une date est dans le futur.**

1. Créez un dossier `ValidationAttributes` à la racine du projet.
2. Créez un fichier `FutureDateAttribute.cs`.

   ```c#
   using System;
   using System.ComponentModel.DataAnnotations;

   namespace MonAppMvc.ValidationAttributes
   {
       public class FutureDateAttribute : ValidationAttribute
       {
           public override bool IsValid(object value)
           {
               // On s'assure qu'on a bien une DateTime
               if (value is DateTime dateValue)
               {
                   // La règle métier : la date doit être supérieure à maintenant
                   return dateValue > DateTime.Now;
               }
               // Si ce n'est pas une date, on considère que c'est valide
               // pour ne pas interférer avec l'attribut [Required].
               return true; 
           }
       }
   }
   ```
3. Utilisez votre nouvel attribut sur un modèle !

   ```c#
   using MonAppMvc.ValidationAttributes;

   public class EventCreateModel
   {
       [Required]
       public string Name { get; set; }

       [Required]
       [FutureDate(ErrorMessage = "La date de l'événement doit être dans le futur.")]
       public DateTime EventDate { get; set; }
   }
   ```

Et voilà ! Vous avez créé une règle de validation réutilisable et parfaitement adaptée à votre métier.

#### Exercice 4 : Validateur personnalisé

Créez un attribut de validation personnalisé `[AllowedDomain("gmail.com", "outlook.com")]` qui vérifie qu'une chaîne de
caractères (un email) se termine par l'un des domaines autorisés.

##### Correction exercice 4 {collapsible='true'}

**`ValidationAttributes/AllowedDomainAttribute.cs`**

```c#
using System.ComponentModel.DataAnnotations;
using System.Linq;

namespace MonAppMvc.ValidationAttributes
{
    public class AllowedDomainAttribute : ValidationAttribute
    {
        private readonly string[] _allowedDomains;

        // Le constructeur accepte les domaines autorisés
        public AllowedDomainAttribute(params string[] allowedDomains)
        {
            _allowedDomains = allowedDomains;
        }

        public override bool IsValid(object value)
        {
            if (value is string email)
            {
                // On extrait le domaine de l'email
                string domain = email.Split('@').LastOrDefault();

                if (domain != null && _allowedDomains.Contains(domain.ToLower()))
                {
                    return true; // Le domaine est autorisé
                }
            }
            
            // Si la valeur n'est pas un string, ou si le domaine n'est pas
            // dans la liste, la validation échoue.
            return false; 
        }
    }
}
```

**Utilisation dans un modèle :**

```c#
public class RegisterViewModel
{
    [Required]
    [EmailAddress]
    [AllowedDomain("gmail.com", "outlook.com", "yahoo.com", 
        ErrorMessage = "Seuls les emails gmail, outlook ou yahoo sont autorisés.")]
    public string Email { get; set; }
}
```

---

### Conclusion

Félicitations ! Vous avez maintenant une compréhension bien plus profonde et nuancée du fonctionnement interne du
routage, des contrôleurs et de la gestion des données. Vous ne vous contentez plus d'utiliser les mécanismes par défaut,
vous êtes capable de les modeler pour qu'ils répondent précisément à vos besoins.

Savoir utiliser les **contraintes de route**, générer des **URLs dynamiquement**, choisir le bon **type de retour
d'action**, spécifier les **sources de liaison** et créer des **règles de validation personnalisées** sont des
compétences qui distinguent un développeur junior d'un développeur confirmé. Elles sont la clé pour bâtir des
applications non seulement fonctionnelles, mais aussi robustes, sécurisées et maintenables.

Dans le prochain module, nous aborderons la façade de notre application : la **Vue**. Nous verrons comment utiliser la
puissance de Razor pour transformer nos données en interfaces web élégantes et interactives.