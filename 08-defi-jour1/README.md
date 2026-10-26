# Jour 1 — Défi final (sans aide)

Mission : déployer, exposer, mettre à jour et revenir en arrière, seul.

0. [x] Cluster propre : supprimer l'appli vitrine (Deployment + 2 Services) ; il ne reste que service/kubernetes
1. [x] Deployment `boutique` : image `httpd:2.4`, 3 replicas (fichier YAML dans ce dossier)
2. [x] Service NodePort `boutique-svc` : nodePort 30090 (fichier YAML dans ce dossier)
3. [x] Site testé depuis le PC avec curl (page « It works! »)
4. [x] Rolling update vers `httpd:2.4-alpine` (méthode déclarative), suivi jusqu'au bout
5. [x] Rollback vers `httpd:2.4`
6. [x] Fichier YAML remis d'accord avec le cluster (aucune dérive)
7. [x] Site re-testé après le rollback

Règles : pas de solution donnée, seulement des indices si je bloque. Je colle mes sorties pour vérification.

## Fichiers
- `deployment-boutique.yaml` : Deployment boutique, 3 replicas, selector/labels app: boutique, httpd:2.4, port 80
- `service-boutique.yaml` : Service NodePort boutique-svc, selector app: boutique, 80 → 80, nodePort 30090

## Déroulé
kubectl delete -f 07-revision-jour1/ && kubectl get all                 # cluster propre
kubectl apply -f 08-defi-jour1/deployment-boutique.yaml && kubectl get all   # 3/3, RS boutique-68f55567b7
kubectl apply -f 08-defi-jour1/service-boutique.yaml
kubectl describe service boutique-svc                                   # NodePort 30090, 3 Endpoints
curl -s http://$(minikube ip):30090                                     # It works!
sed -i 's/httpd:2.4/httpd:2.4-alpine/' 08-defi-jour1/deployment-boutique.yaml   # une seule fois (sinon -alpine-alpine)
kubectl apply -f 08-defi-jour1/deployment-boutique.yaml && kubectl rollout status deployment/boutique
kubectl get rs -o wide && kubectl get pods                              # RS 6dc98d9795 (alpine) 3/3, ancien à 0
kubectl rollout undo deployment/boutique && kubectl rollout status deployment/boutique
kubectl get rs -o wide && kubectl get pods                              # RS 68f55567b7 (httpd:2.4) 3/3
sed -i 's/httpd:2.4-alpine/httpd:2.4/' 08-defi-jour1/deployment-boutique.yaml   # correction de la dérive
kubectl diff -f 08-defi-jour1/deployment-boutique.yaml                  # vide
kubectl apply -f 08-defi-jour1/deployment-boutique.yaml                 # met à jour last-applied-configuration
kubectl get deploy boutique -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'   # httpd:2.4
curl -s http://$(minikube ip):30090                                     # It works!

À retenir :
- Les commandes avec `!` partent de ~/kubernetes : chemin `08-defi-jour1/fichier.yaml`.
- `cat > f <<'EOF'` : la ligne EOF n'est pas du YAML, elle ne doit pas rester dans le fichier.
- `kubectl apply` sur un objet créé en impératif : Warning « missing last-applied-configuration », ajoutée automatiquement.

## Questions
1. Pourquoi `kubectl expose` ne suffisait pas ?
   → Impératif : pas de fichier YAML (pas de source de vérité, rien dans Git) ; sans `nodePort`, le port est
     choisi au hasard (30000–32767) au lieu de 30090.
2. Pourquoi la bascule 1 par 1 avec 3 replicas (2 par 2 avec 5) ?
   → 25 % de 3 = 0,75 : maxSurge arrondi au-dessus = 1, maxUnavailable arrondi en dessous = 0
     (jamais moins de 3 Pods prêts). Avec 5 : 1,25 → maxSurge 2, maxUnavailable 1.
3. Rollback sans corriger le fichier, puis `kubectl apply` par la CI ?
   → httpd:2.4-alpine revient : la version boguée est redéployée sans prévenir personne. D'où : toujours
     remettre le fichier (et Git) sur la bonne version après un rollback.

✅ DÉFI RÉUSSI — JOUR 1 VALIDÉ
