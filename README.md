# API Key Authentication PoC

Authentication is the process of determining a user/service identity. Authentication can be done with: something you know, something you have, something you are. In the case of API key authentication - something you know. 

In this example it will be implementation without third-party NuGet packages. Basic authentication sample. There are two basic ways to authenticate:

1. **Service wants to only accept requests from authenticated services. It doesn't care which service is calling it. Key is always the same.** Service B has a static Key that Service A must provide using the query string parameter or a request header. Any service that is calling it will have the same key.
2. **Dynamic Keys.** Create different keys for different consumers. Use `X-api-key` in the header or query string parameters (e.g.: `?token=your_token`). First login, create an API key and then use that API key.

We have 2 options to pass API key:

1. Use query string parameter: e.g., `?token=your_token`
2. Use headers: e.g. x-api-key Header

Possible scenarios when user/service is authenticating:

- Don't have a key at all
- Have an invalid key
- Have the correct key

## Approach 1: Using Middleware
*Project: APIKeyAuthentication_Middleware*

Middleware is software that's assembled into an app pipeline to handle requests and responses. Each component: Chooses whether to pass the request to the next component in the pipeline. Can perform work before and after the next component in the pipeline. The ASP.NET Core request pipeline consists of a sequence of request delegates, called one after the other. 

This is more of an generic approach. All API endpoints will be covered by this api key authentication approach. Using middleware for API key authentication involves intercepting HTTP requests and checking for a valid API key before allowing access to the requested resource.

Implementation (ApiKeyAuthMiddleware class): 
- Try to retrieve the API key from the request headers
- If the key is not found return with 401 Unauthorized StatusCode
- If the key extracted from Request is not matching the configured API key respond with 401 Unauthorized StatusCode
- If the API key is successfully validated, it proceeds to the next middleware in the pipeline

We need to add middleware to pipeline: Add service to Program.cs:

```app.UseMiddleware<ApiKeyAuthMiddleware>();```

## Approach 2: Using ApiKeyAuthFilter
*Project: APIKeyAuthentication_AuthFilter*

There is sync and async implementation. With this approach you get more control. Authentication filters let you set an authentication scheme for individual controllers or actions. That way, your app can support different authentication mechanisms for different HTTP resources. AuthorizationFilterContext Class have properties like HttpContext, Filters, Result etc. Result is being used to get or set the result of the request.

2 ways to work with filters:
1. In AddControllers() method add filters. This will add AuthFilter for all controllers in system - same as we have with middleware approach.
<pre>
  builder.Services.AddControllers(x => x.Filters.Add<ApiKeyAuthFilter>()); 
</pre>

2. Apply at controller level. This way we can set it for individual endpoints.

Program.cs:
<pre>
    builder.Services.AddScoped<ApiKeyAuthFilter>();
</pre>
Controllers:
<pre>
   [ServiceFilter(typeof(ApiKeyAuthFilter))]
</pre>

Implementation (ApiKeyAuthFilter class): 
- It attempts to retrieve the API key from the request headers using the specified header name
- If the API key is not found, it sets the response result to an UnauthorizedObjectResult with the message "API Key missing" and returns
- It retrieves the expected API key from configuration and compare it with extracted API key from the request
- If they are not equal, it sets the response result to an UnauthorizedObjectResult with the message "Invalid API Key" and returns

Result is being used to get or set the result of the request. Setting Result to a non-null value inside an authorization filter will short-circuit the remainder of the filter pipeline.
<hr/>

Integration with Swagger for both approaches:
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
