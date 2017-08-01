---
title: Migrating Auth and Identity to ASP.NET Core 2.0
author: scottaddie
description: This article outlines the most common steps for migrating ASP.NET Core 1.x authentication and Identity to ASP.NET Core 2.0.
keywords: ASP.NET Core,Identity,authentication
ms.author: scaddie
manager: wpickett
ms.date: 07/31/2017
ms.topic: article
ms.technology: aspnet
ms.prod: asp.net-core
uid: migration/identity-2x
---

# Migrating Authentication and Identity to ASP.NET Core 2.0

By [Scott Addie](https://github.com/scottaddie) and [Hao Kung](https://github.com/HaoK)

ASP.NET Core 2.0 has a new model for authentication and [Identity](xref:security/authentication/identity) which simplifies configuration by using services. ASP.NET Core 1.x applications that use authentication and Identity need to be updated to use the new model as outlined below.

<a name="auth-middleware"></a>

## Authentication Middleware and Services
In 1.x projects, authentication is configured via middleware. A middleware method is invoked for each authentication scheme you want to support.

```csharp
public void Configure(IApplicationBuilder app, ILoggerFactory loggerfactory) {
    app.UseIdentity();
    app.UseCookieAuthentication(new CookieAuthenticationOptions {
        LoginPath = new PathString("/login")
    });
    app.UseFacebookAuthentication(new FacebookOptions { 
        AppId = Configuration["auth:facebook:appid"],
        AppSecret = Configuration["auth:facebook:appsecret"]
    });
} 
```

In 2.0 projects, authentication is configured via services. Each authentication scheme is registered in the `ConfigureServices` method of *Startup.cs*. The `UseIdentity` method is replaced with `UseAuthentication`.

```csharp
public void ConfigureServices(IServiceCollection services) {
    services.AddIdentity<ApplicationUser, IdentityRole>().AddEntityFrameworkStores();
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
            .AddCookie(o => o.LoginPath = new PathString("/login"))
            .AddFacebook(o =>
            {
                o.AppId = Configuration["auth:facebook:appid"];
                o.AppSecret = Configuration["auth:facebook:appsecret"];
            });
}

public void Configure(IApplicationBuilder app, ILoggerFactory loggerfactory) {
    app.UseAuthentication();
}
```

The `UseAuthentication` method discovers the services being used, from [dependency injection](xref:fundamentals/dependency-injection), to determine the authentication schemes you want to use.

Below are 2.0 migration instructions for each major authentication scheme.

### Cookie-based Authentication
Select one of the two options below, and make the necessary changes in *Startup.cs*:

1. Use cookies with Identity
    - Remove the cookie configuration code from the `Configure` method:
 
    ```csharp
    services.Configure<IdentityOptions>(o => { config.Cookies... }
    ```

    - Invoke the `AddIdentity` method in the `ConfigureServices` method to add the cookie authentication services.
    - Invoke the `ConfigureApplicationCookie` or `ConfigureExternalCookie` method in the `ConfigureServices` method to configure the Identity cookie.

    ```csharp
    services.AddIdentity<ApplicationUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();
    services.ConfigureApplicationCookie(o => o.LoginPath = new PathString("/Account/LogIn"));
    ```

2. Use cookies without Identity
    - Remove the `UseCookieAuthentication` method call from the `Configure` method.
    - Invoke the `AddAuthentication` and `AddCookieAuthentication` methods in the `ConfigureServices` method:

    ```csharp
    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
            .AddCookieAuthentication(o => {
                o.LoginPath = "/Account/LogIn";
                o.LogoutPath = "/Account/LogOff";
            });
    ```

### JWT Bearer Authentication
Make the following changes in *Startup.cs*:

- Remove the `UseJwtBearerAuthentication` method call from the `Configure` method.
- Invoke the `AddJwtBearerAuthentication` method in the `ConfigureServices` method:

    ```csharp
    services.AddJwtBearerAuthentication(o => {
        o.Audience = "http://localhost:5001/";
        o.Authority = "http://localhost:5000/";
    });
    ```

### Facebook Authentication
Make the following changes in *Startup.cs*:

- Remove the `UseFacebookAuthentication` method call from the `Configure` method.
- Invoke the `AddFacebookAuthentication` method in the `ConfigureServices` method:
    
    ```csharp
    services.AddFacebookAuthentication(o => {
        o.AppId = Configuration["facebook:appid"];
        o.AppSecret = Configuration["facebook:appsecret"];
    });
    ```

### Google Authentication
Make the following changes in *Startup.cs*:

- Remove the `UseGoogleAuthentication` method call from the `Configure` method.
- Invoke the `AddGoogleAuthentication` method in the `ConfigureServices` method:

    ```csharp
    services.AddGoogleAuthentication(o => {
        o.ClientId = Configuration["auth:google:clientid"];
        o.ClientSecret = Configuration["auth:google:clientsecret"];
    });    
    ```

### Microsoft Account Authentication
Make the following changes in *Startup.cs*:

- Remove the `UseMicrosoftAccountAuthentication` method call from the `Configure` method.
- Invoke the `AddMicrosoftAccountAuthentication` method in the `ConfigureServices` method:

    ```csharp
    services.AddMicrosoftAccountAuthentication(o => {
        o.ClientId = Configuration["auth:microsoft:clientid"];
        o.ClientSecret = Configuration["auth:microsoft:clientsecret"];
    });
    ``` 

### Twitter Authentication
Make the following changes in *Startup.cs*:

- Remove the `UseTwitterAuthentication` method call from the `Configure` method.
- Invoke the `AddTwitterAuthentication` method in the `ConfigureServices` method:

    ```csharp
    services.AddTwitterAuthentication(o => {
        o.ConsumerKey = Configuration["auth:twitter:consumerkey"];
        o.ConsumerSecret = Configuration["auth:twitter:consumersecret"];
    });
    ```

### Multiple Authentication Schemes
If your 1.x application has configured multiple authentication schemes, the default scheme needs to be identified in 2.0. 

<a name="obsolete-interface"></a>

## Use HttpContext Authentication Extensions
The `IAuthenticationManager` interface was the main entry point into the 1.x authentication system. It has been replaced with a new set of `HttpContext` extension methods in the `Microsoft.AspNetCore.Authentication` namespace.

For example, 1.x projects reference an `Authentication` property:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore1.1App/AspNetCoreDotNetCore1.1App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

In 2.0 projects, import the `Microsoft.AspNetCore.Authentication` namespace, and delete the `Authentication` property references:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore2.0App/AspNetCoreDotNetCore2.0App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

<a name="identity-cookie-options"></a>

## IdentityCookieOptions Instances
A side effect of the 2.0 changes is the switch to using named options instead of cookie options instances. The ability to customize the Identity cookie scheme names is removed.

For example, 1.x projects use [constructor injection](xref:mvc/controllers/dependency-injection#constructor-injection) to pass an `IdentityCookieOptions` parameter into *AccountController.cs*. The external cookie authentication scheme is accessed from the provided instance:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore1.1App/AspNetCoreDotNetCore1.1App/Controllers/AccountController.cs?name=snippet_AccountControllerConstructor&highlight=4,11)]

The constructor injection becomes unnecessary in 2.0 projects, and the `_externalCookieScheme` field can be deleted:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore2.0App/AspNetCoreDotNetCore2.0App/Controllers/AccountController.cs?name=snippet_AccountControllerConstructor)]

The `IdentityConstants.ExternalScheme` constant can be used directly:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore2.0App/AspNetCoreDotNetCore2.0App/Controllers/AccountController.cs?name=snippet_AuthenticationProperty)]

<a name="navigation-properties"></a>

## Add IdentityUser POCO Navigation Properties
The Entity Framework Core navigation properties of the base `IdentityUser` POCO (Plain Old CLR Object) have been removed. If the 1.x project used these properties, manually add them back to the 2.0 project:

```csharp
/// <summary>
/// Navigation property for the roles this user belongs to.
/// </summary>
public virtual ICollection<TUserRole> Roles { get; } = new List<TUserRole>();

/// <summary>
/// Navigation property for the claims this user possesses.
/// </summary>
public virtual ICollection<TUserClaim> Claims { get; } = new List<TUserClaim>();

/// <summary>
/// Navigation property for this users login accounts.
/// </summary>
public virtual ICollection<TUserLogin> Logins { get; } = new List<TUserLogin>();
```

<a name="synchronous-method-removal"></a>

## Replace GetExternalAuthenticationSchemes
The synchronous method `GetExternalAuthenticationSchemes` was removed in favor of an asynchronous version. 1.x projects have the following code in *ManageController.cs*:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore1.1App/AspNetCoreDotNetCore1.1App/Controllers/ManageController.cs?name=snippet_GetExternalAuthenticationSchemes)]

This method appears in *Login.cshtml* too:

[!code-cshtml[Main](../1x-to-2x/samples/AspNetCoreDotNetCore1.1App/AspNetCoreDotNetCore1.1App/Views/Account/Login.cshtml?range=62,75-84)]

In 2.0 projects, use the `GetExternalAuthenticationSchemesAsync` method:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore2.0App/AspNetCoreDotNetCore2.0App/Controllers/ManageController.cs?name=snippet_GetExternalAuthenticationSchemesAsync)]

In *Login.cshtml*, the `AuthenticationScheme` property accessed in the `foreach` loop changes to `Name`:

[!code-cshtml[Main](../1x-to-2x/samples/AspNetCoreDotNetCore2.0App/AspNetCoreDotNetCore2.0App/Views/Account/Login.cshtml?range=62,75-84)]

<a name="property-change"></a>

## ManageLoginsViewModel Property Change
A `ManageLoginsViewModel` object is used in the `ManageLogins` action of *ManageController.cs*. In 1.x projects, the object's `OtherLogins` property return type is `IList<AuthenticationDescription>`. This return type requires an import of `Microsoft.AspNetCore.Http.Authentication`:

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore1.1App/AspNetCoreDotNetCore1.1App/Models/ManageViewModels/ManageLoginsViewModel.cs?name=snippet_ManageLoginsViewModel&highlight=2,11)]

In 2.0 projects, the return type changes to `IList<AuthenticationScheme>`. This new return type requires replacing the `Microsoft.AspNetCore.Http.Authentication` import with a `Microsoft.AspNetCore.Authentication` import.

[!code-csharp[Main](../1x-to-2x/samples/AspNetCoreDotNetCore2.0App/AspNetCoreDotNetCore2.0App/Models/ManageViewModels/ManageLoginsViewModel.cs?name=snippet_ManageLoginsViewModel&highlight=2,11)]