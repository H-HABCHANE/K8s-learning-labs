# Jour 2 — Namespaces

## Étape 7 : namespaces
Namespace = « dossier virtuel » dans le cluster : chaque namespace a ses Pods, Deployments, Services, ConfigMaps,
Secrets... Un nom est unique DANS un namespace, pas entre namespaces. Sert à séparer équipes et environnements
(dev, test, prod) dans un même cluster.

kubectl get namespaces                 # (ou get ns) default, kube-system, kube-public, kube-node-lease
kubectl get pods -n kube-system        # les composants de Kubernetes : apiserver, etcd, scheduler,
                                       # controller-manager, coredns, kube-proxy, kindnet, storage-provisioner
                                       # (le kubelet n'est pas un Pod : programme installé sur le nœud)

| Namespace       | Rôle                                                           |
|-----------------|----------------------------------------------------------------|
| default         | là où vont les objets quand on ne précise rien                 |
| kube-system     | composants internes de Kubernetes (ne jamais y toucher)        |
| kube-public     | infos lisibles par tous (presque vide)                         |
| kube-node-lease | « battements de cœur » des nœuds                               |

Fichier : `namespaces.yaml` (2 objets Namespace, dev et prod, séparés par `---`)
kubectl apply -f 12-namespaces/namespaces.yaml && kubectl get ns     # dev et prod Active

Même fichier, deux namespaces :
kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml -n dev    # created
kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml -n prod   # created (objet différent, pas de conflit)
kubectl get deploy                     # No resources found in default namespace.
kubectl get deploy -A                  # colonne NAMESPACE : dev/vitrine, prod/vitrine, kube-system/coredns

Supprimer un namespace :
kubectl delete namespace dev           # supprime TOUT ce qu'il contient (Deployment, ReplicaSets, Pods...)
kubectl get ns && kubectl get deploy -A   # dev a disparu ; prod/vitrine intact

À retenir :
- `-n <ns>` : choisir le namespace (sans -n → default) ; `-A` : tous les namespaces.
- Piège n°1 : « mes Pods ont disparu » → on ne regarde pas dans le bon namespace.
- `kubectl delete namespace` = suppression en cascade de tout le contenu : très dangereux en production.
- Un même YAML (sans `namespace:` dans metadata) peut être déployé en dev puis en prod avec -n.
- Plusieurs objets dans un fichier : séparés par une ligne `---`.

DNS entre namespaces (vitrine + vitrine-svc dans prod, Pod de test dans default) :
kubectl apply -f 07-revision-jour1/service-clusterip.yaml -n prod
kubectl run test --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- -T 3 http://vitrine-svc | grep -i title
    # pod default/test terminated (Error) : le nom court est cherché dans default
kubectl run test --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- -T 3 http://vitrine-svc.prod | grep -i title
    # <title>Welcome to nginx!</title>

### Questions
1. À quoi sert un namespace ?
   → Séparer les environnements (dev, test, prod) dans un même cluster ; éviter les conflits de noms ;
     contrôler les accès (RBAC par namespace) et les ressources par équipe (ResourceQuota : total CPU/mémoire
     autorisé dans un namespace).
2. « kubectl get pods n'affiche rien » ?
   → Mauvais namespace (kubectl regarde default). `kubectl get pods -A` pour trouver où est l'appli, puis
     `kubectl get pods -n <namespace>`. Changer le namespace par défaut :
     `kubectl config set-context --current --namespace=<ns>` (attention à ne pas l'oublier).
3. Le nom court d'un Service marche-t-il depuis un autre namespace ?
   → Non : le nom court ne marche que dans le même namespace. Ailleurs : `<service>.<namespace>`
     (ou le nom complet `<service>.<namespace>.svc.cluster.local`).

✅ Étape 7 validée : namespaces.
