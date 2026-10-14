# Étape 4 : le Deployment

## Le problème
Un Pod seul n'est pas relancé s'il meurt (vu dans `02-pods` avec `kubectl delete pod`).

## C'est quoi ?
Un Deployment est un responsable de salle qui surveille les Pods en permanence.
On lui dit : « je veux toujours 3 Pods nginx ». S'il en manque un, il en recrée un.

## Analogie du restaurant
- Pod = un plateau.
- Deployment = le responsable de salle à qui le gérant a dit « 3 plateaux de pâtes en salle, toujours ».
  Un plateau tombe → il en fait préparer un autre, tout de suite.
- template = la fiche recette affichée en cuisine : chaque nouveau plateau est fait pareil.

## Le fichier deployment-nginx.yaml
- `replicas: 3` → le nombre de Pods voulus (l'état souhaité).
- `template` → le modèle de Pod (c'est presque le contenu de pod-nginx.yaml).
- `selector` + `labels: app: monsite` → l'étiquette qui permet au Deployment de reconnaître « ses » Pods.

## Mes réponses
1. replicas: 3 ? → Le nombre de Pods identiques que Kubernetes maintient en permanence ; si un Pod plante ou est supprimé, le Deployment en recrée un. ✅
2. template ? → La recette pour fabriquer chaque Pod : conteneurs, images, ports, étiquettes sont copiés dans chaque nouveau Pod. ✅
3. Comment il reconnaît ses Pods ? → Grâce au selector (matchLabels: app: monsite), qui doit correspondre à l'étiquette du template. ✅
   (Si selector et labels du template ne correspondent pas, Kubernetes refuse le fichier.)

## Pratique
kubectl apply -f deployment-nginx.yaml   # créer le Deployment
kubectl get deployments                  # voir le Deployment (READY 3/3)
kubectl get pods                         # voir les 3 Pods fabriqués
kubectl delete pod <nom-d-un-pod>        # simuler une panne
kubectl get pods                         # un nouveau Pod apparaît tout seul

## Résultats de ma pratique
- `kubectl apply -f deployment-nginx.yaml` → deployment.apps/monsite created
- `kubectl get deployments` → READY 3/3 en 8s (image nginx déjà sur le nœud)
- Combien de Pods ? → 3 ✅
- Noms des Pods ? → monsite-<série 1>-<série 2> ✅, avec une précision :
  la série du milieu est LA MÊME pour les 3 Pods (elle identifie la version de la recette / template),
  seule la dernière série change d'un Pod à l'autre.
- Prédiction avant de supprimer un Pod (pas encore vérifiée en pratique) :
  1. Toujours 3 Pods : le ReplicaSet (géré par le Deployment) recrée un remplaçant ✅
  2. f56w5 disparaît, un nouveau Pod apparaît avec un autre suffixe ✅
  3. Le nouveau Pod a un AGE très faible (quelques secondes) ✅
  ReplicaSet = l'objet intermédiaire Deployment → ReplicaSet → Pods. La série du milieu du nom (86f69f946d) = le ReplicaSet.
- VÉRIFIÉ en pratique : `kubectl delete pod monsite-86f69f946d-f56w5` puis `kubectl get pods` →
  toujours 3 Pods ; f56w5 a disparu ; nouveau Pod monsite-86f69f946d-4vcxz (AGE 9s, les autres 4m47s).
  Même série du milieu (86f69f946d) = même ReplicaSet, même recette. Prédictions confirmées ✅
  Différence avec l'étape 1 (docker kill) et 3 bis (Pod seul) : cette fois, le Pod est remplacé automatiquement.
- Prédiction passage à replicas: 5 (à vérifier) :
  1. 5 Pods : les 3 existants + 2 nouveaux ✅
  2. Les 3 anciens gardent leur nom : on change le nombre, pas la recette (template) ✅
  3. apply affiche "configured" (le Deployment existe déjà, on met à jour) ✅
