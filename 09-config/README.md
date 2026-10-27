# Jour 2 — Configuration : variables d'environnement, ConfigMaps, Secrets

## Étape 1 : variables d'environnement dans un Pod
Fichier : `pod-env.yaml` (Pod env-demo, busybox:1.36, env APP_COULEUR=bleu et APP_MODE=debug,
command qui affiche les variables puis `sleep 3600`)

kubectl apply -f 09-config/pod-env.yaml && kubectl get pods   # created ; ContainerCreating (téléchargement de l'image)
kubectl get pods && kubectl logs env-demo                     # Running ; Couleur=bleu Mode=debug
kubectl exec env-demo -- env | grep APP                       # APP_COULEUR=bleu, APP_MODE=debug

Syntaxe :
    env:
    - name: MA_VARIABLE
      value: "ma valeur"      # toujours une chaîne (guillemets pour "8080", "true")

À retenir :
- Même image, réglages différents selon l'environnement (dev, test, prod) : pas besoin de reconstruire l'image.
- `kubectl logs` : ce que le conteneur a écrit. `kubectl exec <pod> -- <commande>` : lancer une commande DANS le conteneur.
- busybox s'arrête dès que sa commande est finie : `sleep 3600` le garde en vie.

Modifier une variable :
sed -i 's/bleu/rouge/' 09-config/pod-env.yaml
kubectl apply -f 09-config/pod-env.yaml      # REFUSÉ : spec: Forbidden: pod updates may not change fields other than spec.containers[*].image, ...
kubectl delete pod env-demo                  # il faut supprimer le Pod...
kubectl apply -f 09-config/pod-env.yaml      # ...et le recréer
kubectl wait --for=condition=Ready pod/env-demo --timeout=60s && kubectl logs env-demo   # Couleur=rouge Mode=debug

À retenir :
- Un Pod est presque immuable : on ne peut changer que quelques champs (surtout l'image), pas `env`.
- Les variables sont lues au démarrage du conteneur : pour en changer, il faut un nouveau Pod.
- Avec un Deployment, changer `env` modifie le template → nouveau ReplicaSet → rolling update, sans coupure.
- `kubectl logs` trop tôt → « waiting to start: ContainerCreating ». `kubectl wait --for=condition=Ready` attend
  que le Pod soit prêt (utile dans les scripts et la CI).
- Supprimer un Pod busybox prend ~30 s : `sleep` ignore le signal d'arrêt, Kubernetes attend la fin du délai de grâce.

### Questions
1. Différence entre `kubectl logs` et `kubectl exec <pod> -- env` ?
   → logs affiche ce que le conteneur a déjà écrit (sa sortie) ; exec lance une nouvelle commande à l'intérieur
     du conteneur, maintenant.
2. Pourquoi la config dans des variables d'environnement plutôt que dans l'image ?
   → On change seulement la configuration, sans reconstruire l'image : une seule image pour dev, test et prod
     (séparer la configuration du code, principe des 12-factor apps).
3. Changer une variable dans le fichier d'un Pod puis `kubectl apply` ?
   → Refusé (Forbidden) : le Pod est presque immuable. Il faut le supprimer et le recréer ; avec un Deployment,
     c'est un rolling update automatique.

✅ Étape 1 validée : variables d'environnement.

## Étape 2 : ConfigMap
Fichiers : `configmap-app.yaml` (ConfigMap app-config : APP_COULEUR=vert, APP_MODE=production)
           `pod-configmap.yaml` (Pod cm-demo, busybox, lit toute la ConfigMap avec envFrom)

Syntaxe :
    kind: ConfigMap
    data:
      CLE: "valeur"
  Dans le conteneur, toutes les clés :      Ou une seule clé :
    envFrom:                                  env:
    - configMapRef:                           - name: MA_VARIABLE
        name: app-config                        valueFrom:
                                                  configMapKeyRef:
                                                    name: app-config
                                                    key: CLE

Panne testée : Pod appliqué AVANT la ConfigMap
kubectl apply -f 09-config/pod-configmap.yaml && kubectl get pods   # created, puis 0/1 CreateContainerConfigError
kubectl describe pod cm-demo        # Events : Warning Failed (x12 over 2m20s) kubelet  Error: configmap "app-config" not found
kubectl apply -f 09-config/configmap-app.yaml                       # configmap/app-config created
kubectl wait --for=condition=Ready pod/cm-demo --timeout=60s && kubectl logs cm-demo   # Couleur=vert Mode=production

À retenir :
- ConfigMap = objet séparé avec la configuration (clé = valeur), partagé par plusieurs Pods.
- L'API server accepte un Pod même si sa ConfigMap n'existe pas ; c'est le kubelet qui bloque :
  CreateContainerConfigError (State: Waiting). Cause exacte : `kubectl describe pod` → Events.
- Le kubelet réessaie tout seul : dès que la ConfigMap existe, le Pod démarre, sans le recréer.
- `optional: true` dans configMapRef : le Pod démarrerait sans les variables (par défaut Optional: false).

Modifier la ConfigMap pendant que le Pod tourne :
sed -i 's/vert/jaune/' 09-config/configmap-app.yaml
kubectl apply -f 09-config/configmap-app.yaml        # configmap/app-config configured
kubectl exec cm-demo -- env | grep APP               # APP_COULEUR=vert : le Pod ne voit PAS le changement

À retenir :
- Les variables d'environnement sont lues au démarrage du conteneur : modifier la ConfigMap ne change rien
  pour les Pods déjà lancés. Il faut de nouveaux Pods :
  Pod seul → delete + apply ; Deployment → `kubectl rollout restart deployment/<nom>` (rolling update).
- Une ConfigMap montée comme fichier (volume) est, elle, mise à jour automatiquement dans le Pod (après un délai).

### Questions
1. Avantage d'une ConfigMap par rapport à `env:` dans le Pod ?
   → Une seule ConfigMap partagée par plusieurs Pods : on change un réglage à un seul endroit. La configuration
     est séparée du Deployment (même Deployment, ConfigMap différente en dev et en prod).
2. Pod en CreateContainerConfigError : cause probable et commande ?
   → ConfigMap (ou Secret, ou clé) manquante. `kubectl describe pod <nom>` → Events : configmap "..." not found.
3. ConfigMap modifiée, utilisée en variables par un Deployment : que faire ?
   → `kubectl rollout restart deployment/<nom>` : recrée les Pods (rolling update, sans coupure), qui lisent
     la nouvelle valeur au démarrage.

✅ Étape 2 validée : ConfigMap (envFrom), panne CreateContainerConfigError, variables figées au démarrage.

## Étape 3 : Secret
kubectl create secret generic db-secret --from-literal=DB_USER=admin --from-literal=DB_PASSWORD=SuperSecret123
kubectl describe secret db-secret         # Type: Opaque ; valeurs cachées, seulement la taille (14 bytes, 5 bytes)
kubectl get secret db-secret -o yaml      # data: DB_PASSWORD: U3VwZXJTZWNyZXQxMjM=  DB_USER: YWRtaW4=
echo U3VwZXJTZWNyZXQxMjM= | base64 -d; echo   # SuperSecret123

À retenir :
- Secret = comme une ConfigMap, pour les données sensibles (mots de passe, clés d'API, tokens).
- base64 n'est PAS du chiffrement : n'importe qui ayant le droit de lire le Secret le décode en une commande.
- La vraie protection : droits d'accès (RBAC), chiffrement d'etcd (encryption at rest), ou un gestionnaire
  externe (Vault, AWS Secrets Manager...).
- Ne jamais pousser un Secret en YAML sur GitHub : le créer en ligne de commande (ou via un outil dédié).
- `describe secret` cache les valeurs ; `get secret -o yaml` les montre en base64.

Fichier : `pod-secret.yaml` (Pod secret-demo, busybox, DB_USER et DB_PASSWORD via secretKeyRef)
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
  (ou toutes les clés : envFrom: - secretRef: name: db-secret)

kubectl apply -f 09-config/pod-secret.yaml
kubectl wait --for=condition=Ready pod/secret-demo --timeout=60s && kubectl logs secret-demo
    # Utilisateur=admin, mot de passe de 14 caracteres  (${#VAR} = longueur ; ne jamais écrire un secret dans les logs)
kubectl exec secret-demo -- env | grep DB_     # DB_PASSWORD=SuperSecret123 : dans le conteneur, la valeur est en clair

À retenir :
- Kubernetes décode le base64 avant de donner la valeur au conteneur : l'application reçoit le vrai mot de passe.
- Qui peut faire `kubectl exec` (ou `get secret`) peut lire le secret → limiter ces droits avec RBAC.

### Questions
1. Différence ConfigMap / Secret ? Quand utiliser l'un ou l'autre ?
   → ConfigMap : configuration non sensible (couleur, mode, URL d'un service). Secret : données sensibles
     (mot de passe de base de données, clé d'API, certificats TLS/SSL — type spécial kubernetes.io/tls).
     Les deux permettent de changer une valeur sans reconstruire l'image ; la différence est le caractère sensible.
2. Les Secrets sont-ils chiffrés ? Comment les protéger ?
   → Non, pas par défaut : seulement encodés en base64. Protection : RBAC (qui peut lire les Secrets et faire
     exec), chiffrement d'etcd (encryption at rest), gestionnaire externe (Vault, AWS Secrets Manager).
3. Pourquoi créer le Secret en ligne de commande plutôt qu'en YAML dans le dépôt ?
   → Pour ne pas pousser le mot de passe sur GitHub (le base64 se décode en une commande).
     Attention : `--from-literal` laisse le mot de passe dans l'historique du shell ; alternatives :
     `--from-file`, ou des outils comme Sealed Secrets / External Secrets, ou les secrets de la CI.
   (Ici, SuperSecret123 est un mot de passe de démonstration.)

✅ Étape 3 validée : Secret (base64 ≠ chiffrement, secretKeyRef, protection).
