---
author: "CRH"
date: "2020-04-11"
modified: "2025-08-10"
description: ""
tags: ["keycloak"]
title: "'NAMESPACED' CLAIMS IN KEYCLOAK"
type: "post"
bag: true
draft: true
---

![Keycloak Image](/static/keycloak.png)

 **This is now built into Keycloak!**
 
Keyclock currently handles URI based claim names out of the box. 
 

 Keycloak is an Open Source Identity and Access Management
 application that rivals top IAM SaaS products, including
 [Auth0](https://www.auth0.com).


Something that took me an inordinate amount of time was trying
to add custom 'namespaced' claims to the Keycloak's ```access_token```.

Previously, I've used Auth0 as an IAM; adding a custom claim
with [Auth0](https://www.auth0.com) only worked with 'namespaced' claims --- claims
that were prefixed with a URL. In Auth0's case, ```https://choudeshell.com/role_type``` was considered 'namespaced' claim.

Translating this functionality into Keycloak was actually
much harder than expected. Out of the box, Keycloak allows for many different types of claim mappers -- ways to generate and assign claims to the ```access_token``` or ```id_token```. Generally, all of them behave the same when it comes to defining the claim name. If a given claim name has a ```.``` in it, Keycloak will generate a nested object in
the resulting ```access_token```. 

For example, a claim with the name ```https://choudeshell.com/role_type``` would result in:


```javascript 
{
    "https://choudeshell":
        {
            "com/role_type":"value"
        }
}
```

What I wanted the resulting ```access_token``` to contain was:

```javascript
{
    "https://choudeshell.com/role_type":"value"
}
```

The solution? **Use a Script Mapper**. A Script Mapper allows us to use plain-old Javascript. With the provided objects, we can add any claim, with any name we want, and it isn't effected by the nesting logic.

```javascript
/**
 * Available variables: 
 * user - the current user
 * realm - the current realm
 * token - the current token
 * userSession - the current userSession
 * keycloakSession - the current userSession
 */

token.setOtherClaims("https://choudeshell.com/role_type","value");
```

> Something to keep in mind. The Javascript Script Mapper's script is executed via Java & Nashorn, which means the injected variables are bound both ways. You can even update the user's email address by calling the ```user.setEmail('x')``` function!