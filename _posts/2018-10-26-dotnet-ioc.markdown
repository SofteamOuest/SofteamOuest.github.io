---
title: "Injection de dépendances dans les Azure Functions"
date: 2018-09-18 00:00:00 +001
layout: post
author: Yvan Razafindramanana
description: Injection de dépendances dans les Azure Functions
excerpt_separator: "<!--more-->"
---

<acronym>Tl;DR</acronym> La philosophie fondamentalement statique et procédurale des Azure Functions fait qu'il peut être contre-intuitif d'utiliser des concepts avancés de la programmation objet, tels que l'injection de dépendances.

Cependant plusieurs cas d'utilisation peuvent justifier le recours à ces concepts, pour simplifier l'écriture.

Heureusement, l'arrivée du support de .Net Core 2.x dans le nouveau runtime des Azure Functions (C#) rend possible l'utilisation des outils du le framework – et donc en particulier l'injection de dépendances.

<!--more-->

# Uniquement pour les Functions?

Les outils en question sont dans le package NuGet 
[`Microsoft.Extensions.DependencyInjection`](https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/). Il est fourni par défaut dans le SDK 2.x, mais vous devrez peut-être l'ajouter manuellement.

Dans cet article, je prends pour exemple les Azure Functions, dont les nombreuses fonctionnalités natives peuvent être complétées par tout l'arsenal du framework.

Mais le code ci-après fonctionne également sur toutes les plateformes compatibles .Net standard. On pourrait donc imaginer s'en servir pour une application console, un device IoT, etc.

# Configurer avec ServiceCollection

La première étape est d'instancer un objet `ServiceCollection` qui va nous permettre de configurer l'injection – exactement comme on le ferait dans une application ASP.NET Core classique.

```cs
var servicesCollection = new ServicesCollection();
```

Les plus perspicaces remarqueront que cet objet ressemble étrangement à celui qui est passé en paramètre à la méthode `ConfigureServices(IServiceCollection)` du `Startup.cs` ASP.NET Core. Un indice chez vous : ce n'est pas un hasard 😉.

Une fois cet objet instancié, vous pouvez utiliser exactement les mêmes méthodes `AddSingleton`, `AddScoped`, `AddTransient`, etc. que dans ASP.NET Core, pour configurer la résolution de vos dépendances en fonction de la durée de vie souhaitée.

Imaginons par exemple une interface `ICustomerRepository` et l'implémentation correspondante `CustomerRepository`. Voici comment elle pourrait être configurée dans `ServiceCollection` :

```cs
servicesCollection.AddSingleton<ICustomerRepository, CustomerRepository>();
```

# Instancier avec ServiceProvider 

Imaginons maintenant un `BillingService` qui a besoin d'une instance de `ICustomerRepository`. Pour pouvoir utiliser cette instance, il faut la déclarer dans le constructeur :

```cs
public class BillingService
{
    public BillingService(ICustomerRepository customerRepository) 
    {
        this.customerRepository = customerRepository;
    }
}
```

L'injection de dépendances nous évite d'avoir à instancier manuellement l'implémentation de `ICustomerRepository` à utiliser. C'est le moteur d'injection qui va nous la fournir via le constructeur.

Comment ça 🤔 ?

La `ServiceCollection` créée plus haut fournit une méthode `BuildServiceProvider` qui retourne un provider capable de nous instancier les objets __et__ leurs dépendances de manière automatique.

```cs
var services = serviceCollection.BuildServiceProvider();
var billingService = services.GetRequiredService<BillingService>();
```

L'instance `billingService` est créée dynamiquement et son champ `customerRepository` est valorisé avec une instance de `CustomerRepository` via le constructeur.

Il faut également au préalable déclarer notre `BillingService` dans la `ServiceCollection` :

```cs
serviceCollection.AddSingleton<BillingService>();
```

# Enrichissement

Et là vous allez me dire : ça commence à faire beaucoup de code pour instancer seulement deux objets.

C'est pas faux. Mais mon exemple reste relativement trivial.

Imaginez que votre `BillingService` ait aussi besoin d'un `IOrderRepository` en plus de `ICustomerRepository`. Imaginez en plus que cet `IOrderRepository` ait lui aussi besoin de `ICustomerRepository`.

L'injection de dépendances vous évite de devoir réfléchir à l'ordre et à la manière dont sont instanciés tous ces objets.

Vous pouvez en plus injecter des composants du framework lui-même.

Le gain est réellement significatif quand on a une hiérarchie de composants qui ont besoin les uns des autres, par exemple avec un découpage en couches métier (contrôleur, service, entity) et qui ont également besoin de services tiers.

Un bon vieux code lasagne comme on les aime 🙂🍕.

# Code final

Si on met tout les morceaux ensemble, on obtient ceci :

```cs
private static IServiceProvider ConfigureServices()
{
    var serviceCollection = new ServiceCollection();

    serviceCollection.AddSingleton<BillingService>();
    serviceCollection.AddSingleton<ICustomerRepository, CustomerRepository>();

    return serviceCollection.BuildServiceProvider();
}
```

Dans notre Azure Function, à l'endroit où on a besoin d'une instance de `BillingService` :

```cs
private static readonly IServiceProvider services = ConfigureServices();
private static readonly BillingService billingService = services.GetRequiredService<BillingService>();
```

Pensez bien à déclarer ces variables `static` afin de bénéficier de l'effet de cache des Azure Functions.

En effet, une fois son exécution terminée, la classe contenant votre code reste en mémoire au minimum cinq minutes avant d'être déchargée. Si, dans cet intervalle, votre fonction est à nouveau appelée, l'instance `billingService` déclarée statique sera toujours présente en mémoire et n'aura pas besoin d'être à nouveau initialisée.

Si vous injectez des ressources coûteuses (comme un client http par exemple), vous éviterez un démarrage à froid qui ralentit considérablement les temps de réponse des fonctions.

# Conclusion

Ce mécanisme d'injection de dépendances, initialement présent dans ASP.NET core, fonctionne désormais sur toutes les plateformes compatibles .NET standard, y compris les Azure Functions V2.

Dans un prochain article, je montrerai comment utiliser cette injection, pour paramétrer simplement un client HTTP avec une politique de rejeu et un coupe-circuit.
