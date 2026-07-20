# nestjs-k8s-gitops

Fuente de verdad del estado de los clústeres (GitOps con ArgoCD). Lo que está
en `main` de este repo ES lo que corre en cada entorno: ArgoCD reconcilia
continuamente (`automated + prune + selfHeal`) y cualquier cambio manual en el
clúster se revierte solo.

Repos relacionados:
- [nestjs-k8s-microservice-stack](https://github.com/JosueDev-afk/nestjs-k8s-microservice-stack) — código + chart Helm (el *cómo* se despliega)
- [nestjs-k8s-infrastructure](https://github.com/JosueDev-afk/nestjs-k8s-infrastructure) — Terraform: cloud, EKS, datos, bootstrap de ArgoCD/ESO (el *dónde*)
- este repo — Applications + values por entorno (el *qué* corre)

## Estructura

```
bootstrap/    root Application por entorno (App-of-Apps). Los aplica Terraform.
apps/aws-*/   una Application por componente, ordenadas por sync-wave:
              0 kube-prometheus-stack (CRDs) -> 1 loki,tempo -> 2 agentes+exporters -> 3 microservices
values/aws-*/ values de cada chart por entorno. values/aws-*/microservices.yaml
              contiene los tags de imagen: los escribe el CI, no se editan a mano.
```

## Flujo de despliegue

```
push a main (repo app)
  └─ CI: test -> build 4 imágenes -> push ECR -> yq bump tags en values/aws-dev/microservices.yaml
       └─ ArgoCD (dev) detecta el commit y sincroniza  ✅ deploy a dev

promoción a prod: workflow "promote" (repo app) abre PR aquí copiando los tags
dev -> prod. EL MERGE DE ESE PR ES EL DEPLOY A PROD. Rollback = git revert.
```

## Bootstrap de un entorno (una sola vez)

1. Aplicar el repo de infraestructura en orden: `network -> eks -> data -> platform`.
   La capa platform instala ArgoCD y aplica `bootstrap/root-app-aws-<env>.yaml`.
2. Rellenar los `REPLACE_ME` de `values/aws-<env>/` con los outputs de Terraform:

   | Placeholder | Origen |
   |---|---|
   | `REPLACE_ME_ECR_REGISTRY` | `terraform -chdir=live/aws/shared/ecr-ci output -raw registry_url` |
   | `REPLACE_ME_RDS_ENDPOINT_<ENV>` | `terraform -chdir=live/aws/<env>/data output -raw postgres_endpoint` |
   | `REPLACE_ME_REDIS_ENDPOINT_<ENV>` | `terraform -chdir=live/aws/<env>/data output -raw redis_endpoint` |
   | `REPLACE_ME_LOKI_BUCKET_PROD` / `REPLACE_ME_LOKI_IRSA_ROLE_ARN` | capa platform de prod (bucket + rol IRSA de Loki) |

3. Commit + push: ArgoCD converge solo. `argocd app list` debe mostrar todo `Synced/Healthy`.

Si este repo es privado, ArgoCD necesita credenciales: crear un fine-grained
PAT read-only sobre este repo y el de la app, guardarlo en Secrets Manager
(`nestjs/<env>/gitops/repo-credentials`, JSON `{"username":"git","password":"<PAT>"}`)
y activar `gitops_repo_private = true` en la capa platform.

## Reglas

- Nunca commitear secretos: todos llegan por External Secrets Operator (`existingSecret`).
- `values/aws-prod/**` solo cambia vía PR (branch protection).
- No editar tags de imagen a mano: los gestiona el CI (dev) y el PR de promoción (prod).
- Un cambio de chart de plataforma = bump de `targetRevision` en `apps/` (dev primero, prod por PR).
