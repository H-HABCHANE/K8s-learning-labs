# Étape 3 : installer Kubernetes sur Ubuntu

## Les 2 outils
- minikube : crée un petit cluster Kubernetes sur MON PC (dans un conteneur Docker).
  Analogie : un restaurant miniature pour s'entraîner.
- kubectl : la télécommande pour parler au cluster (se prononce « kube-control »).
  Analogie : le carnet de commandes que je donne au gérant.

## Installer kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client

## Installer minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
minikube version

## Démarrer le cluster
minikube start --driver=docker
kubectl get nodes      # doit afficher "minikube   Ready"

## Ce qui a été fait réellement (sans sudo)
- kubectl v1.37.1 et minikube v1.39.0 installés dans ~/.local/bin (vérifiés par checksum sha256).
- minikube start --driver=docker → cluster Kubernetes v1.37.0, nœud "minikube" Ready.
- Au début, le nœud est NotReady quelques secondes : c'est normal, il démarre.
- Éteindre le cluster : minikube stop  |  Le rallumer : minikube start

## Mes réponses
1. kubectl : la télécommande (outil en ligne de commande) qui envoie mes demandes au cluster.
   ⚠️ Ne pas confondre : c'est KUBERNETES qui déploie, surveille et maintient les répliques.
2. Un seul nœud : minikube. Il pilote le cluster (control-plane) ET exécute les applications.
