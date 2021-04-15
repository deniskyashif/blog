---
title: "Set a Custom Name Claim Type in Azure AD Secured Web API"
date: 2021-04-06T22:16:29+03:00
draft: false
summary: "Overriding the default name claim type configuration in Microsoft Identity."
description: "Overriding the default name claim type configuration in Microsoft Identity."
tags: ["azure", "csharp", "dotnet"]
images: 
- "/images/posts/2021-04-15-name-claim-azure-ad/azure-ad-logo.jpeg"
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2021-04-15-set-custom-name-claim-type-in-azure-ad-auth.md"
---

In this article, we'll learn how to set a custom **name claim type** to our `ClaimsPrincipal`'s primary identity in ASP.NET Core Web API. Our API uses Azure AD as its identity provider. That will make it possible for us to retrieve the request initiator's name in our controllers' action methods by simply calling

```cs
User.Identity.Name
```

which will return the value of the claim we specified.

## The Defaults

In a standard authentication flow, the user signs in to the client app (1) through Azure AD and obtains an access token as a JWT (2). This token is then used as a bearer token to authenticate the user when calling the Web API (3).

<img src="/images/posts/2021-04-15-name-claim-azure-ad/azure-ad-auth-flow.jpeg" alt="Authentication Flow" />
<small style="display: block; text-align: right;"><em>Source: docs.microsoft.com</em></small>


To authenticate the user, the respective handler decodes, verifies the access token, and extracts the claims its payload. When ASP.NET instantiates the `ClaimsPrincipal` object for our request, it uses those claims to establish its identity. We enable the Azure AD authentication into our HTTP request/response pipeline, by simply adding

```cs
services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
	.AddMicrosoftIdentityWebApi(Configuration.GetSection("AzureAd"));
```

to the application startup. `"AzureAd"` is the name of the respective section in our app settings. For this example, we'll use the `"preferred_username"` claim which is included in the access token's payload issued by Azure AD. We can retrieve its value by calling

```cs
User.FindFirst("preferred_username")?.Value;
```

This will search the `User.Claims` collection for a claim with the provided type and return it if one is found.


Our `ClaimsPrincipal`'s primary identity has a `Name` property. It would make sense to get the value of the `"preferred_username"` claim when we call

```cs
User.Identity.Name
```

but instead, we get `null`, even though the authentication is successful and the `"preferred_username"` claim is available.

<img src="/images/posts/2021-04-15-name-claim-azure-ad/name-null.png" alt="Null Identity Name" />

The reason is that the default name claim type is set to something different and since `User.Claims` don't contain it, we end up with `null`.

<img src="/images/posts/2021-04-15-name-claim-azure-ad/find-default-name.png" alt="Find Default Name Claim" />

## The Solution

We set the default name claim type by using a **different overload** of the `AddMicrosoftIdentityWebApi` method:

```cs
services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(options =>
    {
        Configuration.Bind("AzureAd", options);
        options.TokenValidationParameters.NameClaimType = "preferred_username";
    },
    options => Configuration.Bind("AzureAd", options));
```

Now we can easily retrieve the name of our user who initiated the request:

<img src="/images/posts/2021-04-15-name-claim-azure-ad/name-preferred.png" alt="Populated Identity Name" />

**NB**

The `"preferred_username"` claim's value is mutable and might change over time, hence, it **should not** be used as a primary key or for authorization decisions. There are immutable claims for such tasks. Refer to the [docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/access-tokens) for more details.
