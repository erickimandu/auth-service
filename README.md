# Enterprise Authentication Service

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)
![Coverage](https://img.shields.io/badge/coverage-94%25-brightgreen.svg)
![.NET Core](https://img.shields.io/badge/.NET%20Core-8.0-purple.svg)

A robust, enterprise-grade authentication and authorization service designed for modern distributed applications. Built with security, flexibility, and multi-tenancy at its core.

## 🚀 Features

- **OAuth2/OpenID Connect** implementation with OpenIddict for secure token-based authentication
- **Multi-tenant architecture** with hierarchical organization units
- **Fine-grained permission system** with role-based access control
- **API key management** for secure service-to-service communication
- **Complete user onboarding flow** with identity verification integration
- **Email verification** and account management
- **Robust security features** including token revocation, refresh token rotation, and brute force protection

## 📋 Table of Contents

- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Key Components](#-key-components)
- [Getting Started](#-getting-started)
- [API Documentation](#-api-documentation)
- [Configuration](#-configuration)
- [Deployment](#-deployment)
- [Security Considerations](#-security-considerations)
- [Contributing](#-contributing)
- [License](#-license)

## 🏗 Architecture

The service follows a clean, layered architecture:

```
┌─────────────────┐
│   API Layer     │ ← REST controllers, middleware, authentication handlers
├─────────────────┤
│   Core Layer    │ ← Domain entities, interfaces, business logic
├─────────────────┤
│ Infrastructure  │ ← Data access, external service integration
└─────────────────┘
```

The service is designed to be deployed as a microservice providing centralized identity and access management to all components in your application ecosystem.

## 🔧 Tech Stack

- **Framework**: ASP.NET Core 8.0
- **Authentication**: OpenIddict (OAuth2/OpenID Connect)
- **Database**: PostgreSQL with Entity Framework Core
- **Caching**: Redis for token and session caching
- **Identity Verification**: Pluggable provider system (Smile Identity integration included)
- **Messaging**: Event-driven architecture
- **Containerization**: Docker with Kubernetes orchestration

## 🧩 Key Components

### Multi-Tenancy System

The service supports multiple isolated tenants sharing the same infrastructure, with each tenant having its own users, organization units, and permission settings.

```csharp
// Example of tenant-aware query
var userQuery = _dbContext.Users.AsQueryable();
if (!string.IsNullOrEmpty(tenantId))
{
    userQuery = userQuery.Where(u => u.TenantId == tenantId);
}
```

### Hierarchical Organization Units

Each tenant can define its own organizational hierarchy, with inheritance of permissions through the tree structure.

```csharp
// Example of getting effective permissions through org unit hierarchy
public async Task<IEnumerable<Permission>> GetEffectivePermissionsForUserAsync(string userId)
{
    var user = await _userManager.FindByIdAsync(userId);
    var permissions = new List<Permission>();
    
    // Get org unit permissions including ancestors
    if (!string.IsNullOrEmpty(user.OrganizationUnitId))
    {
        var orgUnitPermissions = await _orgUnitService.GetEffectivePermissionsForOrgUnitAsync(user.OrganizationUnitId);
        permissions.AddRange(orgUnitPermissions);
    }
    
    // Add role-based permissions
    var rolePermissions = await GetRolePermissionsForUserAsync(userId);
    permissions.AddRange(rolePermissions);
    
    return permissions.Distinct();
}
```

### JWT Authentication with OpenIddict

The service uses OpenIddict to implement OAuth2/OpenID Connect standards, providing secure token-based authentication.

```csharp
// Example token generation
var principal = await CreateClaimsPrincipalAsync(user, roles, customClaims);
var authorization = await _authorizationManager.CreateAsync(
    principal: principal,
    subject: await _applicationManager.GetIdAsync(application),
    client: await _applicationManager.GetIdAsync(application),
    type: OpenIddictConstants.AuthorizationTypes.Permanent,
    scopes: principal.GetScopes());

var accessToken = await _tokenManager.CreateAsync(
    type: OpenIddictConstants.TokenTypeHints.AccessToken,
    authorizationId: await _authorizationManager.GetIdAsync(authorization));
```

### API Key Management

The service provides API key-based authentication for service-to-service communication, with full lifecycle management.

```csharp
// Example API key validation middleware
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKeyHeaderValues))
        {
            return AuthenticateResult.NoResult();
        }

        var providedApiKey = apiKeyHeaderValues.FirstOrDefault();
        var clientIpAddress = Context.Connection.RemoteIpAddress;
        var validationResult = await _apiKeyService.ValidateApiKeyAsync(providedApiKey, clientIpAddress);

        if (!validationResult.IsValid)
        {
            return AuthenticateResult.Fail("Invalid API key.");
        }

        // Create claims principal with permissions from API key
        // ...
    }
}
```

### Identity Verification

The service includes a pluggable identity verification system, with built-in integration for Smile Identity and other KYC providers.

```csharp
// Example identity verification flow
public async Task<VerificationInitResult> InitiateVerificationAsync(string userId, VerificationType verificationType)
{
    // Create verification session with the configured provider
    var client = _httpClientFactory.CreateClient("IdentityVerification");
    var jobParams = new
    {
        partner_id = _options.PartnerId,
        job_type = GetJobTypeFromVerificationType(verificationType),
        user_id = userId,
        job_id = Guid.NewGuid().ToString()
    };

    // Initiate verification and handle response
    // ...

    return new VerificationInitResult
    {
        Success = true,
        JobId = jobParams.job_id,
        RedirectUrl = result.RedirectUrl
    };
}
```

## 🚀 Getting Started

### Prerequisites

- .NET 8.0 SDK
- PostgreSQL Database
- Redis (optional, recommended for production)
- SMTP Server for emails

### Installation

1. Clone the repository
   ```bash
   git clone https://github.com/your-org/auth-service.git
   cd auth-service
   ```

2. Configure the service
   ```bash
   cp appsettings.example.json appsettings.Development.json
   # Edit the appsettings.Development.json with your configuration
   ```

3. Run database migrations
   ```bash
   dotnet ef database update
   ```

4. Run the service
   ```bash
   dotnet run
   ```

### Running with Docker

```bash
docker build -t auth-service:latest .
docker run -d \
  --name auth-service \
  -p 5000:80 \
  -e ConnectionStrings__DefaultConnection="Host=postgres;Database=authservice;Username=postgres;Password=yourpassword" \
  -e ASPNETCORE_ENVIRONMENT="Production" \
  auth-service:latest
```

## 📚 API Documentation

Full API documentation is available via Swagger when running the service:

```
http://localhost:5000/swagger
```

### Key Endpoints

- **Authentication**: `/api/auth/*` - Login, register, token refresh
- **Tenants**: `/api/tenants/*` - Tenant management
- **Organization Units**: `/api/tenants/{tenantId}/organization-units/*` - Organization structure
- **Permissions**: `/api/permissions/*` - Permission management
- **API Keys**: `/api/apikeys/*` - API key management
- **Onboarding**: `/api/onboarding/*` - User onboarding and identity verification

## ⚙️ Configuration

Configuration is managed through appsettings.json:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=postgres;Database=authservice;Username=postgres;Password=yourpassword"
  },
  "Jwt": {
    "Issuer": "https://auth.your-app.com",
    "Audience": "your-app-api",
    "Key": "[Your secret key here]",
    "TokenExpirationInMinutes": 60
  },
  "EmailSettings": {
    "SenderEmail": "noreply@your-app.com",
    "SenderName": "Your Application",
    "SmtpServer": "smtp.example.com"
  },
  "IdentityVerification": {
    "Provider": "SmileIdentity",
    "ApiUrl": "https://api.provider.com/v1/",
    "PartnerId": "[Your partner ID]"
  }
}
```

## 🚢 Deployment

### Kubernetes Deployment

Example Kubernetes deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: auth-service:latest
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        # Add other environment variables or config maps
```

## 🔒 Security Considerations

- **Secrets Management**: Use a secure vault for storing secrets in production
- **HTTPS**: Always configure HTTPS in production environments
- **Token Security**: Keep token lifetime short and implement proper revocation
- **API Key Management**: Rotate API keys regularly
- **Permission Reviews**: Regularly audit permission assignments

## 👥 Contributing

We welcome contributions to the Auth Service! Please see our [Contributing Guide](CONTRIBUTING.md) for more details.

### Development Workflow

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run tests
5. Submit a pull request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
