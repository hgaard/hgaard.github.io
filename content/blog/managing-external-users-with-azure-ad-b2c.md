+++
author = "Jakob Højgaard"
categories = ["Azure", "Azure AD", "Azure AD B2C", "OpenIdConnect", "Authentication"]
date = 0001-01-01T00:00:00Z
description = ""
draft = false
image = "/images/2016/04/basics_of_auth_in_aad-1.png"
slug = "managing-external-users-with-azure-ad-b2c"
tags = ["Azure", "Azure AD", "Azure AD B2C", "OpenIdConnect", "Authentication"]
title = "Managing external users with Azure AD B2C"

+++

The last few weeks I have been working with a customer to implement Azure AD B2C login for their internal systems and I thought I might share my experience with you.

###What is Azure AD B2C
First of all I might need to explain a few things, like what B2C is (assuming here that you know what Azure and AD are) and what kind of problem I am solving with it. First of all the name B2C simply means business to consumer and is in preview with another new variant of AD called B2B (Business to Business). It is intended to make it easier to set up authentication with multiple social identity providers like Facebook, Google or LinkedIn as easy as can be. But B2C also provides a way to sign up users without a social identity by providing a local account type - which in turn really is just a modified version of an AD account. For the local account B2C provides all the necessary plumbing like signup, password and user management, such that the consuming application never need to store any sensitive information. You can find a lot more information in the B2C documentation [here](https://azure.microsoft.com/en-us/documentation/services/active-directory-b2c/).

###The setup
Very good then, so the customer I'm working with has some internal applications that external partners also needs access to, to cooperate with them. The applications range from web apps and web api's to mobile apps. This in essence make 3 different scenarios I had to solve: 

* Simple login to web app
* Call web api from web app on behalf of user
* Login from mobile app


In this post i will describe how we solved the first two scenarios and I might write about the latter in another post when I get around to actually writing the code. As I mentioned, the setup required internal as well as external users granted access to the applications. B2C only provides a way to integrate identities from other (social media) identity provides and the build in users. It does not provide any way to synchronize the B2C AD with an AD through technology like Azure AD Connect. Nor does it provide a way to federate with other AD based identity providers like "regular" Azure AD (which by the way is called B2E - or business to employee) or internal AD through ADFS. Thus the applications I work on need to know 2 identity providers. External user resides in B2C and internal users in an B2E existing instance. For info about how to setup B2C and the concepts of policies, please visit the [B2C documentation](https://azure.microsoft.com/en-us/documentation/services/active-directory-b2c/). The getting started sections are excellent. 

Just to recap in a simple authentication scenario, have a look at the illustration here[^1]
![Azure AD login scenario](/content/images/2016/04/basics_of_auth_in_aad.png)

###Initial assumption

I'm already using Owin middleware for authentication, so I thought that I might just configure the middleware with another OpenID Connect idp in the Owin startup configuraion. But since it was not obvious to me how to distinguish the 2 when logging in, I decided to add the internal idp as a WS-Federation idp. This turned out to be a feasible solution for login with the web applications, since I could login with identities from both idp and have the application work as expected. All good, except that the application needs to make web api calls on behald of the user. Then I remembered how bad WS-Fed is for the modern web. WS-Fed uses SAML tokens and these won't work with the bearer token authentication in the web api. Back to the drawing board, or rather, do more googling.


###OpenId Connect all the way

It turns out the setting up Owin with multiple idp's with the same protocol is not all that dificoult, actualy it's straight forward, they just have to be given a ``AuthenticationType`` which is just s string. For simplicity I'm just using the tenant uri.

```csharp
app.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
{
    // Idp identifier 
    AuthenticationType = Auth.Config.ExternalUsersTenant,
    ClientId = Config.ExternalUsersClientId,
    RedirectUri = Auth.Config.RedirectUri,
    PostLogoutRedirectUri = Auth.Config.RedirectUri,
    Notifications = new OpenIdConnectAuthenticationNotifications
    {
        AuthenticationFailed = OnAuthenticationFailed,
        RedirectToIdentityProvider = OnRedirectToIdentityProvider,
        AuthorizationCodeReceived = OnAuthorizationCodeReceived
    },
    Scope = "openid offline_access",
   
    TokenValidationParameters = new System.IdentityModel.Tokens.TokenValidationParameters
    {
        NameClaimType = "name",
    }
});
```

And when logging in the same ``AuthenticationType`` is used to point out the idp when calling 'Challenge'

```csharp
if (loginType == AuthType.External)
{
    context.GetOwinContext().Authentication.Challenge(
        new AuthenticationProperties(
            new Dictionary<string, string>
            {
                {Config.PolicyKey, Config.SignInByEmailPolicyId}
            })
        {
            RedirectUri = "/",
            AllowRefresh = true,
            IsPersistent = true
        }, Config.ExternalUsersTenant); // Identifies idp
}
else if (loginType == AuthType.Internal)
{
    context.GetOwinContext()
        .Authentication.Challenge(new AuthenticationProperties
        {
            RedirectUri = "/",
            AllowRefresh = true,
            IsPersistent = true
        }, Config.InternalUsersTenant);
}
```

###Experimental problems
Excellent, now I could login with credentials from both the B2C AD and the B2E one. Yeah, well not really. It turns out that Microsoft is also rolling out an AD v2 app model and that the library for interacting with AD - the Active Directory Access Library (ADAL) in the release with that supports B2C, is hardcoded to use v2 app model. All good except that applications that are already registered in the B2E AD are in effect not visible to the version 2 of the api. They need to be re-registered in the [new application portal](https://aadguide.azurewebsites.net/integration/registerappv2/) (which is also in preview.) in the in order login through the B2C AD. 
Hmmm, I could have started the registration in the new portal, but I decided I had enough places to maintain configuration for now (B2C ADd's needs to be configured in both the old azure [management portal](https://manage.windowsazure.com) and the new ([just portal portal](https://portal.azure.com)). So I decided to have namespaces come to the rescue and use both the new (experimental) ADALv4 library and the old ADALv2. It bloats the code quite a bit in certain places, but who knows ADALv4 might have support for the old api once it gets released. I'll see if I made a wise choice. But if all goes wrong, registering the app in the v2 portal is still an option.

###Producing Bearer tokens
In order to call the web api it's necessary to provide an access token. The acquisition of the access token follows after the sign in flow and the whole protocol is illustrated in the figrure below[^1]

![Acquiring an access token](/content/images/2016/04/web_app_to_web_api.png)

For this to work with both B2C and B2E I'll have to make use of both the v2 and v4 of the ADAL library again. Here though, the difference is more profound since the version 4 has made it easier for the consumer of the api to acquire the access token. Instead of keeping track of validity of id token to determine if a refresh token should be used it is all wrapped in one method ```AcquireTokenSilent()```, which will then only fail if the user needs to reenter credentials i.e. refresh token is lost or has expired.

On the web api side the Owin pipeline needs to be configured to consume OauthBearer tokens form the 2 idp's. As with configuring the OpenIdConnect parts the idp's can be distinguished by setting the ``AuthenticationType`` accordingly.

```csharp
 public void ConfigureAuth(IAppBuilder app)
        {
            // B2E AAD
            app.UseOAuthBearerAuthentication(new OAuthBearerAuthenticationOptions
            {
                AccessTokenFormat = new JwtFormat(
                      new TokenValidationParameters
                      {
                           ValidAudience = Config.InternalUsersClientId
                      }, 
                      new OpenIdConnectCachingSecurityTokenProvider(
                           string.Format(Config.AadInstance,
                             Config.InternalUsersTenant, 
                             string.Empty, 
                             Config.DiscoverySuffix, 
                             string.Empty))),
                 AuthenticationType = Config.InternalUsersTenant
            });

            // B2C AAD
            app.UseOAuthBearerAuthentication(new OAuthBearerAuthenticationOptions
            {
                AccessTokenFormat = new JwtFormat(
                      new TokenValidationParameters
                      {
                           ValidAudience = Config.ExternalUsersClientId
                      }, 
                      new OpenIdConnectCachingSecurityTokenProvider(
                           String.Format(Config.AadInstance, 
                               Config.ExternalUsersTenant, 
                               "v2.0", Config.DiscoverySuffix, "?p="
                               Config.CommonPolicy))),
                AuthenticationType = Config.ExternalUsersTenant
            });
        }
```
Note that the metadata string provided for the 2 idp'S differ in that the B2C one points to version 2 of the AD api and that it also needs to reference a policy. 

 
###A working sample app
I have modified and combined a few of the sample apps provided by the Azure team and published it to [Github](https://github.com/hgaard/azure-b2c-multitenant-demo) 
It has a web app and a web api and illustrates the scenario described here
Note that there are a few classes inherited form ms examples on B2C. These are necessary because ADAL is not yet fully updated with the necessary changes for B2C.
A small note of caution , it is extremely important to use the latest version of Microsoft.IdentityModel.Protocol.Extensions, since the older versions will produce wrong query strings when signing in with B2C.


### In conclusion
Even though it surely has a few rough edges, the prospects are definetly there.
I know there are several and more mature alternatives like [Auth0](http://auth0.com) and [Identity Server](https://github.com/IdentityServer/IdentityServer3) from Thinktecture. They are both great products, but in this case the sales point for the customer was the ability to add new external users to their systems without having to think about how to manage sensitive information like user passwords, since this is managed 100% by Azure AD B2C. As mentioned earlier I have not yet had the chance to work on extending the solution to mobile apps, but it's in the pipeline, so i might update you on that then.

A note for the interested. The pre-release of B2C AD does not yet support Single Page Application (SPA) like Angular apps. Luckily for me this was not necessary this time :). I hope it will be high in the feature list for the team, since most of the projects we do these days are in that realm. 

**Resources**

* Azure AD B2C [documentation](https://azure.microsoft.com/en-us/documentation/services/active-directory-b2c/)
* Azure quick starts on [Github](https://github.com/AzureADQuickStarts/)
* Azure AD App v2 [registration](https://azure.microsoft.com/en-us/documentation/articles/active-directory-appmodel-v2-overview/)
* Authentication scenarios for Azure AD [explained](https://azure.microsoft.com/en-gb/documentation/articles/active-directory-authentication-scenarios/) 
* JWT [ Debugger](https://jwt.io/#debugger-io)

[^1]: Image borrowed from ["Authentication scenarios for Azure AD" explained](https://azure.microsoft.com/en-gb/documentation/articles/active-directory-authentication-scenarios/) 