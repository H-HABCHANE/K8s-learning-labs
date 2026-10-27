# Jour 2 — Défi final (sans aide)

Mission : l'application « magasin » (`magasin.yaml` : ConfigMap + Deployment + Service NodePort 30100) a été
déployée, mais le site ne répond pas. Plusieurs pannes sont cachées.

Pour chaque panne :
1. l'identifier (get, describe, logs, events...) ;
2. la justifier avec la preuve (message exact) ;
3. la corriger de façon déclarative (dans le fichier, puis apply), sauf si la bonne pratique dit autrement.

Validation : `curl -s http://$(minikube ip):30100 | grep -i title` affiche « Welcome to nginx! »,
les 2 Pods sont en 1/1 Running sans redémarrage, et `kubectl diff -f 16-defi-jour2/magasin.yaml` est vide.

Règles : pas de solution donnée, seulement des indices si je les demande.

## Mes pannes trouvées
1. Image inexistante
   Preuve : get pods → ErrImagePull / ImagePullBackOff ; get events --sort-by=.lastTimestamp →
            Failed to pull image "nginx:1.27-alpnie": ... not found
   Cause : faute de frappe dans le tag (alpnie au lieu de alpine).
   Correction : sed -i 's/1.27-alpnie/1.27-alpine/' 16-defi-jour2/magasin.yaml puis apply.
2. Secret manquant
   Preuve : CreateContainerConfigError ; describe pod → Error: secret "magasin-secret" not found
   Cause : le Pod lit la clé API_KEY du Secret magasin-secret, qui n'existe pas.
   Correction : kubectl create secret generic magasin-secret --from-literal=API_KEY=demo-123
                (en ligne de commande, jamais dans le YAML publié sur GitHub) ; le kubelet réessaie tout seul.
3. Readiness sur le mauvais port
   Preuve : Running mais 0/1, anciens Pods non remplacés ; describe pod | tail →
            Readiness probe failed: Get "http://...:8080/": connect: connection refused
   Cause : la readiness teste le port 8080, nginx écoute sur 80 → jamais prêt → rolling update bloqué.
   Correction : sed -i 's/port: 8080/port: 80/' 16-defi-jour2/magasin.yaml puis apply.
4. Selector du Service faux
   Preuve : curl ne renvoie rien alors que les Pods sont 1/1 ; describe svc → Selector: app=magasin-web,
            Endpoints vide
   Cause : le Service cherche app=magasin-web, les Pods portent app=magasin.
   Correction : sed -i 's/app: magasin-web/app: magasin/' 16-defi-jour2/magasin.yaml puis apply.

Validation :
kubectl apply -f 16-defi-jour2/magasin.yaml && kubectl describe svc magasin-svc | grep Endpoints && curl -s http://$(minikube ip):30100 | grep -i title && kubectl diff -f 16-defi-jour2/magasin.yaml; echo "diff: code $?"
    # Endpoints: 10.244.0.39:80,10.244.0.38:80 ; <title>Welcome to nginx!</title> ; diff: code 0

À retenir :
- Les pannes se cachent les unes derrière les autres : on en corrige une, la suivante apparaît
  (image → configuration → readiness → Service).
- Avec plusieurs Pods, `describe pod -l` n'affiche pas les Events : décrire un Pod par son nom,
  ou `kubectl get events --sort-by=.lastTimestamp | grep <nom>`.
- `grep -A 8` peut couper trop tôt : `| tail` pour voir les derniers Events.
- Pods 1/1 mais site muet → regarder le Service : Selector et Endpoints.

✅ DÉFI RÉUSSI — 4 pannes trouvées, justifiées et corrigées.

### Questions
1. Pourquoi corriger les pannes 1, 3, 4 dans le fichier, mais créer le Secret en ligne de commande ?
   → Le fichier est la source de vérité (sinon dérive) ; mais un Secret ne doit jamais aller sur GitHub
     (en YAML il n'est qu'en base64, décodable en une commande).
2. « Mes Pods vont bien mais mon site est muet » : premières vérifications ?
   → Le Selector du Service et ses Endpoints : `kubectl describe svc <nom>`. Endpoints vides = le selector ne
     correspond pas aux labels des Pods (`kubectl get pods --show-labels`) ; vérifier aussi port / targetPort.

✅ JOUR 2 VALIDÉ
