# Keycloak — Create Instance, User, Role, and Assign Role (Step‑by‑Step)

**Purpose:** A concise step‑by‑step guide to run a local Keycloak instance, create a realm, create a client, create roles (realm & client), create users, and assign roles to users. Use this to integrate with an API (e.g., ASP.NET Core) or a frontend.

---

## Prerequisites

* Docker & Docker Compose installed and running.
* Basic familiarity with Keycloak admin console.
* Port `8080` and your app port (e.g., `5000`) available.

---

## 1. Start Keycloak (local instance)

Create a file `docker-compose.yml` with the following content:

```yaml
version: '3.9'

services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0
    command: start-dev
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    ports:
      - "8080:8080"
```

Run Keycloak:

```bash
docker compose up -d
```

Open admin console: `http://localhost:8080` → Login: `admin` / `admin`.

---

## 2. Create a Realm

1. In Admin Console, open realm selector (top-left) → **Create realm**.
2. Set **Name** to `demo-realm`.
3. Click **Create**.

> A realm is an isolated space for clients, users, and roles.

---

## 3. Create a Client (for your API or app)

1. Left menu → **Clients** → **Create client**.

2. Fill:

    * **Client ID:** `demo-api` (or `demo-ui` for frontend)
    * **Client type:** `OpenID Connect`

3. Click **Next**.

4. On settings page, configure:

    * **Client authentication:** ON (if `confidential` client)
    * **Authorization:** ON (if you will use Keycloak Authorization features)
    * **Access Type:** choose `confidential` for server apps (APIs) or `public` for browser SPAs
    * **Valid Redirect URIs:** e.g., `http://localhost:5000/*` or `http://localhost:3000/*` for frontend
    * **Web Origins:** e.g., `http://localhost:5000`

5. Click **Save**.

---

## 4. Get Client Credentials (Secret)

1. With the client open, go to **Credentials** tab.
2. Copy the **Client secret** (for `confidential` clients).
3. Use this secret in server app (Swagger OAuth config or token requests).

---

## 5. Create a Realm Role

1. Left menu → **Roles**.
2. Click **Create role**.
3. Set **Role name**: e.g., `user` (or `admin`, `manager`).
4. Click **Save**.

> This creates a realm-level role visible to all clients under the realm.

---

## 6. (Optional) Create a Client-Level Role

If you want roles scoped to a specific client:

1. Left menu → **Clients** → open `demo-api` → **Roles** tab.
2. Click **Create role**.
3. Add role name (e.g., `api-reader`, `api-writer`) → Save.

Client roles are returned in tokens under `resource_access` (client-specific map).

---

## 7. Create a User

1. Left menu → **Users** → **Add user**.
2. Enter details: `Username: testuser` (set email/name as desired) → Click **Create**.
3. Open the newly created user → **Credentials** tab → **Set password**.

    * Password: `123` (example)
    * Turn **Temporary** OFF → Click **Set password** or **Save**.

---

## 8. Assign Role(s) to the User

### Assign Realm Role

1. User details → **Role mapping** tab.
2. Click **Assign role**.
3. Use **Filter by realm roles** → select `user` → Click **Assign**.

You should see `user` under **Assigned roles**.

### Assign Client Role

1. In **Assign role** modal, switch to **Filter by client roles**.
2. Choose the client (`demo-api`) → select 1+ client roles → Click **Assign**.

Now the user will have both realm and/or client roles as required.

---

## 9. Verify Roles in Access Token

1. Use Swagger or a token request to get an access token (resource owner password or authorization code flow).

Example `curl` (Resource Owner Password Credentials - for testing only):

```bash
curl -X POST "http://localhost:8080/realms/demo-realm/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=demo-api" \
  -d "client_secret=<CLIENT_SECRET>" \
  -d "grant_type=password" \
  -d "username=testuser" \
  -d "password=123"
```

2. Decode `access_token` at [https://jwt.io](https://jwt.io) or using `jq` + `base64` to see claims.

Look for:

* Realm roles under:

```json
"realm_access": { "roles": ["user", ...] }
```

* Client roles under `resource_access`:

```json
"resource_access": {
  "demo-api": { "roles": ["api-reader"] }
}
```

---

## 10. Map Roles for ASP.NET (RoleClaimType)

Keycloak places realm roles under `realm_access.roles`. To make ASP.NET Core `Authorize(Roles = "user")` work, add these roles into claims during token validation.

In `Program.cs` (JwtBearer config) add an `OnTokenValidated` event to copy roles into role claims:

```csharp
options.Events = new JwtBearerEvents
{
    OnTokenValidated = context =>
    {
        var claimsIdentity = context.Principal?.Identity as ClaimsIdentity;
        var realmAccess = context.Principal?.FindFirst("realm_access")?.Value;
        if (!string.IsNullOrEmpty(realmAccess))
        {
            var parsed = JsonDocument.Parse(realmAccess);
            if (parsed.RootElement.TryGetProperty("roles", out var roles))
            {
                foreach (var role in roles.EnumerateArray())
                {
                    claimsIdentity?.AddClaim(new Claim(ClaimsIdentity.DefaultRoleClaimType, role.GetString()!));
                }
            }
        }
        return Task.CompletedTask;
    }
};
```

Also set `TokenValidationParameters.RoleClaimType = ClaimsIdentity.DefaultRoleClaimType;` if needed.

---

## 11. Common Troubleshooting

* **`invalid_token: The audience 'account' is invalid`**: Ensure token `aud` contains your client (`demo-api`). Add an *audience mapper* or set `ValidateAudience = false` for dev.
* **Swagger login fails**: Add `http://localhost:5000/swagger/oauth2-redirect.html` to client **Valid Redirect URIs** and `http://localhost:5000` to **Web Origins**.
* **Roles not recognized by ASP.NET**: Ensure you copy `realm_access.roles` into `roles` claim as shown above.

---

## 12. Cleanup (Stop Keycloak)

```bash
docker compose down
```

---

## Appendix — Useful Endpoints

* OpenID configuration: `http://localhost:8080/realms/demo-realm/.well-known/openid-configuration`
* Authorization endpoint: `http://localhost:8080/realms/demo-realm/protocol/openid-connect/auth`
* Token endpoint: `http://localhost:8080/realms/demo-realm/protocol/openid-connect/token`

---

If you want, I can:

* Add a ready-to-use `Program.cs` snippet that includes role-mapping logic for ASP.NET Core, or
* Generate a one-page printable version of this markdown, or
* Create a `demo-ui` client step for Angular/React login flow.
