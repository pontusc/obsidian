Information about environment vars
https://docs.goauthentik.io/install-config/configuration/

Information for bootstrap
https://docs.goauthentik.io/install-config/automated-install
## Example secret for bootstrap
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
## Example blueprint for Google OAuth
Needs to be mounted, see helm chart values for blueprints. Filename has to be \*.yaml for Authentik to find it.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: google-oauth-blueprint
  namespace: authentik
type: Opaque
stringData:
  google-oauth.yaml: |
    version: 1
    metadata:
      name: google-oauth-source
    entries:
      - model: authentik_policies_expression.expressionpolicy
        identifiers:
          name: google-email-to-username
        attrs:
          expression: |
            email = request.context["prompt_data"]["email"]
            request.context["prompt_data"]["username"] = email
            return True

      - model: authentik_sources_oauth.oauthsource
        identifiers:
          slug: google
        attrs:
          name: Google
          provider_type: google
          consumer_key: clientid from google console
          consumer_secret: secret from google console
          enrollment_flow: !Find [authentik_flows.flow, [slug, default-source-enrollment]]
          authentication_flow: !Find [authentik_flows.flow, [slug, default-source-authentication]]

      - model: authentik_policies.policybinding
        state: created
        identifiers:
          order: 0
          target: !Find [authentik_flows.flow, [slug, default-source-enrollment]]
          policy: !Find [authentik_policies_expression.expressionpolicy, [name, google-email-to-username]]
        attrs:
          enabled: true

      - model: authentik_stages_identification.identificationstage
        identifiers:
          name: default-authentication-identification
        attrs:
          sources:
            - !Find [authentik_sources_oauth.oauthsource, [slug, google]]
```

## Example for ArgoCD SSO
Follow the authentik documentation for the Argo configuration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-sso-blueprint
  namespace: authentik
type: Opaque
stringData:
  argocd-oauth-sso.yaml: |
    version: 1
    metadata:
      name: argocd-application
    entries:
      - model: authentik_core.group
        identifiers:
          name: ArgoCD Admins
        attrs:
          is_superuser: false

      - model: authentik_core.group
        identifiers:
          name: ArgoCD Viewers
        attrs:
          is_superuser: false

      - model: authentik_providers_oauth2.oauth2provider
        identifiers:
          name: ArgoCD
        attrs:
          invalidation_flow: !Find [authentik_flows.flow, [slug, default-provider-invalidation-flow]]
          authorization_flow: !Find [authentik_flows.flow, [slug, default-provider-authorization-implicit-consent]]
          client_type: confidential
          client_id: argocd
          client_secret: generate a secret for this
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

      - model: authentik_core.application
        identifiers:
          slug: argocd
        attrs:
          name: ArgoCD
          provider: !Find [authentik_providers_oauth2.oauth2provider, [name, ArgoCD]]
```