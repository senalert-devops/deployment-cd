# deployment-cd : déploiement de SenAlert sur Kubernetes

SenAlert est une plateforme de signalement des rues, canaux et caniveaux bouchés (Examen DevOps, L3 Big Data, DIT).
Ce dépôt contient les manifestes Kubernetes et le pipeline de déploiement (CD). Il récupère les images publiées sur Docker Hub par les dépôts `backend-ci` et `frontend-ci`.

## Architecture

```
Navigateur -> frontend (nginx, NodePort 30080) -> /api -> backend (FastAPI) -> PostgreSQL (volume persistant)
```

```
backend-ci   : tests -> image Docker -> Docker Hub -> déclenche deployment-cd (backend-updated)
frontend-ci  : tests -> image Docker -> Docker Hub -> déclenche deployment-cd (frontend-updated)
deployment-cd: runner auto-hébergé -> kubectl apply -> rolling update -> vérification
```

| Fichier | Rôle |
|---|---|
| `k8s/namespace.yaml` | Namespace `senalert` |
| `k8s/configmap.yaml` | Variables non sensibles (`POSTGRES_DB`, `ADMIN_USER`, `CORS_ORIGINS`) |
| `k8s/database-pvc.yaml` | Volume persistant de PostgreSQL (1 Gi) |
| `k8s/database-deployment.yaml` | PostgreSQL 17, stratégie `Recreate` |
| `k8s/database-service.yaml` | Service `db-service` (5432) |
| `k8s/backend-deployment.yaml` | API, rolling update sans coupure, sondes sur `/api/health` |
| `k8s/backend-service.yaml` | Service `backend-service` (8000) |
| `k8s/frontend-deployment.yaml` | Frontend, 2 réplicas, rolling update sans coupure |
| `k8s/frontend-service.yaml` | Service `frontend-service` (NodePort 30080) |
| `.github/workflows/cd.yml` | Pipeline de déploiement |

Le Secret `senalert-secret` n'est pas dans le dépôt : le pipeline le crée à partir des secrets GitHub. Aucun mot de passe n'est écrit dans le code, dans les images ni dans ce dépôt.

## Configuration (Settings > Secrets and variables > Actions)

### Dans ce dépôt

| Type | Nom | Valeur |
|---|---|---|
| Secret | `POSTGRES_PASSWORD` | Mot de passe de la base (lettres et chiffres uniquement) |
| Secret | `ADMIN_PASSWORD` | Mot de passe de l'administrateur SenAlert |

### Au niveau de l'organisation (partagés avec les trois dépôts)

| Type | Nom | Valeur | Utilisé par |
|---|---|---|---|
| Variable | `DOCKERHUB_USER` | Compte Docker Hub de l'équipe | les trois dépôts |
| Secret | `DOCKERHUB_TOKEN` | Jeton d'accès Docker Hub (Read & Write) | `backend-ci`, `frontend-ci` |
| Secret | `CD_TRIGGER_TOKEN` | Jeton GitHub (classic, portée `repo`) pour lancer ce dépôt | `backend-ci`, `frontend-ci` |

`DOCKERHUB_USER` doit être une **variable** (et non un secret) : ce pipeline la lit avec `vars.DOCKERHUB_USER`. Un secret du même nom défini dans un dépôt remplace celui de l'organisation.

Les images attendues sur Docker Hub sont `DOCKERHUB_USER/senalert-backend` et `DOCKERHUB_USER/senalert-frontend`.

### Ce que reçoit le backend

| Variable | Origine |
|---|---|
| `DATABASE_URL` | Secret `senalert-secret`, construit avec `POSTGRES_PASSWORD` |
| `ADMIN_PASSWORD` | Secret `senalert-secret`, valeur du secret GitHub `ADMIN_PASSWORD` |
| `ADMIN_USER` | ConfigMap `senalert-config` (`admin`) |
| `CORS_ORIGINS` | ConfigMap `senalert-config` |

Le backend n'a aucune valeur par défaut pour ces variables : s'il en manque une, il refuse de démarrer. La connexion à l'espace administrateur se fait donc avec l'identifiant `admin` et le mot de passe du secret `ADMIN_PASSWORD`.

## Runner auto-hébergé

Le job de déploiement tourne sur la machine qui héberge le cluster (Docker Desktop avec Kubernetes, ou minikube). Le runner est sous Windows : les étapes du workflow utilisent donc PowerShell (`shell: pwsh`).

1. Organisation GitHub > Settings > Actions > Runners > New runner, puis suivre les commandes affichées.
2. Lancer le runner avec `run.cmd` dans la session de l'utilisateur qui possède le fichier `~/.kube/config` (sinon `kubectl` ne trouve pas le cluster), et non comme service Windows.
3. Vérifier que `kubectl get nodes` fonctionne depuis cette session.
4. Le runner doit apparaître « Idle ». L'ordinateur doit rester allumé pendant les déploiements : sans runner, le job reste en attente (« Waiting for a runner »).

## Déclenchement

| Déclencheur | Effet |
|---|---|
| Push sur `main` de ce dépôt | Applique les manifestes et redéploie le backend et le frontend |
| `repository_dispatch` `backend-updated` | Redéploie le backend |
| `repository_dispatch` `frontend-updated` | Redéploie le frontend |
| Lancement manuel (Actions > Run workflow) | Comme un push |

Les évènements `backend-updated` et `frontend-updated` sont envoyés par le dernier job du workflow `.github/workflows/ci.yml` de `backend-ci` et de `frontend-ci`, une fois l'image publiée sur Docker Hub depuis `main`. Il utilise le secret `CD_TRIGGER_TOKEN` et le propriétaire du dépôt (`github.repository_owner`) pour construire l'adresse de l'API :

```yaml
- name: Déclencher le déploiement
  env:
    CD_TRIGGER_TOKEN: ${{ secrets.CD_TRIGGER_TOKEN }}
  run: |
    curl -fsS -X POST \
      -H "Accept: application/vnd.github+json" \
      -H "Authorization: Bearer $CD_TRIGGER_TOKEN" \
      "https://api.github.com/repos/${{ github.repository_owner }}/deployment-cd/dispatches" \
      -d '{"event_type":"backend-updated"}'
```

GitHub répond `204` quand le déclenchement a réussi. Le jeton est valable 30 jours : il faut le renouveler (et mettre à jour le secret de l'organisation) s'il expire.

## Étapes du pipeline (`cd.yml`)

1. Récupération des manifestes.
2. Vérification de l'accès au cluster (`kubectl cluster-info`, `kubectl get nodes`).
3. Remplacement de `DOCKERHUB_USER` par le compte Docker Hub dans `k8s/*.yaml`.
4. Création du namespace.
5. Création ou mise à jour du Secret `senalert-secret` à partir des secrets GitHub.
6. Application de tous les manifestes (`kubectl apply -f k8s/`).
7. Redémarrage progressif des déploiements concernés (rolling update).
8. Attente de la fin du déploiement (`kubectl rollout status`).
9. Vérification : le backend répond sur `/api/health`, directement puis à travers nginx.
10. Résumé du déploiement dans l'onglet du run.

Un seul déploiement à la fois (`concurrency`).

## Premier déploiement à la main (avant le pipeline)

À n'utiliser que pour un test : en fonctionnement normal, le pipeline crée le Secret avec les vraies valeurs. Remplacer les valeurs entre chevrons par les vôtres (lettres et chiffres uniquement pour le mot de passe de la base) :

```bash
kubectl apply -f k8s/namespace.yaml
kubectl create secret generic senalert-secret -n senalert \
  --from-literal=POSTGRES_USER=admin \
  --from-literal=POSTGRES_PASSWORD=<MOT_DE_PASSE_BASE> \
  --from-literal=DATABASE_URL="postgresql://admin:<MOT_DE_PASSE_BASE>@db-service:5432/senalert" \
  --from-literal=ADMIN_PASSWORD=<MOT_DE_PASSE_ADMIN>
# remplacer DOCKERHUB_USER dans les fichiers de k8s/ par le compte Docker Hub, puis :
kubectl apply -f k8s/
kubectl get pods -n senalert -w
```

## Accéder à l'application

Avec un cluster qui expose les NodePorts, l'application est sur http://localhost:30080.

Avec Docker Desktop en mode « kind » (nœud `desktop-control-plane`, notre cas), le NodePort n'est pas relayé vers `localhost` de Windows. On utilise alors un tunnel :

```powershell
kubectl port-forward -n senalert svc/frontend-service 8080:80
```

puis http://localhost:8080. Le tunnel est attaché à un pod et à la fenêtre qui l'a lancé : après un redéploiement du frontend, il faut l'arrêter (Ctrl+C) et le relancer. Pour le lancer sans bloquer une fenêtre :

```powershell
Start-Process kubectl -ArgumentList 'port-forward -n senalert svc/frontend-service 8080:80' -WindowStyle Minimized
```

## Vérifications

```bash
kubectl get all -n senalert
kubectl rollout status deployment/backend -n senalert
kubectl logs -n senalert deploy/backend
kubectl get pvc -n senalert
```

### Rolling update sans coupure

Dans une première fenêtre (PowerShell), appeler l'application en boucle (adresse `localhost:8080` avec le port-forward, `localhost:30080` sinon) :

```powershell
while ($true) { (Invoke-WebRequest -UseBasicParsing http://localhost:8080/api/health).StatusCode; Start-Sleep -Milliseconds 500 }
```

Dans une seconde fenêtre, suivre les pods (`kubectl get pods -n senalert -w`), puis lancer un déploiement dans une troisième :

```powershell
kubectl rollout restart deployment/backend -n senalert
```

Tous les appels doivent renvoyer 200 pendant que les pods sont remplacés un par un (`maxUnavailable: 0`, sondes de disponibilité sur `/api/health`). Avec le port-forward, privilégier le redémarrage du backend : le tunnel est lié à un pod du frontend, et le remplacer le coupe.

### Persistance des données

1. Créer un signalement depuis l'application et noter son code.
2. Supprimer le pod de la base : `kubectl delete pod -n senalert -l app=postgres`.
3. Attendre que le pod revienne (`kubectl get pods -n senalert`), puis relancer un déploiement.
4. Chercher le signalement avec son code sur la page « Suivre » : il existe toujours. Le volume apparaît avec `kubectl get pvc -n senalert`.

## Difficultés rencontrées

| Problème | Cause | Solution |
|---|---|---|
| Le déploiement reste en attente (« Waiting for a runner ») | Le runner auto-hébergé n'était pas lancé | Lancer `run.cmd` dans la session de l'utilisateur et vérifier qu'il est « Idle » |
| Le workflow ne s'exécute pas sur le runner Windows | Étapes écrites pour `bash` | Étapes réécrites en PowerShell (`shell: pwsh`) avec contrôle du code de sortie de `kubectl` |
| Le backend de départ utilisait des identifiants de simulation (`admin` / `demo123`, base en `admin123`) au lieu du Secret Kubernetes | Valeurs par défaut écrites dans le code | Suppression des valeurs par défaut : le backend lit uniquement `DATABASE_URL`, `ADMIN_USER`, `ADMIN_PASSWORD` et `CORS_ORIGINS` fournis par le cluster |
| Le déclenchement depuis `backend-ci` et `frontend-ci` échouait (erreur 403) | Jeton `CD_TRIGGER_TOKEN` incorrect ou sans les droits suffisants | Nouveau jeton classic (portée `repo`) testé en ligne de commande (réponse 204), puis remplacement du secret de l'organisation |
| Le frontend envoyait des positions `x`/`y` que le backend refuse (erreur 422) | Contrat différent entre le front et l'API (latitude et longitude) | Conversion de position dans le frontend, puis remplacement du plan schématique par une carte OpenStreetMap |
| `http://localhost:30080` ne répond pas | Docker Desktop en mode « kind » : les NodePorts ne sont pas exposés à Windows | Accès par `kubectl port-forward` sur le port 8080 |
| Le port-forward continuait à afficher l'ancienne version après un déploiement | Le tunnel reste attaché à l'ancien pod | Arrêter puis relancer le port-forward après chaque redéploiement du frontend |
