## MSALASPNETSample

this is a sample to showcase how to use MSAL to secure a ASP.NET web application. 

## Project Instructure
- Create Azure AD App
- Include the "redirect Uri" to the AAD App registration
- Add permissions. 
- Update the following sessions from web.config
    - ida:ClientId: The App Id
    - ida:ClientSecret: the App Secret
    - ida:TenantId: tenant name. 

## Secure ASP.NET 4.x web application
ASP.NET uses OWIN middleware to secure your ASP.NET 4.x web application using Azure AD V2.0. please see the below steps

- Create ASP.NET web application from Visual Studio 2019
- Install the following package from NuGet library
```PowerShell
Install-Package Microsoft.Owin.Security.OpenIdConnect
Install-Package Microsoft.Owin.Security.Cookies
Install-Package Microsoft.Owin.Host.SystemWeb
#if you need to use MSAL to get token, you can install MSAL
Install-Package Microsoft.Identity.Client
#if you need to call Graph API, you can install Microsoft.Graph
Microsoft.Graph.Core
Microsoft.Graph
```
- Add Utils\AuthenticationConfig.cs and include the following content to read from web.config
```c#
/************************************************************************************************
using System.Configuration;
using System.Globalization;

namespace WebApp.Utils
{
    public static class AuthenticationConfig
    {
        public const string IssuerClaim = "iss";
        public const string TenantIdClaimType = "http://schemas.microsoft.com/identity/claims/tenantid";
        public const string MicrosoftGraphGroupsApi = "https://graph.microsoft.com/v1.0/groups";
        public const string MicrosoftGraphUsersApi = "https://graph.microsoft.com/v1.0/users";
        public const string AdminConsentFormat = "https://login.microsoftonline.com/{0}/adminconsent?client_id={1}&state={2}&redirect_uri={3}";
        public const string BasicSignInScopes = "openid profile offline_access";
        public const string NameClaimType = "name";
        public static string[] Scopes = { "https://graph.microsoft.com/.default" };

        /// <summary>
        /// The Client ID is used by the application to uniquely identify itself to Azure AD.
        /// </summary>
        public static string ClientId { get; } = ConfigurationManager.AppSettings["ida:ClientId"];

        /// <summary>
        /// The ClientSecret is a credential used to authenticate the application to Azure AD.  Azure AD supports password and certificate credentials.
        /// </summary>
        public static string ClientSecret { get; } = ConfigurationManager.AppSettings["ida:ClientSecret"];

        /// <summary>
        /// The Redirect Uri is the URL where the user will be redirected after they sign in.
        /// </summary>
        public static string RedirectUri { get; } = ConfigurationManager.AppSettings["ida:RedirectUri"];

        public static string AADInstance { get; } = ConfigurationManager.AppSettings["ida:AADInstance"];
        public static string TenantId { get; } = ConfigurationManager.AppSettings["ida:TenantId"];

        /// <summary>
        /// The authority
        /// </summary>
        //public static string Authority = string.Format(CultureInfo.InvariantCulture, AADInstance, "common", "/v2.0");
        public static string Authority = string.Format(CultureInfo.InvariantCulture, AADInstance, TenantId, "/v2.0");
    }
}
``` 
- Add OWIN Startup class
- add the following code to the startup.cs
```C#
using System;
using System.Threading.Tasks;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;
using Microsoft.IdentityModel.Tokens;
using Microsoft.Owin;
using Microsoft.Owin.Security;
using Microsoft.Owin.Security.Cookies;
using Microsoft.Owin.Security.Notifications;
using Microsoft.Owin.Security.OpenIdConnect;
using Owin;
using WebApp.Utils;

[assembly: OwinStartup(typeof(MSALWebApplication4.Startup))]

namespace MSALWebApplication4
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);

            app.UseCookieAuthentication(new CookieAuthenticationOptions());
            app.UseOpenIdConnectAuthentication(
                new OpenIdConnectAuthenticationOptions
                {
                    // Sets the ClientId, authority, RedirectUri as obtained from web.config
                    ClientId = AuthenticationConfig.ClientId,
                    Authority = AuthenticationConfig.Authority,
                    RedirectUri = AuthenticationConfig.RedirectUri,
                    // PostLogoutRedirectUri is the page that users will be redirected to after sign-out. In this case, it is using the home page
                    PostLogoutRedirectUri = AuthenticationConfig.RedirectUri,
                    Scope = OpenIdConnectScope.OpenIdProfile,
                    // ResponseType is set to request the id_token - which contains basic information about the signed-in user
                    ResponseType = OpenIdConnectResponseType.IdToken,
                    // ValidateIssuer set to false to allow personal and work accounts from any organization to sign in to your application
                    // To only allow users from a single organizations, set ValidateIssuer to true and 'tenant' setting in web.config to the tenant name
                    // To allow users from only a list of specific organizations, set ValidateIssuer to true and use ValidIssuers parameter
                    TokenValidationParameters = new TokenValidationParameters()
                    {
                        ValidateIssuer = false // Simplification (see note below)
                    },
                    // OpenIdConnectAuthenticationNotifications configures OWIN to send notification of failed authentications to OnAuthenticationFailed method
                    Notifications = new OpenIdConnectAuthenticationNotifications
                    {
                        AuthenticationFailed = OnAuthenticationFailed
                    }
                }
            );
        }
        private Task OnAuthenticationFailed(AuthenticationFailedNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> notification)
        {
            notification.HandleResponse();
            notification.Response.Redirect("/Error?message=" + notification.Exception.Message);
            return Task.FromResult(0);
        }
    }
}

```
