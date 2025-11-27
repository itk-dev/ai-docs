---
title: Installation
parent: Technical documentation
has_children: false
---

# AarhusAI GitOps usage

To install AarhusAI with this GitOps [template](https://github.com/AarhusAI/helm-deployments), you need to have a
Kubernetes cluster with Helm 3 installed and [Ceph](https://rook.io/docs/rook/v1.12/Getting-Started/quickstart/)
filesystem installed.

This installation documentation must be followed and executed in the correct order as described below, because the
different services require other services to be ready before they can be installed. In particular, ArgoCD needs to be
installed before Argo resources and sealed secrets, before Prometheus and Grafana, etc. The application requires you to
generate new API keys, etc., and use sealed secrets to store them before a given service is installed, and some need the
keys/secrets from one service to communicate with another.

Also, because all configuration is stored in GitOps as code, you need to update secrets and URLs to match your domain
and setup. This document will guide you through the process of setting up AarhusAI.

## Bootstrap continuous deployment

The first step is
to [create](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template)
a new GitHub repository based on this [template repository](https://github.com/AarhusAI/helm-deployments).

### ArgoCD

For continuous development and using GitOps to handle updates and configuration
changes, [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) needs to be bootstrapped into the cluster.

Configuration to change in `applications/argo-cd/values.yaml`:

```yaml
global:
  domain: argo.<FQDN>
```

Then install ArgoCD using the following commands:

```shell
helm repo add argocd https://argoproj.github.io/argo-helm
helm repo update
cd applications/argo-cd
helm dependency build
kubectl create namespace argo-cd
helm template argo-cd . -n argo-cd | kubectl apply -f -
```

You can check that ArgoCD is running by opening the ingress URL:

```shell
kubectl get ingress argocd-server -n argo-cd -o jsonpath='{.spec.rules[0].host}' | xargs -I {} open "https://{}"
```

You can log into the web-based user interface with the username `admin` and get the password with this command:

```shell
kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

As an optional configuration, later on, we can install [Authentik](https://goauthentik.io/) as a single-sign-on (SSO)
provider for the internal services in the cluster.

### Argo resources (applications)

Now we can install the Argo resources that define the applications and their configuration. But first, we need to
change the repository URL to our own repository.

Edit `applications/argo-cd-resources/templates/applications.yaml`:

```yaml
spec:
  source:
    repoURL: https://github.com/aarhusai/<YOUR REPO>.git
```

One also needs to change the configuration `sourceRepos` **in every**
`applications/argo-cd-resources/templates/projects/*.yaml` file for each application so Argo knows which repos to stay
in sync with.

The last step in Argo installation is to install the resources:

```shell
cd applications/argo-cd-resources/
helm template argo-cd-resources . -n argo-cd | kubectl apply -f -
```

This will install all the applications from `applications/argo-cd-resources/values.yaml` that are set to
`automated: true`, which are all the applications that do not need configuration changes. All the other applications
will need to have their configuration updated and changed to automatically sync with the repository.

## Observability

Loki, Tempo, and Alloy are automatically installed by ArgoCD above as they do not require configuration changes. But
Grafana needs some basic domain name and URL configuration:

```yaml
grafana:
  ingress:
    hosts:
      - obs.<FQDN>
    tls:
      hosts:
        - obs.<FQDN>

  additionalDataSources:
    grafana.ini:
      server:
        domain: obs.<FQDN>
        root_url: https://obs.<FQDN>
```

Change `automated` to true in `applications/argo-cd-resources/values.yaml` for the `prometheus-stack` application.

You can access Grafana by opening the ingress URL:

```yaml
kubectl get ingress -n monitoring -o jsonpath='{.items[*].spec.rules[*].host}' | tr ' ' '\n' | grep grafana | xargs -I {} open "https://{}"
```

Log into grafana with the username `admin` and password:

```yaml
kubectl get secret -n monitoring prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

## Sealed Secrets

Before installing the rest of the applications, we need to install `kubeseal` to generate sealed secrets that can be
safely committed to public GitHub repositories. (Yes, one can reconfigure ArgoCD to use private repositories, but one
would still use sealed secrets to protect secrets. Accidental exposure happens, and human errors occur.)

Read more about [sealed secrets](https://github.com/bitnami-labs/sealed-secrets).

### Install kubeseal

Follow the [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal) install guide based on
which system you are on to install the CLI tool.

Next, download the public certificate:

```yaml
kubeseal --fetch-cert > public-cert.pem
```

Optionally use the ingress endpoint (`--cert http://sealed-secrets.<FQDN>/v1/cert.pem`). To enable the ingress endpoint
for sealed secrets, edit `applications/sealed-secrets/values.yaml` and change `ingress.enabled` to true. Commit and
push the changes to the repository and wait for the redeployment of sealed secrets by ArgoCD.

### Seal a secret (example)

You can place the unsealed secret anywhere you want, but this repository has the path `applications/**/local-secrets`
in `.gitignore`. So with this in mind, the `kubeseal` commands would be in the form:

```shell
kubectl create -f local-secrets/<NAME_OF_APPLICATION>-secret.yaml --dry-run=client -o yaml | \ 
kubeseal --cert public-cert.pem --format yaml > templates/sealed-<NAME_OF_APPLICATION>-secret.yaml
```

With an unsealed secret like this (here for LiteLLM):

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: litellm-secrets
  namespace: litellm
stringData:
  PROXY_MASTER_KEY: <KEY>
  CA_VLLM_LOCAL_API_KEY: <KEY>
```

When you need to generate a random key, you can use this command:

```shell
echo "$(cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 32 | head -n 1)"
```

## vLLM (Mistral 24B model)

Read more about [vLLM production stack](https://github.com/vllm-project/production-stack).

If you want to use a locally hosted model on local hardware in the cluster, you need to configure the vLLM application,
which requires you to first create a [Hugging Face](https://huggingface.co/) account. Models are downloaded from here,
and it's free. Create an account and generate an access token.

Create an unsealed secret for `hf-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: hf-secret
  namespace: vllm
stringData:
  TOKEN: <TOKEN>
```

Seal it:

```shell
kubectl create -f local-secrets/hf-secret.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-hf-secret.yaml
```

Next, create a secret for the vLLM API. This key is used internally in the cluster to communicate with the vLLM service,
and you should generate a random key:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: vllm-secret
  namespace: vllm
stringData:
  KEY: <GENERATED API-KEY>
```

Seal it:

```shell
kubectl create -f local-secrets/vllm-secret.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-vllm-secret.yaml
```

Change `automated` to true in `applications/argo-cd-resources/values.yaml` for the `vllm` application. Add, commit, and
push the changes to have Argo deploy vLLM automatically. **Note** that it may take some time to deploy vLLM for the
first time as it downloads the model from Hugging Face and loads it into GPU memory.

## LiteLLM

This is an LLM proxy that can be used to generate virtual API keys and keep track of token costs. It is able to talk to
many different LLM servicing frameworks. It extends the ability to connect to Azure, Claude, and OpenAI APIs, which are
not
necessarily supported by the frontend (Open WebUI). It also has a built-in, simple chat interface which can be used to
debug connections to models.

It is also the place to set up guardrails. We currently ship with a single custom guardrail that ensures that the
context window for Mistral is not "overflowed" by ensuring that the context window is not larger than the model's
maximum context window. So if you are using another model, you may need to adjust the guardrail configuration.

[LiteLLM](https://docs.litellm.ai/docs/) uses a PostgreSQL database to store the virtual API keys (if used) and usage
statistics. So we need to create a secret for the database credentials:

Create the file `local-secrets/litellm-cloudnative-pg-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: cloudnative-pg-cluster-litellm
  namespace: litellm
stringData:
  username: litellm
  password: <GENERATED PASSWORD>
```

Seal it:

```shell
kubectl create -f local-secrets/litellm-cloudnative-pg-secret.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-cloudnative-pg-secret.yaml
```

Next, we need to set a master key and API keyes for the confiuraed end-points. Create the file
`local-secrets/litellm-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: litellm-secrets
  namespace: litellm
stringData:
  PROXY_MASTER_KEY: sk-<GENERATE RANDOM KEY>
  CA_VLLM_LOCAL_API_KEY: <SET VLLM SECRET>
```

Seal it:

```shell
kubectl create -f local-secrets/litellm-secrets.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-litellm-secrets.yaml
```

**NOTE**: If you want to access the web UI, you need to edit `litellm-values.yaml` and enable ingress and set a domain
name to access it. You can use the master key from the `litellm-secrets` secret as the password for the web UI.

Change `automated` to true in `applications/argo-cd-resources/values.yaml` for the `litellm` application. Add,
commit, and push the changes to have Argo deploy LiteLLM automatically.

## Document ingestion route (optional recommended)

This is a [FastAPI](https://fastapi.tiangolo.com/) proxy that tries to find the best way to extract texts for RAG before
embedding when files and web search results are processed. Currently, it supports Tika for the backend and makes some
decisions about whether to extract clean text or Markdown. The idea is that it should be used later on to route input
data to the correct extractor based on, e.g., file types, etc. It could use tools such as Marker, MarkItDown, Docling,
and Tika.

It is optional, as Open WebUI can talk directly to Tika or Docling (but it's recommended for better text extraction).

Create the file `local-secrets/secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: secrets
  namespace: doc-ingestion
stringData:
  API_KEY: sk-<RANDOM GENERATED KEY>
```

Seal it:

```shell
kubectl create -f local-secrets/secrets.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-secrets.yaml
```

Change `automated` to true in `applications/argo-cd-resources/values.yaml` for the `doc-ingestion` application. Add,
commit, and push the changes to have Argo deploy automatically.

## SearXNG (optional recommended)

[SearXNG](https://docs.searxng.org/) is a metasearch engine and is used to find relevant documents for a given
query search on the internet. Open WebUI can be configured to use a range of online web search pages (but only one at a
time). SearXNG allows us to search a huge range of different search engines, and it is also possible to add custom
search engines.

It also ensures searches are anonymous and not trackable from one query to the next, thereby ensuring that one does not
affect another. It filters out paid results and AI-generated results.

In Aarhus, we have made an engine that searches [https://aarhus.dk/search](https://aarhus.dk/search) and places its
results high in the final federated search. For reference,
see the [implementation](https://github.com/AarhusAI/aarhusai-docker/blob/main/.docker/searxng/aarhus.py).

Create the file `local-secrets/searxng-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: searxng-secrets
  namespace: searxng
stringData:
  SEARXNG_SECRET: <RANDOM GENERATED SECRET>
```

Seal it:

```shell
kubectl create -f local-secrets/searxng-secret.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-searxng-secret.yaml
```

Change `automated` to true in `applications/argo-cd-resources/values.yaml` for the `searxng` application. Add,
commit, and push the changes to have Argo deploy automatically.

## Open WebUI

[Open WebUI](https://docs.openwebui.com/) is the main application that binds the whole AarhusAI stack together by
providing the framework to chat with the LLM(s) and make RAG-based models. It has a lot of features that can be
controlled
through [environment variables](https://docs.openwebui.com/getting-started/env-configuration). The deployment comes with
some default values that match the current AarhusAI setup, but you can change them to fit your needs in `values.yaml`.

Also note that `was-middleware.yaml` in templates should be updated to match your municipality's
"tilgængelighedserklæringer" to redirect correctly to it.

Create the file `local-secrets/openwebui-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: openwebui-secrets
  namespace: openwebui
stringData:
  OPENAI_API_KEY: <LITELLM API KEY>
  WEBUI_SECRET_KEY: <RANDOM GENERATED SECRET>
  RAG_OPENAI_API_KEY: <EMBEDDING API KEY>
  EXTERNAL_DOCUMENT_LOADER_API_KEY: <DOC INGESTION ROUTE>
```

Create the file `local-secrets/cloudnative-pg-cluster-openwebui-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: cloudnative-pg-cluster-openwebui
  namespace: openwebui
stringData:
  username: openwebui
  password: <GENERATED PASSWORD>
```

Change `automated` to true in `applications/argo-cd-resources/values.yaml` for the `openwebui` application. Add,
commit, and push the changes to have Argo deploy automatically.

-----------------------------

-----------------------------

@TODO: below this point is not yet documented

-----------------------------

-----------------------------

## Authentik (optional)

Only for internal SSO.

### ArgoCD

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

### Grafana/Prometheus

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
