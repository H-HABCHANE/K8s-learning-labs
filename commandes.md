# Commandes

## Terminal Linux
```bash
mkdir -p ~/kubernetes                 # crée un dossier (-p : pas d'erreur s'il existe déjà)
cd ~/kubernetes                       # entre dans le dossier
cat fichier.yaml                      # affiche le contenu d'un fichier
nano fichier.yaml                     # édite un fichier (Ctrl+O enregistrer, Entrée, Ctrl+X quitter) — dans un vrai terminal
sed -i 's/ancien/nouveau/' fichier    # remplace un texte dans un fichier sans l'ouvrir
grep mot fichier                      # affiche les lignes qui contiennent "mot"
hostname                              # affiche le nom de la machine
which kubectl minikube                # indique où sont installés ces programmes (rien = absents)
```

## Docker
```bash
docker run -d --name monsite nginx    # lance un conteneur nginx en arrière-plan, nommé monsite
docker ps                             # liste les conteneurs en marche
docker ps -a                          # liste tous les conteneurs, même arrêtés
docker logs monsite                   # affiche ce que le conteneur a écrit
docker kill monsite                   # arrête brutalement le conteneur (simule une panne)
docker rm monsite                     # supprime un conteneur arrêté
```

## Installation
```bash
curl -LO <url>                                        # télécharge un fichier (-L suit les redirections, -O garde son nom)
sudo install -m 0755 kubectl /usr/local/bin/kubectl   # installe un programme dans le système et le rend exécutable
kubectl version --client                              # affiche la version de kubectl
minikube version                                      # affiche la version de minikube
```

## minikube (le cluster)
```bash
minikube start --driver=docker        # crée ou rallume le cluster dans un conteneur Docker
minikube start                        # rallume le cluster (après chaque redémarrage du PC)
minikube status                       # indique si le cluster est allumé (Running) ou éteint (Stopped)
minikube stop                         # éteint le cluster pour libérer la RAM
```

## kubectl : voir
```bash
kubectl get nodes                     # liste les machines du cluster (attendre Ready)
kubectl get pods                      # liste les Pods et leur état (STATUS doit être Running)
kubectl get pods -o wide              # liste les Pods avec leur IP et leur nœud
kubectl get deployments               # liste les Deployments (READY 3/3 = 3 Pods prêts sur 3 voulus)
kubectl get replicasets               # liste les ReplicaSets créés par les Deployments
kubectl describe pod <nom>            # fiche détaillée d'un Pod (IP, image, Events en bas)
kubectl logs <nom-du-pod>             # affiche ce que le conteneur du Pod a écrit
```

## kubectl : agir
```bash
kubectl apply -f fichier.yaml                   # envoie l'état souhaité au cluster (created / configured / unchanged)
kubectl apply --dry-run=client -f fichier.yaml  # vérifie le fichier sans rien créer
kubectl delete pod <nom>                        # supprime un Pod (remplacé seulement s'il appartient à un Deployment)
kubectl delete -f fichier.yaml                  # supprime tout ce que décrit le fichier
kubectl port-forward pod/<nom> 8080:80          # rend le port 80 du Pod accessible sur http://localhost:8080 (Ctrl+C pour arrêter)
```

## Git
```bash
git status                            # montre ce qui a changé depuis le dernier commit
git add -A                            # prépare tous les changements pour le prochain commit
git commit -m "message"               # enregistre une version avec un message
git push                              # envoie les commits sur GitHub (jamais --force)
git log --oneline                     # liste les commits
```
