# K8s learning labs

Mes TP et exercices pour apprendre Kubernetes en partant de zéro, en parallèle avec Docker.
Chaque dossier correspond à une notion, avec mes notes (README) et les manifestes YAML utilisés.

## Objectif
Comprendre les bases de Kubernetes par la pratique sur un cluster local : à quoi il sert,
la différence avec Docker, puis Pod, Deployment, Service et les commandes `kubectl` essentielles.

## Outils (versions réellement installées)
| Outil | Version |
|---|---|
| Ubuntu | 26.04 |
| Docker | 29.9.0 |
| kubectl (client) | v1.37.1 |
| minikube | v1.39.0 (driver `docker`) |
| Kubernetes (serveur, via minikube) | v1.37.0 |

## Labs
| Dossier | Notion | Manifeste | Testé sur minikube |
|---|---|---|---|
| [`01-bases`](01-bases/) | À quoi sert Kubernetes, différence avec Docker | aucun (exercice Docker) | exercice Docker uniquement |
| [`02-pods`](02-pods/) | Le Pod : créer, inspecter, lire les logs, supprimer | `pod-nginx.yaml` | ✅ oui |
| [`03-installation`](03-installation/) | Installer kubectl et minikube sur Ubuntu | aucun | ✅ cluster démarré, nœud `Ready` |
| [`04-deployments`](04-deployments/) | Le Deployment : répliques et remplacement automatique des Pods | `deployment-nginx.yaml` | ✅ oui (3 répliques, un Pod supprimé est recréé) |

À venir : Services, ConfigMaps, Secrets, volumes…

Tous les manifestes ont été validés avec `kubectl apply --dry-run=client` et `--dry-run=server`.

## Relancer les labs
```bash
# Démarrer le cluster local (à refaire après chaque redémarrage du PC)
minikube start --driver=docker
kubectl get nodes                      # attendre STATUS = Ready

# Lab Pod
kubectl apply -f 02-pods/pod-nginx.yaml
kubectl get pods
kubectl describe pod mon-premier-pod
kubectl logs mon-premier-pod
kubectl delete pod mon-premier-pod     # il ne revient pas : personne ne le surveille

# Lab Deployment
kubectl apply -f 04-deployments/deployment-nginx.yaml
kubectl get deployments                # READY 3/3
kubectl get pods
kubectl delete pod <nom-d-un-pod>      # un remplaçant apparaît tout seul
kubectl get pods

# Nettoyer
kubectl delete -f 04-deployments/deployment-nginx.yaml
minikube stop
```

## Limites
- Cluster **local d'apprentissage** (minikube, un seul nœud qui est à la fois control-plane et nœud d'exécution).
  Ce n'est pas un cluster de production multi-machine : pas de haute disponibilité, pas de vraie répartition entre machines.
- Les manifestes utilisent l'image `nginx` sans version fixée ni limites de ressources : c'est volontaire pour rester simple,
  mais à éviter en production.
- Aucun Secret n'est versionné pour l'instant. Si un manifeste de type Secret est ajouté, il contiendra uniquement
  des valeurs d'exemple évidentes (`changeme`).
- Le fichier `PROGRESSION.md` suit mon avancement dans le cours.
