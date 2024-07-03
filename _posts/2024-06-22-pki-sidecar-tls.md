---
layout: post
title: TLS PKI sidecar container
date: 2024-07-03 12:00:00-0000
description: Utiliser un sidecar pour gérer le TLS avec des certificats dynamiques Vault
tags: tls sidecar vault kubernetes
categories: education
giscus_comments: true
related_posts: false
toc:
  beginning: true
---

# Introduction

Dans cet article, nous allons voir comment configurer un sidecar container pour gérer le TLS en utilisant le secret engine PKI de Vault. Contrairement au [sidecar TLS simple](/blog/2024/sidecar-tls), nous allons faire de sorte que les certificats se renouvellent automatiquement sans interruption de service.

Notre Pod va contenir :
- un serveur web qui répond "hello world" sur le port 80
- un sidecar qui va gérer le TLS sur le port 443 avec des certificats téléchargés depuis Vault
- un sidecar Vault Agent qui va télécharger les certificats et les maintenir à jour dans le volume Kubernetes

Nous allons utiliser un Vault avec le [secret engine PKI activé et configuré](/blog/2024/vault-pki).

Lorsqu'un certificat est généré à l'aide de la fonction de modèle `pkiCert`, le modèle de Vault Agent adopte les comportements suivants pour la récupération et la réémission des certificats :
- il récupère un nouveau certificat au démarrage de l'Agent s'il n'en a pas déjà généré un précédemment ou si le certificat actuellement généré a expiré
- lors d'une ré-authentification automatique de l'Agent (par exemple, en cas d'expiration du jeton), il ne récupère pas de nouveau certificat à moins que le certificat actuellement généré ait expiré


# Configuration de Vault
Nous allons configurer Vault afin d'activer l'authentification Kubernetes.

```bash
SA_CA_CRT=$(kubectl config view --raw --minify --flatten -o jsonpath={.clusters[].cluster.certificate-authority-data} | base64 -d)

vault auth enable kubernetes

vault write auth/kubernetes/config \
     disable_local_ca_jwt="true" \
     kubernetes_host="https://192.168.0.20:6443" \
     kubernetes_ca_cert="$SA_CA_CRT" \
     issuer="https://kubernetes.default.svc.cluster.local"

vault read auth/kubernetes/config

vault write auth/kubernetes/role/authkube \
     bound_service_account_names=app-sa \
     bound_service_account_namespaces=poc-sidecar-tls-pki \
     token_policies=pki \
     ttl=24h
```

# Configuration de Kubernetes

## Création du namespace

ns.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-sidecar-tls-pki
```

```bash
kubectl apply -f ns.yaml
kubectl config set-context --current --namespace=poc-sidecar-tls-pki
```

## Déploiement des configmaps

cm-agent-sidecar.yaml : contient la configuration du vault agent qui va permettre de maintenir l'authentification à Vault et obtenir les fichiers tls.crt et tls.key pour le domaine foo.home
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-sidecar-cm
  namespace: poc-sidecar-tls-pki
data:
  vault-agent-config.hcl: |
    # Comment this out if running as sidecar instead of initContainer
    exit_after_auth = false

    pid_file = "/home/vault/pidfile"

    auto_auth {
        method "kubernetes" {
            mount_path = "auth/kubernetes"
            config = {
                role = "authkube"
            }
        }

        sink "file" {
            config = {
                path = "/home/vault/.vault-token"
            }
        }
    }


    template {
    destination = "/etc/nginx/ssl/tls.crt"
    contents = <<EOT
    {{- with pkiCert "pki_intermediate/issue/home" "common_name=foo.home" "alt_names=localhost,bar.home" "ttl=24h" -}}
    {{ .Data.Cert }}
    {{ .Data.CA }}
    {{ if .Data.Key }}
    {{ .Data.Key | writeToFile "/etc/nginx/ssl/tls.key" "" "" "0644" }}
    {{ end }}
    {{ end }}    
    EOT
    perms = 0644
    }
```


cm-app.yaml : contient des variables d'environnement
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-cm
  namespace: poc-sidecar-tls-pki
data: 
  VAULT_ADDR: "https://vault.home"
  VAULT_CACERT: "/etc/ssl/certs/vault.pem"
```

cm-chain-ca.yaml : contient les certificats Intermediate CA et Root CA concaténés ensemble
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: chain-ca-cm
  namespace: poc-sidecar-tls-pki
data:
  vault.pem: |
    -----BEGIN CERTIFICATE-----
    MIIC7DCCApKgAwIBAgIUPtQcuNvBFdJzqo3AoRFgOSsiaG4wCgYIKoZIzj0EAwIw
    LDEYMBYGA1UEChMPRGFuZyBDb25zdWx0aW5nMRAwDgYDVQQDEwdSb290IENBMB4X
    DTI0MDYxOTEyMjgzOVoXDTI5MDYxODEyMjkwOVowGjEYMBYGA1UEAxMPSW50ZXJt
    ZWRpYXRlIENBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvP/UEhbp
    4OCr9tzSJw3h8GtaZpXCel6raybntq66uZwGTMbWFqUdFqTzBca+5EJgkz7cGLse
    h4zJu0JYFB1cE0NFzyMaonVdv5T1cLjvoiklzZcVQrC8XQrImasT8Z3qgoZRLQ79
    emed0ubafaKxRn+0srTmFZEQ8ANguFLyQI6XqI4n/9wy5JKJK8y6iAMCzidkx2+r
    pCYzeS0CkiFTV9wbB89haspZdPOeB/4oJuZiMotBd+lav5P4DzCfT+Zy7donJtcB
    91+SvOeDMmhKR/Ruir+1sFYXDPMq+gVf9hjavuDu82ylF+D2Vpz4VQ/WA8DzlP/4
    5yK/K8P9zRDBBwIDAQABo4HYMIHVMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
    BTADAQH/MB0GA1UdDgQWBBTqnRa/wFhgeQCNyptE7c3LHVOKWDAfBgNVHSMEGDAW
    gBRkw7RXwBwEJ/UsgMkz6qRiVvPVhTA9BggrBgEFBQcBAQQxMC8wLQYIKwYBBQUH
    MAKGIWh0dHBzOi8vdmF1bHQuaG9tZS92MS9wa2lfcm9vdC9jYTAzBgNVHR8ELDAq
    MCigJqAkhiJodHRwczovL3ZhdWx0LmhvbWUvdjEvcGtpX3Jvb3QvY3JsMAoGCCqG
    SM49BAMCA0gAMEUCIQCh7DpEdllzu/1+HITWQdTbOOfJ7IDe6i/n8lCRniRqeAIg
    LJTfEaxnbER0Qm+Mb4JB0yINqesGzsIdKoLCzp9dUas=
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIBmzCCAUGgAwIBAgIRAPz9FOe+IXCzfMCkQu5ImUcwCgYIKoZIzj0EAwIwLDEY
    MBYGA1UEChMPRGFuZyBDb25zdWx0aW5nMRAwDgYDVQQDEwdSb290IENBMCAXDTIz
    MTEyNzExMTgyMloYDzIxMjMxMTI4MTExODIyWjAsMRgwFgYDVQQKEw9EYW5nIENv
    bnN1bHRpbmcxEDAOBgNVBAMTB1Jvb3QgQ0EwWTATBgcqhkjOPQIBBggqhkjOPQMB
    BwNCAAR5BP2rtk2YImzxHBQPnsvvAk7b5HetisGtIu6rtfy3I6Q98pgVOq3PVyCY
    Y3KR4mZWosAjaeOS/rK0W40YgxiSo0IwQDAOBgNVHQ8BAf8EBAMCAqQwDwYDVR0T
    AQH/BAUwAwEB/zAdBgNVHQ4EFgQUZMO0V8AcBCf1LIDJM+qkYlbz1YUwCgYIKoZI
    zj0EAwIDSAAwRQIgDdrAKVlOaZY1LIQqJJnaiBIxVtiVb8hcIWQZfb91CscCIQC/
    PY73ybZ9VbmIskEy59C8VRQdsUA6JocIUlEfTZHQJg==
    -----END CERTIFICATE-----   
```

```bash
kubectl apply -f cm-agent-sidecar.yaml,cm-app.yaml,cm-chain-ca.yaml
```

## Création du Service Account et du JWT associé

Un Service Account représente une identité pour un Pod. Le JWT associé au service account va contenir 2 informations qui vont permettre à Vault d'authentifier le Pod : le nom du Namespace et le nom du Service Account

secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: poc-sidecar-tls-pki
  annotations:
    kubernetes.io/service-account.name: app-sa
type: kubernetes.io/service-account-token
```

sa.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: poc-sidecar-tls-pki
```

```bash
kubectl apply -f secret.yaml,sa.yaml
```

## Affectation des permissions au Service Account

Pour définir des permissions pour un service account, on utilise :
- un Role (objet "RoleBindings") pour un namespace particulier
- ou ClusterRole (objet "ClusterRoleBinding") pour tout le cluster

crb.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: app-crb
  namespace: poc-sidecar-tls-pki
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: poc-sidecar-tls-pki
```

```bash
kubectl apply -f crb.yaml

# check
kubectl describe clusterrolebinding.rbac.authorization.k8s.io/app-crb
```


# Troubleshooting

## Vérification que le cluster à les droits pour inspecter le JWT 

pour rappel, ces droits sont accordés dans crb.yaml

```bash
kubectl get clusterrole system:auth-delegator -o yaml

jwt=$(kubectl get secrets app-secret -o json | jq -r '.data.token' | base64 -d)

tee tokenreview.yaml <<EOF
kind: TokenReview
apiVersion: authentication.k8s.io/v1
metadata:
  name: test
spec:
  token: $jwt
EOF

# should return a yaml with authenticated: true
kubectl apply -o yaml -f tokenreview.yaml
```

## Vérification que le JWT permet bien de s'authentifier à Vault

```bash
# JWT is in token field 
kubectl describe secret app-secret

# => should respond with a json with the client_token
curl --request POST --data '{"jwt": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjZIb2M2RnVUSy12TlhqYzJ3VnE4MkVfeDFQaFNQdTlpUkR5NjdyZTlTUGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJwb2Mtc2lkZWNhci10bHMtcGtpIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFwcC1zZWNyZXQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiYXBwLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZGVjZTE0MzAtYzZkOC00NDUzLWE2YzQtNjFiYzY5MzA5ZTRlIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OnBvYy1zaWRlY2FyLXRscy1wa2k6YXBwLXNhIn0.Gil1a32Yyleibo4PyBQ2rTDnjNXs7rA10r7EhyQbirIt2wGZNz6Xbypl3akYWL-9cKaqNUrSxw9rnapRue6TXjN0lzxDSNT3ZcdwzxImNl_wdc7SuCGmYWYrgKMvgLQQlbCKMRUwDl18sImCbsAv06Bu3Kwv3jpgerTeiU3KaFZXhuTDmwUWAuPN6SooQNJW0BRkjlLTTYpHsQT28fyBbGWBj5jLqDADqOsP926_iv3HaIJ22LYABy5eaIIt4K9QizCE7Io8QF2orf_RbZuQea2xHgmVWk9MMrjhMplClq6b145asG6tpBUR7HwetrnyJTZaOKWmn2B2hbrsI9iVDw", "role": "authkube"}' https://vault.home/v1/auth/kubernetes/login
```

## Vérification que tls.key et tls.crt correspondent

```bash
cd troubleshooting
kubectl cp poc-sidecar-tls-pki-deploy-bc75b6b6-5966n:/etc/nginx/ssl/tls.key tls.key -c vault
kubectl cp poc-sidecar-tls-pki-deploy-bc75b6b6-5966n:/etc/nginx/ssl/tls.crt tls.crt -c vault

openssl pkey -pubout -in tls.key | openssl sha256  
SHA2-256(stdin)= ab1f1ed68ebd181ac3212c875fa4a107c1fc5ef81ec9301b259cfbb584df73f1

openssl x509 -pubkey -in tls.crt -noout | openssl sha256
SHA2-256(stdin)= ab1f1ed68ebd181ac3212c875fa4a107c1fc5ef81ec9301b259cfbb584df73f1
```

## Vérification des dates d'expiration des certificats en se connectant au container

```bash
kubectl exec -it deployment.apps/poc-sidecar-tls-pki-deploy -c nginx -- bash

# manual check
echo | openssl s_client -showcerts -servername foo.home -connect localhost:443 | openssl x509 -noout -enddate
notAfter=Jun 24 18:37:37 2024 GMT

# used by liveness (exit code 1 if expired)
echo | openssl s_client -showcerts -servername foo.home -connect localhost:443 | openssl x509 -noout -enddate -checkend 0
```


# Références
- <https://www.hashicorp.com/blog/kubernetes-vault-integration-via-sidecar-agent-injector-vs-csi-provider>
- <https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template>
- <https://developer.hashicorp.com/vault/tutorials/vault-agent/agent-env-vars>
- <https://support.hashicorp.com/hc/en-us/articles/4404389946387-Kubernetes-auth-method-Permission-Denied-error>
- <https://wlwan.medium.com/why-a-cluster-role-binding-is-needed-in-k8s-vault-integration-82b5aefc4d81>
- <https://www.hashicorp.com/blog/certificate-management-with-vault>