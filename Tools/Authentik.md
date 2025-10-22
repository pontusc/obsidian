Information about environment vars
https://docs.goauthentik.io/install-config/configuration/

Information for bootstrap
https://docs.goauthentik.io/install-config/automated-install
## Example secret for bootstrap
Used to bypass the normal setup.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: authentik-secrets
  namespace: authentik
type: Opaque
stringData:
  AUTHENTIK_SECRET_KEY: "yourkey"
  AUTHENTIK_BOOTSTRAP_PASSWORD: "yourpw"
  AUTHENTIK_BOOSTRAP_TOKEN: "yourtoken"
  AUTHENTIK_BOOTSTRAP_EMAIL: "admin@authentik.local"
```
## Blueprints for Authentik in K8s
Needs to be mounted, see helm chart values for blueprints. Filename has to be 
```*.yaml``` for Authentik to source it, even if mounted.
## Google OAuth provider
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: google-oauth-source-blueprint
  namespace: authentik
type: Opaque
stringData:
  google-oauth-source.yaml: |
    version: 1
    metadata:
      name: google-oauth-source
    entries:
      - model: authentik_sources_oauth.oauthsource
        state: created
        identifiers:
          slug: google
        attrs:
          name: Google
          provider_type: google
          consumer_key: clientId from google console
          consumer_secret: secret from google console
          enrollment_flow: !Find [authentik_flows.flow, [slug, default-source-enrollment]]
          authentication_flow: !Find [authentik_flows.flow, [slug, default-source-authentication]]
```

## OAuth Provider for Argo
Store below as a file in a secret
```yaml
version: 1
metadata:
  name: argocd-oauth-provider
entries:
  - model: authentik_providers_oauth2.oauth2provider
    state: created
    identifiers:
      name: ArgoCD
    attrs:
      invalidation_flow: !Find [authentik_flows.flow, [slug, default-provider-invalidation-flow]]
      authorization_flow: !Find [authentik_flows.flow, [slug, default-provider-authorization-implicit-consent]]
      client_type: confidential
      client_id: argocd
      client_secret: GENERATE_SELF
      redirect_uris:
        - matching_mode: strict
          url: https://argocd.cloud.nejlikka.com/api/dex/callback
        - matching_mode: strict
          url: https://localhost:8085/auth/callback
      signing_key: !Find [authentik_crypto.certificatekeypair, [name, authentik Self-signed Certificate]]
      property_mappings:
        - !Find [authentik_providers_oauth2.scopemapping, [scope_name, openid]]
        - !Find [authentik_providers_oauth2.scopemapping, [scope_name, email]]
        - !Find [authentik_providers_oauth2.scopemapping, [scope_name, profile]]

```

## OAuth Application for Argo

```yaml
version: 1
metadata:
  name: argocd-integration
entries:
  # Dependencies
  - model: authentik_blueprints.metaapplyblueprint
    attrs:
      identifiers:
        name: argocd-oauth-provider

  - model: authentik_core.group
    state: created
    identifiers:
      name: ArgoCD Admins
    attrs:
      is_superuser: false

  - model: authentik_core.group
    state: created
    identifiers:
      name: ArgoCD Viewers
    attrs:
      is_superuser: false

  - model: authentik_core.application
    state: created
    identifiers:
      slug: argocd
    attrs:
      name: ArgoCD
      provider: !Find [authentik_providers_oauth2.oauth2provider, [name, ArgoCD]]

```
## Forward Auth
```yaml
version: 1
metadata:
  name: forward-auth
entries:
  # Create application and provider
  - model: authentik_providers_proxy.proxyprovider
    identifiers:
      name: Forward Auth Provider
    attrs:
      authorization_flow: !Find [authentik_flows.flow, [slug, default-provider-authorization-implicit-consent]]
      invalidation_flow: !Find [authentik_flows.flow, [slug, default-provider-invalidation-flow]]
      mode: forward_domain
      cookie_domain: cloud.nejlikka.com
      external_host: https://authentik.cloud.nejlikka.com

  - model: authentik_core.application
    identifiers:
      slug: forward-auth
    attrs:
      name: Forward Auth
      provider: !Find [authentik_providers_proxy.proxyprovider, [name, Forward Auth Provider]]
      policy_engine_mode: all

  - model: authentik_outposts.outpost
    identifiers:
      name: authentik Embedded Outpost
    attrs:
      providers:
        - !Find [authentik_providers_proxy.proxyprovider, [name, Forward Auth Provider]]

  - model: authentik_policies.policybinding
    identifiers:
      target: !Find [authentik_core.application, [slug, forward-auth]]
      group: !Find [authentik_core.group, [name, authentik Admins]]
      order: 0
    attrs:
      enabled: true

```