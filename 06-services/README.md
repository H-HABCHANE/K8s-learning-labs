# Jour 1 — Les Services

## C'est quoi ?
Les Pods changent d'IP quand ils sont recréés. Un Service donne un nom et une IP fixes devant un groupe de Pods
(choisis par étiquette, `selector`) et répartit les requêtes entre eux. Analogie : l'accueil du restaurant.
- ClusterIP (par défaut) : accessible seulement depuis l'intérieur du cluster.
- NodePort : ouvre aussi un port (30000–32767) sur le nœud, accessible de l'extérieur.

## Exercice 8 : Service ClusterIP devant monsite
Fichier : `service-clusterip.yaml` (écrit seul avec nano à partir d'un squelette) :
apiVersion v1, kind Service, name monsite-svc, type ClusterIP, selector app: monsite, port 80 → targetPort 80.
Remarque : pour un Service, l'étiquette se met directement sous `selector` (pas de `matchLabels`).

kubectl apply --dry-run=client -f 06-services/service-clusterip.yaml   # → created (dry run)
kubectl apply -f 06-services/service-clusterip.yaml                    # → service/monsite-svc created
kubectl get services                                                   # monsite-svc ClusterIP 10.109.84.17, 80/TCP
kubectl describe service monsite-svc                                   # Endpoints: 10.244.0.22:80,10.244.0.23:80,10.244.0.20:80 + 2 more
kubectl get pods -o wide                                               # IP des 5 Pods : .19 .20 .21 .22 .23

Piège rencontré : quitter nano sans enregistrer (fichier absent, seul le verrou `.service-clusterip.yaml.swp`
existait). Toujours Ctrl+O, Entrée (« Wrote X lines »), puis Ctrl+X.

### Mes réponses
1. 5 Endpoints → ce sont les 5 Pods de monsite (IP:port). Le Service les trouve grâce à app=monsite,
   et la liste se met à jour automatiquement quand un Pod est remplacé. ✅
2. Le Service `kubernetes` (10.96.0.1:443) → l'accès à l'API server, en HTTPS. Créé automatiquement par
   Kubernetes ; les Pods qui veulent parler à l'API passent par lui. Ne pas le supprimer. ✅
3. EXTERNAL-IP `<none>` → ClusterIP est interne, pas accessible depuis l'extérieur du cluster. ✅
   Remarque : l'IP du Service (10.109.x.x) n'est pas dans la plage des Pods (10.244.x.x) : c'est une adresse virtuelle.

### Test depuis l'intérieur du cluster
kubectl run test --image=busybox --restart=Never -- wget -qO- http://monsite-svc   # Pod client jetable
kubectl get pods test     # Completed, READY 0/1 = la commande est finie (pas une erreur)
kubectl logs test         # page « Welcome to nginx! » → le Service fonctionne

Chemin : Pod test → nom monsite-svc → 10.109.84.17:80 → un des 5 Pods nginx (port 80).

### Mes réponses
1. Quel Pod a répondu ? → Non, impossible à savoir : les 5 Pods servent exactement la même page,
   et le Service choisit un Pod à chaque connexion (répartition de charge). ✅ (complété)
2. Qui traduit monsite-svc en IP ? → le DNS du cluster (CoreDNS). Nom complet :
   monsite-svc.default.svc.cluster.local. ✅
3. wget depuis mon PC ? → Non : ClusterIP est interne, et le nom monsite-svc n'existe que dans le DNS du cluster. ✅

## Exercice 9 : Service NodePort (accès depuis mon PC)
Fichier : `service-nodeport.yaml`, copie du ClusterIP puis 3 changements faits avec sed :
name → monsite-nodeport ; type → NodePort ; ajout de `nodePort: 30080` sous targetPort.
Sans changer le nom, `apply` aurait répondu `monsite-svc unchanged` (même objet) au lieu d'en créer un nouveau.

cp 06-services/service-clusterip.yaml 06-services/service-nodeport.yaml
sed -i 's/name: monsite-svc/name: monsite-nodeport/' 06-services/service-nodeport.yaml
sed -i 's/type: ClusterIP/type: NodePort/' 06-services/service-nodeport.yaml
sed -i 's/^      targetPort: 80$/&\n      nodePort: 30080/' 06-services/service-nodeport.yaml   # & = la ligne trouvée, \n = nouvelle ligne
kubectl apply -f 06-services/service-nodeport.yaml   # → service/monsite-nodeport created
kubectl get services                                 # monsite-nodeport NodePort 10.111.148.216 80:30080/TCP

Chemin : PC → IP du nœud:30080 (nodePort) → Service:80 (port) → Pod:80 (targetPort).

### Mes réponses
1. 80:30080 → 80 = port du Service (dans le cluster), 30080 = port ouvert sur le nœud (accès extérieur). ✅
2. NodePort a aussi une CLUSTER-IP → oui : NodePort inclut ClusterIP et ajoute une porte sur le nœud.
   Les Pods peuvent toujours l'utiliser en interne (monsite-nodeport:80). ✅

### Test depuis mon PC
minikube ip                          # → 192.168.49.2 (IP du nœud)
curl -s http://192.168.49.2:30080    # page « Welcome to nginx! » (-s cache la barre de progression)

### Ma réponse
Pourquoi ça marche ici et pas `wget http://monsite-svc` depuis le PC ?
→ On utilise l'IP du nœud (connue du PC) au lieu du nom monsite-svc (qui n'existe que dans le DNS du cluster),
  et le type NodePort ouvre un port accessible de l'extérieur, contrairement à ClusterIP. ✅

## Exercice 10 : port-forward + panne d'un Pod derrière le Service
kubectl port-forward service/monsite-svc 8080:80 > ~/pf.log 2>&1 &   # tunnel PC:8080 → Service:80, en arrière-plan (&)
cat ~/pf.log                                                          # Forwarding from 127.0.0.1:8080 -> 80
curl -s http://localhost:8080 | grep title                            # <title>Welcome to nginx!</title>
kubectl get endpointslices -l kubernetes.io/service-name=monsite-svc  # liste des Pods derrière le Service
kubectl describe endpointslice -l kubernetes.io/service-name=monsite-svc | grep Addresses   # toutes les IP, sans "+ 2 more"
kubectl delete pod monsite-86f69f946d-mf7nv                           # panne : Pod .19 supprimé

Pièges rencontrés :
- `port-forward` bloque le terminal : soit un 2ᵉ terminal, soit `&` (arrière-plan). Sans tunnel, `curl -s` échoue en silence.
- Résultat observé : remplaçant 7t7wk (IP .25) ; Endpoints : .19 retiré, .25 ajouté automatiquement ;
  NodePort (curl 192.168.49.2:30080) a continué de répondre ; MAIS le port-forward est mort
  (« error: lost connection to pod ») : il était branché sur mf7nv, justement le Pod supprimé.
- `port-forward service/...` choisit UN Pod au démarrage et s'y branche directement (pas de répartition).
  → outil de débogage, jamais pour exposer un vrai site. Il a fallu le relancer.

### Mes réponses
1. Pourquoi le tunnel vers un ClusterIP interne marche depuis le PC ? → kubectl relaie le trafic via l'API server. ✅
2. Le Service a-t-il suivi la panne ? → oui, les Endpoints se mettent à jour seuls (par étiquette).
   Complété : le port-forward, lui, n'a PAS suivi (fixé sur un seul Pod). ✅ (complété)
3. Le client du NodePort a-t-il dû changer quelque chose ? → non, rien : même IP, même port. ✅

## Questions d'entretien (Jour 1)
Q1. Comment Kubernetes maintient 3 replicas ?
→ Ma réponse : grâce au ReplicaSet, qui surveille le nombre de Pods en cours d'exécution. ✅
→ Enrichie : le Deployment crée un ReplicaSet qui compte en permanence les Pods portant son étiquette et compare
  à l'état souhaité (boucle de réconciliation) ; il en manque → il crée, trop → il supprime. Testé : Pod supprimé,
  remplaçant en quelques secondes (nouveau nom, nouvelle IP).

Q2. Pourquoi un Service plutôt que l'IP des Pods ?
→ Ma réponse : IP stable + répartition du trafic, car les IP des Pods changent. ✅
→ Enrichie : il offre aussi un nom DNS (monsite-svc) et ne garde dans ses Endpoints que les Pods prêts.

Q3. ClusterIP / NodePort / port-forward ?
→ Ma réponse : interne / externe via un port du nœud / accès temporaire depuis le PC pour tester. ✅
→ Enrichie : NodePort inclut ClusterIP ; en production on utilise plutôt LoadBalancer ou Ingress ;
  port-forward se branche sur UN seul Pod (vu à l'Ex. 10) → débogage uniquement.

Q4. Comment le Service trouve ses Pods ?
→ Ma réponse : selector qui identifie les Pods par leurs labels. ✅
→ Enrichie : le résultat est la liste des Endpoints (EndpointSlice), mise à jour automatiquement.

Q5. Service inaccessible mais Pods Running ?
→ Ma réponse : vérifier le Service, labels/selectors, endpoints, ports, puis logs des Pods. ✅
→ Enrichie, dans l'ordre : 1) kubectl get endpointslices → vide = selector ≠ labels ; 2) targetPort = containerPort ?
  3) Pods Ready ? 4) tester depuis l'intérieur (busybox wget) pour séparer problème Service / problème d'accès externe ;
  5) kubectl logs / describe pod.

Cas concrets (suite de l'entretien) :
1. selector app: monsite mais Pods labellisés app: mon-site → Endpoints vides, le client n'accède pas au site. ✅
2. targetPort: 8080 alors que nginx écoute sur 80 → Endpoints remplis (le selector correspond),
   mais le site ne marche pas : le trafic part vers un port où personne n'écoute. ✅
   → Endpoints remplis ≠ Service qui fonctionne : il faut aussi vérifier les ports.
