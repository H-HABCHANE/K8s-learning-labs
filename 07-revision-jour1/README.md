# Jour 1 — Révision complète, refaite de zéro

Objectif : refaire tout le Jour 1 avec une nouvelle appli, « vitrine », avant le défi final (`08-defi-jour1/`).

Plan :
0. Repartir d'un cluster propre
1. Pod seul
2. Deployment 3 replicas + suppression d'un Pod
3. ReplicaSet (hash, Controlled By, -l)
4. scale 3 → 5 (déclaratif)
5. Rolling update
6. Rollback + correction de la dérive
7. Service ClusterIP + test busybox
8. Service NodePort + curl
9. port-forward + panne d'un Pod
10. Mini-entretien oral

## Étape 0 : repartir d'un cluster propre
minikube start                # rallume le cluster (après extinction du PC : host/kubelet/apiserver Stopped)
kubectl get nodes             # minikube Ready, control-plane, AGE 11d (même cluster, pas recréé)
kubectl get all               # avant : 5 Pods monsite, 2 ReplicaSets (dont l'ancien à 0), 1 Deployment, 3 Services
kubectl delete -f 04-deployments/deployment-nginx.yaml -f 06-services/service-clusterip.yaml -f 06-services/service-nodeport.yaml
kubectl get all               # après : il ne reste que service/kubernetes

Dépannage : si kubectl affiche `dial tcp 192.168.49.2:8443: connect: no route to host`, kubectl n'arrive pas à
joindre l'API server → `minikube status` (Stopped) → `minikube start`.

À retenir :
- Supprimer le Deployment supprime en cascade ses ReplicaSets et ses Pods.
- `delete -f` supprime les objets du cluster, pas les fichiers YAML du disque.
- Le Service `kubernetes` (API server) n'est dans aucun de mes fichiers, donc il reste.

### Questions
1. Pourquoi les Pods ont « survécu » à l'extinction du PC ?
   → L'état souhaité est stocké dans etcd (la base de données du cluster), sur le disque du nœud, donc il survit
     à l'arrêt. Les objets Pod sont aussi dans etcd : au redémarrage, le kubelet a relancé leurs conteneurs
     (mêmes noms de Pods, RESTARTS 1). Kubernetes répare tout seul l'écart entre l'état réel et l'état souhaité
     (réconciliation). Si un Pod avait disparu, le ReplicaSet en aurait recréé un.
2. Pourquoi les ReplicaSets et les Pods ont disparu alors que j'ai supprimé seulement le Deployment ?
   → Le Deployment a créé les ReplicaSets, qui ont créé les Pods. Chaque objet garde l'identité de son
     propriétaire (ownerReferences, visible dans `kubectl describe` sous « Controlled By »). Quand on supprime
     le propriétaire, le ramasse-miettes (garbage collector) supprime ses enfants : cascade
     Deployment → ReplicaSets → Pods. Un Pod créé seul n'a pas de propriétaire.
3. Pourquoi le Service `kubernetes` est toujours là ?
   → Il a été créé par Kubernetes, pas par mes fichiers ; `delete -f` ne supprime que les objets décrits dans
     les fichiers donnés. C'est l'accès à l'API server (10.96.0.1:443) ; même supprimé, Kubernetes le recréerait.

✅ Étape 0 validée : cluster propre.

## Étape 1 : un Pod seul, « vitrine »
Fichier : `pod-vitrine.yaml` (kind: Pod, name: vitrine, label app: vitrine, image nginx:1.27, containerPort 80)

kubectl apply -f 07-revision-jour1/pod-vitrine.yaml   # pod/vitrine created
kubectl get pods -o wide --show-labels                # 1/1 Running, RESTARTS 0, IP 10.244.0.9, NODE minikube, app=vitrine
kubectl describe pod vitrine                          # pas de ligne « Controlled By » ; Events : Scheduled → Pulled → Created → Started
kubectl delete pod vitrine                            # pod "vitrine" deleted
kubectl get pods                                      # No resources found → le Pod n'est PAS revenu

À retenir :
- `Pulled ... already present on machine` : image déjà en cache sur le nœud, donc pas d'étape `Pulling`.
- IP du nœud (`Node: minikube/192.168.49.2`) ≠ IP du Pod (`10.244.0.9`).

### Questions
1. Pourquoi le Pod n'a pas été recréé ?
   → Il n'a pas de contrôleur (pas de ligne « Controlled By »). Sans ReplicaSet/Deployment, personne ne surveille
     le nombre de Pods, donc personne n'en recrée.
2. Qui a choisi le nœud, qui a démarré le conteneur ?
   → Colonne « From » des Events : `default-scheduler` choisit le nœud (Scheduled) ; le `kubelet` (l'agent du
     nœud) récupère l'image, crée et démarre le conteneur (Pulled, Created, Started).
3. Pourquoi l'IP 10.244.0.9 pose problème pour accéder à l'application ?
   → L'IP d'un Pod est temporaire : un Pod recréé est un nouveau Pod, avec une nouvelle IP. Solution : un Service
     (IP et nom DNS stables, qui retrouve les Pods grâce aux labels, ex. app=vitrine) → étapes 7 et 8.

✅ Étape 1 validée : un Pod seul n'est jamais recréé.

## Étape 2 : Deployment 3 replicas + suppression d'un Pod
Fichier : `deployment-vitrine.yaml`
(apiVersion apps/v1, kind Deployment, replicas 3, selector app=vitrine = labels du template, image nginx:1.27)

kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml   # deployment.apps/vitrine created
kubectl get deploy,rs,pods -o wide                           # Deployment 3/3 → ReplicaSet vitrine-768475f9f6 (3/3/3) → 3 Pods
kubectl delete pod vitrine-768475f9f6-cxmk4                  # on supprime 1 Pod (IP 10.244.0.10)
kubectl get pods -o wide                                     # toujours 3 Pods : nouveau vitrine-768475f9f6-7bcns, IP 10.244.0.13, AGE 9s

À retenir :
- Nom d'un Pod = <deployment>-<pod-template-hash>-<suffixe aléatoire>.
- Le hash (768475f9f6) est l'empreinte du template ; il est aussi dans le selector du ReplicaSet.
- Le remplaçant est un NOUVEAU Pod (nouveau nom, nouvelle IP, AGE remis à zéro), pas l'ancien relancé.
- Le hash du remplaçant est le même : c'est le même ReplicaSet (même template) qui l'a créé.

### Questions
1. Quel objet a remplacé le Pod, et comment le voir dans le nom ?
   → Le ReplicaSet : il a vu qu'il manquait un Pod et en a créé un nouveau. Dans vitrine-768475f9f6-7bcns,
     `768475f9f6` est le hash du ReplicaSet propriétaire (le Pod seul s'appelait juste `vitrine`).
2. Le nouveau Pod est-il l'ancien relancé ?
   → Non : nouveau nom (7bcns ≠ cxmk4), nouvelle IP (.13 ≠ .10), nouvel âge (9s ≠ 6m).
3. Que se passe-t-il si le selector dit `app: vitrine` mais les labels du template `app: boutique` ?
   → Kubernetes REFUSE le Deployment : le label sert au ReplicaSet à reconnaître SES Pods ; sinon il ne les
     verrait jamais et en créerait sans fin. Test (simulation, fichier non modifié) :
     sed '13s/vitrine/boutique/' 07-revision-jour1/deployment-vitrine.yaml | kubectl apply --dry-run=server -f -
     → The Deployment "vitrine" is invalid: spec.template.metadata.labels: Invalid value: {"app":"boutique"}:
       `selector` does not match template `labels`
   `--dry-run=server` : l'API server vérifie l'objet mais ne crée rien. `-f -` : lire le YAML depuis le tuyau.

✅ Étape 2 validée : le Deployment (via son ReplicaSet) remplace tout Pod supprimé par un nouveau.

## Étape 3 : le ReplicaSet de plus près
kubectl describe pod vitrine-768475f9f6-7bcns   # Controlled By: ReplicaSet/vitrine-768475f9f6 ; Labels : app=vitrine + pod-template-hash=768475f9f6
kubectl describe rs vitrine-768475f9f6          # Controlled By: Deployment/vitrine ; Replicas 3 current / 3 desired ; revision: 1
kubectl get pods -l app=vitrine --show-labels   # -l = filtre par label : les 3 Pods

À retenir :
- `pod-template-hash` est ajouté automatiquement par le Deployment (je ne l'ai pas écrit dans le YAML).
- Selector du ReplicaSet = app=vitrine ET pod-template-hash=768475f9f6 : il ne compte que les Pods de SA version.
- Events du ReplicaSet (From: replicaset-controller) : 3 Pods créés, puis 7bcns (le remplaçant de cxmk4).
- Annotation `deployment.kubernetes.io/revision: 1` : numéro de version, utile pour le rollback.

### Questions
1. Chaîne de propriétaires, et où la voir ?
   → Deployment/vitrine → ReplicaSet/vitrine-768475f9f6 → Pods.
     `kubectl describe pod ...` → `Controlled By: ReplicaSet/vitrine-768475f9f6` ;
     `kubectl describe rs ...` → `Controlled By: Deployment/vitrine`.
2. Qui ajoute `pod-template-hash`, et à quoi sert-il ?
   → Le Deployment. C'est l'empreinte de la VERSION du template : chaque ReplicaSet ne compte que les Pods de sa
     version. Pendant un rolling update, 2 ReplicaSets ont le même app=vitrine : le hash les empêche de se
     « voler » les Pods.
3. Pourquoi un Deployment plutôt qu'un ReplicaSet directement ?
   → Un ReplicaSet ne fait que maintenir N Pods : si on change son image, les Pods existants restent sur
     l'ancienne image. Le Deployment gère les VERSIONS : à chaque changement de template, il crée un nouveau
     ReplicaSet et bascule progressivement les Pods (rolling update, sans coupure), et garde l'ancien
     ReplicaSet pour revenir en arrière (rollback).

✅ Étape 3 validée : chaîne Deployment → ReplicaSet → Pods, rôle du pod-template-hash.

## Étape 4 : scale 3 → 5 (déclaratif)
Méthode déclarative : on modifie le fichier, puis apply (le fichier reste la source de vérité, versionnée dans Git).
Méthode impérative (à éviter ici) : `kubectl scale deploy vitrine --replicas=5` → le fichier dit encore 3 = dérive.

sed -i 's/replicas: 3/replicas: 5/' 07-revision-jour1/deployment-vitrine.yaml   # modifie le fichier (le cluster n'est pas encore au courant)
kubectl diff -f 07-revision-jour1/deployment-vitrine.yaml                       # prévisualise : -replicas: 3 +replicas: 5, generation 1 → 2
kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml                      # deployment.apps/vitrine configured
kubectl get rs,pods                                                              # même ReplicaSet 768475f9f6 à 5/5/5 ; 2 nouveaux Pods (AGE 29s)

À retenir :
- `kubectl diff` : compare LIVE (cluster) et MERGED (après apply), ne modifie rien. Code de sortie 1 = différences.
- `generation` : compteur de modifications de l'objet.
- `last-applied-configuration` : copie du dernier fichier appliqué, gardée dans l'objet.
- Changer replicas ne touche pas au template → pas de nouveau ReplicaSet, pas de nouveau hash ;
  les 3 anciens Pods restent, 2 Pods sont ajoutés.

### Questions
1. Pourquoi modifier le fichier puis apply, plutôt que `kubectl scale` ?
   → Le fichier reste la source de vérité. Avec `scale` sans modifier le fichier (qui dit replicas: 3), le
     prochain `apply` du fichier remettrait 3 Pods : c'est la dérive, le changement manuel est écrasé.
2. Pourquoi le scale n'a pas créé de nouveau ReplicaSet ?
   → Le pod-template-hash est l'empreinte du `template` uniquement (labels, conteneurs, image...).
     `replicas` est en dehors du template : le template ne change pas, donc même hash, même ReplicaSet,
     qui ajoute juste 2 Pods. Changer l'image modifie le template → nouveau hash → nouveau ReplicaSet.
3. À quoi sert `kubectl diff` ?
   → Il montre la différence entre ce qui tourne dans le cluster (LIVE) et le fichier (MERGED), sans rien
     modifier. On le lance avant `apply` pour vérifier ce qui va changer et éviter les surprises.

Vérification (nouveau ReplicaSet ?) : replicas 5 → 2 : non ; image 1.27 → 1.28 : oui ; label ajouté dans le template : oui.

✅ Étape 4 validée : scale déclaratif 3 → 5, même ReplicaSet.

## Étape 5 : rolling update (nginx:1.27 → nginx:1.28)
sed -i 's/nginx:1.27/nginx:1.28/' 07-revision-jour1/deployment-vitrine.yaml        # change l'image DANS le template
kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml && kubectl rollout status deployment/vitrine
    # suit la bascule en direct : 2 → 3 → 4 out of 5 new replicas, old replicas pending termination, successfully rolled out
kubectl get rs -o wide && kubectl get pods
    # vitrine-645d8b6885 (nginx:1.28) 5/5/5 ; vitrine-768475f9f6 (nginx:1.27) 0/0/0 ; 5 Pods avec le hash 645d8b6885

À retenir :
- Rolling update : nouveau ReplicaSet créé, Pods remplacés petit à petit, l'appli ne s'arrête jamais.
- Réglages par défaut : maxSurge 25 % (arrondi au-dessus : 2 Pods en plus pour 5 replicas),
  maxUnavailable 25 % (arrondi en dessous : au plus 1 Pod indisponible).
- L'ancien ReplicaSet reste, à 0 replica (revisionHistoryLimit: 10).

### Questions
1. Pourquoi Kubernetes garde l'ancien ReplicaSet vide ?
   → Pour le rollback : il suffit de remonter l'ancien ReplicaSet (template 1.27) de 0 à 5 et de descendre le
     nouveau à 0, sans rien recréer. Il en garde jusqu'à 10 (revisionHistoryLimit: 10).
2. L'application a-t-elle été coupée pendant le rolling update ?
   → Non. maxUnavailable = 1 : le Deployment s'interdit d'arrêter plus d'un Pod de trop à la fois, il reste
     donc toujours au moins 5 − 1 = 4 Pods prêts. maxSurge = 2 : au plus 5 + 2 = 7 Pods en même temps.
3. Rolling update vers une image qui n'existe pas (nginx:9.99) : que se passe-t-il pour les utilisateurs ?
   → Pas de coupure : les nouveaux Pods restent en ImagePullBackOff et ne sont jamais prêts, donc le Deployment
     ne supprime pas les anciens (au plus 1, à cause de maxUnavailable) : au moins 4 anciens Pods continuent
     de servir. Le rollout reste bloqué ; après progressDeadlineSeconds (600 s), il est marqué en échec
     (ProgressDeadlineExceeded). Correction : `kubectl rollout undo` (ou remettre une image valide).

✅ Étape 5 validée : rolling update 1.27 → 1.28 sans coupure.

## Étape 6 : rollback + correction de la dérive
kubectl rollout history deployment/vitrine                   # REVISION 1 (nginx:1.27) et 2 (nginx:1.28), CHANGE-CAUSE <none>
kubectl rollout undo deployment/vitrine && kubectl rollout status deployment/vitrine
    # rolled back → Warning : last-applied-configuration n'est pas mise à jour par un rollback
kubectl get rs -o wide && kubectl rollout history deployment/vitrine
    # 768475f9f6 (1.27) remonté à 5, 645d8b6885 (1.28) à 0 ; historique : 2 et 3 (la révision 1 est devenue la 3)
sed -i 's/nginx:1.28/nginx:1.27/' 07-revision-jour1/deployment-vitrine.yaml     # corrige la dérive : le fichier redit 1.27
kubectl diff -f 07-revision-jour1/deployment-vitrine.yaml    # vide (code 0) : le spec du fichier = celui du cluster
kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml && kubectl get rs   # configured, aucun rolling update
kubectl get deploy vitrine -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'
    # l'annotation dit maintenant "image":"nginx:1.27", "replicas":5

À retenir :
- `rollout undo` revient à la révision précédente (`--to-revision=N` pour en viser une précise) ; c'est aussi
  un rolling update, sans coupure.
- Un rollback réutilise l'ancien ReplicaSet et lui donne un NOUVEAU numéro de révision (1 → 3) ;
  un même template n'apparaît qu'une fois dans l'historique.
- Après un rollback, le fichier YAML dit encore la nouvelle version = dérive. Il faut remettre le fichier
  d'accord (puis apply), sinon le prochain `apply` relancerait la mauvaise version.
- `kubectl diff` compare le contenu de l'objet, pas l'annotation last-applied-configuration.
- `-o jsonpath='{...}'` : extraire un seul champ ; `\.` pour les points à l'intérieur d'un nom de clé.

### Questions
1. Après `rollout undo`, pourquoi la révision 1 a disparu et une révision 3 est apparue ?
   → Kubernetes ne recule pas dans l'historique, il avance toujours : le rollback est une nouvelle mise à jour.
     Il réutilise l'ancien ReplicaSet (même template, même hash 768475f9f6) et lui donne le numéro suivant (3).
     Un même template n'apparaît qu'une seule fois dans l'historique.
2. Après un rollback, pourquoi modifier aussi le fichier YAML ?
   → Le rollback change le cluster (nginx:1.27), mais pas le fichier, qui dit encore nginx:1.28 (dérive).
     Au prochain `kubectl apply` du fichier, Kubernetes relancerait un rolling update vers nginx:1.28 :
     la version boguée reviendrait et le rollback serait annulé. On remet donc le fichier en 1.27, puis apply.
3. En production, la v2 provoque des erreurs : premières commandes ?
   → D'abord rétablir le service, puis comprendre :
     kubectl rollout undo deployment/vitrine          # revenir tout de suite à la version qui marchait
     kubectl rollout status deployment/vitrine        # vérifier que le retour est terminé
     kubectl get pods / describe pod / logs <pod>     # comprendre pourquoi la v2 échouait
     puis corriger le fichier YAML (et le commit Git) pour qu'il redise la bonne version.

Vérification : rollback sans corriger le fichier, puis `apply` → nginx:1.28 revient, car le fichier n'a pas été mis à jour. ✅

✅ Étape 6 validée : rollback, historique des révisions, dérive corrigée (fichier + last-applied-configuration).

## Étape 7 : Service ClusterIP + test busybox
Fichier : `service-clusterip.yaml`
(apiVersion v1, kind Service, name vitrine-svc, type ClusterIP, selector app: vitrine, port 80 → targetPort 80)

kubectl apply -f 07-revision-jour1/service-clusterip.yaml && kubectl get svc   # vitrine-svc ClusterIP 10.100.48.48, 80/TCP
kubectl describe svc vitrine-svc && kubectl get pods -o wide                     # Endpoints = les 5 IP des Pods (10.244.0.21–25:80)
kubectl run test --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- http://vitrine-svc
    # Pod jetable dans le cluster → page « Welcome to nginx! », puis pod "test" deleted

À retenir :
- Service = IP fixe + nom DNS fixe + répartition vers les Pods choisis par le selector (labels).
- ClusterIP : joignable seulement depuis l'intérieur du cluster ; son IP (10.100.x.x) n'est pas une IP de Pod (10.244.x.x).
- Endpoints : liste IP:port des Pods derrière le Service, mise à jour automatiquement.
- Le DNS du cluster traduit `vitrine-svc` en 10.100.48.48 (nom complet : vitrine-svc.default.svc.cluster.local).
- kubectl run --rm -i --restart=Never : Pod de test supprimé à la fin ; `-t` impossible avec `!` (pas de vrai terminal).
- Page affichée 2 fois : wget a fini avant que kubectl s'attache → kubectl réaffiche les logs du Pod. Pas une erreur.

### Questions
1. Pourquoi un Service alors que chaque Pod a déjà une IP ?
   → L'IP d'un Pod change à chaque remplacement ; le Service garde une IP fixe et un nom DNS fixe (vitrine-svc),
     et répartit les requêtes entre les Pods (load balancing).
2. Comment le Service choisit-il ses Pods ? Que se passe-t-il si un Pod est supprimé ?
   → Grâce au selector app=vitrine (les labels des Pods). Les Endpoints se mettent à jour tout seuls :
     l'IP du Pod supprimé sort de la liste, celle du remplaçant y entre.
3. Pourquoi tester avec un Pod busybox et pas avec curl depuis le PC ?
   → Un Service ClusterIP est accessible seulement depuis l'intérieur du cluster.

✅ Étape 7 validée : Service ClusterIP vitrine-svc, Endpoints, test DNS depuis un Pod busybox.

## Étape 8 : Service NodePort + curl
Fichier : `service-nodeport.yaml`
(type NodePort, selector app: vitrine, port 80 → targetPort 80, nodePort 30080)

kubectl apply -f 07-revision-jour1/service-nodeport.yaml && kubectl get svc   # vitrine-nodeport NodePort 10.110.195.167, 80:30080/TCP
curl -s http://$(minikube ip):30080 | grep -i title                           # depuis le PC : <title>Welcome to nginx!</title>

À retenir :
- NodePort ouvre le même port (30000–32767) sur chaque nœud : PC → IP du nœud:30080 → Service:80 → Pod:80.
- Un NodePort a aussi une ClusterIP (il est joignable de l'intérieur ET de l'extérieur).
- Plusieurs Services peuvent viser les mêmes Pods (même selector).
- `80:30080` = port du Service : port du nœud. Sans `nodePort:`, Kubernetes choisit un port au hasard.
- `$(minikube ip)` : remplacé par l'IP du nœud (192.168.49.2).

### Questions
1. Différence ClusterIP / NodePort ?
   → ClusterIP : joignable seulement depuis l'intérieur du cluster. NodePort : joignable depuis l'extérieur
     (IP du nœud:30080) ET de l'intérieur (il a aussi une ClusterIP).
2. Dans `80:30080/TCP`, que représentent 80 et 30080 ? Et targetPort ?
   → 80 = port du Service (le comptoir), 30080 = port ouvert sur le nœud (la porte sur la rue),
     targetPort 80 = port du conteneur nginx (= containerPort du Deployment).
3. Pourquoi pas de NodePort pour une vraie appli en production sur Internet ?
   → Ports bizarres (30000–32767, pas 80/443), l'IP du nœud peut changer, et avec plusieurs nœuds il faut
     choisir une IP de nœud (si ce nœud tombe, plus d'accès) : pas de point d'entrée unique, pas de nom de
     domaine ni de HTTPS. En production : Service LoadBalancer (IP publique du cloud) ou Ingress (HTTP/HTTPS,
     noms de domaine, un seul point d'entrée pour plusieurs Services).

✅ Étape 8 validée : Service NodePort vitrine-nodeport, accès depuis le PC par l'IP du nœud.

## Étape 9 : port-forward + panne d'un Pod
nohup kubectl port-forward pod/vitrine-768475f9f6-wr4f4 8080:80 > /tmp/pf.log 2>&1 &   # tunnel PC:8080 → Pod wr4f4:80, en arrière-plan
curl -s http://localhost:8080 | grep -i title; cat /tmp/pf.log   # Welcome to nginx! ; Forwarding from 127.0.0.1:8080 -> 80
kubectl delete pod vitrine-768475f9f6-wr4f4                      # panne du Pod relié au tunnel
curl -s -m 3 http://localhost:8080 ...                           # tunnel : plus de réponse ; log : error: lost connection to pod
curl -s http://$(minikube ip):30080 | grep -i title             # NodePort : Welcome to nginx! (toujours OK)
kubectl get pods -o wide && kubectl describe svc vitrine-svc | grep Endpoints
    # remplaçant r9gfj (10.244.0.27) ; Endpoints : .21 disparu, .27 ajouté (.22 .23 .24 .25 .27)

À retenir :
- port-forward = tunnel temporaire de débogage vers UN seul Pod (format port_local:port_du_Pod).
  Si ce Pod disparaît, le tunnel s'arrête (lost connection to pod) et ne bascule jamais vers un autre Pod.
  Même `port-forward svc/...` choisit un seul Pod au démarrage.
- Le Service, lui, suit le label app=vitrine : ses Endpoints se mettent à jour tout seuls, l'appli reste accessible.
- Une commande lancée avec `&` tourne en arrière-plan ; `nohup` la garde en vie après la fin du shell.
- `pgrep -f motif` peut se trouver lui-même (le motif est dans sa propre ligne de commande) ;
  `pgrep -a kubectl` est plus fiable.
- Voir toutes les IP derrière un Service :
  kubectl get endpointslices -l kubernetes.io/service-name=vitrine-svc

### Questions
1. Pourquoi le tunnel a cassé alors que le NodePort marchait toujours ?
   → Le port-forward est lié à un seul Pod précis ; le NodePort (Service) suit le label app=vitrine et ses
     Endpoints se mettent à jour tout seuls.
2. Pourquoi le port-forward ne s'est pas reconnecté au Pod remplaçant ?
   → Le remplaçant est un nouveau Pod avec un autre nom (r9gfj ≠ wr4f4) et une autre IP. Le port-forward visait
     wr4f4 : il s'arrête (lost connection to pod) et doit être relancé à la main.
3. À quoi sert `kubectl port-forward`, pourquoi jamais pour de vrais utilisateurs ?
   → Débogage temporaire (tester un Pod ou une base de données sans l'exposer). Pas pour des utilisateurs :
     lié à un seul Pod (pas de répartition, casse si le Pod disparaît), tourne sur MON PC (s'arrête si je
     ferme le terminal), écoute seulement sur localhost et demande un accès kubectl au cluster.
     Pour les utilisateurs : Service NodePort / LoadBalancer ou Ingress.

✅ Étape 9 validée : port-forward lié à un seul Pod, Service qui suit les labels.

## Étape 10 : mini-entretien oral

1. Différence Pod / ReplicaSet / Deployment ? Lequel en pratique ?
   → Pod : la plus petite unité de Kubernetes ; il enveloppe un (ou plusieurs) conteneurs qui partagent la même
     IP. Seul, il n'est jamais recréé. ReplicaSet : garde N Pods identiques en vie (recrée ceux qui manquent).
     Deployment : gère les ReplicaSets et donc les versions (rolling update, rollback, historique).
     En pratique : toujours un Deployment.
2. Déployer une nouvelle version sans interruption ?
   → Rolling update : on change l'image dans le YAML puis `kubectl apply` (suivi avec `kubectl rollout status`).
     Le Deployment crée un nouveau ReplicaSet et remplace les Pods petit à petit : il n'arrête un ancien Pod
     que quand un nouveau est prêt (maxUnavailable / maxSurge, 25 % par défaut), donc il reste toujours des
     Pods qui servent.
3. La nouvelle version provoque des erreurs : que faire ?
   → `kubectl rollout undo` pour revenir tout de suite à la version qui marchait (`rollout status` pour vérifier),
     puis analyser avec get pods / describe / logs. Ensuite, remettre le fichier YAML (et Git) sur la bonne version
     et faire `kubectl apply`, pour éviter la dérive : sinon le prochain apply relancerait la version boguée.
4. Les Pods changent d'IP : comment les joindre ? Et depuis l'extérieur ?
   → Un Service : IP fixe et nom DNS fixe (ex. vitrine-svc), qui trouve les Pods par leur label (selector).
     ClusterIP pour l'intérieur du cluster, NodePort pour l'extérieur (IP du nœud:30000–32767).
     En production : LoadBalancer ou Ingress.
5. « Mon Service ne renvoie rien, mes Pods tournent bien » : première vérification ?
   → Le selector et les Endpoints : `kubectl describe svc <nom>`. Endpoints vides = le selector ne correspond
     à aucun label de Pod → comparer avec `kubectl get pods --show-labels`. Vérifier aussi que le targetPort
     correspond au port du conteneur.

✅ Étape 10 validée : mini-entretien oral (5 questions).

✅ RÉVISION DU JOUR 1 TERMINÉE (étapes 0 à 10). Suite : défi final sans aide dans `08-defi-jour1/`.
