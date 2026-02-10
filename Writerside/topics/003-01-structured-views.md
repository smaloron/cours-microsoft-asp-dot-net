# Module 3 : Vues Structurées et Composants Avancés - Pour aller plus loin

### Objectifs Pédagogiques

À la fin de ce module complémentaire, vous serez capable de :

* **Maîtriser** la gestion des sections Razor pour une personnalisation fine des layouts.
* **Utiliser** `ViewData` et `ViewBag` et comprendre leurs limites par rapport aux modèles fortement typés.
* **Créer** et **utiliser** des View Components pour encapsuler une logique complexe dans un composant de vue
  réutilisable.
* **Différencier** une Vue Partielle d'un View Component et savoir quand utiliser chacun.
* **Organiser** vos fichiers de vues de manière logique et maintenable.

### Introduction : De l'artisan à l'ingénieur

Dans la partie essentielle, vous avez appris les techniques de l'artisan : utiliser des gabarits (layouts) et des pièces
préfabriquées (vues partielles) pour construire vos interfaces plus rapidement. C'est très bien pour des constructions
simples.

Maintenant, vous allez adopter la mentalité de l'ingénieur. Comment construire un gratte-ciel ? Vous ne pouvez pas vous
contenter de simples gabarits. Vous avez besoin de systèmes complexes et autonomes : un système de climatisation, un
système d'ascenseurs... Chacun a sa propre logique, fonctionne de manière indépendante, mais s'intègre parfaitement dans
la structure globale. Les **View Components** sont ces systèmes. Apprendre à les utiliser, ainsi que les autres
techniques de cette section, vous permettra de construire des applications web vastes et complexes tout en gardant un
code propre, organisé et facile à maintenir.

---

### 1. Personnalisation avancée des Layouts avec les Sections

**Le problème :** Votre `_Layout.cshtml` charge les fichiers JavaScript globaux à la fin du fichier. Mais une de vos
pages, et une seule, a besoin d'un fichier JavaScript supplémentaire (par exemple, une librairie de graphiques). Comment
l'ajouter depuis la page spécifique sans modifier le layout ?

**La solution :** Les sections Razor. Une section est un "emplacement réservé" que vous définissez dans votre layout, et
que n'importe quelle vue peut décider de remplir. C'est comme si votre cadre photo (`_Layout`) avait un petit
emplacement optionnel pour une étiquette de description. Seules les photos qui en ont besoin l'utiliseront.

**Étape 1 : Définir la section dans le Layout**
Dans `_Layout.cshtml`, à l'endroit où vous voulez que le contenu optionnel apparaisse, ajoutez `@RenderSection()`.

```html
<!-- Views/Shared/_Layout.cshtml -->
...
</footer>

<script src="~/js/site.js"></script>

@* On déclare une section nommée "Scripts".
Le deuxième paramètre 'required: false' signifie que les vues
ne sont pas obligées de la définir. *@
@await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

**Étape 2 : Remplir la section dans la Vue**
Dans la vue qui a besoin du script supplémentaire (ex: `Views/Dashboard/Index.cshtml`), utilisez la directive
`@section`.

```html
@* Views/Dashboard/Index.cshtml *@
@{
ViewData["Title"] = "Tableau de Bord";
}

<h1>Tableau de Bord</h1>
<div id="chart"></div>

@section Scripts {
<script src="https://cdn.jsdelivr.net/npm/apexcharts"></script>
<script>
    // Votre code JS pour initialiser le graphique
    // qui utilise la librairie apexcharts
</script>
}
```

Au moment du rendu, le contenu de ce bloc `@section Scripts` sera injecté précisément à l'endroit du
`@RenderSectionAsync("Scripts")` dans le layout. C'est une méthode très propre pour gérer les dépendances spécifiques à
une page.

---

### 2. `ViewData` vs `ViewBag` : Les anciens messagers

Avant l'avènement des modèles fortement typés, c'étaient les seules façons de passer des données du contrôleur à la vue.
Vous devez les connaître car vous les rencontrerez dans du code existant, mais leur usage est aujourd'hui **déconseillé
** au profit des ViewModels.

Imaginez que vous voulez envoyer un message.

* **ViewModel (fortement typé) :** Vous écrivez une lettre structurée, avec des champs clairs (destinataire, objet,
  contenu). Le facteur sait exactement ce qu'il transporte. C'est sûr et sans erreur.
* **`ViewData` :** Vous prenez une boîte et vous y mettez des objets en leur collant une étiquette (`string`).
  `ViewData["TitrePage"] = "Mon Titre";`. Pour récupérer l'objet, il faut connaître l'étiquette et souvent le "caster" (
  convertir) dans le bon type. C'est un dictionnaire.
* **`ViewBag` :** Vous prenez un sac magique (`dynamic`). Vous y mettez des objets en leur donnant un nom.
  `ViewBag.TitrePage = "Mon Titre";`. C'est plus simple à écrire, mais encore plus risqué. `ViewBag` est une surcouche
  dynamique de `ViewData`.

**Le problème avec `ViewData` et `ViewBag` :**

* **Pas de vérification à la compilation :** Si vous faites une faute de frappe (`ViewBag.TitrePge` au lieu de
  `ViewBag.TitrePage`), vous ne le saurez qu'à l'exécution, quand votre page plantera.
* **Pas d'IntelliSense :** Votre éditeur de code ne peut pas vous aider à retrouver les noms des propriétés.
* **Nécessite des "magic strings" :** Vous devez utiliser des chaînes de caractères qui doivent être identiques dans le
  contrôleur et la vue.

**Quand les utiliser (avec parcimonie) ?**
Pour des petites données qui ne font pas partie du modèle principal de la page, comme le titre de la page (
`ViewData["Title"]`), ou un message de statut temporaire.

**Exemple :**

```c#
// Contrôleur
public IActionResult Index()
{
    ViewData["WelcomeMessage"] = "Bienvenue sur notre site !";
    ViewBag.ItemsInCart = 5;
    return View();
}

// Vue
<h1>@ViewData["WelcomeMessage"]</h1>
<p>Vous avez @ViewBag.ItemsInCart articles dans votre panier.</p>
```

<warning>

**Bonne pratique :** Pour toutes les données principales nécessaires à votre vue, utilisez toujours un **ViewModel fortement typé**. Réservez `ViewData`/`ViewBag` pour des cas très limités et non critiques.

</warning>

---

### 3. Les View Components : Les Widgets Intelligents

**Le problème :** Une vue partielle est super pour réutiliser du HTML. Mais que faire si le morceau de code que vous
voulez réutiliser a besoin de sa propre logique ? Par exemple, un panier d'achat affiché dans le header. Pour savoir
combien d'articles il contient, il doit interroger une base de données ou une session. Mettre cette logique dans le
`_Layout.cshtml` ou dans chaque contrôleur serait une très mauvaise pratique.

**La solution :** Le View Component. C'est l'évolution de la vue partielle. Un View Component est un mini-contrôleur +
une mini-vue, totalement autonome.

Un View Component est idéal pour des éléments comme :

* Un panier d'achat
* Un nuage de tags
* Un menu de navigation dynamique
* Une liste des "derniers articles" dans une barre latérale

**Comment créer un View Component ?**

1. **La Classe :** Créez un dossier `ViewComponents` à la racine. Créez une classe qui hérite de `ViewComponent`. Son
   nom doit se terminer par `ViewComponent`. Elle doit avoir une méthode `InvokeAsync`.

   ```c#
   // ViewComponents/ShoppingCartViewComponent.cs
   using Microsoft.AspNetCore.Mvc;
   using System.Threading.Tasks;
   
   // Simule un service qui gère le panier
   public class ShoppingCartService 
   {
       public int GetItemCount() => 5; // Logique métier ici
   }

   public class ShoppingCartViewComponent : ViewComponent
   {
       private readonly ShoppingCartService _cartService;

       public ShoppingCartViewComponent()
       {
           // Dans une vraie app, ce service serait injecté
           _cartService = new ShoppingCartService();
       }

       public async Task<IViewComponentResult> InvokeAsync()
       {
           int itemCount = _cartService.GetItemCount();
           return View(itemCount); // On passe le nombre d'articles à la vue
       }
   }
   ```
2. **La Vue :** Créez une vue pour le composant. Elle doit se trouver dans
   `Views/Shared/Components/{NomDuComposant}/Default.cshtml`.

   **`Views/Shared/Components/ShoppingCart/Default.cshtml`**
   ```html
   @model int

   <div class="shopping-cart">
       Panier (@Model articles)
   </div>
   ```

3. **L'Appel :** Dans n'importe quelle vue (par exemple, le `_Layout`), appelez le composant.

   ```html
   @* Dans _Layout.cshtml *@
   @await Component.InvokeAsync("ShoppingCart")
   ```

#### Vue Partielle vs. View Component

| Caractéristique | Vue Partielle | View Component | Analogie |
| :--- | :--- | :--- | :--- |
| **Logique** | Aucune. Reçoit ses données de la vue parente. | Possède sa propre logique (peut appeler des services, BDD...). | Un simple morceau de LEGO. |
| **Autonomie** | Totalement dépendante. | Totalement autonome. | Un moteur LEGO complet. |
| **Quand l'utiliser ?** | Pour réutiliser un simple bloc de HTML statique ou qui affiche des données déjà disponibles. | Pour des composants réutilisables qui ont besoin de leurs propres données et de leur propre logique. |
| **Exemple** | Une carte de produit. | Un panier d'achat. |

#### Exercice 2 : Créer un View Component "Derniers Articles"

Pour le projet de blog, créez un View Component `LatestPostsViewComponent` qui affiche les titres des 2 derniers
articles sous forme de liste `<ul>`.

1. Créez la classe du View Component. Elle devra accéder à la liste statique des articles du `BlogController`.
2. Créez la vue du composant.
3. Appelez ce composant depuis la barre latérale de votre `_Layout.cshtml`.

##### Correction exercice 2 {collapsible='true'}

1. **Classe `LatestPostsViewComponent.cs` (dans `ViewComponents/`)**

   ```c#
   using Microsoft.AspNetCore.Mvc;
   using MonBlog.Controllers; // Pour accéder à la liste statique
   using System.Linq;
   using System.Threading.Tasks;

   namespace MonBlog.ViewComponents
   {
       public class LatestPostsViewComponent : ViewComponent
       {
           public async Task<IViewComponentResult> InvokeAsync(int count = 2)
           {
               // On récupère tous les posts
               var allPosts = BlogController.GetPosts();

               // On trie par date et on prend les 'count' plus récents
               var latestPosts = allPosts
                   .OrderByDescending(p => p.PublicationDate)
                   .Take(count)
                   .ToList();
               
               // On passe cette liste filtrée à notre vue
               return View(latestPosts);
           }
       }
   }
   ```
2. **Vue `Default.cshtml` (dans `Views/Shared/Components/LatestPosts/`)**

   ```html
   @model IEnumerable<MonBlog.Models.BlogPost>

   <h4>Derniers Articles</h4>
   <ul>
       @foreach (var post in Model)
       {
           <li>
               <a asp-controller="Blog" asp-action="Details" asp-route-id="@post.Id">
                   @post.Title
               </a>
           </li>
       }
   </ul>
   ```
3. **Appel dans `_Layout.cshtml`**
   Vous pouvez ajouter une barre latérale simple dans votre layout pour le placer.

   ```html
   <div class="container">
       <div class="row">
           <div class="col-md-9">
               <main role="main" class="pb-3">
                   @RenderBody()
               </main>
           </div>
           <aside class="col-md-3">
               @* Voici l'appel à notre composant, on peut lui passer un paramètre ! *@
               @await Component.InvokeAsync("LatestPosts", new { count = 3 })
           </aside>
       </div>
   </div>
   ```

---

### Conclusion

Vous avez maintenant ajouté des outils d'ingénierie logicielle à vos compétences de développement d'interface. Vous
savez comment structurer vos vues de manière encore plus modulaire avec les **sections**, vous comprenez l'historique et
l'usage (limité) de `ViewData` et `ViewBag`, et surtout, vous avez découvert la puissance des **View Components** pour
créer des morceaux d'interface réutilisables et autonomes.

La distinction entre une Vue Partielle et un View Component est fondamentale pour concevoir des applications ASP.NET
Core complexes et maintenables. En choisissant le bon outil pour chaque situation, vous écrirez un code plus propre,
plus découplé et beaucoup plus facile à faire évoluer.

Dans le prochain module, nous allons nous attaquer à un sujet central mais souvent redouté : la gestion des services et
l'injection de dépendances. C'est le mécanisme qui permet à toutes les parties de votre application de collaborer de
manière élégante et efficace, sans être "soudées" les unes aux autres.