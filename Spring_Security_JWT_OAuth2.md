# Spring Security 

## 1. What is Spring Security?

Spring Security is a framework used to secure Java applications. It mainly provides:

- Authentication
- Authorization
- Security filters
- Protection against common attacks
- Integration with JWT and OAuth 2.0

### Memory

```text
Authentication → Who are you?
Authorization  → What are you allowed to do?
```

---

## 2. Authentication vs Authorization

### Authentication

Authentication verifies **who the user is**.

```text
Username + Password
        ↓
Is this user valid?
```

### Authorization

Authorization checks **what the authenticated user is allowed to access**.

```text
ROLE_USER  → /user/**
ROLE_ADMIN → /admin/**
```

### Interview Answer

> Authentication identifies the user, while authorization determines whether that authenticated user has permission to access a resource.

---

# 3. Explain Spring Security Request Flow

This is a very important 5-year interview question.

### Interview Answer

> When a request comes to the application, it first passes through the Spring Security filter chain. The filters check authentication information such as a session or JWT. If authentication succeeds, the authenticated user is stored in the SecurityContext. Then authorization checks whether the user has the required role or authority. Finally, the request reaches the controller.

```text
Client
  |
  | HTTP Request
  ↓
Security Filter Chain
  |
  ├── Authentication Filter
  |
  ├── JWT Filter
  |
  ├── SecurityContext
  |
  └── Authorization
          |
          ├── Allowed → Controller
          |
          └── Denied → 403
```

---

# 4. What is SecurityFilterChain?

`SecurityFilterChain` defines how Spring Security should secure incoming HTTP requests.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/auth/login").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
            .anyRequest().authenticated()
        );

    return http.build();
}
```

### Memory

```text
SecurityFilterChain
        ↓
Which URL?
        ↓
Who can access?
        ↓
Authenticated / Role
```

---

# 5. Login API is public, but other APIs are protected. How do you configure it?

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/auth/login").permitAll()
    .requestMatchers("/auth/register").permitAll()
    .anyRequest().authenticated()
)
```

### Interview Answer

> Login and registration are public endpoints because the user does not have a token yet. All other endpoints require authentication.

---

# 6. How does JWT Authentication work?

```text
                  LOGIN
                    |
                    ↓
          Username + Password
                    |
                    ↓
             Authentication
                    |
                    ↓
                JWT Token
                    |
                    ↓
               Client
                    |
                    ↓
        Authorization: Bearer JWT
                    |
                    ↓
               JWT Filter
                    |
                    ↓
              Validate JWT
                    |
                    ↓
            SecurityContext
                    |
                    ↓
               Controller
```

### Interview Answer

> During login, the server validates the username and password. If authentication succeeds, it generates a JWT containing information such as the user identity and authorities. The client sends this token with subsequent requests using the `Authorization: Bearer <token>` header. A security filter validates the token and creates an authenticated SecurityContext.

---

# 7. Why do we create a JWT Filter?

### Interview Answer

> The JWT filter intercepts incoming requests, extracts the JWT from the Authorization header, validates it, and if valid, creates an authenticated object and stores it in the SecurityContext.

Example:

```java
String header = request.getHeader("Authorization");

if (header != null && header.startsWith("Bearer ")) {

    String token = header.substring(7);

    // Validate token
    // Extract username
    // Create Authentication

    SecurityContextHolder
        .getContext()
        .setAuthentication(authentication);
}
```

---

# 8. What is SecurityContextHolder?

### Interview Answer

> `SecurityContextHolder` stores the security information of the currently authenticated user.

Example:

```java
Authentication authentication =
        SecurityContextHolder
            .getContext()
            .getAuthentication();

String username = authentication.getName();
```

It allows application code to access the currently authenticated principal.

---

# 9. What is UserDetails?

`UserDetails` represents the user's security-related information, such as:

- Username
- Password
- Authorities
- Account status

Example:

```java
public class CustomUserDetails implements UserDetails {

    private String username;
    private String password;
    private Collection<? extends GrantedAuthority> authorities;

}
```

### Flow

```text
Database
   ↓
UserDetailsService
   ↓
UserDetails
   ↓
Authentication
```

---

# 10. What is UserDetailsService?

### Interview Answer

> `UserDetailsService` is used by Spring Security to load user information, usually from a database, based on the username.

Example:

```java
@Service
public class CustomUserDetailsService
        implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) {

        User user = userRepository
                .findByUsername(username)
                .orElseThrow();

        return new CustomUserDetails(user);
    }
}
```

### Real-time Flow

```text
Login
 ↓
Username
 ↓
UserDetailsService
 ↓
Database
 ↓
User
 ↓
Password Verification
```

---

# 11. What is PasswordEncoder?

### Interview Answer

> `PasswordEncoder` securely hashes passwords instead of storing plain-text passwords.

Example:

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

During registration:

```java
String encodedPassword =
        passwordEncoder.encode(user.getPassword());
```

During authentication:

```java
passwordEncoder.matches(
    rawPassword,
    encodedPassword
);
```

### Important

Do not say:

> We encrypt the password.

Better:

> We hash the password using a password hashing algorithm such as BCrypt.

---

# 12. Why BCrypt?

### Interview Answer

> BCrypt is designed for password hashing and includes salting and configurable computational cost, making brute-force attacks more difficult.

---

# 13. 401 vs 403

This is a very common interview question.

## 401 Unauthorized

Authentication problem.

Examples:

```text
No token
Invalid token
Expired token
```

Meaning:

> I don't know who you are.

## 403 Forbidden

Authentication succeeded but authorization failed.

Example:

```text
User = authenticated
Role = USER
Required = ADMIN
```

Meaning:

> I know who you are, but you don't have permission.

### Memory

```text
401 → Authentication problem
403 → Authorization problem
```

---

# 14. Real-time Scenario: USER calls ADMIN API

Request:

```http
GET /admin/users
Authorization: Bearer <USER_TOKEN>
```

Configuration:

```java
.requestMatchers("/admin/**")
.hasRole("ADMIN")
```

User has:

```text
ROLE_USER
```

Flow:

```text
JWT valid
    ↓
Authentication successful
    ↓
Authorization check
    ↓
ROLE_ADMIN required
    ↓
User has ROLE_USER
    ↓
403 Forbidden
```

### Interview Answer

> The token is valid, so authentication succeeds, but the user does not have the required ADMIN authority. Therefore Spring Security returns 403 Forbidden.

---

# 15. `hasRole()` vs `hasAuthority()`

### hasRole

```java
.hasRole("ADMIN")
```

Spring commonly maps this to:

```text
ROLE_ADMIN
```

### hasAuthority

```java
.hasAuthority("ROLE_ADMIN")
```

This checks the authority value directly.

### Memory

```text
hasRole("ADMIN")
       ↓
ROLE_ADMIN

hasAuthority("ROLE_ADMIN")
       ↓
Exact authority
```

---

# 16. What is CSRF?

CSRF means **Cross-Site Request Forgery**.

### Interview Answer

> CSRF is an attack where a malicious website tricks a user's browser into sending an authenticated request to another application.

It is especially important for cookie/session-based authentication.

### JWT API

If an application uses JWT in the Authorization header and does not rely on browser cookies for authentication, CSRF protection may not be required in the same way as a cookie-based application.

Do not say:

> JWT means CSRF is always unnecessary.

The correct answer depends on how authentication credentials are stored and transmitted.

---

# 17. What is CORS?

### Interview Answer

> CORS controls whether a browser allows a frontend from one origin to make requests to a backend on another origin.

Example:

```text
Frontend:
http://localhost:3000

Backend:
http://localhost:8080
```

These are different origins.

---

# 18. CORS vs CSRF

| CORS | CSRF |
|---|---|
| Cross-origin access control | Request forgery attack |
| Controls allowed browser origins | Protects authenticated requests |
| Browser security mechanism | Application security attack |
| Common in frontend/backend separation | Especially important with cookie/session authentication |

---

# 19. Why do we use `@PreAuthorize`?

It provides method-level authorization.

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) {
    // ...
}
```

Only an ADMIN can execute the method.

Another example:

```java
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
```

---

# 20. How would you secure an e-commerce application?

### Strong 5-year answer

> I would separate authentication and authorization. The login endpoint would authenticate the user and issue an access token. Protected APIs would use the Spring Security filter chain to validate the token. Roles and authorities would control access to resources. For example, customers can access their orders, while ADMIN users can manage products and users. Passwords would be stored using BCrypt rather than plain text. I would also configure CORS, CSRF based on the authentication mechanism, exception handling, and secure token expiration and refresh mechanisms.

```text
                  Client
                    |
                    ↓
              API Gateway
                    |
                    ↓
        Spring Security Filters
                    |
                    ↓
              JWT Validation
                    |
                    ↓
             SecurityContext
                    |
                    ↓
              Authorization
              /           \
          USER             ADMIN
           |                 |
       Order APIs       Admin APIs
```

---

# 21. What happens when JWT expires?

```text
Access Token
     ↓
Expired
     ↓
API Request
     ↓
JWT Validation Fails
     ↓
401 Unauthorized
     ↓
Client Uses Refresh Token
     ↓
New Access Token
```

### Important

```text
Access Token  → Short-lived
Refresh Token → Longer-lived
```

---

# 22. Access Token vs Refresh Token

| Access Token | Refresh Token |
|---|---|
| Short-lived | Longer-lived |
| Used to access APIs | Used to obtain a new access token |
| Sent frequently | Sent less frequently |
| More exposed | Must be protected carefully |

### Interview Answer

> I keep access tokens short-lived and use refresh tokens to obtain new access tokens without forcing the user to log in again.

---

# 23. How do you handle authentication failures?

Spring Security provides mechanisms such as:

```text
AuthenticationEntryPoint
AccessDeniedHandler
```

### Authentication failure

```text
Unauthenticated
      ↓
401
      ↓
AuthenticationEntryPoint
```

### Authorization failure

```text
Authenticated
      ↓
Insufficient permission
      ↓
403
      ↓
AccessDeniedHandler
```

### Memory

```text
AuthenticationEntryPoint → 401
AccessDeniedHandler     → 403
```

---

# 24. What is OAuth 2.0?

### Interview Answer

> OAuth 2.0 is an authorization framework that allows an application to obtain limited access to a protected resource on behalf of a user or another client without sharing the user's password with that application.

Example:

```text
Your Application
       |
       ↓
Login with Google
       |
       ↓
Google Authorization Server
       |
       ↓
User gives consent
       |
       ↓
Authorization Code
       |
       ↓
Access Token
       |
       ↓
Protected Resource
```

### Important

OAuth 2.0 is primarily an **authorization framework**.

It is not itself a user authentication protocol.

For user authentication, **OpenID Connect (OIDC)** is commonly used on top of OAuth 2.0.

---

# 25. OAuth 2.0 Important Components

Remember these four:

```text
Resource Owner
Client
Authorization Server
Resource Server
```

### Example

```text
Resource Owner
     ↓
     User

Client
     ↓
     Your application

Authorization Server
     ↓
     Google / Keycloak / Okta / Auth0

Resource Server
     ↓
     Protected API
```

---

# 26. OAuth 2.0 Authorization Code Flow

This is one of the most important OAuth interview topics.

```text
User
 |
 | 1. Login / authorize
 ↓
Client Application
 |
 | 2. Authorization Request
 ↓
Authorization Server
 |
 | 3. User Login + Consent
 |
 | 4. Authorization Code
 ↓
Client Application
 |
 | 5. Code + Client Authentication
 ↓
Authorization Server
 |
 | 6. Access Token
 ↓
Client
 |
 | 7. Access Token
 ↓
Resource Server
 |
 | 8. Protected Resource
 ↓
Response
```

### Interview Answer

> In the Authorization Code flow, the client redirects the user to the authorization server. After successful authentication and consent, the authorization server sends an authorization code back to the client. The client exchanges that code for an access token and then uses the access token to access protected resources.

---

# 27. What is PKCE?

PKCE means **Proof Key for Code Exchange**.

### Interview Answer

> PKCE adds protection to the OAuth 2.0 Authorization Code flow by using a dynamically generated code verifier and challenge. It helps prevent an attacker who intercepts the authorization code from exchanging it for an access token.

Flow:

```text
Client
 |
 | Generate code_verifier
 |
 | code_challenge
 ↓
Authorization Server
 |
 | Authorization Code
 ↓
Client
 |
 | code + code_verifier
 ↓
Authorization Server
 |
 ↓
Access Token
```

For modern public clients such as mobile and browser applications, Authorization Code + PKCE is the recommended approach.

---

# 28. OAuth 2.0 Grant Types

Know these:

### Authorization Code

Used for user authorization flows.

```text
User → Authorization Server → Code → Token
```

### Client Credentials

Used for machine-to-machine communication.

```text
Service A
   |
   | client credentials
   ↓
Authorization Server
   |
   ↓
Access Token
   |
   ↓
Service B
```

There is no end-user involved.

### Refresh Token

Used to obtain a new access token.

### Resource Owner Password Credentials

This legacy flow should generally **not be used for new applications**.

---

# 29. OAuth 2.0 vs JWT

This is a very important interview question.

| OAuth 2.0 | JWT |
|---|---|
| Authorization framework | Token format |
| Defines authorization flows | Defines token structure |
| Can use different token formats | JWT is a specific token format |
| Uses authorization server/resource server concepts | Can be used independently |
| Often used with OIDC for authentication | Can carry claims about a user/client |

### Best interview answer

> OAuth 2.0 and JWT are not alternatives. OAuth 2.0 defines how access is obtained and delegated, while JWT is a token format that can be used to represent access tokens or identity information. An OAuth 2.0 system can use JWT access tokens, but OAuth 2.0 does not require JWT.

### Memory

```text
OAuth 2.0 → HOW do I get access?
JWT       → WHAT format is the token?
OIDC      → WHO is the user?
```

---

# 30. OAuth 2.0 vs OIDC

### OAuth 2.0

Primarily:

```text
Authorization
```

### OpenID Connect

Built on OAuth 2.0 and adds:

```text
Authentication + Identity
```

### Memory

```text
OAuth 2.0 → Authorization
OIDC      → Authentication / Identity
```

Example:

```text
"Login with Google"
        ↓
OIDC
        ↓
ID Token + Access Token
```

---

# 31. Access Token vs ID Token

This is a common OAuth/OIDC interview question.

### Access Token

Used to access APIs.

```text
Client
  ↓
Access Token
  ↓
Resource Server
```

### ID Token

Used by the client to obtain information about the authenticated user in an OIDC flow.

```text
Authorization Server
       ↓
    ID Token
       ↓
     Client
```

### Memory

```text
Access Token → API
ID Token     → Identity
```

Do not use an ID token as an API access token.

---

# 32. What is an OAuth Resource Server?

### Interview Answer

> A Resource Server hosts protected APIs and validates access tokens presented by clients.

Example:

```text
Client
  |
  | Bearer Access Token
  ↓
Resource Server
  |
  | Validate token
  ↓
Protected API
```

In a Spring application, Spring Security can configure the application as an OAuth 2.0 Resource Server.

Conceptually:

```text
Authorization Server
        |
        | Access Token
        ↓
Spring Resource Server
        |
        ↓
Protected APIs
```

---

# 33. What is an Authorization Server?

### Interview Answer

> An Authorization Server authenticates/authorizes clients and issues tokens according to the configured OAuth/OIDC flow.

Examples of products commonly used in enterprise systems include:

```text
Keycloak
Okta
Auth0
Microsoft Entra ID
Google Identity
```

---

# 34. JWT Structure

A JWT normally contains three Base64URL-encoded parts:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example:

```text
xxxxx.yyyyy.zzzzz
```

### Header

Contains metadata such as the signing algorithm.

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

### Payload

Contains claims.

```json
{
  "sub": "12345",
  "roles": ["USER"],
  "exp": 1788500000
}
```

### Signature

Used to verify integrity/authenticity.

```text
Header
   +
Payload
   +
Secret/Private Key
   ↓
Signature
```

---

# 35. Is JWT encrypted?

**Usually, no.**

A signed JWT (JWS) is normally **encoded and signed, not encrypted**.

Therefore:

> Do not put passwords, secrets, or sensitive information into an ordinary signed JWT payload.

If confidentiality is required, an encrypted JWT/JWE or another secure mechanism may be appropriate.

---

# 36. Symmetric vs Asymmetric JWT Signing

## HS256

```text
Same secret key

Issuer
  ↓
Secret
  ↓
Sign

Resource Server
  ↓
Same Secret
  ↓
Verify
```

## RS256

```text
Private Key
    ↓
Authorization Server
    ↓
Sign JWT

Public Key
    ↓
Resource Server
    ↓
Verify JWT
```

### Enterprise interview answer

> In distributed systems, asymmetric signing such as RSA can be useful because the authorization server keeps the private key while resource servers only need the public key for verification.

---

# 37. Real-time Microservices OAuth/JWT Architecture

A strong system-design answer:

```text
                         User
                           |
                           ↓
                    Frontend / Mobile
                           |
                           ↓
                     API Gateway
                           |
              +------------+------------+
              |                         |
              ↓                         ↓
        Order Service             User Service
              |                         |
              ↓                         ↓
        Product Service          Payment Service


             Authorization Server
                    |
                    |
              Access Token
                    |
                    ↓
             API Gateway
                    |
                    ↓
       Each Resource Server validates
             the access token
```

### Example

```text
User
 ↓
Login
 ↓
Identity Provider / Authorization Server
 ↓
Access Token
 ↓
API Gateway
 ↓
Order Service
 ↓
JWT validation
 ↓
Authorization
 ↓
Business logic
```

---

# 38. Real-time: How do you secure microservices?

### Interview Answer

> I would use a centralized identity provider or authorization server for issuing tokens. The client obtains an access token and sends it to the API gateway. The gateway can perform initial validation and routing, while resource services should still enforce their own authorization and validate tokens according to the architecture. Services use scopes, roles, or authorities to control access. Communication between internal services should also be secured rather than blindly trusting headers from the client.

---

# 39. Scope vs Role

### Role

Usually represents application-level responsibilities.

```text
ROLE_ADMIN
ROLE_USER
ROLE_MANAGER
```

### Scope

OAuth 2.0 scopes represent delegated permissions.

```text
orders:read
orders:write
users:read
```

Example:

```text
Access Token
     ↓
scope = orders:read
     ↓
GET /orders
     ↓
Allowed
```

### Memory

```text
Role   → Who is the user?
Scope  → What permission was delegated?
```

In real systems, exact semantics depend on the organization's authorization model.

---

# 40. Production Issue: Every API returns 401

Debug in this order:

```text
1. Check Authorization header
2. Check "Bearer " prefix
3. Check token expiration
4. Check JWT signature
5. Check issuer
6. Check audience
7. Check signing key / JWKS
8. Check JWT filter/resource-server configuration
9. Check SecurityContext
10. Check clock/time synchronization
```

Request should look like:

```http
Authorization: Bearer eyJhbGciOi...
```

---

# 41. Production Issue: Login works but API returns 403

Think:

```text
JWT valid?
    ↓
YES
    ↓
Authentication successful
    ↓
403
    ↓
Authorization problem
```

Check:

```text
Role mapping
Authority mapping
ROLE_ prefix
hasRole vs hasAuthority
GrantedAuthority
Endpoint authorization
Method-level authorization
Scopes
```

---

# 42. Production Issue: JWT signature validation fails

Check:

```text
Issuer signing key
Public key
JWKS endpoint
Algorithm
Token corruption
Key rotation
Wrong environment configuration
```

For asymmetric JWT:

```text
Authorization Server
       |
       | Private Key
       ↓
     Sign JWT
       |
       ↓
Resource Server
       |
       | Public Key / JWKS
       ↓
   Verify Signature
```

---

# 43. What happens during Key Rotation?

In production, signing keys may be rotated.

```text
Old Private Key
       ↓
Old tokens

New Private Key
       ↓
New tokens
```

The resource server should obtain the appropriate public keys, often through a JWKS endpoint, and support the transition according to the identity provider's rotation strategy.

### Interview Answer

> I would avoid hardcoding a single public key when the authorization infrastructure supports key rotation. The resource server should be configured to retrieve and cache signing keys appropriately, commonly through JWKS.

---

# 44. How would you secure passwords?

```text
Registration
     ↓
Raw Password
     ↓
PasswordEncoder
     ↓
BCrypt Hash
     ↓
Database
```

Never:

```text
Plain Password → Database
```

Never log:

```text
password
access token
refresh token
client secret
```

---

# 45. JWT Security Best Practices

Remember these in interviews:

```text
✓ Short-lived access tokens
✓ Secure refresh-token handling
✓ Strong signing keys
✓ Validate issuer
✓ Validate audience where applicable
✓ Validate expiration
✓ Validate signature
✓ HTTPS
✓ Avoid sensitive data in JWT claims
✓ Key rotation
✓ Least privilege scopes/roles
✓ Do not log tokens
✓ Protect client secrets
```

---

# 46. Spring Security Interview Rapid-Fire

### Q: Who are you?

```text
Authentication
```

### Q: What can you access?

```text
Authorization
```

### Q: Where does request security happen?

```text
SecurityFilterChain
```

### Q: Where is current authentication stored?

```text
SecurityContext
```

### Q: How is database user loaded?

```text
UserDetailsService
```

### Q: How is password hashed?

```text
PasswordEncoder / BCrypt
```

### Q: No/invalid authentication?

```text
401
```

### Q: Authenticated but insufficient permission?

```text
403
```

### Q: OAuth 2.0?

```text
Authorization framework
```

### Q: OIDC?

```text
Authentication / identity layer on OAuth 2.0
```

### Q: JWT?

```text
Token format
```

### Q: Access Token?

```text
Access protected API
```

### Q: ID Token?

```text
Identity information in OIDC
```

### Q: Refresh Token?

```text
Obtain new access token
```

### Q: API protection?

```text
Resource Server
```

### Q: Token issuing system?

```text
Authorization Server
```

### Q: Modern browser/mobile OAuth flow?

```text
Authorization Code + PKCE
```

---

# 47. Ultimate Memory Map

```text
                         SPRING SECURITY
                               |
             +-----------------+-----------------+
             |                                   |
       Authentication                      Authorization
             |                                   |
       "Who are you?"                    "What can you do?"
             |                                   |
     +-------+-------+                    +------+------+
     |               |                    |             |
  Session           JWT                 Role          Scope
     |               |                    |             |
     |          JWT Filter              USER          orders:read
     |               |                  ADMIN         orders:write
     |               |
     |        SecurityContext
     |               |
     +---------------+
             |
             ↓
       SecurityFilterChain
             |
      +------+------+
      |             |
     401           403
      |             |
 Authentication   Authorization
   failure          failure


                    OAUTH 2.0
                       |
          +------------+------------+
          |            |            |
       Client    Authorization   Resource
                    Server        Server
                       |
                    Token
                       |
              +--------+--------+
              |                 |
         Access Token       Refresh Token
              |
           API Access


                     OIDC
                       |
                  OAuth 2.0 +
                  Identity
                       |
                   ID Token


                      JWT
                       |
             HEADER.PAYLOAD.SIGNATURE
                       |
              +--------+--------+
              |                 |
          HS256             RS256
        Shared Key       Private/Public
```

---

# 48. 30-Second Interview Summary

If the interviewer says:

**"Explain your Spring Security implementation."**

Say:

> In our application, we use Spring Security to handle authentication and authorization. For stateless API security, the client authenticates through our identity/authentication mechanism and receives an access token. For JWT-based authentication, the token is sent as a Bearer token in the Authorization header. Spring Security validates the token through the security filter/resource-server infrastructure and establishes the SecurityContext. Authorization is then handled using roles, authorities, or OAuth scopes. Public endpoints such as login are permitted, while protected endpoints require authentication. We return 401 for authentication failures and 403 when an authenticated user lacks the required permissions. For enterprise identity and delegated authorization, we use OAuth 2.0/OIDC, commonly with Authorization Code + PKCE for user-facing clients.

---

# Final Interview Formula

```text
Spring Security
      ↓
Authentication
      ↓
JWT / OAuth2 / OIDC
      ↓
SecurityFilterChain
      ↓
SecurityContext
      ↓
Authorization
      ↓
Role / Authority / Scope
      ↓
Controller / Service
      ↓
401 / 403 when required
```

## Must-know for 5 Years

```text
★★★★★ SecurityFilterChain
★★★★★ JWT Authentication
★★★★★ JWT Filter
★★★★★ SecurityContext
★★★★★ UserDetailsService
★★★★★ PasswordEncoder
★★★★★ 401 vs 403
★★★★★ Roles vs Authorities

★★★★★ OAuth 2.0
★★★★★ Authorization Code + PKCE
★★★★★ Access Token vs Refresh Token
★★★★★ OAuth vs JWT
★★★★★ OAuth vs OIDC
★★★★★ Access Token vs ID Token
★★★★★ Resource Server
★★★★★ Authorization Server
★★★★★ Scopes
★★★★★ JWT signature / RS256
★★★★★ JWKS / Key Rotation

★★★★ CORS
★★★★ CSRF
★★★★ Exception Handling
★★★★ Production troubleshooting
★★★★ Microservices security
```
