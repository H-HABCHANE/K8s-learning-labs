# Jour 1 — Le ReplicaSet

## C'est quoi ?
Un ReplicaSet garantit qu'un nombre précis de Pods identiques tourne en permanence.
Il compte les Pods qui portent une étiquette (selector) : s'il en manque, il en crée ; s'il y en a trop, il en supprime.
Chaîne : Deployment → ReplicaSet → Pods.

## Exercice 1 : le ReplicaSet créé par le Deployment monsite
kubectl get deployments,replicasets,pods          # les 3 types d'objets en une commande
kubectl describe replicaset monsite-86f69f946d    # fiche du ReplicaSet (Controlled By, Replicas, Selector)

Sortie observée (19/10, après redémarrage du PC + minikube start) :
- replicaset.apps/monsite-86f69f946d : DESIRED 3, CURRENT 3, READY 3
- Pods 4vcxz, kfrzp, kjxxs : mêmes noms qu'au 14/10, RESTARTS 1

### Mes réponses
1. Nom monsite-86f69f946d → monsite = Deployment ; 86f69f946d = hash du Pod template ✅
2. Preuve d'appartenance → `Controlled By: Deployment/monsite` ✅ — vérifié dans describe
3. RESTARTS 1 → c'est le CONTENEUR qui a redémarré, pas le Pod : l'objet Pod est conservé (même nom, même AGE),
   ses conteneurs ont été relancés au redémarrage de minikube. Un Pod supprimé puis remplacé aurait un autre nom. ✅
   Autres lignes vues dans describe :
   - Selector: app=monsite,pod-template-hash=86f69f946d → le Deployment ajoute pod-template-hash pour séparer les versions
   - Replicas: 3 current / 3 desired
   - deployment.kubernetes.io/revision: 1 → numéro de version (servira au rollback)

## Exercice 2 : écrire mon propre ReplicaSet (web-rs)
Fichier : `replicaset-web.yaml` (écrit seul avec nano, sans erreur, indentation correcte).
kubectl apply --dry-run=client -f 05-replicasets/replicaset-web.yaml   # vérifie sans créer → created (dry run)
kubectl apply -f 05-replicasets/replicaset-web.yaml                    # crée le ReplicaSet
kubectl get replicasets,pods                                           # web-rs : DESIRED 2, CURRENT 2, READY 2

Sortie observée : Pods web-rs-ktz59 et web-rs-pjdsm (Running), à côté des 3 Pods de monsite.

### Mes réponses
1. Pas de hash au milieu du nom (web-rs-ktz59) : c'est le Deployment qui ajoute pod-template-hash et l'utilise
   dans le nom du ReplicaSet qu'il crée. Ici, le ReplicaSet a été créé directement. ✅
2. Lignes changées : kind Deployment → ReplicaSet ; nom → web-rs ; replicas → 2 ;
   app: monsite → app: web (dans selector.matchLabels ET template.metadata.labels) ; apiVersion apps/v1 identique. ✅
3. Pas app: monsite → éviter que deux contrôleurs ciblent le même groupe de Pods (sélecteurs qui se chevauchent). ✅
   Précision : ici le ReplicaSet de monsite sélectionne aussi pod-template-hash, ce qui limite le conflit,
   mais un ReplicaSet "nu" avec app: monsite verrait quand même les Pods de monsite. Règle : 1 sélecteur = 1 contrôleur.

## Exercice 3A : le Pod intrus
Fichier : `pod-intrus.yaml` (copie de 02-pods/pod-nginx.yaml : name: intrus, label app: web).
cp 02-pods/pod-nginx.yaml 05-replicasets/pod-intrus.yaml   # copier un fichier pour partir d'une base
kubectl get pods -l app=web                                # -l = filtrer par étiquette (le filtre du ReplicaSet)

Prédiction : le ReplicaSet supprime un Pod pour revenir à 2, et ce sera l'intrus (le plus récent). ✅ CONFIRMÉ
Events de web-rs : `SuccessfulDelete  replicaset-controller  Deleted pod: intrus`
→ Le ReplicaSet ne regarde que les étiquettes : il a compté 3 Pods app=web, en a supprimé 1.
Précision : l'ordre de suppression privilégie d'abord les Pods pas encore prêts (pas Ready, Pending),
puis ceux prêts depuis le moins longtemps → en pratique, le plus récent.

### Mes réponses
1. Comment un ReplicaSet reconnaît ses Pods ? → Seulement par les étiquettes (selector), pas par qui les a créés. ✅
2. Intrus créé AVANT web-rs (replicas 2) → web-rs crée un seul nouveau Pod : il ADOPTE l'intrus. ✅ (prédiction, à vérifier)

## Exercice 3B : prouver l'adoption
kubectl delete replicaset web-rs                     # supprime le ReplicaSet ET ses Pods
kubectl apply -f 05-replicasets/pod-intrus.yaml      # intrus créé en premier (Pod orphelin)
kubectl apply -f 05-replicasets/replicaset-web.yaml  # web-rs recréé
kubectl get pods -l app=web                          # intrus (23s) + web-rs-jh8q9 (10s) → 1 seul Pod créé
kubectl describe pod intrus                          # Controlled By: ReplicaSet/web-rs → ADOPTION prouvée ✅
Prédiction "un seul nouveau Pod, il adopte l'intrus" : CONFIRMÉE.

## Exercice 4 : la limite du ReplicaSet (changer l'image)
sed -i 's/image: nginx$/image: nginx:1.27/' 05-replicasets/replicaset-web.yaml   # modifie la recette dans le fichier
grep image 05-replicasets/replicaset-web.yaml                                      # vérifie la modification
kubectl apply -f 05-replicasets/replicaset-web.yaml                                # → configured (ReplicaSet mis à jour)
kubectl get pods -l app=web -o custom-columns=NOM:.metadata.name,IMAGE:.spec.containers[0].image
# → intrus : nginx ; web-rs-jh8q9 : nginx   (les Pods existants N'ONT PAS changé)
Prédiction "les Pods existants gardent nginx" : CONFIRMÉE ✅
Pourquoi : le ReplicaSet ne vérifie que le NOMBRE de Pods qui ont la bonne étiquette (2/2 → rien à faire).
Le nouveau template ne sert qu'aux Pods créés ENSUITE.

### Mes réponses
1. Forcer le passage à nginx:1.27 avec un ReplicaSet seul → supprimer les Pods : il les recrée avec la nouvelle recette. ✅
2. Problème avec 50 Pods → coupure du site si on les supprime tous d'un coup. ✅
   Et un par un : long, manuel, risque d'erreur, pas d'historique ni de retour arrière → c'est le travail du Deployment.
Mise à jour MANUELLE, un Pod à la fois :
kubectl delete pod intrus         # → web-rs-jh8q9 (nginx) + web-rs-z6ls7 (nginx:1.27) : 2 versions en même temps
kubectl delete pod web-rs-jh8q9   # → web-rs-25j7t + web-rs-z6ls7, tous en nginx:1.27 ✅
Question : 2 versions en même temps, quel problème ? → Le visiteur voit des versions différentes selon le Pod. ✅
Précision : c'est aussi le cas pendant un rolling update de Deployment → les versions successives doivent rester compatibles.

## Conclusion : ReplicaSet seul vs Deployment
| | ReplicaSet seul | Deployment |
|---|---|---|
| Maintenir N Pods | oui | oui (via un ReplicaSet) |
| Changer l'image des Pods existants | non : il faut supprimer les Pods à la main | oui, automatiquement (rolling update) |
| Historique / retour arrière | non | oui (revisions, rollback) |
→ En pratique, on n'écrit jamais de ReplicaSet à la main : on écrit un Deployment.

## Question d'entretien n°1 : Pod / ReplicaSet / Deployment
Ma réponse : Le Pod exécute l'application, le ReplicaSet maintient le nombre de Pods souhaité,
et le Deployment gère leur déploiement et leurs mises à jour. ✅
Version enrichie (le LIEN entre les trois) : le Deployment crée et pilote des ReplicaSets (un par version du template),
chaque ReplicaSet maintient N Pods repérés par leurs étiquettes, et le Pod est la plus petite unité qui fait tourner
les conteneurs. Le Deployment ajoute le rolling update et le rollback.
