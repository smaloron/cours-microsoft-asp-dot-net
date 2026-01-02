# Correction du Projet "Mini-Lien"

### Étape 0 : Création du Projet et Installation des Dépendances

Ouvrez un terminal et exécutez les commandes suivantes.

1.  **Créer le projet MVC :**
    ```bash
    dotnet new mvc -o MiniLien
    cd MiniLien
    ```

2.  **Installer les paquets NuGet nécessaires :**
    Nous avons besoin d'Entity Framework Core et du fournisseur pour SQLite, qui est une base de données fichier très simple à utiliser pour ce type de projet.

    ```bash
    dotnet add package Microsoft.EntityFrameworkCore.Sqlite
    dotnet add package Microsoft.EntityFrameworkCore.Tools
    ```

### Étape 1 : Création du Modèle (l'Entité)

Nous commençons par définir la structure de nos données. Créez un dossier `Entities` à la racine du projet.

<tabs>
<tab title="Entities/ShortenedUrl.cs">

```c#
using System.ComponentModel.DataAnnotations;
using Microsoft.EntityFrameworkCore;

namespace MiniLien.Entities
{
    // L'attribut Index garantit que la recherche par ShortCode sera rapide
    // et que la base de données imposera l'unicité.
    [Index(nameof(ShortCode), IsUnique = true)]
    public class ShortenedUrl
    {
        [Key] // Marque cette propriété comme la clé primaire
        public int Id { get; set; }
        
        [Required] // L'URL originale est obligatoire
        public string OriginalUrl { get; set; } = string.Empty;

        [Required]
        public string ShortCode { get; set; } = string.Empty;
        
        public DateTime CreatedAt { get; set; }
        
        public int ClickCount { get; set; } = 0;
    }
}
```

</tab>
</tabs>

### Étape 2 : Création du Contexte de Base de Données (DbContext)

Le `DbContext` est notre pont entre le code C# et la base de données. Créez un dossier `Data` à la racine.

<tabs>
<tab title="Data/ApplicationDbContext.cs">

```c#
using Microsoft.EntityFrameworkCore;
using MiniLien.Entities;

namespace MiniLien.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
        
        // Chaque DbSet correspond à une table dans notre base de données.
        public DbSet<ShortenedUrl> ShortenedUrls { get; set; }
    }
}
```

</tab>
</tabs>

### Étape 3 : Configuration dans `Program.cs`

C'est ici que nous assemblons toutes les pièces : configuration de la base de données, enregistrement des services et définition des routes.

<tabs>
<tab title="Program.cs">

```c#
using Microsoft.EntityFrameworkCore;
using MiniLien.Data;
using MiniLien.Services;

var builder = WebApplication.CreateBuilder(args);

// --- Configuration des Services ---

// 1. Enregistrement du DbContext pour Entity Framework Core
// On utilise SQLite et on récupère la chaîne de connexion depuis appsettings.json
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection"))
);

// 2. Enregistrement de notre service de génération de code court
// Singleton est approprié car le service n'a pas d'état.
builder.Services.AddSingleton<UrlShorteningService>();

// 3. Ajout des services MVC (Contrôleurs, Vues)
builder.Services.AddControllersWithViews();

var app = builder.Build();

// --- Configuration du Pipeline de Requêtes HTTP ---

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthorization();

// --- Configuration des Routes ---

// Route personnalisée pour la redirection.
// Elle intercepte les requêtes courtes comme "http://.../abcdef"
// et les envoie à l'action RedirectToUrl du RedirectController.
app.MapControllerRoute(
    name: "ShortUrlRedirect",
    pattern: "{shortCode}",
    defaults: new { controller = "Redirect", action = "RedirectToUrl" }
);

// Route MVC par défaut
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

</tab>
<tab title="appsettings.json">

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=minilien.db"
  }
}
```
*   **Note :** `Data Source=minilien.db` est la chaîne de connexion la plus simple pour SQLite. Elle créera un fichier `minilien.db` à la racine de votre projet.

</tab>
</tabs>

### Étape 4 : Création de la Base de Données via les Migrations

Ouvrez un terminal à la racine du projet et exécutez ces deux commandes.

1.  **Créer la migration :**
    ```bash
    dotnet ef migrations add InitialCreate
    ```
    Cette commande va inspecter votre `DbContext` et vos entités, et créer un fichier dans un nouveau dossier `Migrations` qui décrit comment créer la base de données.

2.  **Appliquer la migration :**
    ```bash
    dotnet ef database update
    ```
    Cette commande exécute la migration. Vous devriez maintenant voir un fichier `minilien.db` apparaître à la racine de votre projet.

### Étape 5 : Création du Service de Logique Métier

Pour garder notre contrôleur propre, nous externalisons la logique de génération de code dans un service. Créez un dossier `Services`.

<tabs>
<tab title="Services/UrlShorteningService.cs">

```c#
namespace MiniLien.Services
{
    public class UrlShorteningService
    {
        private const string Alphabet = 
            "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
        private readonly Random _random = new();

        public string GenerateShortCode()
        {
            // Génère un code de 6 caractères en piochant au hasard
            // dans notre alphabet.
            var codeChars = new char[6];
            for (int i = 0; i < 6; i++)
            {
                codeChars[i] = Alphabet[_random.Next(Alphabet.Length)];
            }
            return new string(codeChars);
        }
    }
}
```
*   **Note :** Pour ce projet simple, le service génère juste le code. La vérification de l'unicité sera faite dans le contrôleur, qui a accès au `DbContext`.

</tab>
</tabs>

### Étape 6 : Création du ViewModel

Le ViewModel est l'objet qui transporte les données entre notre formulaire (la Vue) et notre action (le Contrôleur). Créez un dossier `ViewModels`.

<tabs>
<tab title="ViewModels/ShortenUrlViewModel.cs">

```c#
using System.ComponentModel.DataAnnotations;

namespace MiniLien.ViewModels
{
    public class ShortenUrlViewModel
    {
        [Required(ErrorMessage = "Veuillez fournir une URL.")]
        [Url(ErrorMessage = "L'URL fournie n'est pas valide.")]
        [Display(Name = "URL à raccourcir")]
        public string Url { get; set; } = string.Empty;
    }
}
```

</tab>
</tabs>

### Étape 7 : Création des Contrôleurs

Nous aurons besoin de deux contrôleurs : un pour la page d'accueil et la logique de création, et un autre dédié à la redirection pour une meilleure séparation des responsabilités.

<tabs>
<tab title="Controllers/HomeController.cs">

```c#
using System.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MiniLien.Data;
using MiniLien.Entities;
using MiniLien.Services;
using MiniLien.ViewModels;

namespace MiniLien.Controllers
{
    public class HomeController : Controller
    {
        private readonly UrlShorteningService _urlShorteningService;
        private readonly ApplicationDbContext _context;

        public HomeController(
            UrlShorteningService urlShorteningService, 
            ApplicationDbContext context)
        {
            _urlShorteningService = urlShorteningService;
            _context = context;
        }

        [HttpGet]
        public IActionResult Index()
        {
            var model = new ShortenUrlViewModel();
            return View(model);
        }

        [HttpPost]
        public async Task<IActionResult> Index(ShortenUrlViewModel model)
        {
            if (!ModelState.IsValid)
            {
                return View(model);
            }

            string shortCode;
            bool isCodeUnique;

            // Boucle pour s'assurer que le code généré est bien unique
            do
            {
                shortCode = _urlShorteningService.GenerateShortCode();
                isCodeUnique = !await _context.ShortenedUrls
                    .AnyAsync(u => u.ShortCode == shortCode);
            } 
            while (!isCodeUnique);

            var shortenedUrl = new ShortenedUrl
            {
                OriginalUrl = model.Url,
                ShortCode = shortCode,
                CreatedAt = DateTime.UtcNow
            };

            _context.ShortenedUrls.Add(shortenedUrl);
            await _context.SaveChangesAsync();

            return RedirectToAction("Success", new { shortCode = shortCode });
        }

        [HttpGet]
        public async Task<IActionResult> Success(string shortCode)
        {
            var shortenedUrl = await _context.ShortenedUrls
                .FirstOrDefaultAsync(u => u.ShortCode == shortCode);

            if (shortenedUrl == null)
            {
                return NotFound();
            }
            
            // On reconstruit l'URL complète pour l'afficher à l'utilisateur
            var fullShortUrl = 
                $"{Request.Scheme}://{Request.Host}/{shortenedUrl.ShortCode}";
            
            ViewBag.FullShortUrl = fullShortUrl;
            return View(shortenedUrl);
        }
        
        [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, 
            NoStore = true)]
        public IActionResult Error()
        {
            return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? 
                HttpContext.TraceIdentifier });
        }
    }
}
```

</tab>
<tab title="Controllers/RedirectController.cs">

```c#
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MiniLien.Data;

namespace MiniLien.Controllers
{
    public class RedirectController : Controller
    {
        private readonly ApplicationDbContext _context;

        public RedirectController(ApplicationDbContext context)
        {
            _context = context;
        }

        [HttpGet]
        public async Task<IActionResult> RedirectToUrl(string shortCode)
        {
            var shortenedUrl = await _context.ShortenedUrls
                .FirstOrDefaultAsync(u => u.ShortCode == shortCode);

            if (shortenedUrl == null)
            {
                return NotFound();
            }

            // Incrémente le compteur de clics
            shortenedUrl.ClickCount++;
            await _context.SaveChangesAsync();
            
            // Effectue une redirection permanente (HTTP 301)
            return RedirectPermanent(shortenedUrl.OriginalUrl);
        }
    }
}
```

</tab>
</tabs>

### Étape 8 : Création des Vues

Enfin, nous créons l'interface utilisateur.

<tabs>
<tab title="Views/Shared/_Layout.cshtml">

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - MiniLien</title>
    <link rel="stylesheet" 
        href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm 
                    navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container-fluid">
                <a class="navbar-brand" asp-area="" asp-controller="Home" 
                    asp-action="Index">MiniLien</a>
            </div>
        </nav>
    </header>
    <div class="container">
        <main role="main" class="pb-3">
            @RenderBody()
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; 2024 - MiniLien
        </div>
    </footer>
    <script src="~/lib/jquery/dist/jquery.min.js"></script>
    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>
    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

</tab>
<tab title="Views/Home/Index.cshtml">

```html
@model MiniLien.ViewModels.ShortenUrlViewModel

@{
    ViewData["Title"] = "Accueil";
}

<div class="text-center">
    <h1 class="display-4">Mini-Lien</h1>
    <p class="lead">Un raccourcisseur d'URL simple et efficace.</p>
</div>

<div class="row justify-content-center mt-5">
    <div class="col-md-8">
        <form asp-controller="Home" asp-action="Index" method="post">
            <div class="input-group">
                <input asp-for="Url" class="form-control form-control-lg" 
                    placeholder="Collez votre URL longue ici..." />
                <button type="submit" class="btn btn-primary btn-lg">
                    Raccourcir
                </button>
            </div>
            <span asp-validation-for="Url" class="text-danger mt-2 d-block"></span>
        </form>
    </div>
</div>
```

</tab>
<tab title="Views/Home/Success.cshtml">

```html
@model MiniLien.Entities.ShortenedUrl

@{
    ViewData["Title"] = "Lien créé !";
    var fullShortUrl = ViewBag.FullShortUrl;
}

<div class="text-center">
    <h1 class="display-4 text-success">Votre lien a été raccourci !</h1>
    <p class="lead mt-4">
        Voici votre nouveau Mini-Lien. Vous pouvez le copier et le partager.
    </p>

    <div class="card mt-4">
        <div class="card-body">
            <h5 class="card-title">Votre lien :</h5>
            <div class="input-group">
                <input type="text" class="form-control" 
                    value="@fullShortUrl" id="shortUrlInput" readonly>
                <button class="btn btn-outline-secondary" type="button" 
                    onclick="copyToClipboard()">Copier</button>
            </div>
            <p class="card-text mt-3">
                <small class="text-muted">
                    Redirige vers : @Model.OriginalUrl
                </small>
            </p>
        </div>
    </div>
    
    <div class="mt-4">
        <a asp-controller="Home" asp-action="Index" class="btn btn-secondary">
            Raccourcir une autre URL
        </a>
    </div>
</div>

@section Scripts {
    <script>
        function copyToClipboard() {
            var copyText = document.getElementById("shortUrlInput");
            copyText.select();
            copyText.setSelectionRange(0, 99999); // Pour mobile
            document.execCommand("copy");
            alert("Lien copié : " + copyText.value);
        }
    </script>
}
```

</tab>
<tab title="Views/_ViewImports.cshtml">

```c#
@using MiniLien
@using MiniLien.Models
@using MiniLien.Entities
@using MiniLien.ViewModels
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```
*   **Note :** Ajoutez les `using` pour les entités et ViewModels pour qu'ils soient accessibles dans toutes vos vues.

</tab>
</tabs>

### Étape 9 : Lancer et Tester le Projet

Vous avez terminé ! Lancez le projet avec la commande :
```bash
dotnet run
```
Ouvrez votre navigateur et testez les fonctionnalités :
1.  Allez à la page d'accueil.
2.  Essayez de soumettre une URL invalide, vérifiez l'erreur.
3.  Soumettez une URL valide (ex: `https://fr.wikipedia.org/wiki/Uniform_Resource_Locator`).
4.  Vous devriez être redirigé vers la page de succès. Copiez le lien.
5.  Collez le lien raccourci dans une nouvelle barre d'adresse. Vous devriez être redirigé vers la page Wikipedia.
6.  (Bonus) Vous pouvez utiliser un outil comme "DB Browser for SQLite" pour ouvrir votre fichier `minilien.db` et vérifier que le `ClickCount` a bien été incrémenté.