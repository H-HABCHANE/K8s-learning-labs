# Jour 2 — Probes : readiness, liveness, startup

## Étape 4 : readiness et liveness
Fichiers : `deployment-probes.yaml` (Deployment probe-demo, nginx:1.27, 3 replicas, readiness + liveness httpGet / port 80)
           `service-probes.yaml` (Service ClusterIP probe-svc, app: probe-demo, 80 → 80)

| Probe     | Question                                | Si elle échoue                                        |
|-----------|-----------------------------------------|-------------------------------------------------------|
| readiness | prêt à recevoir des clients ?           | retiré des Endpoints du Service (pas de redémarrage)  |
| liveness  | encore vivant ?                         | le kubelet redémarre le conteneur                     |

Syntaxe (dans le conteneur, au niveau de image/ports) :
    readinessProbe:            # ou livenessProbe
      httpGet:                 # autres types : tcpSocket, exec
        path: /
        port: 80
      initialDelaySeconds: 5   # attente avant le 1er test
      periodSeconds: 5         # fréquence
      # par défaut : timeout 1s, failureThreshold 3, successThreshold 1
  httpGet réussit si le code HTTP est entre 200 et 399.

kubectl apply -f 10-probes/ && kubectl get pods       # Pending / ContainerCreating, puis Running 0/1, puis 1/1
kubectl describe pod -l app=probe-demo | grep -E "Liveness|Readiness" | head -2   # réglages + valeurs par défaut

Readiness cassée (chemin inexistant → 404) :
sed -i '22s|path: /|path: /pas-la|' 10-probes/deployment-probes.yaml   # | comme séparateur (le texte contient des /)
kubectl apply -f 10-probes/ && kubectl get pods
    # nouveau Pod 6fd78789cc : Running 0/1, RESTARTS 0 ; les 3 anciens restent en 1/1 → rolling update bloqué
kubectl describe pod <nouveau-pod>     # Warning Unhealthy (x22 over 106s) Readiness probe failed: HTTP probe failed with statuscode: 404
kubectl describe svc probe-svc | grep Endpoints   # seulement les 3 anciens (.10 .11 .12) ; le nouveau (.13) n'y est pas

À retenir :
- READY 1/1 = la readiness a réussi (pas seulement « le conteneur tourne »).
- Readiness en échec : le Pod tourne mais ne reçoit aucun trafic ; pendant un rolling update, les anciens Pods
  ne sont pas supprimés → une version cassée ne remplace jamais celle qui marche.
- `Unhealthy` dans les Events = une probe échoue ; le message dit laquelle et pourquoi.

Liveness cassée (readiness OK) :
sed -i '22s|path: /pas-la|path: /|' 10-probes/deployment-probes.yaml
sed -i '28s|path: /|path: /pas-la|' 10-probes/deployment-probes.yaml
kubectl apply -f 10-probes/ && kubectl get pods
    # nouveau RS 65d88c6469 ; les 3 anciens Pods SONT remplacés (la readiness réussit) ;
    # RESTARTS qui montent (1er redémarrage ≈ 10 s + 3 × 10 s = 40 s), puis CrashLoopBackOff
kubectl describe pod <pod>
    # Warning Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
    # Normal  Killing    Container nginx failed liveness probe, will be restarted
    # Warning BackOff    Back-off restarting failed container nginx
    # Last State: Terminated, Reason: Completed, Exit Code: 0 (nginx s'arrête proprement, il ne plante pas)
Réparation déclarative : ligne 28 remise sur `/`, puis
kubectl apply -f 10-probes/deployment-probes.yaml   # le template redevient celui du début → RS 6fcd8b4b85 réutilisé
kubectl get pods                                    # 3 Pods 6fcd8b4b85 en 1/1, RESTARTS 0

À retenir :
- La readiness protège un rolling update ; la liveness NON : une liveness fausse peut casser toute l'appli.
- CrashLoopBackOff = le conteneur redémarre en boucle ; le kubelet espace les essais (10 s, 20 s, 40 s... max 5 min).
- Distinguer la cause : Events « Killing ... failed liveness probe » → c'est la probe ;
  Exit Code ≠ 0 sans liveness en cause → c'est l'application (voir `kubectl logs --previous`).
- Un fichier revenu à un ancien template réutilise l'ancien ReplicaSet (même hash), comme un rollback.

### Questions
1. Différence readiness / liveness ? Que fait Kubernetes quand chacune échoue ?
   → Readiness en échec : le Pod est retiré des Endpoints du Service (plus de trafic), sans être tué.
     Liveness en échec : le kubelet tue et redémarre le conteneur.
2. Pourquoi une readiness cassée bloque le rolling update, et une liveness cassée remplace tous les anciens Pods ?
   → La readiness est le feu vert du rolling update : le Deployment ne supprime un ancien Pod que quand un nouveau
     est prêt ; si elle reste rouge, tout s'arrête et les anciens servent toujours. La liveness ne gère que la
     survie du conteneur : le Deployment ne la regarde pas ; si la readiness a dit « prêt », il continue, même
     si la liveness tue ensuite les nouveaux Pods. → liveness plus tolérante que la readiness.
3. CrashLoopBackOff : l'application plante ou une probe est mal configurée ?
   → `kubectl describe pod` → Events : « Killing ... failed liveness probe » = la probe.
     Sinon, c'est l'application : `kubectl logs <pod> --previous` (logs du conteneur précédent, celui qui a planté)
     et l'Exit Code dans Last State.

✅ Étape 4 validée : readiness et liveness, readiness cassée (rolling update bloqué), liveness cassée (CrashLoopBackOff).

## Étape 5 : startup probe
Fichier : `deployment-lent.yaml` (Deployment lent, 1 replica, app: lent, nginx:1.27 qui démarre après `sleep 30`,
liveness stricte : delay 5, period 5, failureThreshold 3)

| Probe     | Quand elle tourne                         | Si elle échoue trop              |
|-----------|-------------------------------------------|----------------------------------|
| startup   | seulement au démarrage, jusqu'au 1er succès | redémarrage                     |
| liveness  | après le succès de la startup             | redémarrage                      |
| readiness | après le succès de la startup             | retiré du Service                |

Sans startup probe :
kubectl apply -f 10-probes/deployment-lent.yaml
kubectl get pods -l app=lent        # CrashLoopBackOff, 7 restarts : la liveness tue à 5 + 3 × 5 = 20 s, nginx démarre à 30 s

Avec startup probe (ajoutée sous la liveness, même indentation) :
    startupProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
      failureThreshold: 12    # budget de démarrage = 12 × 5 = 60 s
kubectl apply -f 10-probes/deployment-lent.yaml
kubectl get pods -l app=lent                          # 1/1 Running, RESTARTS 0
kubectl describe pod -l app=lent | grep -A 10 Events
    # Warning Unhealthy (x6 over 2m7s) Startup probe failed: ... connect: connection refused ; aucun Killing

À retenir :
- La startup probe protège le démarrage d'une appli lente : tant qu'elle n'a pas réussi, liveness et readiness
  sont en pause. Ensuite, elle ne sert plus.
- Budget de démarrage = failureThreshold × periodSeconds (à prévoir plus grand que le temps de démarrage réel).
- On garde ainsi une liveness stricte sans tuer une appli lente à démarrer.
- `grep -A 10 motif` : afficher la ligne trouvée et les 10 suivantes.

### Questions
1. À quoi sert une startup probe ?
   → Protéger le démarrage d'une appli lente : tant qu'elle n'a pas réussi, la liveness et la readiness sont en
     pause ; après son 1er succès, elle ne tourne plus et les deux autres prennent le relais.
2. Appli qui démarre en 90 s max, periodSeconds: 10 : quel failureThreshold ?
   → Plus que 9 (9 × 10 = 90 s, sans marge : un démarrage à 92 s serait tué). Prévoir 1,5 à 2 fois le temps
     habituel, par ex. 15 (150 s) ou 18 (180 s). La marge ne coûte rien : la startup s'arrête dès le 1er succès.
3. Pourquoi pas `initialDelaySeconds: 120` sur la liveness ?
   → Ce délai est fixe : si l'appli démarre en 10 s puis se bloque, la liveness attend quand même 120 s avant de
     détecter le blocage (et ce délai recommence à chaque redémarrage). La startup probe s'adapte : elle laisse
     jusqu'à N secondes, mais rend la main à la liveness dès que l'appli a démarré.

✅ Étape 5 validée : startup probe.
