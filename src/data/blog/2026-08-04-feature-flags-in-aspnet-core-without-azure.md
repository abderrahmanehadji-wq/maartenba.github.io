---
layout: post
title: "Feature Flags in ASP.NET Core Without Azure"
pubDatetime: 2026-08-04T08:00:00Z
comments: true
published: false
categories: ["post"]
tags: ["General", ".NET", "dotnet", "web"]
author: Maarten Balliauw
---

Feature flags let you toggle functionality at runtime without redeploying. The `Microsoft.FeatureManagement` is a library that helps with implementing feature flags. It library lives under the [Azure App Configuration docs on Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference), so most developers (including myself, until recently) assume it needs Azure. As it turns out, the whole thing is built on `IConfiguration`, which means `appsettings.json`, environment variables, or any configuration provider you already have will do.

I've been using it in [SpeakerTravel](https://speaker.travel) for a while and kept finding capabilities I didn't expect from what looks like a simple boolean flag library. Percentage-based rollouts with consistent user targeting, conditional middleware pipelines, variant-driven configuration objects, all running on a JSON file or environment variables.

## How does it work?

After installing the `Microsoft.FeatureManagement.AspNetCore` NuGet package into your project, you can register the feature flag services in your `Program.cs`:

```csharp
builder.Services.AddFeatureManagement();
```

If you're in Blazor or need per-request evaluation, use `AddScopedFeatureManagement()` instead so the feature state is evaluated once per request rather than once per singleton lifetime.

Feature flags can be defined in your `IConfiguration`, where the library looks for a `feature_management` section. When using `appsettings.json`, a simple boolean flag will look like this:

```json
{
  "feature_management": {
    "feature_flags": [
      {
        "id": "NewDashboard",
        "enabled": true
      }
    ]
  }
}
```

In your code, you can then inject `IVariantFeatureManager` wherever you need to check this flag and light up specific functionality:

```csharp
public class DashboardController : Controller
{
    private readonly IVariantFeatureManager _featureManager;

    public DashboardController(IVariantFeatureManager featureManager)
    {
        _featureManager = featureManager;
    }

    public async Task<IActionResult> Index()
    {
        if (await _featureManager.IsEnabledAsync("NewDashboard"))
        {
            return View("NewDashboard");
        }

        return View();
    }
}
```

That's really all you need for a basic on/off flag.

## Beyond on/off feature flags

Beyond simple booleans, the library ships with default filters you can apply to a flag. A time window filter, for example, lets you schedule a feature to be active during a specific period without deploying anything:

```json
{
  "id": "HolidayBanner",
  "enabled": true,
  "conditions": {
    "client_filters": [
      {
        "name": "Microsoft.TimeWindow",
        "parameters": {
          "Start": "2026-12-20T00:00:00Z",
          "End": "2027-01-02T00:00:00Z"
        }
      }
    ]
  }
}
```

Set a start and end-date, and you can check the `"HolidayBanner"` in your code to see whether a holiday banner should be displayed.

You can write custom `IFeatureFilter` implementations for your own logic (time-of-day, tenant ID, request header, whatever you need). The built-in filters cover the common cases, but you can extend when needed.

## The `<feature>` tag helper in Razor views

This one I did not know. Instead of injecting `IVariantFeatureManager` into your view and wrapping things in `@if` blocks, there's a tag helper you can use to conditionally render content based on the presence of a feature flag:

```html
<feature name="NewDashboard">
    <div class="dashboard-v2">
        <!-- new dashboard content -->
    </div>
</feature>

<feature name="NewDashboard" negate="true">
    <div class="dashboard-classic">
        <!-- old dashboard, shown when flag is off -->
    </div>
</feature>
```

Flags aren't just on or off, they can also have *variants* (I'll cover those in more detail below). The tag helper handles that too, so you can render different markup depending on which variant a user gets:

```html
<feature name="Checkout" variant="StreamlinedFlow">
    <!-- streamlined checkout UI -->
</feature>
```

Less ceremony than `@if/else`, although I could also understand why you'd keep using those in views to have the conditional logic stand out more.

## Consistent percentage rollouts without a feature flag service

This is where it gets interesting. Sometimes, you may want to roll out new functionality to a percentage of your users. Microsoft's library ships with two percentage-based filters, and the difference between them matters a lot.

`Microsoft.Percentage` is a random percentage (that may return a different on/off state for a flag per evaluation). A user at a 50% rollout might see the feature on one request and not the next, and when not using `AddScopedFeatureManagement()` they may even see the flag evaluated differently within one request. This may be fine for non-user-facing feature toggles, but probably bad to use these for anything a user can see.

`Microsoft.Targeting` solves this properly. It hashes the user ID (or whatever identifier you provide) into a deterministic bucket. The same user (or identifier) will see the same result on every evaluation. The hash is server-side, computed from the user identity through `ITargetingContextAccessor`, and the default ASP.NET Core implementation pulls from `HttpContext.User.Identity.Name`.

Here's a targeting configuration that gives your beta testers full access and rolls out to 10% of everyone else:

```json
{
  "id": "NewCheckout",
  "enabled": true,
  "conditions": {
    "client_filters": [
      {
        "name": "Microsoft.Targeting",
        "parameters": {
          "Audience": {
            "Groups": [
              {
                "Name": "BetaTesters",
                "RolloutPercentage": 100
              }
            ],
            "DefaultRolloutPercentage": 10
          }
        }
      }
    ]
  }
}
```

For anonymous users, you'd implement and register your own `ITargetingContextAccessor` that provides a stable identifier (a cookie, a session ID, whatever makes sense for your app).

## Conditional middleware pipelines

Most teams don't think of middleware as something you can toggle at runtime. But it may make sense. One example could be when you want to more aggressively rate limit based on a feature flag:

```csharp
app.UseMiddlewareForFeature<RateLimitingMiddleware>("AggressiveRateLimiting");
```

In this case, the rate limiting middleware only runs when the `AggressiveRateLimiting` flag is on.

For more complex scenarios, you can branch the entire pipeline:

```csharp
app.UseForFeature("DiagnosticsMode", appBuilder =>
{
    appBuilder.UseMiddleware<DetailedTimingMiddleware>();
    appBuilder.UseMiddleware<RequestBodyLoggingMiddleware>();
});
```

If you are using ASP.NET MVC or Razor Pages, you can also use the `[FeatureGate("Flag")]` attribute on a controller or action (or page/handler) as a quick way to hide entire endpoints. Your app will return a 404 when the endpoint is requested while the flag is off.

## Variants as configuration objects

A feature flag variant isn't limited to "on" or "off" or even string labels. Each variant can return a full `IConfigurationSection`, so you can attach an entire JSON block to it.

```json
{
  "id": "Checkout",
  "enabled": true,
  "variants": [
    {
      "name": "Classic",
      "configuration_value": {
        "Steps": 4,
        "ShowGiftWrapping": true,
        "PaymentProviders": ["stripe", "paypal"]
      }
    },
    {
      "name": "Streamlined",
      "configuration_value": {
        "Steps": 2,
        "ShowGiftWrapping": false,
        "PaymentProviders": ["stripe"]
      }
    }
  ],
  "allocation": {
    "default_when_enabled": "Classic",
    "percentile": [
      { "variant": "Streamlined", "from": 0, "to": 20 }
    ]
  }
}
```

In this example, feature flags are used for A/B testing. 20% of users will see a new streamlined checkout experience (with options specific to that variant), others will remain on the default. In your code, you'll only have to check for the `"Checkout"` feature flag, and the feature management libraryu will take care of injecting each variant's configuration section into the ASP.NET Core configuration system.

## Wrapping up

The `Microsoft.FeatureManagement` library is useful for soft launches, kill switches, trunk-based development where unfinished features are deployed but remain hidden in production, and gradual rollouts. It plays nicely with any `IConfiguration` source, so if you later decide you do want Azure App Configuration, Consul, HashiCorp Vault, or whatever to manage your flags centrally, you can swap the configuration provider without needing to change the feature management code.

I would also recommend reading through the repo and docs for more details about various aspects of the library:
* [Microsoft.FeatureManagement on GitHub](https://github.com/microsoft/FeatureManagement-Dotnet)
* [Official docs](https://learn.microsoft.com/en-us/azure/azure-app-configuration/feature-management-dotnet-reference)