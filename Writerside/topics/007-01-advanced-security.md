# Module 7 : Sécurité Avancée et Gestion des Identités - Pour aller plus loin

### Objectifs Pédagogiques

À la fin de ce module complémentaire, vous serez capable de :

* **Comprendre** le flux d'authentification basé sur les tokens JWT et son utilité pour les APIs.
* **Implémenter** la génération et la validation de tokens JWT dans une API ASP.NET Core.
* **Gérer** l'autorisation basée sur les rôles et les "policies" (stratégies).
* **Comprendre** les principes de base d'OAuth 2.0 et d'OpenID Connect (OIDC).
* **Configurer** une application pour utiliser un fournisseur d'identité externe comme Google ou Microsoft.

### Introduction : La sécurité dans un monde connecté

Dans la partie essentielle, nous avons construit une forteresse avec un pont-levis et des gardes. C'est parfait pour une
application monolithique. Mais que se passe-t-il dans une ville moderne où plusieurs services (APIs), des applications
mobiles et des sites partenaires doivent communiquer de manière sécurisée ? On ne peut pas simplement partager des
cookies partout.

Cette section vous apprend à devenir un diplomate de la sécurité. Vous allez apprendre à émettre des "passeports"
sécurisés (JWT) pour que vos APIs puissent authentifier les clients sans état. Vous créerez des niveaux
d'accréditation (rôles et policies) pour une gestion fine des permissions. Et enfin, vous apprendrez à faire confiance à
d'autres "nations" (fournisseurs d'identité externes) pour qu'elles authentifient les utilisateurs à votre place. Ce
sont les compétences essentielles pour construire des systèmes distribués et des applications modernes.

---

### 1. L'authentification pour les APIs : Les Tokens JWT

**Le problème :** Les cookies fonctionnent bien pour les navigateurs, mais ils sont peu pratiques pour les autres
clients (applications mobiles, scripts, autres serveurs). De plus, ils créent un "état" sur le serveur (une session), ce
qui va à l'encontre du principe REST d'API "stateless".

**La solution :** Le **JSON Web Token (JWT)**. Comme nous l'avons vu dans le module sur les APIs, un JWT est un
standard (RFC 7519) pour créer des jetons d'accès qui contiennent des informations (les "claims") et qui sont signés
numériquement.

Un JWT est une longue chaîne de caractères composée de trois parties séparées par des points :
`[Header].[Payload].[Signature]`

* **Header :** Métadonnées sur le token (type, algorithme de signature).
* **Payload (charge utile) :** Les informations sur l'utilisateur, appelées **claims**. Exemples de claims : `sub` (
  sujet/ID de l'utilisateur), `name` (nom), `exp` (date d'expiration), `role` (rôle de l'utilisateur).
* **Signature :** La partie la plus importante. C'est un hachage du Header et du Payload, signé avec une **clé secrète**
  que seul le serveur connaît.

**Le flux JWT est stateless :**

1. Le client s'authentifie une fois (login/pass).
2. Le serveur valide les identifiants, génère un JWT signé avec la clé secrète, et le renvoie au client.
3. Le client stocke ce token (ex: dans le `localStorage` ou un stockage sécurisé sur mobile).
4. Pour chaque requête suivante, le client envoie le token dans l'en-tête `Authorization: Bearer <token>`.
5. Le serveur reçoit le token. Il n'a pas besoin de chercher quoi que ce soit en base de données. Il vérifie simplement
   que la signature est valide avec sa clé secrète. Si c'est le cas, il peut faire confiance aux informations contenues
   dans le Payload.

#### Mise en place

1. **Installer le paquet NuGet :** `Microsoft.AspNetCore.Authentication.JwtBearer`.
2. **Configurer dans `appsettings.json` :** On y place la configuration du token.
   ```json
   "Jwt": {
     "Key": "CeciEstUneCléSecrèteSuperLongueEtDifficileÀDeviner",
     "Issuer": "https://mon-api.com",
     "Audience": "https://mon-client.com"
   }
   ```
3. **Configurer dans `Program.cs` :**
   ```c#
   using Microsoft.AspNetCore.Authentication.JwtBearer;
   using Microsoft.IdentityModel.Tokens;
   using System.Text;

   // ...
   builder.Services.AddAuthentication(options => {
       options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
       options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
   })
   .AddJwtBearer(options => {
       options.TokenValidationParameters = new TokenValidationParameters
       {
           ValidateIssuer = true,
           ValidateAudience = true,
           ValidateLifetime = true,
           ValidateIssuerSigningKey = true,
           ValidIssuer = builder.Configuration["Jwt:Issuer"],
           ValidAudience = builder.Configuration["Jwt:Audience"],
           IssuerSigningKey = new SymmetricSecurityKey(
               Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
       };
   });
   // ...
   ```
4. **Créer un endpoint pour générer le token :**
   Ce endpoint sera public. Il prendra les identifiants, les vérifiera (avec Identity par exemple), et si c'est bon,
   générera le JWT. La logique de génération est un peu verbeuse mais toujours la même.

---

### 2. Autorisation Avancée : Rôles et Policies

`[Authorize]` est bien, mais souvent on a besoin de plus de granularité.

#### Autorisation basée sur les Rôles

C'est la plus simple. On associe des utilisateurs à des "rôles" (Admin, Editor, Member...) et on protège les actions en
fonction de ces rôles.

1. **Avec ASP.NET Core Identity :** On peut créer des rôles et les assigner aux utilisateurs.
2. **Dans le contrôleur :**
   ```c#
   // Seuls les Admins peuvent supprimer.
   [HttpDelete("{id}")]
   [Authorize(Roles = "Admin")]
   public IActionResult DeleteProduct(int id) { /* ... */ }

   // Les Admins ET les Editors peuvent créer.
   [HttpPost]
   [Authorize(Roles = "Admin,Editor")]
   public IActionResult CreateProduct(ProductCreateModel model) { /* ... */ }
   ```
   C'est simple et efficace pour des cas d'usage courants.

#### Autorisation basée sur les Policies (Stratégies)

**Le problème :** Les rôles sont rigides. Que faire si une règle d'autorisation est plus complexe ? Par exemple : "Seuls
les employés qui sont dans l'entreprise depuis plus d'un an peuvent accéder à ce document". Ce n'est pas un "rôle".

**La solution :** Les **Policies**. Une policy est un ensemble de conditions (de "requirements") qu'un utilisateur doit
remplir. C'est beaucoup plus flexible.

1. **Définir la Policy dans `Program.cs` :**
   ```c#
   builder.Services.AddAuthorization(options =>
   {
       // Une policy simple basée sur un claim
       options.AddPolicy("MustBeFromFrance", policy =>
           policy.RequireClaim("Country", "France"));

       // Une policy plus complexe qui vérifie l'âge
       options.AddPolicy("AtLeast18", policy =>
           policy.RequireAssertion(context =>
           {
               var birthDateClaim = context.User.FindFirst(c => c.Type == ClaimTypes.DateOfBirth);
               if (birthDateClaim == null) return false;

               var birthDate = Convert.ToDateTime(birthDateClaim.Value);
               return birthDate.AddYears(18) <= DateTime.Today;
           }));
   });
   ```
2. **Appliquer la Policy sur le contrôleur/action :**
   ```c#
   [HttpGet]
   [Authorize(Policy = "AtLeast18")]
   public IActionResult GetContentForAdults() { /* ... */ }
   ```

Les Policies décuplent la puissance de votre système d'autorisation en le déconnectant de simples noms de rôles pour le
lier à de véritables règles métier.

---

### 3. OAuth 2.0 et OpenID Connect (OIDC) : L'authentification fédérée

**Le problème :** Vous ne voulez pas forcément forcer vos utilisateurs à créer un énième compte sur votre site. La
plupart ont déjà un compte Google, Microsoft, Facebook, GitHub... Ne serait-il pas plus simple de leur permettre de se
connecter avec un compte qu'ils possèdent déjà ?

**La solution :** Les protocoles **OAuth 2.0** et **OpenID Connect (OIDC)**.

* **OAuth 2.0 :** C'est un protocole d'**autorisation**. Il permet à un utilisateur de donner à une application (la
  vôtre) la permission d'accéder à des ressources lui appartenant sur un autre service (ex: accéder à ses contacts
  Google), sans jamais donner son mot de passe Google à votre application.
* **OpenID Connect (OIDC) :** C'est une surcouche d'**authentification** construite sur OAuth 2.0. C'est le protocole
  qui permet de dire "Je veux utiliser Google pour me connecter à KanbanFlow".

**Comment ça marche (simplifié) :**

1. L'utilisateur clique sur "Se connecter avec Google" sur votre site.
2. Votre site le redirige vers la page de connexion de Google.
3. L'utilisateur se connecte sur le site de Google et autorise votre application à accéder à ses informations de profil.
4. Google le redirige vers votre site, avec un "code d'autorisation".
5. Votre serveur échange ce code avec Google (de manière sécurisée, de serveur à serveur) contre un "token d'identité" (
   un JWT) contenant les informations de l'utilisateur.
6. Votre application utilise ce token pour connecter l'utilisateur.

**Mise en place avec ASP.NET Core :**
C'est étonnamment simple grâce aux middlewares fournis.

1. **Installer le paquet :** `Microsoft.AspNetCore.Authentication.Google`
2. **Configurer dans `Program.cs` :**
   ```c#
   builder.Services.AddAuthentication()
       .AddGoogle(options =>
       {
           // Ces secrets doivent venir de votre configuration,
           // jamais en dur !
           options.ClientId = builder.Configuration["Authentication:Google:ClientId"];
           options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"];
       });
   ```

Le framework gère tout le flux complexe pour vous !

---

### Conclusion

Vous avez maintenant une vision complète et moderne de la sécurité des applications. Vous savez non seulement comment
sécuriser une application web classique, mais aussi comment concevoir la sécurité pour un écosystème distribué d'APIs et
de clients grâce aux **tokens JWT**.

Vous maîtrisez des techniques d'autorisation avancées avec les **rôles** et les **policies**, qui vous permettent
d'implémenter des règles d'accès fines et complexes. Enfin, vous comprenez les principes de l'**authentification fédérée
** avec OAuth 2.0 et OIDC, vous permettant d'offrir à vos utilisateurs des options de connexion modernes et pratiques.

Ces compétences sont au cœur des attentes pour un développeur d'applications senior. Elles vous permettent de construire
des systèmes qui sont non seulement fonctionnels, mais aussi sécurisés, interopérables et dignes de confiance.

Il est maintenant temps d'aborder la dernière étape : le déploiement de ces applications robustes et sécurisées.