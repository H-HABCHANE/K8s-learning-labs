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

## Jour 1 (19/10) — Exercice 5 : passer de 3 à 5 replicas (VÉRIFIÉ)
sed -i 's/replicas: 3/replicas: 5/' 04-deployments/deployment-nginx.yaml   # changer l'état souhaité dans le fichier
grep replicas 04-deployments/deployment-nginx.yaml                           # vérifier
kubectl apply -f 04-deployments/deployment-nginx.yaml                        # → deployment.apps/monsite configured
kubectl get deployments,replicasets,pods                                     # → 5/5, ReplicaSet monsite-86f69f946d DESIRED 5
Résultats :
1. 5 Pods : 3 anciens + 2 nouveaux (85k7z, gdk8x, AGE 11s) ✅
2. Les 3 anciens gardent leur nom (4vcxz, kfrzp, kjxxs) ✅
3. apply affiche configured ✅
4. Toujours le même ReplicaSet (86f69f946d) : replicas n'est pas dans le template, le hash ne change pas ✅
   Le Deployment a simplement changé DESIRED de son ReplicaSet de 3 à 5.

### Déclaratif vs impératif
kubectl scale deployment monsite --replicas=2   # IMPÉRATIF : ordre direct, le fichier n'est pas modifié
1. Fichier à 5, cluster à 2 ; un collègue relance apply → retour à 5 Pods ✅
   Problème : le cluster ne correspond plus au fichier (on appelle ça une DÉRIVE, "drift") ; la modification est perdue sans prévenir.
2. Meilleure pratique → déclaratif (modifier le fichier + apply) : le fichier (dans Git) fait foi ✅
   (kubectl scale reste utile en urgence, à condition de reporter ensuite le changement dans le fichier.)

## Exercice 6 : rolling update nginx → nginx:1.27
sed -i 's/image: nginx$/image: nginx:1.27/' 04-deployments/deployment-nginx.yaml   # changer la recette (template)
kubectl apply -f 04-deployments/deployment-nginx.yaml                              # → configured, la mise à jour démarre
kubectl rollout status deployment monsite                                          # suivre la mise à jour → successfully rolled out
kubectl get replicasets                                                            # 2 ReplicaSets
kubectl get pods -o custom-columns=NOM:.metadata.name,IMAGE:.spec.containers[0].image
Résultats :
1. 2 ReplicaSets : monsite-676bf75868 (nouveau, nginx:1.27) DESIRED 5 ; monsite-86f69f946d (ancien) DESIRED 0 ✅
2. Les Pods changent de nom : 5 nouveaux Pods monsite-676bf75868-xxxxx, tous en nginx:1.27 ✅
3. Pas de coupure ✅ prouvé par kubectl describe deployment monsite :
   Events : l'ancien ReplicaSet descend petit à petit (5→4→3→2→1→0) pendant que le nouveau monte (0→2→3→4→5).
   RollingUpdateStrategy: 25% max unavailable, 25% max surge (sur 5 Pods) :
   - max surge 25 % → arrondi au-dessus = 2 : droit de créer 2 Pods EN PLUS du nombre voulu (jusqu'à 7 Pods au total) ✅
   - max unavailable 25 % → arrondi au-dessous = 1 : au moins 4 Pods disponibles en permanence ✅
   annotation deployment.kubernetes.io/revision: 2 → 2e version du Deployment.

## Exercice 7 : rollback
Prédictions (avant d'exécuter) :
1. Pourquoi garder l'ancien ReplicaSet à 0 ? → pour pouvoir faire un rollback
2. Après undo, qui passe à 5 ? → l'ancien ReplicaSet 86f69f946d (pas un 3e)
3. Fichier toujours en nginx:1.27 après undo → dérive (drift) entre le fichier et le cluster
Exécution :
kubectl rollout history deployment monsite                                         # révisions 1 (nginx) et 2 (nginx:1.27)
kubectl rollout undo deployment monsite && kubectl rollout status deployment monsite   # retour arrière, suivi en direct
→ Warning de kubectl : le rollback ne met pas à jour la config "apply" → confirme la DÉRIVE (prédiction 3) ✅
kubectl get replicasets   # 86f69f946d (ancien) DESIRED 5 ; 676bf75868 DESIRED 0 → prédictions 1 et 2 ✅
kubectl get pods ...      # 5 Pods NEUFS (5v55w, 76ntt...) avec l'ancien hash 86f69f946d, image nginx
kubectl rollout history   # révisions 2 et 3 (la 1 a disparu)
Interprétation :
1. Révision 3 = nginx : un rollback est un NOUVEAU déploiement qui réutilise une ancienne recette ✅
2. La 1 est devenue la 3 ✅ : 1 révision = 1 ReplicaSet ; le ReplicaSet 86f69f946d a été réutilisé et renuméroté 3.
Corriger la dérive — prédictions :
1. apply du fichier (nginx:1.27) demain → retour à 1.27, les clients retrouvent le problème ✅
2. Corriger → sed pour remettre image: nginx dans le fichier ✅
3. apply après correction → "unchanged" (à vérifier)
Exécution : sed -i 's/image: nginx:1.27/image: nginx/' 04-deployments/deployment-nginx.yaml ; apply → "configured" (et non unchanged) ❌ prédiction 3
Pourquoi : apply compare le fichier à sa MÉMOIRE du dernier apply (annotation kubectl.kubernetes.io/last-applied-configuration),
pas seulement au cluster. Le rollback n'avait pas mis cette mémoire à jour (elle disait encore nginx:1.27) → apply a mis
à jour l'annotation → "configured". Les Pods, eux, sont déjà en nginx : pas de nouveau rolling update.
Leçon : après un rollback, toujours remettre le fichier (Git) d'accord avec le cluster.
kubectl rollout history deployment monsite   # toujours 2 et 3 : pas de révision 4 ✅ (template inchangé → pas de nouveau ReplicaSet)
