# Commandes

## Terminal Linux
```bash
mkdir -p ~/kubernetes                 # crée un dossier (-p : pas d'erreur s'il existe déjà)
cd ~/kubernetes                       # entre dans le dossier
cat fichier.yaml                      # affiche le contenu d'un fichier
cp source destination                 # copie un fichier
echo "texte" >> fichier               # ajoute une ligne à la fin d'un fichier (> écraserait tout)
cmd1 && cmd2                          # lance cmd2 seulement si cmd1 a réussi
nano fichier.yaml                     # édite un fichier (Ctrl+O enregistrer, Entrée, Ctrl+X quitter) — dans un vrai terminal
sed -i 's/ancien/nouveau/' fichier    # remplace un texte dans un fichier sans l'ouvrir
grep mot fichier                      # affiche les lignes qui contiennent "mot"
cmd | grep mot                        # filtre la sortie d'une commande (|  = envoie la sortie à la suivante)
cmd &                                 # lance la commande en arrière-plan
nohup cmd > fichier.log 2>&1 &        # arrière-plan qui survit à la fin du shell, messages dans un fichier log
echo $?                               # code de sortie de la commande précédente (0 = réussite, autre = échec)
sleep 2                               # attend 2 secondes
$(commande)                           # remplacé par la sortie de la commande (ex. $(minikube ip) → 192.168.49.2)
cat > fichier.yaml <<'EOF'            # écrit un fichier sans éditeur : le contenu suit, puis EOF seul sur la dernière ligne
pgrep -a kubectl                      # liste les processus kubectl en cours (rien + code 1 = aucun)
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
kubectl get all                       # tout ce qui tourne (Pods, Services, Deployments, ReplicaSets)
kubectl get pods                      # liste les Pods et leur état (STATUS doit être Running)
kubectl get pods -o wide              # liste les Pods avec leur IP et leur nœud
kubectl get deployments               # liste les Deployments (READY 3/3 = 3 Pods prêts sur 3 voulus)
kubectl get replicasets               # liste les ReplicaSets (DESIRED / CURRENT / READY)
kubectl get deployments,replicasets,pods   # plusieurs types d'objets en une commande
kubectl get deploy,rs,pods -o wide    # même chose en abrégé, avec images et selectors
kubectl get rs -o wide                # ReplicaSets avec leur image (voir quelle version a les Pods)
kubectl get pods --show-labels        # Pods avec leurs étiquettes (app=..., pod-template-hash=...)
kubectl get pods -l app=web           # seulement les Pods qui ont l'étiquette app=web
kubectl get pods -o custom-columns=NOM:.metadata.name,IMAGE:.spec.containers[0].image   # colonnes choisies
kubectl describe replicaset <nom>     # fiche d'un ReplicaSet (Controlled By, Selector, Events)
kubectl describe deployment <nom>     # fiche d'un Deployment (RollingUpdateStrategy, Events de scaling)
kubectl describe pod <nom>            # fiche détaillée d'un Pod (IP, image, Events en bas)
kubectl logs <nom-du-pod>             # affiche ce que le conteneur du Pod a écrit
kubectl get deploy <nom> -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'
                                      # lit un seul champ (ici la copie du dernier fichier appliqué) ; \. = point dans un nom de clé
```

## kubectl : agir
```bash
kubectl apply -f fichier.yaml                   # envoie l'état souhaité au cluster (created / configured / unchanged)
kubectl apply --dry-run=client -f fichier.yaml  # vérifie le fichier sans rien créer
kubectl apply --dry-run=server -f fichier.yaml  # le cluster vérifie vraiment le fichier (ex. selector ≠ labels) sans rien créer
sed '13s/a/b/' f.yaml | kubectl apply --dry-run=server -f -   # teste une version modifiée sans toucher au fichier (-f - = lire le tuyau)
kubectl diff -f fichier.yaml                    # montre ce que changerait apply (LIVE vs MERGED), ne modifie rien ; vide = d'accord
kubectl delete pod <nom>                        # supprime un Pod (remplacé seulement s'il appartient à un Deployment)
kubectl delete -f fichier.yaml                  # supprime tout ce que décrit le fichier
kubectl delete -f dossier/                      # supprime les objets de tous les YAML du dossier (NotFound = déjà supprimé)
kubectl port-forward pod/<nom> 8080:80          # rend le port 80 du Pod accessible sur http://localhost:8080 (Ctrl+C pour arrêter)
kubectl scale deployment <nom> --replicas=2     # change le nombre de Pods (impératif : le fichier n'est pas modifié → dérive)
kubectl delete replicaset <nom>                 # supprime un ReplicaSet ET ses Pods
```

## kubectl : mises à jour (rollout)
```bash
kubectl rollout status deployment <nom>         # suit une mise à jour en direct jusqu'à la fin
kubectl rollout history deployment <nom>        # historique des révisions (1 révision = 1 ReplicaSet)
kubectl rollout undo deployment <nom>           # revient à la révision précédente (nouvelle révision créée)
kubectl rollout undo deployment <nom> --to-revision=N   # revient à une révision précise
kubectl apply -f f.yaml && kubectl rollout status deployment/<nom>   # lance la mise à jour et la suit
# Après un undo : remettre l'ancienne image dans le fichier, puis kubectl diff (vide) et kubectl apply → plus de dérive
```

## kubectl : Services
```bash
kubectl get services                                                    # liste les Services (TYPE, CLUSTER-IP, PORT(S))
kubectl describe service <nom>                                          # fiche du Service : Selector, Endpoints
kubectl get endpointslices -l kubernetes.io/service-name=<nom>          # Pods derrière le Service
kubectl describe endpointslice -l kubernetes.io/service-name=<nom> | grep Addresses   # toutes leurs IP
kubectl run test --image=busybox --restart=Never -- wget -qO- http://<service>   # client jetable dans le cluster
kubectl run test --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- http://<service>   # même chose, supprimé tout seul (--rm)
kubectl delete pod test                                                 # supprime le client une fois fini
minikube ip                                                             # IP du nœud (pour NodePort)
curl -s http://$(minikube ip):30080                                     # accès NodePort depuis le PC (-s : silencieux)
kubectl port-forward service/<nom> 8080:80 > ~/pf.log 2>&1 &            # tunnel PC:8080 → Service:80 en arrière-plan (lié à UN Pod)
nohup kubectl port-forward pod/<nom> 8080:80 > /tmp/pf.log 2>&1 &      # tunnel vers un Pod précis (s'arrête si ce Pod disparaît)
curl -s -m 3 http://localhost:8080 | grep -i title                      # teste le tunnel (-m 3 : abandon après 3 s)
pkill -f port-forward                                                   # ferme le tunnel
```

## Git
```bash
git status                            # montre ce qui a changé depuis le dernier commit
git add -A                            # prépare tous les changements pour le prochain commit
git commit -m "message"               # enregistre une version avec un message
git push                              # envoie les commits sur GitHub (jamais --force)
git log --oneline                     # liste les commits
```
