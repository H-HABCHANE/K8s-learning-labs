# Ma progression Kubernetes

## Plan du cours
- [x] Étape 1 : à quoi sert Kubernetes, différence avec Docker → `01-bases/`
- [x] Étape 2 : le Pod (théorie + fichier `pod-nginx.yaml`) → `02-pods/`
- [x] Étape 3 : installation de kubectl + minikube → `03-installation/`
- [x] Étape 3 bis : lancer mon premier Pod pour de vrai → `02-pods/`
- [x] Étape 4 : le Deployment (relance automatique, répliques) → `04-deployments/`
- [x] ReplicaSet (Jour 1 du programme DevOps) → `05-replicasets/`
- [x] Étape 5 : le Service (accéder à mon application) → `06-services/`, `07-revision-jour1/`
- [ ] Étape 6 : les commandes kubectl essentielles (récapitulatif)

## Où je me suis arrêté
- Suite du cours : voir `PROGRESSION_DEVOPS.md` (programme DevOps 10 jours).
- Étape 3 bis terminée : apply, get pods, describe, logs, delete → le Pod seul ne revient pas.
- Étape 4 : Deployment lancé (3/3), remplacement automatique vérifié (nouveau Pod 4vcxz). Prochaine action : changer le nombre de répliques (replicas: 5) dans `04-deployments/deployment-nginx.yaml` (avec `sed`, car `nano` ne marche pas avec `!`), puis `kubectl apply -f 04-deployments/deployment-nginx.yaml`.

## Points à retenir
- kubectl = la télécommande, Kubernetes = le gérant qui fait le travail (j'avais confondu).
- Si `connection refused` ou `no route to host` → lancer `minikube status` (cluster sûrement éteint).
- Un Pod seul n'est jamais relancé. Pour la relance automatique, il faut un Deployment.
