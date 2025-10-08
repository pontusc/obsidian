Information about environment vars
https://docs.goauthentik.io/install-config/configuration/

Information for bootstrap
https://docs.goauthentik.io/install-config/automated-install

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
          consumer_key: clientid
          consumer_secret: secret
          enrollment_flow: !Find [authentik_flows.flow, [slug, default-source-enrollment]]
          authentication_flow: !Find [authentik_flows.flow, [slug, default-authentication-flow]]

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