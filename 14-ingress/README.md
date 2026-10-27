# Jour 2 — Ingress

## Étape 9 : Ingress
Ingress = un seul point d'entrée HTTP (port 80/443) pour tout le cluster ; il aiguille les requêtes vers le bon
Service selon le nom de domaine (Host) ou le chemin (path).
Deux morceaux : l'objet Ingress (les règles, en YAML) + l'Ingress Controller (le programme qui les applique,
ici ingress-nginx). Sans controller, un Ingress ne fait rien.

minikube addons enable ingress            # installe le controller ingress-nginx
kubectl get ns                            # nouveau namespace ingress-nginx
kubectl get pods -n ingress-nginx         # controller 1/1 Running ; 2 Pods admission en Completed (tâches finies, normal)

Applis derrière l'Ingress :
kubectl apply -f 07-revision-jour1/deployment-vitrine.yaml -f 07-revision-jour1/service-clusterip.yaml -f 08-defi-jour1/

Fichier : `ingress.yaml` (Ingress mon-ingress, ingressClassName nginx,
          vitrine.local → vitrine-svc:80, boutique.local → boutique-svc:80)
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    spec:
      ingressClassName: nginx
      rules:
      - host: vitrine.local
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vitrine-svc
                port:
                  number: 80

kubectl apply --dry-run=server -f 14-ingress/ingress.yaml        # simulation : created (server dry run)
kubectl apply -f 14-ingress/ingress.yaml && kubectl get ingress  # CLASS nginx, HOSTS vitrine.local,boutique.local, PORTS 80
curl -s -H "Host: vitrine.local"  http://$(minikube ip)/ | grep -i title   # Welcome to nginx!
curl -s -H "Host: boutique.local" http://$(minikube ip)/ | grep -i title   # It works! Apache httpd
curl -s -H "Host: inconnu.local"  http://$(minikube ip)/ | grep -i title   # 404 Not Found (aucune règle)

À retenir :
- Même IP, même port 80 : seul l'en-tête HTTP Host change, et l'Ingress choisit le Service avec.
- Un Ingress peut viser n'importe quel type de Service (en production : ClusterIP derrière l'Ingress).
- Host sans règle → 404 renvoyé par le controller lui-même.
- En vrai : un nom de domaine (DNS) pointe vers l'IP de l'Ingress ; en local, on peut l'ajouter dans /etc/hosts
  ou passer l'en-tête avec curl -H "Host: ...".
- Ingress gère aussi HTTPS (certificats TLS, avec un Secret de type kubernetes.io/tls).

### Questions
1. Objet Ingress / Ingress Controller ? Ingress sans controller ?
   → L'objet Ingress contient les règles d'aiguillage ; le controller (ici ingress-nginx) les lit et reçoit le
     trafic. Sans controller, l'Ingress est accepté (created) mais ne fait rien : personne n'écoute.
2. Même IP, même port, deux sites : comment l'Ingress choisit ?
   → Avec l'en-tête HTTP Host de la requête, comparé aux règles `host:` (le navigateur l'envoie automatiquement).
3. Pourquoi un Ingress plutôt qu'un NodePort par application ?
   → Un seul point d'entrée sur les ports standard 80/443, au lieu d'ouvrir un port aléatoire (30000–32767) par
     appli ; routage par nom de domaine ou par chemin ; HTTPS (certificats TLS) géré à un seul endroit.

✅ Étape 9 validée : Ingress.
