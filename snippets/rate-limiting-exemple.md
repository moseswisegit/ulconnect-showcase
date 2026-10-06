# Limitation de débit et CORS dans ASP.NET Core 8

Le pattern : une limite globale par client, des politiques nommées plus strictes sur les routes sensibles, une réponse 429 propre, et une politique CORS fondée sur une liste d'origines lue dans la configuration.

```csharp
// Exemple simplifié et générique
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Limite globale : seau à jetons par utilisateur authentifié, sinon par adresse IP.
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
    {
        var cle = ctx.User.Identity?.IsAuthenticated == true
            ? "u:" + ctx.User.FindFirst("sub")?.Value
            : "ip:" + ctx.Connection.RemoteIpAddress;

        return RateLimitPartition.GetTokenBucketLimiter(cle, _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,
            TokensPerPeriod = 10,
            ReplenishmentPeriod = TimeSpan.FromSeconds(1),
            QueueLimit = 0,
        });
    });

    // Politique stricte pour une route sensible (ex. demande de code de connexion).
    options.AddPolicy("connexion", ctx => RateLimitPartition.GetFixedWindowLimiter(
        partitionKey: ctx.Connection.RemoteIpAddress?.ToString() ?? "inconnu",
        factory: _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 3,
            Window = TimeSpan.FromMinutes(3),
            QueueLimit = 0,
        }));
});

builder.Services.AddCors(options =>
{
    options.AddPolicy("console", policy =>
    {
        var origines = builder.Configuration.GetSection("Cors:Origines").Get<string[]>() ?? [];
        policy.WithOrigins(origines)
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

var app = builder.Build();

// Derrière un proxy, configurer les en-têtes transférés AVANT le limiteur,
// sinon toutes les requêtes semblent venir de la même adresse.
app.UseForwardedHeaders();
app.UseCors("console");
app.UseRateLimiter();

app.MapPost("/connexion/code", () => Results.Accepted())
   .RequireRateLimiting("connexion");

app.Run();
```

## Points d'attention

- Derrière un proxy ou un équilibreur de charge, les réseaux et proxys de confiance doivent être déclarés explicitement dans `ForwardedHeadersOptions`. Sans cela, l'adresse vue par l'API est celle du proxy et une limite « par IP » devient une limite globale.
- Une limite par IP pénalise les réseaux partagés (Wi-Fi d'un campus). La combiner avec une limite par compte ou par adresse courriel sur les routes de connexion.
- `AllowCredentials` est incompatible avec une origine générique : la liste doit être explicite.
