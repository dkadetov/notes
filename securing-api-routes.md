# Securing API routes with OAuth2-Proxy and Emissary-ingress

Organizations frequently need to expose Kubernetes application endpoints to external users or systems. Unfortunately, many such applications weren't designed with robust authentication and authorization mechanisms, either because they're legacy systems no longer actively maintained or because security wasn't a primary consideration during their initial development.

This practical guide demonstrates how to implement a security layer for microservice APIs using JSON Web Tokens (`JWT`) with **Microsoft Entra ID** group membership validation. By implementing this approach, you'll create an effective security barrier ensuring that only properly authenticated and authorized users can access your sensitive API endpoints. We'll focus on integrating the [Emissary-ingress](https://github.com/emissary-ingress/emissary) controller with [OAuth2-Proxy](https://github.com/oauth2-proxy/oauth2-proxy) as a `JWT` validation middleware to achieve this goal.

## API Protection Architecture

```text
╔══════════════════════════════════════════════════════════════════════════════╗
║                                                                              ║
║  ┌─────────────────────┐                              ┌────────────────────┐ ║
║  │                     │                              │    MS Entra ID     │ ║
║  │     API client      │                              │     Provider       │ ║
║  │                     │                              │   (JWT Issuer)     │ ║
║  └──────────┬──────────┘                              └──────────┬─────────┘ ║
║             │                                                    │           ║
║             │ Request with                                       │ Provides  ║
║             │ JWT Bearer Token                                   │ public    ║
║             v                                                    v keys      ║
║  ┌──────────┴──────────┐   Request JWT validation    ┌───────────┴─────────┐ ║
║  │                     ├────────────────────────────>│                     │ ║
║  │  emissary-ingress   │                             │    OAuth2-Proxy     │ ║
║  │   (API Gateway)     │   Respond 200 or 401        │     (Emissary       │ ║
║  │                     │<────────────────────────────│    AuthService)     │ ║
║  └──────────┬──────────┘                             └─────────────────────┘ ║
║             │                                                                ║
║             │ Forward authenticated                                          ║
║             │ requests only                                                  ║
║             v                                                                ║
║  ┌──────────┴──────────┐                                                     ║
║  │  Protected Backend  │                                                     ║
║  │    Microservices    │                                                     ║
║  └─────────────────────┘                                                     ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## A bit of theory

JSON Web Token (`JWT`) has emerged as the industry standard for secure data exchange and authentication between parties. Its popularity stems from its elegant design that combines simplicity, efficiency, and strong security guarantees.

At its core, a `JWT` consists of three distinct components: the `header`, the `payload`, and the `signature`. The header and payload are JSON objects encoded with base64, while the signature provides cryptographic validation. This signature is generated using a secret key known only to the issuing server.

The security model relies on cryptographic verification - the signature mathematically binds the token's contents to prevent tampering. Any modification to even a single character in the header or payload invalidates the signature completely. Without access to the secret signing key, it's cryptographically infeasible to forge a valid signature.

Security best practices dictate that `JWT` tokens should always be transmitted exclusively over `HTTPS` connections. Additionally, since JWTs are typically not designed to be revoked once issued, they're usually configured with relatively short lifespans to minimize the window of opportunity if a token is somehow compromised.

## Setting up JWT token acquisition

To authenticate API requests, we'll use a `JWT` token signed by Microsoft.

This means each API request should contain a header: `Authorization: Bearer <JWT_TOKEN>`

It's quite obvious that for obtaining a `JWT` token, the simplest solution is to use the [Azure CLI](https://learn.microsoft.com/bs-latn-ba/cli/azure/install-azure-cli?view=azure-cli-latest) utility.

Let's describe the typical steps for creating and configuring a linked Enterprise application.

### App registration

- Navigate to: Azure Portal → Microsoft Entra ID → App registrations
- Create an application: New registration → Name: `az-cli-jwt` → Register

### Token configuration

#### Groups claim

- Navigate to: Token configuration → Add groups claim → Edit groups claim
- Select type: All groups (or optionally: Groups assigned to the application) → Add

#### Email claim

- Navigate to: Token configuration → Add groups claim → Add optional claim
- Select type: email → Add (Turn on the Microsoft Graph email permission)

### Expose an API for Azure CLI

#### Add a scope

- Navigate to: Expose an API → Add a scope → Save and continue
- Fill in the fields (for example):
  ```text
  Scope name: App.Get.Access
  Who can consent: Admins and users
  Admin consent display name: Access as the signed-in user
  Admin consent description: Allows Azure CLI to access the app as the signed-in user
  User consent display name: Access as the signed-in user
  User consent description: Allows Azure CLI to access the app
  ```
- Click: Add scope

#### Add an application

- Navigate to: Expose an API → Add a client application
- Fill in the field: `Client ID: 04b07795-8ddb-461a-bbee-02f9e1bf7b46` - this is the Azure CLI ID
- Select: `api://<Application ID URI>/App.Get.Access`
- Click: Add application

### Grant API permissions

- Navigate to: API permissions → Add a permission → My APIs → `az-cli-jwt`
- Select type: Delegated permissions
- Select permissions: `App.Get.Access`
- Click: Add permissions
- (Optional) click: Grant admin consent

### Assign groups to the application (optional: if the corresponding groups claim was selected)

- Navigate to: Azure Portal → Microsoft Entra ID → Enterprise applications → Choose `az-cli-jwt` → Users and groups → Add user/group
- Click: Add Assignment → Users and groups (None Selected) → Select → Assign

### Test

- Obtain the token: `az account get-access-token --resource <Application (client) ID> --tenant <Directory (tenant) ID> --query accessToken -o tsv`
- Navigate to [JWT Decoder](https://jwt.io/) and check "Decoded Payload"
- Pay attention to the fields:
  ```text
  "aud": <Application (client) ID>
  "appid": "04b07795-8ddb-461a-bbee-02f9e1bf7b46"
  "email"
  "groups"
  "unique_name"
  ```

Note that the token lifetime is approximately 1 hour.

## Configure OAuth2-Proxy

For this purpose, let's use the official [Helm chart](https://github.com/oauth2-proxy/manifests) (version `7.12.5` is used here)

I'll skip the installation process details and provide only an example of the configuration part needed to use OAuth2-Proxy as an `AuthService` for Emissary-ingress.

<details>
<summary>OAuth2-Proxy example configuration</summary>

```yaml
service:
  portNumber: 8080

metrics:
  enabled: false

config:
  configFile: false

proxyVarsAsSecrets: false

extraArgs:
  provider: entra-id
  prefer-email-to-user: true
  allowed-group: <Assigned groups> # List of allowed groups from 'Assign groups to the application' step (optional)
  client-secret: 00000000-0000-0000-0000-000000000000 # not used in the current implementation
  client-id: <Application (client) ID>
  oidc-issuer-url: https://login.microsoftonline.com/<Directory (tenant) ID>/v2.0
  extra-jwt-issuers: https://sts.windows.net/<Directory (tenant) ID>/=<Application (client) ID>
  skip-jwt-bearer-tokens: true
  skip-auth-strip-headers: true
  pass-access-token: true
  set-xauthrequest: true
  pass-basic-auth: false

  # cookies are not used in the current implementation
  cookie-secret: 1ZCX4DyXFQLU3T8qiEKRattEkokjP-OjACB5wPwAxBo=
  cookie-samesite: lax
  cookie-refresh: 0

  reverse-proxy: true
  proxy-prefix: /oauth2
  http-address: 0.0.0.0:4180
  upstream: static://200
  api-route: '^/protected-api/(.*)$'
  email-domain: '*'
  skip-provider-button: true

  silence-ping-logging: true
  auth-logging: true
  request-logging: true
  standard-logging: true
```

</details>

For more details, see the official documentation:
- [Overview Config Options](https://oauth2-proxy.github.io/oauth2-proxy/configuration/overview/)
- [Microsoft Entra ID Config](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/ms_entra_id/)

Thus, upon successful validation of the `JWT` token, we'll receive a response with code `200` containing headers:

```text
X-Auth-Request-User
X-Auth-Request-Groups
X-Auth-Request-Email
```

We can then use these headers in our backend, for example, for additional authorization.

## Configure Emissary-ingress

In my infrastructure, the Emissary-ingress controller is already present, so I won't describe its installation here.

Let's just look at the configuration process for the previously mentioned goals.

To understand the configuration, it's recommended to refer to the official documentation:
- [Authentication service](https://www.getambassador.io/docs/emissary/latest/topics/running/services/auth-service)
- [ExtAuth protocol](https://www.getambassador.io/docs/emissary/latest/topics/running/services/ext-authz)
- [Introduction to the Mapping resource](https://www.getambassador.io/docs/emissary/latest/topics/using/intro-mappings)
- [Advanced Mapping configuration](https://www.getambassador.io/docs/emissary/latest/topics/using/mappings)

The official OAuth2-Proxy Helm chart provides the ability to apply custom manifests.

Let's use this capability, assuming that Emissary-ingress and OAuth2-Proxy are installed in the same `namespace`.

Keep in mind that for existing `Mapping` resources, you need to add the `bypass_auth: true` parameter to skip the authentication process.

<details>
<summary>Emissary-ingress example configuration</summary>

```yaml
extraObjects:
  - apiVersion: getambassador.io/v3alpha1
    kind: AuthService
    metadata:
      name: oauth2-proxy
      namespace: '{{ .Release.Namespace }}'
    spec:
      auth_service: 'http://{{ .Release.Name }}-oauth2-proxy.{{ .Release.Namespace }}.svc.cluster.local:8080'
      allowed_authorization_headers:
        - X-Auth-Request-User
        - X-Auth-Request-Groups
        - X-Auth-Request-Email
        - X-Auth-Request-Preferred-Username

  - apiVersion: getambassador.io/v3alpha1
    kind: Mapping
    metadata:
      name: protected-api-mapping
      namespace: '{{ .Release.Namespace }}'
    spec:
      hostname: "*"
      service: <your_backend_microservice_uri>
      prefix: /protected-api/
      rewrite: /
      bypass_auth: false
      remove_request_headers:
        - authorization
```

</details>

## Install and Test

Let's install the Helm chart with the prepared configuration:

```bash
helm upgrade <release-name> oauth2-proxy --repo https://oauth2-proxy.github.io/manifests --version 7.12.5 --values <customized-values-file> --namespace <release-namespace> --install
```

Let's do a simple connection test. For this example, we'll use Elasticsearch as a backend service:

```bash
TOKEN=$(az account get-access-token --resource <Application (client) ID> --tenant <Directory (tenant) ID> --query accessToken -o tsv)
curl -v -H "Authorization: Bearer $TOKEN" https://<your_domain_name>/protected-api/
```

In case of success, you'll get a response like this:

```json
{
  "name": "elasticsearch-master-0",
  "cluster_name": "elasticsearch",
  "cluster_uuid": "XXXXXXXXXXXXX",
  "version": {},
  "tagline": "You Know, for Search"
}
```

In case of failure (the `JWT` token failed validation), you'll get a response with code `401`.

## Conclusion

In this article, we've examined a basic example of securing microservice APIs using OAuth2-Proxy functioning as a `JWT` validation middleware. This setup adds an essential security layer to our microservices, ensuring that only authenticated users can access our APIs.

However, OAuth2-Proxy's capabilities extend beyond what we've covered. With minor configuration adjustments, you can enable it to function as a complete authentication middleware. This allows you to reuse our `AuthService` to protect frontend applications as well.

That's all there is to it. I wish you success in building secure API endpoints!
