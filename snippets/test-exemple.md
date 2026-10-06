# Test d'intégration d'une API ASP.NET Core

Le pattern : démarrer l'API complète en mémoire, remplacer uniquement les dépendances externes, puis tester par de vraies requêtes HTTP.

```csharp
// Exemple simplifié et générique
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Xunit;

public class ApiFactory : WebApplicationFactory<Program>
{
    public FauxEnvoiCourriel Courriels { get; } = new();

    protected override void ConfigureWebHost(Microsoft.AspNetCore.Hosting.IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureServices(services =>
        {
            // Seule la dépendance externe est remplacée ; le reste du pipeline est réel.
            services.AddSingleton<IEnvoiCourriel>(Courriels);
        });
    }
}

public class ConnexionTests : IClassFixture<ApiFactory>
{
    private readonly ApiFactory _factory;

    public ConnexionTests(ApiFactory factory) => _factory = factory;

    [Fact]
    public async Task Un_code_errone_cinq_fois_invalide_le_code()
    {
        var client = _factory.CreateClient();
        await client.PostAsJsonAsync("/connexion/code", new { courriel = "etudiant@exemple.test" });

        for (var i = 0; i < 5; i++)
        {
            await client.PostAsJsonAsync("/connexion/verification",
                new { courriel = "etudiant@exemple.test", code = "000000" });
        }

        var bonCode = _factory.Courriels.DernierCode("etudiant@exemple.test");
        var reponse = await client.PostAsJsonAsync("/connexion/verification",
            new { courriel = "etudiant@exemple.test", code = bonCode });

        Assert.Equal(HttpStatusCode.BadRequest, reponse.StatusCode);
    }
}
```

## Pourquoi ce style de test

- Il exerce le routage, l'authentification, la validation et la limitation de débit réels, ce qu'un test unitaire de contrôleur ne voit pas.
- Les dépendances externes (courriel, stockage, notifications) sont les seules doublures, ce qui garde les tests rapides et déterministes.
- Une base dédiée par classe de test évite les interférences entre tests.
