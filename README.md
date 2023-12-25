# API Key Authentication

Authentication is the process of determining a user/service identity. Authentication can be done with: something you know, something you have, something you are. In the case of API key authentication - something you know. 

No third-party NuGet packages, basic authentication sample. There are two basic ways to authenticate:

1. **Service wants to only accept requests from authenticated services. It doesn't care which service is calling it. Key is always the same.** Service B has a static Key that Service A must provide using the query string parameter or a request header. Any service that is calling it will have the same key.
2. **Dynamic Keys.** Create different keys for different consumers. Use `X-api-key` in the header or query string parameters (e.g., `?token=your_token`). First login, create an API key and then use that API key.

We have 2 options to pass API key:

1. Use query string parameter: e.g., `?token=your_token`
2. Use headers: e.g. x-api-key Header

Possible scenarios when user/service is authenticating:

- Don't have a key at all
- Have an invalid key
- Have the correct key

## Approach 1: Using Middleware

Using middleware for API key authentication involves intercepting HTTP requests and checking for a valid API key before allowing access to the requested resource.
Use something designed to store secrets not appSettings.json. 

It's more of an generic approach. You are going to cover everything in your API behind this api key authentication.

Implementation: get key from configuration, compare it with Request x-api-key header and generate adequate StatusCode and response. 
   
Middleware is sequential. Have it before Authorization.
<pre>
app.UseMiddleware<ApiKeyAuthMiddleware>();
</pre>

## Approach 2: Using ApiKeyAuthFilter

You get more control. AuthorizationFilter. In AuthorizationFilterContext there are Filters, Result, HttpContext attributes. Just set up context.Result and return.
ApiKeyAuthFilter => Async or NotAsync implementation.

2 ways to work with filters:
1. In AddControllers() add filters - this will be added for all controllers in system.
<pre>
  builder.Services.AddControllers(x => x.Filters.Add<ApiKeyAuthFilter>()); 
</pre>

2. To apply at controller level:
Program.cs:
<pre>
    builder.Services.AddScoped<ApiKeyAuthFilter>();
</pre>
Controllers:
<pre>
[ServiceFilter(typeof(ApiKeyAuthFilter))]
</pre>


Integration with Swagger:
<pre>
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("ApiKey", new OpenApiSecurityScheme
    {
        Description = "The API Key to access the API.",
        Type = SecuritySchemeType.ApiKey,
        Name = "x-api-key",
        In = ParameterLocation.Header,
        Scheme = "ApiKeyScheme"
    });
    var scheme = new OpenApiSecurityScheme
    {
        Reference = new OpenApiReference
        {
            Type = ReferenceType.SecurityScheme,
            Id = "ApiKey"
        },
        In = ParameterLocation.Header
    };
    var requirement = new OpenApiSecurityRequirement
    {
        { scheme, new List<string>() }
    };
    c.AddSecurityRequirement(requirement);
});
</pre>
