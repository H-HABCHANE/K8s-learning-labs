# Étape 2 : le Pod

## C'est quoi ?
- Un Pod est la plus petite chose que Kubernetes sait gérer.
- Un Pod est une enveloppe autour d'un conteneur (parfois plusieurs).
- Kubernetes ne gère jamais un conteneur « tout nu » : il gère des Pods.

## Analogie
- Conteneur = le plat.
- Pod = le plateau sur lequel on pose le plat.
  Le serveur (Kubernetes) transporte des plateaux, pas des plats à la main.
  Parfois, un plateau porte un plat + une sauce (2 conteneurs qui vont toujours ensemble).

## Ce que le Pod ajoute au conteneur
- Une adresse IP (dans le cluster).
- Un nom.
- Des conteneurs qui partagent le même réseau (ils se parlent via localhost).

## Important : un Pod est jetable
- Si un Pod meurt, il n'est PAS réparé : on en crée un nouveau (nouveau nom, nouvelle IP).
- Un Pod seul n'est pas relancé automatiquement → c'est le rôle du Deployment (étape 4).

## Le fichier YAML
Voir `pod-nginx.yaml` (dans ce dossier) : on décrit le Pod qu'on veut (l'état souhaité).

## Mes réponses
1. Pod ou conteneur, que gère Kubernetes ?
2. Si un Pod meurt, que se passe-t-il ?
→ 1. Des Pods.  2. Il en crée un nouveau (nouveau nom, nouvelle IP).

## Pratique : lancer le Pod (cluster minikube démarré)
kubectl apply -f pod-nginx.yaml          # envoyer l'état souhaité au cluster
kubectl get pods                         # lister les Pods (STATUS doit être Running)
kubectl get pods -o wide                 # + l'IP et le nœud
kubectl describe pod mon-premier-pod     # fiche détaillée (Events en bas)
kubectl logs mon-premier-pod             # ce que le conteneur affiche
kubectl port-forward pod/mon-premier-pod 8080:80   # ouvrir http://localhost:8080 (Ctrl+C pour arrêter)
kubectl delete pod mon-premier-pod       # supprimer le Pod
kubectl get pods                         # il ne revient PAS : un Pod seul n'est pas relancé

## Résultats de ma pratique (étape 3 bis)
- `kubectl apply` → `pod/mon-premier-pod created` (créé ≠ déjà en marche).
- `kubectl get pods` à 10s → READY 0/1, STATUS ContainerCreating (téléchargement de l'image nginx).
- `kubectl get pods` à 21s → READY 1/1, STATUS Running ✅
- `kubectl describe pod mon-premier-pod` :
  1. IP du Pod ? → 10.244.0.3 ✅ (adresse interne au cluster)
  2. Events ? → 5 lignes, la dernière est Started ✅ (Scheduled → Pulling → Pulled → Created → Started)
  3. Qui choisit le nœud ? → default-scheduler ✅ (le kubelet, lui, télécharge l'image et démarre le conteneur)
- `kubectl logs mon-premier-pod` :
  1. Équivalent Docker ? → docker logs ✅
  2. Ligne qui montre que nginx est prêt ? → "Configuration complete; ready for start up" ✅
  (8 "start worker process" = 1 par cœur du PC ; heure des logs en UTC)
- `kubectl delete pod mon-premier-pod` puis `kubectl get pods` → "No resources found".
  1. Le Pod revient-il ? → Non ✅
  2. Pourquoi ? → Il faut un Deployment pour le relancer ✅ (un Pod seul n'a personne pour le surveiller)
