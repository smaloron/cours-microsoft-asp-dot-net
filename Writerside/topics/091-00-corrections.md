# Réponses aux Auto-évaluations

## Module 1 : Introduction et Fondamentaux d'ASP.NET Core

* **1. Quel est le principal avantage de .NET Core/.NET par rapport à .NET Framework ?**
    * **Réponse : c) Il est multiplateforme et open-source.**
    * **Explication :** Le changement majeur a été de libérer .NET de son carcan Windows pour le rendre compatible avec
      macOS et Linux, tout en ouvrant son code source à la communauté, ce qui a accéléré son innovation et son adoption.

* **2. Quel est le rôle du fichier `Program.cs` dans une application ASP.NET Core ?**
    * **Réponse : b) Configurer les services et le pipeline de requêtes HTTP.**
    * **Explication :** C'est le point d'entrée de l'application où l'on enregistre les services pour l'injection de
      dépendances (`builder.Services`) et où l'on assemble la chaîne de middlewares qui traitera les requêtes (
      `app.Use...`).

* **3. Dans le patron MVC, quelle partie est responsable de la logique métier et de l'accès aux données ?**
    * **Réponse : c) Le Modèle.**
    * **Explication :** Le Modèle représente les données et les règles métier de l'application. Le Contrôleur l'utilise
      pour effectuer des actions, et la Vue l'utilise pour afficher les données.

* **4. Quel est le nom du serveur web par défaut, intégré à ASP.NET Core ?**
    * **Réponse : d) Kestrel.**
    * **Explication :** Kestrel est le serveur web interne, rapide et multiplateforme qui exécute votre application.
      IIS, Nginx et Apache sont souvent utilisés comme reverse proxies devant Kestrel.

* **5. (Ouverte) Expliquez ce qu'est un middleware.**
    * **Réponse modèle :** Un middleware est un composant logiciel qui se place dans le pipeline de traitement des
      requêtes HTTP. Chaque middleware a la possibilité de traiter la requête entrante, puis de passer la main au
      middleware suivant dans la chaîne. Sur le chemin du retour, il peut aussi traiter la réponse. Un exemple est le
      middleware `UseAuthentication`, qui examine la requête pour identifier l'utilisateur avant de la laisser continuer
      vers le reste de l'application.

* **6. (Ouverte) Décrivez le flux d'une requête dans une application MVC.**
    * **Réponse modèle :** 1. L'utilisateur effectue une action (clic) qui envoie une requête HTTP. 2. Le système de
      routage d'ASP.NET Core analyse l'URL et sélectionne le bon Contrôleur et la bonne Action. 3. L'Action du
      Contrôleur est exécutée. Elle interagit avec le Modèle pour récupérer ou modifier des données. 4. Le Contrôleur
      choisit une Vue et lui transmet les données du Modèle. 5. La Vue génère le code HTML final. 6. Le serveur renvoie
      cette page HTML au navigateur de l'utilisateur.

* **7. (Ouverte) Pourquoi l'ordre des middlewares dans le pipeline est-il important ?**
    * **Réponse modèle :** L'ordre est crucial car chaque middleware dépend potentiellement du travail effectué par les
      précédents. Par exemple, `UseAuthorization` (qui vérifie les permissions) doit être placé après
      `UseAuthentication` (qui identifie l'utilisateur), car on ne peut pas vérifier les droits de quelqu'un si on ne
      sait pas qui il est. De même, `UseRouting` doit venir avant `UseEndpoints` pour que la route soit déterminée avant
      d'être exécutée.

## Module 2 : Routage, Contrôleurs et Actions

* **1. Quelle est la route par convention par défaut dans un projet MVC ?**
    * **Réponse : b) `"{controller=Home}/{action=Index}/{id?}"`**
    * **Explication :** Cette route définit le contrôleur `Home` et l'action `Index` comme valeurs par défaut si des
      segments sont absents de l'URL, et un paramètre `id` optionnel.

* **2. Quel `IActionResult` utiliser pour renvoyer une erreur "Non trouvé" (404) ?**
    * **Réponse : c) `NotFound()`**
    * **Explication :** C'est la méthode helper standard qui génère une `NotFoundResult`, se traduisant par une réponse
      HTTP 404.

* **3. Quel est le rôle du "Model Binding" ?**
    * **Réponse : c) Convertir automatiquement les données d'une requête HTTP en un objet C#.**
    * **Explication :** C'est le mécanisme qui prend les valeurs d'un formulaire, de la query string ou de la route et
      les utilise pour peupler les propriétés d'un objet passé en paramètre d'une action.

* **4. À quoi sert `ModelState.IsValid` ?**
    * **Réponse : b) À vérifier si le modèle de données respecte les règles de validation (Data Annotations).**
    * **Explication :** Après le Model Binding, le framework exécute les validateurs (comme `[Required]`,
      `[StringLength]`) et stocke les résultats dans `ModelState`. La propriété `IsValid` est un simple booléen qui
      résume si des erreurs ont été trouvées.

* **5. (Ouverte) Expliquez la différence entre une action `[HttpGet]` et `[HttpPost]` pour une même route.**
    * **Réponse modèle :** Pour une même route comme `/Products/Create`, l'action `[HttpGet]` est utilisée pour la
      requête initiale qui affiche la page avec le formulaire vide. L'action `[HttpPost]` est utilisée pour traiter les
      données lorsque l'utilisateur soumet ce formulaire. On a besoin des deux pour séparer la logique d'affichage de la
      logique de traitement des données.

* **6. (Ouverte) Décrivez le flux de validation d'un formulaire.**
    * **Réponse modèle :** 1. L'utilisateur soumet le formulaire. 2. Le Model Binder tente de peupler le modèle en
      paramètre de l'action POST. 3. Le framework applique les Data Annotations du modèle et remplit `ModelState` avec
      d'éventuelles erreurs. 4. L'action vérifie `if (ModelState.IsValid)`. 5. Si c'est faux, l'action retourne la même
      vue en lui repassant le modèle. 6. La vue utilise des Tag Helpers comme `asp-validation-for` pour afficher les
      messages d'erreur à côté des champs correspondants.

* **7. (Ouverte) Différence entre données de route et de query string.**
    * **Réponse modèle :** Les données de route (ex: `/product/5`) font partie intégrante de la structure de l'URL et
      identifient généralement une ressource unique. C'est sémantiquement important. La query string (ex:
      `/product/search?name=livre`) contient des paramètres optionnels qui filtrent ou modifient la requête (tri,
      recherche, pagination). On utilise la route pour les identifiants et la query string pour les options.

## Module 3 : La Vue avec Razor et Interactions Dynamiques

* **1. Quel symbole est utilisé en Razor pour passer du HTML au C# ?**
    * **Réponse : d) `@`**
    * **Explication :** Le symbole `@` est le marqueur de transition qui indique au moteur Razor que ce qui suit est une
      expression ou un bloc de code C#.

* **2. Quel est le principal avantage d'un Tag Helper comme `<a asp-controller="Home" asp-action="Index">` ?**
    * **Réponse : b) Il génère une URL qui s'adapte automatiquement aux changements de configuration du routage.**
    * **Explication :** Il ne code pas l'URL en dur. Il demande au système de routage de la générer, ce qui rend les
      liens robustes aux changements futurs.

* **3. À quoi sert la directive `@RenderBody()` dans un fichier `_Layout.cshtml` ?**
    * **Réponse : c) À marquer l'emplacement où le contenu de la vue spécifique sera inséré.**
    * **Explication :** C'est le "trou" dans le gabarit principal qui sera rempli par le contenu de la vue en cours de
      rendu (ex: `Index.cshtml`, `Details.cshtml`).

* **4. Lors d'un appel AJAX, quel type de réponse une action de contrôleur doit-elle typiquement retourner ?**
    * **Réponse : c) `JsonResult`**
    * **Explication :** Les clients AJAX (JavaScript) attendent des données brutes, et non du HTML. JSON est le format
      standard pour cet échange de données. `JsonResult` (ou `Ok(data)`) sérialise un objet C# en JSON.

* **5. (Ouverte) Expliquez le principe DRY.**
    * **Réponse modèle :** DRY signifie "Don't Repeat Yourself" (Ne vous répétez pas). C'est un principe de
      développement qui vise à réduire la duplication de code. Les layouts et les vues partielles y contribuent en
      centralisant le code commun (comme le header, le footer, ou une carte de produit) à un seul endroit, au lieu de le
      copier-coller dans plusieurs pages.

* **6. (Ouverte) Décrivez les 5 étapes d'un appel AJAX.**
    * **Réponse modèle :** 1. **Déclencheur :** Un événement utilisateur (ex: clic) se produit. 2. **Requête JS :**
      JavaScript (via `fetch`) envoie une requête HTTP en arrière-plan à une action de contrôleur. 3. **Traitement
      Serveur :** L'action exécute sa logique. 4. **Réponse Serveur :** L'action retourne des données au format JSON. 5.
      **Mise à jour DOM :** JavaScript reçoit la réponse JSON et met à jour dynamiquement une partie de la page HTML
      sans rechargement complet.

* **7. (Ouverte) Différence entre vue et vue partielle.**
    * **Réponse modèle :** Une vue complète (`.cshtml`) est généralement la cible d'une action de contrôleur et
      représente une page entière (qui utilise souvent un layout). Une vue partielle (`_Partial.cshtml`) est un morceau
      de code Razor réutilisable, conçu pour être rendu à l'intérieur d'une autre vue. Elle n'a pas sa propre URL et
      dépend de la vue parente pour lui passer ses données.

## Module 4 : L'Architecture Interne : Services et Injection de Dépendances

* **1. Quel est le principal bénéfice de l'Injection de Dépendances ?**
    * **Réponse : b) Réduire le couplage entre les composants.**
    * **Explication :** Le DI permet à un composant de dépendre d'une abstraction (interface) plutôt que d'une
      implémentation concrète, ce qui rend les composants interchangeables et plus faciles à tester.

* **2. Quelle durée de vie de service est la plus appropriée pour un contexte de base de données EF Core (`DbContext`) ?
  **
    * **Réponse : b) `Scoped`**
    * **Explication :** `Scoped` garantit qu'il n'y aura qu'une seule instance du `DbContext` par requête HTTP. C'est le
      comportement attendu pour EF Core afin d'assurer la cohérence des données au sein d'une même opération.

* **3. Où enregistre-t-on les services pour l'injection de dépendances ?**
    * **Réponse : d) Dans le fichier `Program.cs`.**
    * **Explication :** C'est dans la phase de configuration de l'application (`builder.Services`) que l'on "enseigne"
      au conteneur de DI comment résoudre les dépendances.

* **4. Un filtre d'action (`IActionFilter`) s'exécute :**
    * **Réponse : c) Avant et après l'action.**
    * **Explication :** `IActionFilter` expose deux méthodes : `OnActionExecuting` (avant) et `OnActionExecuted` (
      après), permettant d'encadrer l'exécution de l'action.

* **5. (Ouverte) Expliquez la différence entre `Singleton`, `Scoped` et `Transient`.**
    * **Réponse modèle :** `Singleton` : Une seule instance est créée pour toute la durée de vie de l'application.
      `Scoped` : Une nouvelle instance est créée pour chaque requête HTTP, mais elle est réutilisée au sein de cette
      même requête. `Transient` : Une nouvelle instance est créée chaque fois que le service est demandé.

* **6. (Ouverte) Pourquoi est-il préférable d'injecter une interface plutôt qu'une classe concrète ?**
    * **Réponse modèle :** Injecter une interface permet de découpler le code. La classe qui reçoit la dépendance ne
      connaît que le "contrat" (l'interface), pas les détails de l'implémentation. Cela permet de changer facilement
      l'implémentation (ex: passer d'un dépôt en mémoire à un dépôt SQL) sans modifier le code qui l'utilise. C'est
      aussi essentiel pour les tests unitaires, où l'on peut injecter une fausse implémentation (un "mock").

* **7. (Ouverte) Cas d'utilisation pour un filtre autre que le logging ou la validation.**
    * **Réponse modèle :** Un cas d'utilisation pourrait être un filtre de gestion de cache. Dans `OnActionExecuting`,
      le filtre pourrait vérifier si le résultat de l'action est déjà en cache. Si oui, il court-circuite l'action et
      retourne directement le résultat mis en cache. Sinon, il laisse l'action s'exécuter. Dans `OnActionExecuted`, il
      prend le résultat, le met en cache pour les futures requêtes, puis retourne la réponse.

## Module 5 : Accès aux Données avec Entity Framework Core

* **1. Que signifie "Code-First" ?**
    * **Réponse : b) On écrit les classes C# avant de générer la base de données.**
    * **Explication :** C'est l'approche où le code C# (les classes d'entités) est la source de vérité, et EF Core est
      utilisé pour dériver le schéma de la base de données à partir de ce code.

* **2. Quelle classe représente votre session avec la base de données ?**
    * **Réponse : c) `DbContext`**
    * **Explication :** Le `DbContext` est le point central pour interagir avec la base de données : il gère la
      connexion, le suivi des modifications et l'exécution des requêtes.

* **3. Quelle commande applique une migration en attente à la base de données ?**
    * **Réponse : c) `Update-Database`**
    * **Explication :** `Add-Migration` génère le script de migration, mais c'est `Update-Database` (ou
      `dotnet ef database update`) qui l'exécute réellement sur la base de données cible.

* **4. Quand la requête SQL est-elle réellement envoyée dans ce
  code : `var query = context.Products.Where(p => p.Price > 10);` ?**
    * **Réponse : c) Lorsque la variable `query` est utilisée dans une boucle ou convertie (ex: `.ToList()`).**
    * **Explication :** C'est le principe de l'exécution différée (deferred execution) de LINQ. La première ligne ne
      fait que construire l'expression de la requête. La requête n'est envoyée à la BDD que lorsque le résultat est
      matérialisé.

* **5. (Ouverte) Expliquez le rôle de `Add-Migration`.**
    * **Réponse modèle :** La commande `Add-Migration` compare le modèle de données C# actuel avec l'état du dernier
      instantané de migration. Elle génère ensuite un nouveau fichier de migration contenant le code C# (utilisant les
      API de migration) nécessaire pour mettre à jour le schéma de la base de données pour qu'il corresponde au nouveau
      modèle. Elle produit également un fichier de métadonnées qui est un instantané du nouveau modèle.

* **6. (Ouverte) Pourquoi utiliser `Find(id)` plutôt que `FirstOrDefault(p => p.Id == id)` ?**
    * **Réponse modèle :** `Find(id)` est optimisé pour la recherche par clé primaire. Il va d'abord chercher dans le
      cache du `DbContext` (le Change Tracker) pour voir si l'entité a déjà été chargée. Si c'est le cas, il la retourne
      immédiatement sans aller à la base de données. `FirstOrDefault()` ira toujours exécuter une requête SQL à la base
      de données.

* **7. (Ouverte) Décrivez le "Patron d'Unité de Travail" et comment `SaveChanges()` l'implémente.**
    * **Réponse modèle :** Le patron d'Unité de Travail (Unit of Work) consiste à regrouper une série d'opérations (
      ajouts, modifications, suppressions) en une seule transaction. Le `DbContext` d'EF Core implémente ce patron.
      Lorsque vous appelez `.Add()`, `.Update()`, ou `.Remove()`, EF Core ne fait que suivre ces changements en mémoire.
      L'appel à `.SaveChanges()` analyse tous ces changements et les exécute en une seule transaction atomique sur la
      base de données. Si une seule opération échoue, toutes sont annulées.

## Module 6 : Construire et Sécuriser une API RESTful

* **1. Quel verbe HTTP est utilisé pour mettre à jour une ressource existante ?**
    * **Réponse : c) `PUT`**
    * **Explication :** `PUT` est utilisé pour remplacer entièrement une ressource à une URL donnée. `PATCH` (non listé)
      est utilisé pour des mises à jour partielles.

* **2. Quel code de statut HTTP indique qu'une ressource a été créée avec succès ?**
    * **Réponse : b) `201 Created`**
    * **Explication :** C'est le code standard pour un `POST` réussi. Il est souvent accompagné d'un en-tête `Location`
      pointant vers l'URL de la nouvelle ressource.

* **3. Quelle est la différence entre l'Authentification et l'Autorisation ?**
    * **Réponse : b) L'authentification vérifie qui vous êtes, l'autorisation vérifie ce que vous pouvez faire.**
    * **Explication :** L'authentification (AuthN) est la preuve de l'identité. L'autorisation (AuthZ) est la
      vérification des permissions associées à cette identité.

* **4. Quel attribut place-t-on sur une action pour exiger que l'utilisateur soit authentifié ?**
    * **Réponse : d) `[Authorize]`**
    * **Explication :** C'est l'attribut standard du framework pour déclencher le middleware d'autorisation, qui par
      défaut exige une identité authentifiée.

* **5. (Ouverte) Qu'est-ce qu'une ressource dans le contexte de REST ?**
    * **Réponse modèle :** Une ressource est toute information qui peut être nommée et identifiée. C'est le concept
      fondamental de REST. Ça peut être un objet unique (un produit, un utilisateur) ou une collection d'objets (la
      liste de tous les produits). Chaque ressource est accessible via une URL unique (endpoint). Par exemple,
      `/api/users/123` est l'URL de la ressource "utilisateur avec l'ID 123".

* **6. (Ouverte) Expliquez le flux d'authentification basé sur les tokens JWT.**
    * **Réponse modèle :** 1. **Login :** Le client envoie ses identifiants (ex: email/mot de passe) à un endpoint
      d'authentification. 2. **Token Generation :** Si les identifiants sont valides, le serveur génère un token JWT
      signé qui contient des informations sur l'utilisateur (claims) et une date d'expiration. Il renvoie ce token au
      client. 3. **Requêtes Autorisées :** Pour chaque requête ultérieure à un endpoint sécurisé, le client inclut le
      token JWT dans l'en-tête HTTP `Authorization` (ex: `Authorization: Bearer <token>`). Le serveur valide la
      signature et l'expiration du token avant d'autoriser la requête.

* **7. (Ouverte) Pourquoi l'attribut `[ApiController]` est-il si utile ?**
    * **Réponse modèle :** Il active plusieurs conventions qui simplifient le développement d'API. Deux avantages
      majeurs sont : 1. La validation automatique du modèle : si `ModelState.IsValid` est faux, il retourne
      automatiquement une réponse `400 Bad Request` sans qu'on ait besoin d'écrire le `if` dans chaque action. 2.
      L'inférence des sources de liaison : il est capable de déduire que les objets complexes viennent du corps de la
      requête (`[FromBody]`), ce qui rend le code plus propre.

## Module 7 : Déploiement et Hébergement

* **1. Quel est le rôle principal d'un reverse proxy (comme Nginx) devant Kestrel ?**
    * **Réponse : a) Gérer les requêtes entrantes d'Internet et les transmettre à Kestrel.**
    * **Explication :** Le reverse proxy agit comme une passerelle robuste, gérant des tâches comme la terminaison SSL,
      la répartition de charge et la sécurité, avant de passer une requête "propre" à Kestrel.

* **2. Quelle variable d'environnement contrôle le comportement de l'application (ex: pages d'erreur détaillées) ?**
    * **Réponse : c) `ASPNETCORE_ENVIRONMENT`**
    * **Explication :** C'est la variable clé qui permet au code de s'adapter en fonction de l'environnement (
      `Development`, `Staging`, `Production`).

* **3. Quelle commande CLI est utilisée pour empaqueter une application pour le déploiement ?**
    * **Réponse : b) `dotnet publish`**
    * **Explication :** Cette commande compile l'application et rassemble tous les fichiers et dépendances nécessaires à
      son exécution dans un dossier de sortie.

* **4. Quel est le principal avantage de l'utilisation de Docker pour le déploiement ?**
    * **Réponse : d) Créer un environnement d'exécution portable et cohérent qui élimine le problème du "ça marche sur
      ma machine".**
    * **Explication :** Docker empaquète l'application ET son environnement d'exécution (runtime, dépendances système)
      dans un conteneur, garantissant qu'elle se comportera de la même manière partout.

* **5. (Ouverte) Où et comment devriez-vous stocker une chaîne de connexion à une base de données en production ?**
    * **Réponse modèle :** La chaîne de connexion ne doit JAMAIS être stockée dans le code source ou dans
      `appsettings.json`. En production, la méthode standard et sécurisée est de la stocker en tant que variable
      d'environnement sur le serveur hôte ou via un service de gestion de secrets fourni par la plateforme cloud (ex:
      Azure Key Vault, AWS Secrets Manager). L'application la lira ensuite via le système de configuration.

* **6. (Ouverte) Qu'est-ce qu'un `Dockerfile` ?**
    * **Réponse modèle :** Un `Dockerfile` est un fichier texte qui contient une liste d'instructions, comme une
      recette. Ces instructions décrivent étape par étape comment construire une image Docker pour une application. Il
      spécifie l'image de base, où copier les fichiers de l'application, les commandes à exécuter (comme
      `dotnet restore` et `dotnet publish`), et la commande finale pour démarrer l'application.

* **7. (Ouverte) Expliquez brièvement ce qu'est le CI/CD.**
    * **Réponse modèle :** CI/CD signifie Intégration Continue / Déploiement Continu. C'est une pratique DevOps qui
      automatise le cycle de vie de la mise en production d'un logiciel. **CI** est le processus automatisé de
      compilation et de test du code à chaque fois qu'un développeur pousse une modification. **CD** est le processus
      qui prend le code validé par la CI et le déploie automatiquement sur les serveurs de production. L'ensemble vise à
      rendre les déploiements plus fréquents, plus rapides et plus fiables.