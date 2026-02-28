# API Key Authentication PoC

A proof-of-concept solution demonstrating two different approaches to implement API Key authentication in ASP.NET Core 8 Web APIs without third-party packages.

## Table of Contents
- [Overview](#overview)
- [Solution Structure](#solution-structure)
- [Approach 1: Middleware](#approach-1-using-middleware)
- [Approach 2: Authorization Filter](#approach-2-using-authorization-filter)
- [Comparison: Middleware vs Filter](#comparison-middleware-vs-filter)
- [Other API Key Authentication Approaches](#other-api-key-authentication-approaches)
- [Other Authentication Methods](#other-authentication-methods)
- [Configuration](#configuration)
- [Swagger Integration](#swagger-integration)

---

## Overview

**Authentication** is the process of determining a user/service identity. It answers the question: "Who are you?"

Authentication can be based on:
- **Something you know** (password, API key, PIN)
- **Something you have** (security token, smart card, phone)
- **Something you are** (fingerprint, face recognition, biometrics)

API Key authentication falls under "something you know" - the client must possess a secret key to access the API.

### Ways to Pass API Key

| Method | Example | Pros | Cons |
|--------|---------|------|------|
| **Header** | `x-api-key: your-key` | Clean URL, not logged in browser history | Requires header manipulation |
| **Query String** | `?api_key=your-key` | Easy to use | Visible in logs, browser history |
| **Cookie** | `Cookie: api_key=your-key` | Automatic on subsequent requests | Vulnerable to CSRF |

### Possible Authentication Scenarios
- No key provided ? 401 Unauthorized
- Invalid key provided ? 401 Unauthorized  
- Valid key provided ? Request proceeds

---

## Solution Structure

This solution contains **two projects**, each demonstrating a different approach:

| Project | Approach | Scope | Use Case |
|---------|----------|-------|----------|
| `APIKeyAuthentication_Middleware` | Custom Middleware | Global (all endpoints) | Protect entire API uniformly |
| `APIKeyAuthentication_AuthFilter` | Authorization Filter | Selective (per endpoint/controller) | Fine-grained control |

---

## Approach 1: Using Middleware

**Project:** `APIKeyAuthentication_Middleware`

### What is Middleware?

Middleware is software assembled into the ASP.NET Core request pipeline. Each middleware component:
- Chooses whether to pass the request to the next component
- Can perform work before and after the next component

### How It Works

```
Request ? [ApiKeyAuthMiddleware] ? [Other Middleware] ? Controller ? Response
              ?
         Validates API Key
         (401 if invalid)
```

### Implementation Details

The `ApiKeyAuthMiddleware` class:
1. Intercepts every incoming HTTP request
2. Extracts the API key from the `x-api-key` header
3. Compares it against the configured key in `appsettings.json`
4. Returns 401 Unauthorized if missing or invalid
5. Calls `next(context)` to proceed if valid

### Registration

```csharp
app.UseMiddleware<ApiKeyAuthMiddleware>();
```

### When to Use Middleware
- All endpoints require the same authentication
- You want authentication to run before MVC/routing
- Simple, uniform security policy across the API

---

## Approach 2: Using Authorization Filter

**Project:** `APIKeyAuthentication_AuthFilter`

### What is an Authorization Filter?

Authorization filters run after model binding but before action execution. They implement `IAuthorizationFilter` and allow fine-grained control over which endpoints require authentication.

### How It Works

```
Request ? Routing ? [Model Binding] ? [ApiKeyAuthFilter] ? Action Method
                                            ?
                                    Validates API Key
                                    (401 if invalid)
```

### Implementation Details

The `ApiKeyAuthFilter` class:
1. Implements `IAuthorizationFilter`
2. Extracts the API key from request headers
3. Sets `context.Result` to `UnauthorizedObjectResult` if validation fails
4. Setting `Result` short-circuits the pipeline (action never executes)

### Registration Options

**Option 1: Global (all controllers)**
```csharp
builder.Services.AddControllers(x => x.Filters.Add<ApiKeyAuthFilter>());
```

**Option 2: Per-endpoint (selective)**
```csharp
// Program.cs
builder.Services.AddScoped<ApiKeyAuthFilter>();

// Controller
[ServiceFilter(typeof(ApiKeyAuthFilter))]
[HttpGet("protected")]
public IActionResult GetProtectedData() { }

[HttpGet("public")]  // No filter - publicly accessible
public IActionResult GetPublicData() { }
```

### When to Use Filters
- Different endpoints need different authentication
- Mix of public and protected endpoints
- Need access to MVC context (model binding, route data)

---

## Comparison: Middleware vs Filter

| Aspect | Middleware | Authorization Filter |
|--------|------------|---------------------|
| **Scope** | All requests | Selected endpoints |
| **Pipeline Position** | Early (before routing) | After routing/model binding |
| **Granularity** | Coarse (global) | Fine (per action/controller) |
| **MVC Context** | Not available | Full access |
| **Performance** | Slightly faster (less overhead) | Minimal difference |
| **Use Case** | Uniform API protection | Mixed public/private endpoints |

---

## Other API Key Authentication Approaches

### 1. Custom Authentication Handler (Recommended for Production)

Integrates with ASP.NET Core's authentication system using `AuthenticationHandler<T>`:

```csharp
public class ApiKeyAuthHandler : AuthenticationHandler<ApiKeyAuthOptions>
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("x-api-key", out var apiKey))
            return Task.FromResult(AuthenticateResult.Fail("API Key missing"));

        if (!ValidateApiKey(apiKey))
            return Task.FromResult(AuthenticateResult.Fail("Invalid API Key"));

        var claims = new[] { new Claim(ClaimTypes.Name, "ApiUser") };
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

**Pros:** Integrates with `[Authorize]`, supports policies, claims-based identity  
**Cons:** More complex setup

### 2. Custom Attribute with Type Filter

```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class ApiKeyAttribute : TypeFilterAttribute
{
    public ApiKeyAttribute() : base(typeof(ApiKeyAuthFilter)) { }
}

// Usage
[ApiKey]
[HttpGet("protected")]
public IActionResult Get() { }
```

### 3. Endpoint Filter (Minimal APIs)

For .NET 7+ Minimal APIs:

```csharp
app.MapGet("/protected", () => "Secret data")
   .AddEndpointFilter(async (context, next) =>
   {
       if (!context.HttpContext.Request.Headers.TryGetValue("x-api-key", out var key))
           return Results.Unauthorized();
       
       return await next(context);
   });
```

### 4. API Gateway / Reverse Proxy

Offload authentication to infrastructure:
- **Azure API Management** - Built-in API key policies
- **AWS API Gateway** - API key validation
- **Kong / NGINX** - Plugin-based authentication

---

## Other Authentication Methods

### 1. JWT (JSON Web Tokens)

**How it works:** Client receives a signed token after login, includes it in subsequent requests.

```
POST /login { username, password } ? { "token": "eyJhbG..." }
GET /api/data + Header: Authorization: Bearer eyJhbG...
```

**Pros:** Stateless, contains claims, industry standard  
**Cons:** Token size, cannot be revoked easily  
**Use case:** SPAs, mobile apps, microservices

### 2. OAuth 2.0 / OpenID Connect

**How it works:** Delegated authorization using authorization servers (Identity Providers).

**Flows:**
- **Authorization Code** - Web apps with server
- **PKCE** - SPAs and mobile apps
- **Client Credentials** - Service-to-service

**Pros:** Industry standard, supports SSO, third-party identity providers  
**Cons:** Complex setup  
**Use case:** Enterprise apps, third-party integrations

### 3. Basic Authentication

**How it works:** Username:password encoded in Base64 in Authorization header.

```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Pros:** Simple to implement  
**Cons:** Credentials sent with every request, must use HTTPS  
**Use case:** Simple internal APIs, legacy systems

### 4. Certificate-Based (Mutual TLS)

**How it works:** Client presents X.509 certificate; server validates it.

**Pros:** Very secure, no credentials transmitted  
**Cons:** Certificate management complexity  
**Use case:** High-security environments, IoT, banking

### 5. Windows Authentication (Kerberos/NTLM)

**How it works:** Uses Windows credentials from Active Directory.

**Pros:** Seamless in Windows environments, SSO  
**Cons:** Windows-only, intranet focused  
**Use case:** Corporate intranets, internal tools

### 6. API Key + HMAC Signature

**How it works:** Request signed with secret key; server verifies signature.

```
x-api-key: your-key
x-signature: HMAC-SHA256(request-body + timestamp + secret)
x-timestamp: 1699999999
```

**Pros:** Prevents replay attacks, verifies request integrity  
**Cons:** Complex implementation  
**Use case:** Payment APIs, high-security public APIs

### Authentication Method Comparison

| Method | Security | Complexity | Stateless | Best For |
|--------|----------|------------|-----------|----------|
| API Key | Low-Medium | Low | Yes | Internal services, simple APIs |
| JWT | Medium-High | Medium | Yes | SPAs, mobile, microservices |
| OAuth 2.0 | High | High | Depends | Enterprise, third-party auth |
| Basic Auth | Low | Low | Yes | Internal/legacy systems |
| mTLS | Very High | High | Yes | IoT, banking, B2B |
| HMAC Signature | High | Medium | Yes | Payment/financial APIs |

---

## Configuration

Add your API key to `appsettings.json`:

```json
{
  "Authentication": {
    "ApiKey": "your-secret-api-key-here"
  }
}
```

> **Security Note:** In production, use Azure Key Vault, AWS Secrets Manager, or environment variables instead of appsettings.json.

---

## Swagger Integration

Both projects include Swagger UI configuration for API key input:

```csharp
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("ApiKey", new OpenApiSecurityScheme
    {
        Description = "API Key required to access endpoints",
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
    
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        { scheme, new List<string>() }
    });
});
```

---

## Testing

### Using .http files (Visual Studio / VS Code REST Client)

```http
### Protected endpoint - with valid key
GET {{host}}/WeatherForecast/weather
x-api-key: your-secret-api-key

### Without key - returns 401
GET {{host}}/WeatherForecast/weather
```

### Using curl

```bash
# With API key
curl -H "x-api-key: your-secret-api-key" https://localhost:5001/WeatherForecast/weather

# Without API key (401)
curl https://localhost:5001/WeatherForecast/weather
```

---

## License

This project is for educational purposes demonstrating API authentication patterns in ASP.NET Core.
