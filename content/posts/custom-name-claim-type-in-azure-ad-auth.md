---
title: "Set a Custom Name Claim Type in Azure AD Secured Web API"
date: 2021-04-06T22:16:29+03:00
draft: true
summary: "A how-to guide on overriding the default configuration."
---

In this article, we'll learn how to set a custom **name claim type** to our `ClaimsPrincipal` when authenticating requests in ASP.NET Web API using Azure AD as an identity provider. That will let us to retreive the request initiator's username by simply calling

```cs
User.Identity.Name
```

in our controllers' action methods and it will return the value of the claim we specified.

## The Default Behavior

In a standard authentication flow, the client logs in (1) using Azure AD and receives an access (bearer) token as a JWT (2), which it includes as a header in the requests to the our Web API (3).

<img src="/images/posts/azure-ad/azure-ad-auth-flow.jpeg" alt="Authentication Flow" />
<small style="display: block; text-align: right;"><em>Source: docs.microsoft.com</em></small>


To authenticate the request, the access token gets decoded and its claims are extracted from its payload. When ASP.NET instantiates the `ClaimsPrincipal` object for our request, it uses those claims to populate some of its fields. We enable the Azure AD authentication, by simply adding

```cs
services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
	.AddMicrosoftIdentityWebApi(Configuration.GetSection("AzureAd"));
```

to our our application startup. `"AzureAd"` is the name of the respective section our app settings config. In this example, we'll use the `"preferred_username"` claim which is included by default in the payload of the access tokens that Azure AD issues. We can retreive its value by simple calling

```cs
User.FindFirst("preferred_username")?.Value;
```

This will search the `User.Claims` collection for a claim with the provided type and return it if found.


The claims identity of the request -`User.Identity` implements the `IIdentity` interface which contains the `Name` property:

```cs
public interface IIdentity
{
    ...
    string? Name { get; }
}
```

It would make sense to get the value of the `"preferred_username"` claim when we call

```cs
User.Identity.Name
```

but instead we get `null`, even though the authentication is successful and the `"preferred_username"` claim is available.

<img src="/images/posts/azure-ad/name-null.png" alt="Authentication Flow" />

The reason is that the default name claim type is set to something different and since `User.Claims` doesn't contain this claim, `User.Identity.Name` returns in `null`.

<img src="/images/posts/azure-ad/find-default-name.png" alt="Find Default Name Claim" />

## The Solution

We set the default name claim type by using a different overload of the `AddMicrosoftIdentityWebApi` method:

```cs
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(options =>
    {
        Configuration.Bind("AzureAd", options);
        options.TokenValidationParameters.NameClaimType = "preferred_username";
    },
    options => Configuration.Bind("AzureAd", options));
```

Now we can retreive the name of our `ClaimsPrincipal`

<img src="/images/posts/azure-ad/name-preferred.png" alt="Authentication Flow" />

**NB**

The `"preferred_username"` claim's value is mutable and might change over time, hence, it **shouldn not** be used for authorization decisions or as an identifier. Refer to the [docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/access-tokens) for more info about the access token claims.
