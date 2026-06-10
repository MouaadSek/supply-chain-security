# Supply Chain Security — Kubernetes

Sécurisation de la chaîne d'approvisionnement logicielle d'un cluster Kubernetes avec scan d'images et contrôle d'admission.

## Architecture

```
git push → GitLab CI/CD
             ├── Build image Docker
             ├── Trivy (scan CVE CRITICAL/HIGH)
             ├── Safety (scan dépendances Python)
             ├── Push registre (si scans OK)
             └── Deploy → Kubernetes
                           └── OPA Gatekeeper (admission controller)
                                ├── Registre approuvé uniquement
                                ├── Tag latest interdit
                                └── Exécution root interdite
```

## Stack

- **App** : Flask (Python)
- **CI/CD** : GitLab CI/CD
- **Scan image** : Trivy
- **Scan deps** : Safety
- **Orchestration** : Kubernetes (Kind en local)
- **Admission control** : OPA Gatekeeper (Rego)
- **Conteneurisation** : Docker (multi-stage, non-root)

## Sécurité appliquée

- Image Docker multi-stage slim, utilisateur non-root
- Trivy bloque le pipeline si CVE CRITICAL ou HIGH
- Safety vérifie les dépendances Python (complémentaire à Trivy)
- Image jamais pushée si un scan échoue
- OPA Gatekeeper refuse les pods non conformes au niveau cluster
- Manifest K8s : readOnlyRootFilesystem, drop ALL capabilities, resource limits

## Structure

```
.
├── app/                        # Application Flask + Dockerfile
├── gatekeeper/
│   ├── templates/              # ConstraintTemplates (logique Rego)
│   └── constraints/            # Constraints (paramètres concrets)
├── k8s/                        # Manifests Kubernetes
├── .gitlab-ci.yml              # Pipeline CI/CD
└── .trivyignore                # CVE acceptées
```

## Lancer en local

```bash
# Créer le cluster
kind create cluster --name supply-chain-lab

# Installer OPA Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.16.0/deploy/gatekeeper.yaml

# Attendre que Gatekeeper soit prêt
kubectl -n gatekeeper-system wait --for=condition=ready pod -l control-plane=controller-manager --timeout=120s

# Déployer les policies
kubectl apply -f gatekeeper/templates/
kubectl apply -f gatekeeper/constraints/

# Build et charger l'image
docker build -t registry.gitlab.com/mouaadsek/supply-chain-security:v0.1.0 -f app/Dockerfile app/
kind load docker-image registry.gitlab.com/mouaadsek/supply-chain-security:v0.1.0 --name supply-chain-lab

# Déployer l'app
kubectl apply -f k8s/deployment.yaml
```

## Tester les policies

```bash
# Doit être REFUSÉ (tag latest)
kubectl run test --image=nginx:latest --dry-run=server

# Doit être REFUSÉ (registre non approuvé)
kubectl run test --image=docker.io/nginx:1.25 --dry-run=server

# Doit être ACCEPTÉ (image conforme)
cat <<EOF | kubectl apply --dry-run=server -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-ok
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
    - name: app
      image: registry.gitlab.com/mouaadsek/supply-chain-security:v0.1.0
      securityContext:
        allowPrivilegeEscalation: false
EOF
```

## Auteur

Mouaad Sekkouri — M1 Cybersécurité, Supinfo Lille
