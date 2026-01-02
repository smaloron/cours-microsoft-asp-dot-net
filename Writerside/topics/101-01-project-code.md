# Correction du Projet "KanbanFlow"

### Étape 0 : Création du Projet et Installation des Dépendances

Ouvrez un terminal et exécutez les commandes suivantes.

1. **Créer le projet MVC :**
   ```bash
   dotnet new mvc -o KanbanFlow
   cd KanbanFlow
   ```

2. **Installer les paquets NuGet nécessaires :**
   ```bash
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite
   dotnet add package Microsoft.EntityFrameworkCore.Tools
   ```

### Étape 1 : Création des Entités

Créez un dossier `Entities` à la racine du projet et ajoutez les trois classes suivantes.

<tabs>
<tab title="Entities/Board.cs">

```c#
using System.ComponentModel.DataAnnotations;

namespace KanbanFlow.Entities
{
    public class Board
    {
        [Key]
        public int Id { get; set; }

        [Required]
        [MaxLength(100)]
        public string Name { get; set; } = string.Empty;

        // Propriété de navigation : un tableau a une collection de colonnes
        public ICollection<Column> Columns { get; set; } = new List<Column>();
    }
}
```

</tab>
<tab title="Entities/Column.cs">

```c#
using System.ComponentModel.DataAnnotations;

namespace KanbanFlow.Entities
{
    public class Column
    {
        [Key]
        public int Id { get; set; }

        [Required]
        [MaxLength(100)]
        public string Name { get; set; } = string.Empty;

        // Pour ordonner les colonnes dans l'affichage
        public int Position { get; set; }
        
        // Clé étrangère et propriété de navigation vers Board
        public int BoardId { get; set; }
        public Board Board { get; set; } = null!;

        // Propriété de navigation : une colonne a une collection de cartes
        public ICollection<Card> Cards { get; set; } = new List<Card>();
    }
}
```

</tab>
<tab title="Entities/Card.cs">

```c#
using System.ComponentModel.DataAnnotations;

namespace KanbanFlow.Entities
{
    public class Card
    {
        [Key]
        public int Id { get; set; }

        [Required]
        [MaxLength(200)]
        public string Title { get; set; } = string.Empty;

        public string? Description { get; set; }
        
        // Pour ordonner les cartes dans une colonne
        public int Position { get; set; }
        
        // Clé étrangère et propriété de navigation vers Column
        public int ColumnId { get; set; }
        public Column Column { get; set; } = null!;
    }
}
```

</tab>
</tabs>

### Étape 2 : Création du DbContext

Créez un dossier `Data` et ajoutez-y le contexte.

<tabs>
<tab title="Data/ApplicationDbContext.cs">

```c#
using KanbanFlow.Entities;
using Microsoft.EntityFrameworkCore;

namespace KanbanFlow.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
        
        public DbSet<Board> Boards { get; set; }
        public DbSet<Column> Columns { get; set; }
        public DbSet<Card> Cards { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            
            // Configure explicitement les relations pour plus de clarté
            
            modelBuilder.Entity<Board>()
                .HasMany(b => b.Columns) // Un Board a plusieurs Columns
                .WithOne(c => c.Board)    // Une Column a un seul Board
                .HasForeignKey(c => c.BoardId); // La clé étrangère est BoardId

            modelBuilder.Entity<Column>()
                .HasMany(c => c.Cards)    // Une Column a plusieurs Cards
                .WithOne(ca => ca.Column) // Une Card a une seule Column
                .HasForeignKey(ca => ca.ColumnId); // La clé est ColumnId
        }
    }
}
```

</tab>
</tabs>

### Étape 3 : Configuration dans `Program.cs` et `appsettings.json`

<tabs>
<tab title="Program.cs">

```c#
using KanbanFlow.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllersWithViews();

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite(builder.Configuration
        .GetConnectionString("DefaultConnection"))
);

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Board}/{action=Index}/{id?}");

app.Run();
```

* **Note :** Nous avons modifié la route par défaut pour que l'application démarre directement sur la liste des
  tableaux.

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
    "DefaultConnection": "Data Source=kanbanflow.db"
  }
}
```

</tab>
</tabs>

### Étape 4 : Création de la Base de Données via les Migrations

Ouvrez un terminal à la racine du projet et exécutez ces deux commandes :

```bash
dotnet ef migrations add InitialSchema
dotnet ef database update
```

Un fichier `kanbanflow.db` devrait maintenant être présent à la racine de votre projet.

### Étape 5 : Gestion des Tableaux (User Story 1)

<tabs>
<tab title="Controllers/BoardController.cs">

```c#
using KanbanFlow.Data;
using KanbanFlow.Entities;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace KanbanFlow.Controllers
{
    public class BoardController : Controller
    {
        private readonly ApplicationDbContext _context;

        public BoardController(ApplicationDbContext context)
        {
            _context = context;
        }

        // GET: /Board ou /Board/Index
        [HttpGet]
        public async Task<IActionResult> Index()
        {
            var boards = await _context.Boards.OrderBy(b => b.Name).ToListAsync();
            return View(boards);
        }

        // POST: /Board/Create
        [HttpPost]
        public async Task<IActionResult> Create(string name)
        {
            if (!string.IsNullOrWhiteSpace(name))
            {
                var newBoard = new Board { Name = name };
                
                // Création des colonnes par défaut
                newBoard.Columns.Add(
                    new Column { Name = "À faire", Position = 1 });
                newBoard.Columns.Add(
                    new Column { Name = "En cours", Position = 2 });
                newBoard.Columns.Add(
                    new Column { Name = "Terminé", Position = 3 });
                
                _context.Boards.Add(newBoard);
                await _context.SaveChangesAsync();
            }
            return RedirectToAction("Index");
        }
        
        // La suite (Details) sera ajoutée à l'étape suivante
    }
}
```

</tab>
<tab title="Views/Board/Index.cshtml">

```html
@model IEnumerable
<KanbanFlow.Entities.Board>

    @{
    ViewData["Title"] = "Mes Tableaux";
    }

    <h1>@ViewData["Title"]</h1>
    <hr/>

    <div class="row">
        <div class="col-md-4">
            <h4>Créer un nouveau tableau</h4>
            <form asp-action="Create" method="post">
                <div class="input-group">
                    <input type="text" name="name" class="form-control"
                           placeholder="Nom du projet..." required/>
                    <button type="submit" class="btn btn-primary">Créer</button>
                </div>
            </form>
        </div>
    </div>

    <hr/>

    <h4>Tableaux existants</h4>
    <div class="list-group mt-3">
        @if (!Model.Any())
        {
        <p>Aucun tableau n'a encore été créé.</p>
        }
        else
        {
        @foreach (var board in Model)
        {
        <a asp-action="Details" asp-route-id="@board.Id"
           class="list-group-item list-group-item-action">
            @board.Name
        </a>
        }
        }
    </div>
```

</tab>
</tabs>

### Étape 6 : Vue Détaillée et Ajout de Cartes (User Story 2)

<tabs>
<tab title="Controllers/BoardController.cs (suite)">

```c#
// ... (code précédent)

// GET: /Board/Details/5
[HttpGet]
public async Task<IActionResult> Details(int id)
{
    var board = await _context.Boards
        .Include(b => b.Columns.OrderBy(c => c.Position))
            .ThenInclude(c => c.Cards.OrderBy(card => card.Position))
        .FirstOrDefaultAsync(b => b.Id == id);

    if (board == null)
    {
        return NotFound();
    }
    
    // Pour la logique des boutons de déplacement dans la vue
    ViewBag.MaxColumnPosition = board.Columns.Max(c => (int?)c.Position) ?? 0;
    
    return View(board);
}
```

</tab>
<tab title="Controllers/CardController.cs (nouveau fichier)">

```c#
using KanbanFlow.Data;
using KanbanFlow.Entities;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace KanbanFlow.Controllers
{
    public class CardController : Controller
    {
        private readonly ApplicationDbContext _context;

        public CardController(ApplicationDbContext context)
        {
            _context = context;
        }

        // POST: /Card/Create
        [HttpPost]
        public async Task<IActionResult> Create(Card newCard)
        {
            if (ModelState.IsValid)
            {
                // Calcule la position de la nouvelle carte
                var maxPosition = await _context.Cards
                    .Where(c => c.ColumnId == newCard.ColumnId)
                    .MaxAsync(c => (int?)c.Position) ?? 0;
                newCard.Position = maxPosition + 1;

                _context.Cards.Add(newCard);
                await _context.SaveChangesAsync();

                // On doit récupérer l'ID du tableau pour la redirection
                var column = await _context.Columns
                    .FindAsync(newCard.ColumnId);
                
                if (column != null)
                {
                    return RedirectToAction("Details", "Board", 
                        new { id = column.BoardId });
                }
            }
            // En cas d'erreur, rediriger vers la liste des tableaux
            return RedirectToAction("Index", "Board");
        }
    }
}
```

</tab>
<tab title="Views/Board/Details.cshtml (nouveau fichier)">

```html
@model KanbanFlow.Entities.Board

@{
ViewData["Title"] = $"Tableau : {Model.Name}";
}

<h1>@Model.Name</h1>

<div class="container-fluid mt-4">
    <div class="row flex-nowrap" style="overflow-x: auto;">
        @* Boucle sur chaque colonne *@
        @foreach (var column in Model.Columns)
        {
        <div class="col-3">
            <div class="card bg-light h-100">
                <div class="card-header">@column.Name</div>
                <div class="card-body">
                    @* Boucle sur chaque carte de la colonne *@
                    @foreach (var card in column.Cards)
                    {
                    <div class="card mb-2">
                        <div class="card-body">
                            <h6 class="card-title">@card.Title</h6>
                            <p class="card-text">
                                <small>@card.Description</small>
                            </p>
                            @* Formulaires pour le déplacement (Étape 7) *@
                        </div>
                    </div>
                    }
                </div>
                <div class="card-footer">
                    @* Formulaire pour ajouter une carte *@
                    <form asp-controller="Card" asp-action="Create"
                          method="post">
                        <input type="hidden" name="ColumnId"
                               value="@column.Id"/>
                        <input type="text" name="Title"
                               class="form-control mb-2"
                               placeholder="Titre de la carte..." required/>
                        <textarea name="Description"
                                  class="form-control mb-2"
                                  placeholder="Description..." rows="2"></textarea>
                        <button type="submit" class="btn btn-primary w-100">
                            Ajouter
                        </button>
                    </form>
                </div>
            </div>
        </div>
        }
    </div>
</div>
```

</tab>
</tabs>

### Étape 7 : Déplacement de Cartes (User Story 3)

<tabs>
<tab title="Controllers/CardController.cs (suite)">

```c#
// ... (code précédent)

// POST: /Card/Move
[HttpPost]
public async Task<IActionResult> Move(int id, string direction)
{
    var cardToMove = await _context.Cards
        .Include(c => c.Column)
        .FirstOrDefaultAsync(c => c.Id == id);
        
    if (cardToMove == null)
    {
        return NotFound();
    }
    
    var currentPosition = cardToMove.Column.Position;
    var targetPosition = direction == "right" 
        ? currentPosition + 1 
        : currentPosition - 1;
        
    var targetColumn = await _context.Columns.FirstOrDefaultAsync(c => 
        c.BoardId == cardToMove.Column.BoardId && c.Position == targetPosition);

    if (targetColumn != null)
    {
        cardToMove.ColumnId = targetColumn.Id;
        // Optionnel : remettre la carte en dernière position
        var maxPos = await _context.Cards
            .Where(c => c.ColumnId == targetColumn.Id)
            .MaxAsync(c => (int?)c.Position) ?? 0;
        cardToMove.Position = maxPos + 1;

        await _context.SaveChangesAsync();
    }
    
    return RedirectToAction("Details", "Board", 
        new { id = cardToMove.Column.BoardId });
}
```

</tab>
<tab title="Views/Board/Details.cshtml (mise à jour)">

Ajoutez ce bloc de code à l'intérieur de `<div class="card-body">` de la carte, juste après la description.

```html
@* ... après <p class="card-text"> ... *@

<div class="d-flex justify-content-between mt-2">
    <!-- Bouton pour aller à gauche -->
    <form asp-controller="Card" asp-action="Move" method="post">
        <input type="hidden" name="id" value="@card.Id"/>
        <input type="hidden" name="direction" value="left"/>
        <button type="submit" class="btn btn-sm btn-outline-secondary"
                disabled="@(column.Position == 1)">
            &lt;
        </button>
    </form>

    <!-- Bouton pour aller à droite -->
    <form asp-controller="Card" asp-action="Move" method="post">
        <input type="hidden" name="id" value="@card.Id"/>
        <input type="hidden" name="direction" value="right"/>
        <button type="submit" class="btn btn-sm btn-outline-secondary"
                disabled="@(column.Position >= ViewBag.MaxColumnPosition)">
            &gt;
        </button>
    </form>
</div>
```

</tab>
</tabs>

### Étape 8 : Finalisation des Vues et Lancement

<tabs>
<tab title="Views/Shared/_Layout.cshtml">

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>@ViewData["Title"] - KanbanFlow</title>
    <link rel="stylesheet"
          href="~/lib/bootstrap/dist/css/bootstrap.min.css"/>
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true"/>
</head>
<body>
<header>
    <nav class="navbar navbar-expand-sm navbar-toggleable-sm 
                    navbar-light bg-white border-bottom box-shadow mb-3">
        <div class="container-fluid">
            <a class="navbar-brand" asp-controller="Board"
               asp-action="Index">KanbanFlow</a>
        </div>
    </nav>
</header>
<div class="container-fluid"> @* Changé en container-fluid *@
    <main role="main" class="pb-3">
        @RenderBody()
    </main>
</div>

<footer class="border-top footer text-muted">
    <div class="container">
        &copy; 2024 - KanbanFlow
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
<tab title="Views/_ViewImports.cshtml">

```c#
@using KanbanFlow
@using KanbanFlow.Models
@using KanbanFlow.Entities
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

</tab>
</tabs>

### Lancement et Test

Vous avez terminé la version de base du projet !

```bash
dotnet run
```

Testez le flux complet :

1. Allez sur la page d'accueil (qui est maintenant la liste des tableaux).
2. Créez un nouveau tableau.
3. Cliquez sur ce tableau pour voir la vue détaillée avec les 3 colonnes.
4. Ajoutez quelques cartes dans la colonne "À faire".
5. Déplacez une carte vers "En cours", puis vers "Terminé".
6. Vérifiez que les boutons de déplacement se désactivent correctement aux extrémités.