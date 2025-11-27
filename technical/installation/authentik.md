---
title: Authentik
parent: Installation
has_children: false
nav_order: 4
---

# Authentik (optional)

Only for internal SSO.

## ArgoCD

argo-cd/templates/sealed-oidc-authentik-client-secret.yaml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: oidc-authentik-client-secret
  namespace: argo-cd
spec:
  encryptedData:
    oidc.authentik.clientSecret: AgCrTymTzo5MvUTprglpRRtGzgMo8R7UGAuu4iOoiev0ayQBGLOZnGM5PhLFVQ+ZlgSXZjOR5L5e3iySp/wn4YzTy9z04WVvrk/rf3Y1TbIZLV8zJYc77Av2TgWv3F/XPJ69nTbUNwOzI3Mn5yMh+1CwkEI3dzvW76kbrphfKe0HOmyuhSSGH/6aW40x/7S6XjGGLtfqqTpOdfuAwTPs+gojU83RqwnU+Fj9gzWHqVtBSEN/dtsr75btp38XiTWXmo5dkp0uYJayCHIm3F45UzuBCU2dgKr6DZufpeK69/qtUt0ngPf2qOXjprRKLWywTm94DaZ1jehXaR5u9lk8DmISxfaoZuWo3Ls5iFUeG6wNxPYi8Xc9kKxbnrXFAGMBljy16EaRSPNYK4cDvUPHfbRllZvctozAt+bHUx6Hp9m64T7OmtcyFnNCdakYXq7T2PmyJn1nY+DqhnU6E7pJBxyqLlrYoqNfaoRQYgGC1B5Z9fjP2IuZ5nxroLHj9ldjvF5dAvSOlNrb+joUcA3EYlk4aol+Npl1Obd1aFPEFVD3sQj0l3cKr4z0qa4EozWbSV8Tgvl0lo7o97nlZGUbtko+hMzdG9zwTHkYvWipU8aDXBvBCsISJYt3MiGRVdBoHlZwYtYuqkTrusf7ZhSU6H/iwB3lIexEs8nV4C5+1aUXSOGHbbC82uDqpVj9dHYdmHV5bmxHYN+MAMva2BDjd0lWDx7oH98KsUguG7tMwrBKmFFHHzhWu2F0WZSuh1IFyCMRmO19TLhDTJL36JS8EoCm/ye0dfzGQ7lW2DOmfeqS2fE2Jh7v3I5YEd8IkIvGejF9ATmhDDt12/0+t5JZXSOzdzY3WBCtUhwdd8MCXO0CwA==
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/part-of: argocd
      name: oidc-authentik-client-secret
      namespace: argo-cd
    type: Opaque
```

Configuration to change in `applications/argo-cd/values.yaml`:

```yaml
global:
  configs:
    cm:
      oidc.config: |
        name: AarhusAI SSO
        issuer: https://auth.kom1.deranged.dk/application/o/argocd-kom1/
        clientID: bJDCFviyQcUXVIr2uXqbos5l6ZjIjBZlY0j647T4
        clientSecret: $oidc-authentik-client-secret:oidc.authentik.clientSecret
        requestedScopes: [openid, profile, email, groups]
```

## Grafana/Prometheus

prometheus-stack/templates/sealed-auth-generic-oauth-secret.yaml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: auth-generic-oauth-secret
  namespace: monitoring
spec:
  encryptedData:
    client_id: AgAsCr1A2QChQSocZtiy8dZPfsA2AtRU7CIFH/5eaVwRB0BZJ8NbpifFC+MJs7nKZOjeEJB3A+x/AB4WyJvf+dSgiYWpKmtMQjx+DobtErIfEQyQq6rdpjYzefNG9KdNWA5j5LspcGhrcXyq3kJIKNZHgn0PPALU3Vo3wGkoCj5MBxcMi1QoKaSOdbwh75JZN5ASrgRi1NHY17izIPOB7bEKDR+LTLe1AW9bj0xZAdCOg5MSde9W7gSBmVErdIWvvpd3H1/1IkDHHJ9hvhGjyLDS8kGlIdXmAYG+14C0sRgSYeV6lTpirv7sJkprYIafIvXIhn8M3y8aXtmMUwyW6n3kqpQQfuQdm9p+g345zLwC/EJU3oizosI8N8AAJRgMGsZFT9W2DGApCp+j0TQBPk/RiQRTnaA7BDdeLoYUHSk0tFaQdsMZ9jfaRD/mgw3xZS0ds++uJUKTtmOdBbIwOGKif9UAP/OfYNMfVD8PJTRg04VwZ+X88d9DcGAbuLO3knEa8+HjpTSYtaP+raVezM1LqhfgIYlTZFJlTSLZj50KQjhd/LTFD2nu79t2c6rCW/1qw4+Nd+AiI5NNNl9l2t4yJycCswGIFLoxlmRW7KiogjMvGHCFPniPS5K3nY37LtvcVxqa9Qr4DIztcr0PBIq5O8WQ2njZlvsDHSXBS6eLFcrO2ojcnLFzMQZ6i8GGW9vyYRjC0CylCPjFUWB4pvcLW3jChsp7eXWWwFDiqH9+L0tI/ntKT5O2
    client_secret: AgAWOHFBqsp5tYjDLrJeBeBO8ehvY8HufAyrdBM+MT/rl1dSxEVrK2U1FtP1w1GAd4IdKJ+zI961jZs9iz3yo3m4VNEqEwOpEhhYALWJ6ah4rCS3uj210vYW2FwdTjMv2YtTydw6IFMT818yn0Y/ndt7DMtlmqLAJNiT+JltRglZPnIlMhG59hAri/OIWN/KC8MZjNwlwUFwlwTxJ0+OpNCntpZDaMFTo55jBM6pmNj/+MQJ6Mb7tAlNabWfVzDWqwOq6le5GcinbMUwum+7ztHy8oKQ8DkLMHZdHB0mv5tTqpm337MdrVDkqcvWvu7MguZvI5DBVogGUH9O3Ae3p8rKpPRwdup9gtTLEPpUi5DLv12+Ka3NAefWzVIY1QRQlwCK6YPxUoQ6TrbD7LEK4nCeuOJot7u/oo65kiicI5vnQYjMc4VFOBsCgi391/Sd8baSH68Hd9P8aRzn9aBCGkC4NO91GRCaj1OU/m5gkRKk8+k2PwWP70xzo27ZiDKz/Yc/BGeI0EbvlAqsfJflfWF88blUMF64mbSPButzmIzbCRWtAi7z02KhOLFDwBHTnImRl5cgxGSR0mT89yk9W5wLXNWAsmzBUzBAxlOHHVOgEr8yHbBg6txj5PVcjbbu9JFDQAm1oGfuOm6J+/g/drJk9SkCMPbx1u+xKTbAUEdSmdtnM4/fimhKjmcKqM/1OU0tkAq8gSYaK7o38BQzVTfsefFMaHjg/EnxbQ3rnQJsvr7zf/eVV/tIdrhQG5RGUjQSLGG2WMZLplmTtVVqaAZsBgp72CxLemk25oQ8CyytU/Shb+R9AC4MOEM6aM1JmGWs4QbmfsVmG+qwF/s95ho2mQMKCGPYMPCnObrqxms/tA==
  template:
    metadata:
      creationTimestamp: null
      name: auth-generic-oauth-secret
      namespace: monitoring
    type: Opaque
```

prometheus-stack/values.yaml

```yaml
grafana.ini:
  server:
    auth.basic:
      disable_login_form: true
    auth:
      signout_redirect_url: https://auth.<FQDN>/application/o/grafana-kom1/end-session/
      oauth_auto_login: true
    auth.generic_oauth:
      name: AarhusAI SSO
      enabled: true
      client_id: $__file{/etc/secrets/auth_generic_oauth/client_id}
      client_secret: $__file{/etc/secrets/auth_generic_oauth/client_secret}
      scopes: openid profile email
      auth_url: https://auth.<FQDN>/application/o/authorize/
      token_url: https://auth.<FQDN>/application/o/token/
      api_url: https://auth.<FQDN>/application/o/userinfo/
      # Optionally map user groups to Grafana roles
      role_attribute_path: contains(groups, 'grafana') && 'Admin' || 'Viewer'
extraSecretMounts:
  - name: auth-generic-oauth-secret-mount
    secretName: auth-generic-oauth-secret
    defaultMode: 0440
    mountPath: /etc/secrets/auth_generic_oauth
    readOnly: true
```
