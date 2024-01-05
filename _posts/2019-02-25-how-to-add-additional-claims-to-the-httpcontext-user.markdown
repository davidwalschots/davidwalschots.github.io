---
layout: post
title:  "How to add additional claims to the HttpContext.User"
date:   2019-02-25 08:46:08 +0100
categories: .net
---

I recently had to extend the claims coming from an IdentityServer4 authentication authority. The claims provided by the authority did not include sufficient information to determine the application specific roles of the user. Those additional claims had to come from a web service. I had not worked with claims within .NET Core 2+ before, so some investigation was necessary.

<!--more-->

## Unsuccessful attempts

My first attempt was to set the `OnTokenValidated` method which is part of the `JwtBearerEvents`. Even though the method is called after successful authentication, it did not allow me to use dependency injection to use the web service.

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(IdentityServerAuthenticationDefaults.AuthenticationScheme)
        .AddIdentityServerAuthentication(options =>
        {
            options.Event.OnTokenValidated = async context => {
                // Add additional claims here.
            };
        });
}
{% endhighlight %}

Afterwards I found an alternative option, which is to supply `JwtBearerOptions.EventsType` with a type overriding the `OnTokenValidated` method. This would allow for dependency injection to be used as shown below.

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(IdentityServerAuthenticationDefaults.AuthenticationScheme)
        .AddIdentityServerAuthentication(options =>
        {
            options.EventsType = typeof(ClaimsExtender);
        });
}

public class ClaimsExtender : JwtBearerEvents 
{
    public ClaimsExtender(IServiceImpl service)
    {
        // ...
    }
    
    public override async Task TokenValidated(TokenValidatedContext context)
    {
        // Add additional claims here.
    }
}
{% endhighlight %}

Sadly, it turns out that IdentityServer4’s `AddIdentityServerAuthentication` method does not support the `EventsType` property and it [seems unlikely](https://github.com/IdentityServer/IdentityServer4.AccessTokenValidation/issues/109) it will soon be supported.

## Using IClaimsTransformation

Luckily it turns out that ASP.NET Core comes with an `IClaimsTransformation` interface that when implemented and registered can handle dependency injection of services:

{% highlight csharp %}
public class ClaimsExtender : IClaimsTransformation
{
    public ClaimsExtender(IServiceImpl service)
    {
        // ...
    }

    public Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        // Add additional claims here.
        return Task.FromResult(principal);
    }
}
{% endhighlight %}

Within the `Startup` class, add the `ClaimsExtender` as an injectable:

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddTransient<IClaimsTransformation, ClaimsExtender>();
}
{% endhighlight %}

Do note that as indicated in the [documentation](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authentication.iclaimstransformation.transformasync?view=aspnetcore-2.2) the `TransformAsync` method is potentially called multiple times per `AuthenticateAsync` call. Therefore, you should add logic which detects whether or not a `ClaimsPrincipal` already went through the `ClaimsExtender` at an earlier time and if it did simply returns the `principal` without repeating the same modification again.

## Comments

Post your comments [here](https://gist.github.com/davidwalschots/d25a3fb6cae93ad36c04eecd4892f366).
