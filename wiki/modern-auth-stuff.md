---
title: "Modern Authentication"
date: 2023-12-23
description: "informationa about how modern authentication works and how to set it up"
tags: authentication, code
importance: 5
abstract: ""
---

# Disclaimer

Page will be refined later. For now, this is just my quick notes on how I got authentication & authorization working on a simple endpoint. First version was Client Credentials Flow, which only required client ID and client secret; I've since added Authorization Flow, which does require the user to log in somewhere.

None of this should be considered "This is how we should do this"; it's just "this is how I did it".

# Orchard

- [Homepage](https://orchardcore.net/)
- [Docs for OpenID stuff](https://docs.orchardcore.net/en/main/docs/reference/modules/OpenId/)
- Enable OpenID Connect
- OpenID Connect | Settings | Authorization server
  - Endpoints
    - Enable Token Endpoint
    - Enable Authorization Endpoint (needed for Authorization Flow)
    - Enable User Info Endpoint (needed for Authorization Flow)
  - Flows
    - Allow Authorization Code Flow (typically used for end-user authentication & authorization)
    - Allow Client Credentials Flow (typically used for machine-to-machine authentication & authorization)
  - Advanced options
    - turn on `Disable Access Token Encryption` if you want to be able to verify the token in https://jwt.io ; might also be needed to avoid having to fuss with certificates etc
- OpenID Connect | Management | Scopes
  - Add whatever custom scope you find interesting
    - Record the scope name
    - In "Additional resources", add some text that we'll later refer to as "audience"
  - Add a scope called `roles`
    - In "Additional resources", add the same audience
    - Needed for Authorization Flow
  - Add a scope called `profile`
    - In "Additional resources", add the same audience
    - Only needed if you want the web service to have access to the user's name and such; if you're not using that, you can leave it out.
  - Other predefined scopes:
    - `email`
    - `phone`
    - Like with `profile`, this is only needed if you want the web service to have access to that user's info
- OpenID Connect | Management | Applications
  - Add an application
    - Type: "Confidential client"
    - Record whatever you're using for client secret
    - Allow Authorization Flow
    - Allow client Credentials Flow
    - Allowed Scopes:
      - Whatever custom scopes you want (ie whatever was added above)
      - `roles` (needed for Authorization Flow)
      - `profile` , `email`, and `phone` if desired

# C#

- appSettings.Development.json
  - I ended up with two sections - OAuth and JwtSettings. We can add more as we go if we want to add additional ways to authenticate (e.g. Microsoft Entra ID).
  - If you only have a single value for `Scope`, you can just enter it by itself instead of creating an array.
  - All scopes listed must have been included in the application you created in Orchard.
  - Sample JSON:

```` json
  "JwtSettings": {
    "ClientId": "<application name>",
    "ClientSecret": "<client secret from when you added the application>",
    "Issuer": "<url of Orchard>",
    "Authority": "<url of Orchard>",
    "Scope": [ "<custom scope 1>", "<custom scope 2>", "roles", "profile", "email", "phone" ],
    "Audience": "<audience you added in Additional resources when creating the scope>"
    "RequireHttpsMetadata": "false",
  },
  "OAuth": {
    "CallbackPath":  "/oauth/callback",
    "ClientId": "<application name>",
    "ClientSecret": "<client secret from when you added the application>",
    "AuthorizationEndpoint": "http://localhost:5000/connect/authorize",
    "TokenEndpoint": "http://localhost:5000/connect/token",
    "UserInformationEndpoint":  "http://localhost:5000/connect/userinfo",
    "Scope": [ "testscope", "roles" ],
    "SaveTokens": "true"
  },
````

- Program.cs
```` csharp
builder.Services.AddAuthorization();
builder.Services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = "OAuth";
        options.RequireAuthenticatedSignIn = false;
    })
    .AddCookie()
    // Earlier sample from when I got Client Credentials flow working
    //.AddJwtBearer(JwtBearerDefaults.AuthenticationScheme,
    //    options =>
    //    {
    //        builder.Configuration.Bind("JwtSettings", options);
    //    })
    .AddOAuth("OAuth", options =>
    {
        builder.Configuration.Bind("OAuth", options);

        // .AddOAuth is super generic and the general expectation is that you'll normally use a more specific thing
        // (e.g. .AddFacebook(..)) instead of calling this directly... Which means we need to be extra explicit and
        // tell the code exactly where to find what info. In this case, we need to tell it that the role info is in
        // a field called "roles".
        options.ClaimActions.MapJsonKey(ClaimTypes.Role, "roles");

        options.Events = new OAuthEvents()
        {
            OnCreatingTicket = async context =>
            {
                // Go back to the user endpoint and ask about the user associated with the token we've been given
                var request = new HttpRequestMessage(HttpMethod.Get, context.Options.UserInformationEndpoint);
                request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", context.AccessToken);
                request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

                var response = await context.Backchannel.SendAsync(request, context.HttpContext.RequestAborted);
                response.EnsureSuccessStatusCode();

                using var user = JsonDocument.Parse(await response.Content.ReadAsStringAsync());

                // This call is needed to process the JSON and populate our info about the user - basically, it's where
                // we actually use the mapped JSON fields defined above.
                context.RunClaimActions(user.RootElement);
            }
        };
    });
````
Notice that "OAuth" was used both as the name of a section in the config file and in the call to Bind. I believe the name can be arbitrary, they just have to match. There's also an extra JwtSettings section in the config file but the corresponding `AddJwtBearer` call has been commented out; my belief is that we can use both, I just don't know how they might interact and don't want to mess around enough to find out.
- Add a class:
```` csharp
public class AuthFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        if (!context.ApiDescription
                .ActionDescriptor
                .EndpointMetadata
                .OfType<AuthorizeAttribute>()
                .Any())
        {
            return;
        }

        operation.Security = new List<OpenApiSecurityRequirement>
        {
            new OpenApiSecurityRequirement
            {
                [new OpenApiSecurityScheme
                {
                    Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "OpenId" }
                }] = new List<string>()
            }
        };
    }
}
````
- Back in Program.cs to make Swagger UI look a little better
```` csharp
builder.Services.AddSwaggerGen(c =>
{
    var authority = "http://localhost:5000/";
    c.AddSecurityDefinition("OpenId", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Name = HeaderNames.Authorization,
        In = ParameterLocation.Header,
        Scheme = "Bearer",
        Flows = new OpenApiOAuthFlows
        {
            AuthorizationCode = new OpenApiOAuthFlow
            {
                AuthorizationUrl = new Uri($"{authority}connect/authorize"),
                TokenUrl = new Uri($"{authority}connect/token"),
                Scopes = new Dictionary<string, string>
                {
                    {
                        "openid", "openid"
                    },
                },
            },
        },
        OpenIdConnectUrl = new Uri($"{authority}.well-known/openid-configuration"),
    });
    c.OperationFilter<AuthFilter>();
});
````
Is all of this useful? I honestly don't know. It might be that none of it is aside from the OperationFilter. This is really just about the Swagger page, that's all.
- Your controller
```` csharp
[Authorize(Roles = "Administrator")]
````
The Roles thing is optional; if you do use Roles to control access to an endpoint, you'll need to make sure you've updated your auth config to get role info back & have called the MapJsonKey and RunClaimActions stuff.

# Postman, Client Credentials flow

- Create an environment, and make sure you're operating in it (there's a dropdown at top right once you've created an environment and saved it)
  - Also add a variable `jwt_token` so you don't have to fuss with copying & pasting the token later
- Optional: Create a collection
  - Create a variable in the collection called `baseUrl`, and set it to the base URL of Orchard (in my case, `http://localhost:5000`)
- Create a new request
    - Method: POST
    - Url: <url of Orchard>/connect/token
    - Body: set to `x-www-form-urlencoded` and add variables:
        - client_id: <name of the application you added in Orchard>
        - client_secret: <client secret you recorded when you set up the application in Orchard>
        - grant_type: `client_credentials`
        - scope: <name of the scope you created in Orchard>
    - Tests: Add the following script:
````
const response = pm.response.json();

pm.environment.set("jwt_token", response.access_token);
````
- This step isn't technically necessary but it'll make life a lot easier if you use Postman to do further testing
  - Now hit Send and ensure you get back a token. Copy the contents of the `access_token` field from the response.
- Create a new request for whatever endpoint you've created that needs authorization
  - Authorization type: "Bearer Token"
  - Token: `{{jwt_token}}` (should be pulled automatically from what you got from the request to create a token)

# Postman, Authorization Code flow (not actually working yet but useful for testing auth)

- Create an environment
- Optionally, create a Collection as well, with a variable `baseUrl` set to the base URL of Orchard (e.g. `http://localhost:5000`)
- Create a new query, and set the URL to whatever endpoint you want to use that requires OAuth
- On the Authorization tab:
  - Left panel
    - Type: OAuth 2.0
    - Add authorization data to: Request headers
  - Right panel
    - If you've already got a token (that hasn't expired) you can just use it
    - Otherwise, scroll down a bit and enter the information you need
    - If you use browser auth, you'll want to check your browser window when you try to generate a new key because it needs to be able to open popups in order for the browser auth to work


# Swagger (if you prefer to test there)

- Run your C# web app and open Swagger
- On the Swagger page, click Authorize. You should have a box where you can paste the access token from above
- Do whatever it is you feel like doing
