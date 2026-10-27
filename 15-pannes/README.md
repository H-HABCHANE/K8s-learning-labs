# Jour 2 — Diagnostiquer les pannes

## Étape 10 : pannes
Méthode : `kubectl get pods` (le STATUS) → `kubectl describe pod` (Events, State, Last State) → `kubectl logs`.

| STATUS                         | Signification                         | Où trouver la cause                                      |
|--------------------------------|---------------------------------------|----------------------------------------------------------|
| Pending                        | aucun nœud ne convient                | describe → FailedScheduling (Insufficient cpu/memory)    |
| CreateContainerConfigError     | ConfigMap / Secret / clé manquant     | describe → configmap "..." not found                     |
| ErrImagePull / ImagePullBackOff| image impossible à télécharger        | describe → Failed to pull image ... (not found, denied...) |
| Running mais 0/1               | la readiness échoue                   | describe → Unhealthy ... Readiness probe failed          |
| CrashLoopBackOff (probe)       | la liveness tue le conteneur          | describe → Killing ... failed liveness probe             |
| CrashLoopBackOff (appli)       | l'application plante au démarrage     | logs --previous + Last State (Exit Code ≠ 0)             |
| OOMKilled                      | limite mémoire dépassée               | describe → Last State: OOMKilled, Exit Code 137          |

### ImagePullBackOff
kubectl create deployment panne-image --image=nginx:9.99 && sleep 15 && kubectl get pods
    # ImagePullBackOff, 0/1, RESTARTS 0 (le conteneur n'a jamais été créé)
kubectl describe pod -l app=panne-image
    # Status Pending, State Waiting, Container ID vide
    # Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found
    # ErrImagePull (1er échec) → ImagePullBackOff (le kubelet réessaie avec des pauses croissantes)
kubectl set image deployment/panne-image nginx=nginx:1.27      # conteneur nginx → nouvelle image
kubectl get pods -l app=panne-image                            # 1/1 Running

| Message                              | Cause                                    | Correction                     |
|--------------------------------------|------------------------------------------|--------------------------------|
| not found                            | faute de frappe dans le nom ou le tag    | corriger l'image               |
| pull access denied / unauthorized    | registre privé sans identifiants         | imagePullSecret                |
| i/o timeout / toomanyrequests        | réseau, ou limite Docker Hub             | réseau, attendre, miroir       |

À retenir :
- `kubectl create` refuse un objet qui existe déjà (already exists) ; `kubectl apply` le met à jour.
- `kubectl set image` corrige vite, mais en impératif : avec un fichier YAML, corriger le fichier puis apply.

### CrashLoopBackOff dû à l'application
kubectl create deployment panne-app --image=busybox:1.36 -- sh -c "echo 'Demarrage...'; echo 'ERREUR FATALE : variable DATABASE_URL manquante'; exit 1"
kubectl get pods -l app=panne-app            # Error, RESTARTS 2 en 20 s (alterne avec CrashLoopBackOff)
kubectl logs <pod>                           # Demarrage... / ERREUR FATALE : variable DATABASE_URL manquante
kubectl logs <pod> --previous                # unable to retrieve container logs (l'avant-dernier conteneur est déjà nettoyé)
kubectl describe pod -l app=panne-app | grep -A 5 -E "State:"
    # State: Terminated, Reason: Error, Exit Code: 1  (conteneur actuel, déjà mort)
    # Last State: Terminated, Reason: Error, Exit Code: 1 (le précédent)

À retenir :
- STATUS Error = le conteneur vient de s'arrêter avec un code ≠ 0 ; CrashLoopBackOff = attente avant le prochain essai.
- Quel log lire :
  conteneur qui vient de planter (Error / CrashLoopBackOff) → `kubectl logs <pod>` ;
  conteneur qui a redémarré et tourne de nouveau (Running, RESTARTS > 0) → `kubectl logs <pod> --previous`.
  Réflexe : essayer les deux.
- Exit Code 1 = erreur de l'application ; 137 = tué (SIGKILL, souvent OOM) ; 0 = arrêt normal.
- La cause est dans l'application ou sa configuration (ici une variable manquante) : on corrige la config
  (env, ConfigMap, Secret) ou le code, pas Kubernetes.

### Questions
1. ImagePullBackOff ou CrashLoopBackOff ? Deux indices.
   → RESTARTS : reste à 0 pour ImagePullBackOff (le conteneur n'a jamais été créé), augmente pour CrashLoopBackOff.
     describe : ImagePullBackOff → State Waiting, Container ID vide, Events « Failed to pull image » ;
     CrashLoopBackOff → Last State Terminated avec un Exit Code (le conteneur a bien tourné).
2. ImagePullBackOff avec « pull access denied » ?
   → Image dans un registre privé, sans identifiants (ou nom d'image faux). Créer un Secret de type registre
     (`kubectl create secret docker-registry ...`) et l'ajouter dans le Pod avec `imagePullSecrets`.
3. Pod qui redémarre en boucle : méthode de diagnostic ?
   → 1) `kubectl get pods` : STATUS et RESTARTS.
     2) `kubectl describe pod` : Events (Killing ... failed liveness probe = probe ; FailedScheduling, Failed to
        pull...) et Last State : Exit Code 137 / OOMKilled = mémoire ; 1 = erreur de l'appli.
     3) `kubectl logs <pod>` et `kubectl logs <pod> --previous` : le message d'erreur de l'application.
     4) Corriger la cause (probe, limits, configuration, code), de préférence dans le fichier YAML puis apply.

✅ Étape 10 validée : diagnostic des pannes.

## Étape 11 : mini-entretien Kubernetes (Jour 2)
1. URL d'API et mot de passe de base de données : comment les fournir ? Le mot de passe est-il protégé ?
   → L'URL dans une ConfigMap (non sensible), le mot de passe dans un Secret, injectés en variables
     d'environnement (envFrom / valueFrom). Par défaut, un Secret est seulement encodé en base64 : le protéger
     avec RBAC, le chiffrement d'etcd (encryption at rest) ou un outil externe (Vault), et ne jamais le
     mettre dans Git.
2. Readiness, liveness, startup ? Exemple de liveness mal configurée ?
   → Startup : laisse le temps de démarrer (liveness et readiness en pause). Readiness : autorise le trafic
     (sinon retiré des Endpoints). Liveness : redémarre le conteneur bloqué. Liveness qui teste une dépendance
     externe (base de données) : un ralentissement de la base fait redémarrer l'appli en boucle → panne totale.
     La liveness ne doit tester que l'appli elle-même.
3. OOMKilled, Exit Code 137 ? requests / limits ?
   → Le conteneur a dépassé sa limite mémoire, le noyau l'a tué (137 = 128 + 9, SIGKILL). Analyser la
     consommation (fuite mémoire ?) ou augmenter la limit. Requests : minimum réservé, utilisé par le scheduler ;
     limits : plafond (CPU : ralenti, mémoire : tué).
4. PostgreSQL : ne pas perdre les données ?
   → Les fichiers de la base sur un PVC, monté dans le conteneur (volumes + volumeMounts sur le dossier des
     données) ; la StorageClass crée le PV. Le disque est indépendant du Pod : un nouveau Pod rebranche le même
     PVC. En production : StatefulSet, reclaimPolicy Retain, et des sauvegardes (un PVC n'est pas une sauvegarde).
5. Nouveaux Pods Running 0/1, anciens toujours là, site OK ?
   → La readiness des nouveaux Pods échoue : ils ne sont pas dans les Endpoints du Service, donc ne reçoivent
     pas de trafic. Le rolling update est bloqué (il ne supprime un ancien Pod que quand un nouveau est prêt) :
     les anciens continuent de servir. Cause : `kubectl describe pod` → Events « Unhealthy ... Readiness probe
     failed » + le détail (404, connection refused...). Puis corriger la probe ou l'appli, ou `kubectl rollout undo`.

✅ Étape 11 validée : mini-entretien Jour 2.
