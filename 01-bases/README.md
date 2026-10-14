# Étape 1 : à quoi sert Kubernetes ?

## Docker
- Met une application dans une boîte : le conteneur.
- Analogie : le cuisinier qui prépare un plat.
- Limite : si le conteneur plante, personne ne le relance.

## Kubernetes (K8s)
- Chef d'orchestre de conteneurs.
- Analogie : le gérant du restaurant qui veille à ce qu'il y ait toujours le bon nombre de plats.
- Idée clé : l'ÉTAT SOUHAITÉ.
  Je décris ce que je veux (« toujours 3 copies de mon site »),
  Kubernetes s'arrange pour que la réalité y corresponde, en permanence.

## Docker vs Kubernetes
| | Docker | Kubernetes |
|---|---|---|
| Rôle | Crée et lance des conteneurs | Organise et surveille beaucoup de conteneurs |
| Conteneur qui plante | Reste arrêté | Recréé automatiquement |
| Plusieurs copies | À la main | « Je veux 5 copies » |
| Échelle | Une machine | Un cluster (une ou plusieurs machines) |

Ce ne sont pas des concurrents : Kubernetes utilise des conteneurs.

## Exercice : le problème de Docker seul
docker run -d --name monsite nginx   # lancer nginx en arrière-plan
docker ps                            # voir les conteneurs en marche
docker kill monsite                  # simuler une panne
docker ps                            # monsite a disparu, personne ne le relance
docker rm monsite                    # nettoyer

## Mes réponses
1. Différence Docker / Kubernetes : Docker gère les conteneurs, Kubernetes les surveille et les relance automatiquement (et aussi : il les lance, les multiplie et les répartit).
2. État souhaité : réponse B, « je veux toujours 3 nginx ». On décrit ce qu on veut en permanence, Kubernetes s en occupe.
