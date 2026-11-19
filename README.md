# deployment-cd : déploiement de SenAlert sur Kubernetes

SenAlert est une plateforme de signalement des rues, canaux et caniveaux bouchés (Examen DevOps, L3 Big Data, DIT).
Ce dépôt contient les manifestes Kubernetes et le pipeline de déploiement (CD). Il récupère les images publiées sur Docker Hub par les dépôts `backend-ci` et `frontend-ci`.

## Pourquoi GitHub Actions

Un seul outil CI/CD pour les trois dépôts, comme l'exige l'énoncé. Nous avons choisi GitHub Actions parce que l'organisation GitHub est contrôlée par toute l'équipe (le dépôt GitLab d'origine appartenait à un seul membre, ce qui empêchait de configurer les secrets du pipeline), que les workflows sont versionnés avec le code, et que le runner auto-hébergé permet de déployer sur le cluster local.

## Architecture

```
Navigateur -> frontend (nginx, NodePort 30080) -> /api -> backend (FastAPI) -> PostgreSQL (volume persistant)
```

| Fichier | Rôle |
|---|---|
| `k8s/namespace.yaml` | Namespace `senalert` |
| `k8s/configmap.yaml` | Variables non sensibles |
| `k8s/database-pvc.yaml` | Volume persistant de PostgreSQL (1 Gi) |
| `k8s/database-deployment.yaml` | PostgreSQL 17, stratégie `Recreate` |
| `k8s/database-service.yaml` | Service `db-service` (5432) |
| `k8s/backend-deployment.yaml` | API, rolling update sans coupure, sondes sur `/api/health` |
| `k8s/backend-service.yaml` | Service `backend-service` (8000) |
| `k8s/frontend-deployment.yaml` | Frontend, 2 réplicas, rolling update sans coupure |
| `k8s/frontend-service.yaml` | Service `frontend-service` (NodePort 30080) |
| `.github/workflows/cd.yml` | Pipeline de déploiement |

Le Secret `senalert-secret` n'est pas dans le dépôt : le pipeline le crée à partir des secrets GitHub.

## Configuration du dépôt (Settings > Secrets and variables > Actions)

| Type | Nom | Valeur |
|---|---|---|
| Variable | `DOCKERHUB_USER` | Compte Docker Hub de l'équipe |
| Secret | `POSTGRES_PASSWORD` | Mot de passe de la base (lettres et chiffres uniquement) |
| Secret | `ADMIN_PASSWORD` | Mot de passe de l'administrateur SenAlert |

Les images attendues sur Docker Hub sont `DOCKERHUB_USER/senalert-backend` et `DOCKERHUB_USER/senalert-frontend`.

## Runner auto-hébergé

Le job de déploiement tourne sur la machine qui héberge le cluster (Docker Desktop avec Kubernetes, ou minikube).

1. Organisation GitHub > Settings > Actions > Runners > New runner, puis suivre les commandes affichées.
2. Lancer le runner avec la session de l'utilisateur qui possède le fichier `~/.kube/config` (sinon `kubectl` ne trouve pas le cluster).
3. Vérifier que `kubectl get nodes` fonctionne depuis cette session.

## Déclenchement

| Déclencheur | Effet |
|---|---|
| Push sur `main` de ce dépôt | Applique les manifestes et redéploie le backend et le frontend |
| `repository_dispatch` de `backend-ci` | Redéploie le backend |
| `repository_dispatch` de `frontend-ci` | Redéploie le frontend |
| Lancement manuel | Comme un push |

À ajouter à la fin du workflow de `backend-ci` (et `frontend-ci` avec `frontend-updated`), après le push de l'image :

```yaml
- name: Déclencher le déploiement
  if: github.ref == 'refs/heads/main'
  run: |
    curl -fsS -X POST \
      -H "Accept: application/vnd.github+json" \
      -H "Authorization: Bearer ${{ secrets.CD_TRIGGER_TOKEN }}" \
      https://api.github.com/repos/ORGANISATION/deployment-cd/dispatches \
      -d '{"event_type":"backend-updated"}'
```

`CD_TRIGGER_TOKEN` est un jeton d'accès personnel (portée `repo`) enregistré comme secret dans `backend-ci` et `frontend-ci`.

## Premier déploiement à la main (avant le pipeline)

```bash
kubectl apply -f k8s/namespace.yaml
kubectl create secret generic senalert-secret -n senalert --from-literal=POSTGRES_USER=admin --from-literal=POSTGRES_PASSWORD=MotDePasse123 --from-literal=DATABASE_URL="postgresql://admin:MotDePasse123@db-service:5432/senalert" --from-literal=ADMIN_PASSWORD=AutreMotDePasse
# remplacer DOCKERHUB_USER dans les fichiers de k8s/ par le compte Docker Hub, puis :
kubectl apply -f k8s/
kubectl get pods -n senalert -w
```

Application : http://localhost:30080. Si le NodePort ne répond pas : `kubectl port-forward -n senalert svc/frontend-service 8080:80`, puis http://localhost:8080.

## Vérifications

```bash
kubectl get all -n senalert
kubectl rollout status deployment/backend -n senalert
kubectl logs -n senalert deploy/backend
kubectl get pvc -n senalert
```

### Rolling update sans coupure

Dans une première fenêtre (PowerShell), appeler l'application en boucle :

```powershell
while ($true) { (Invoke-WebRequest -UseBasicParsing http://localhost:30080/api/health).StatusCode; Start-Sleep -Milliseconds 500 }
```

Dans une seconde fenêtre, suivre les pods (`kubectl get pods -n senalert -w`), puis lancer un déploiement (push sur `main` ou `kubectl rollout restart deployment/frontend -n senalert`). Tous les appels doivent renvoyer 200 pendant que les pods sont remplacés un par un.

### Persistance des données

1. Créer un signalement depuis l'application.
2. Supprimer le pod de la base : `kubectl delete pod -n senalert -l app=postgres`.
3. Relancer un déploiement, puis vérifier que le signalement existe toujours.

## Difficultés rencontrées

À compléter au fil du projet.
