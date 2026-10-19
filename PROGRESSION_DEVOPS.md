# Progression — Programme DevOps intensif (10 jours)

Source : `programme_devops_10_jours.pdf` · Objectif : entretien de stage DevOps / Cloud (M2).
Légende : [ ] à faire · [~] en cours · [x] validé (pratique + questions réussies)

## Bilan de départ (19/10/2026)
- Docker, Docker Compose, Docker Hub : terminés (hors de ce dépôt, pas refaits ici).
- Pods : acquis et pratiqués (`02-pods/`) : apply, get, describe, logs, delete ; Pod seul non recréé.
- Deployment : vu une fois (`04-deployments/`) : 3/3, suppression d'un Pod → remplaçant. Plusieurs réponses données
  par prédiction avant d'exécuter → À REPRENDRE, non considéré comme acquis.
- ReplicaSet : seulement cité, jamais manipulé directement → À APPRENDRE.
- Pas encore vus : Services, rolling update, rollback, port-forward.

## Jour 1 — Kubernetes : fondamentaux  [~]
Objectif : déployer une application, la rendre accessible et gérer ses mises à jour.
- [x] Révision Pods, ReplicaSets, Deployments, YAML (partie ReplicaSet)
  - [x] Ex. 1 : observer le ReplicaSet créé par un Deployment (describe, Controlled By)
  - [x] Ex. 2 : écrire et appliquer un ReplicaSet à la main
  - [x] Ex. 3 : ReplicaSet et labels (suppression de l intrus, adoption prouvée)
  - [x] Ex. 4 : pourquoi un Deployment plutôt qu'un ReplicaSet (mise à jour manuelle, 2 versions en même temps)
- [~] Déployer Nginx avec 3 replicas et vérifier les Pods (sans tutoriel) — scale 3→5 vérifié
- [ ] Tester ClusterIP, NodePort et kubectl port-forward
- [x] Réaliser un rolling update puis un rollback (rollout status / history / undo, dérive corrigée)
- [ ] Supprimer un Pod et observer sa recréation
- [ ] Questions d'entretien Kubernetes
Validation : déploiement, exposition et rollback sans tutoriel.

## Jour 2 — Kubernetes : configuration et dépannage  [ ]
- [ ] ConfigMaps, Secrets, variables d'environnement
- [ ] Readiness, liveness, startup probes
- [ ] Requests, limits, namespaces
- [ ] Volumes, PVC et bases d'Ingress
- [ ] Simuler CrashLoopBackOff et ImagePullBackOff ; analyser describe, logs, events
- [ ] Mini-entretien Kubernetes
Validation : identifier une panne et justifier sa correction.

## Jour 3 — GitHub Actions et CI/CD  [ ]
- [ ] Workflow, event, job, step, runner
- [ ] `.github/workflows/ci.yml` pour push et pull request
- [ ] Tests Python et lint
- [ ] Build et push d'image vers Docker Hub sur la branche principale
- [ ] Secrets GitHub, permissions minimales, gestion des échecs
- [ ] Corriger volontairement un pipeline cassé
Validation : un push déclenche les tests et la publication conditionnelle.

## Jour 4 — Linux, Git, Bash et réseau  [ ]
- [ ] chmod, chown, ps, top, systemctl, journalctl
- [ ] DNS, HTTP, TCP/IP, ports, curl, ss, ping
- [ ] SSH, clés, utilisateurs et permissions
- [ ] Bash : variables, conditions, boucles, codes de sortie
- [ ] Git : branches, merge, rebase, conflits
- [ ] Cas pratique : service inaccessible
Validation : retrouver les logs et isoler une panne réseau/service.

## Jour 5 — AWS : IAM, VPC, EC2 et S3  [ ]
- [ ] IAM, rôles, policies, moindre privilège
- [ ] VPC, subnet, route, Security Group, Internet Gateway
- [ ] Créer EC2 et accéder par SSH
- [ ] Déployer une application sur EC2
- [ ] S3, CloudWatch, responsabilité partagée
- [ ] Questions AWS, budget et nettoyage des ressources
Validation : application accessible avec règles réseau maîtrisées.

## Jour 6 — Terraform  [ ]
- [ ] Providers, resources, variables, outputs, fichiers .tf
- [ ] init, fmt, validate, plan, apply
- [ ] EC2 et Security Group avec Terraform
- [ ] State, modules, drift ; ne jamais publier secrets/state
- [ ] terraform destroy et contrôle des coûts
- [ ] Expliquer le plan et les choix d'architecture
Validation : infrastructure créée et détruite proprement.

## Jour 7 — Prometheus, Grafana et diagnostic  [ ]
- [ ] Prometheus : métriques, endpoints, scraping
- [ ] Grafana : datasource, dashboard, panels
- [ ] Alertes, logs, disponibilité
- [ ] Simuler une panne ; corréler logs et métriques
- [ ] Questions d'entretien monitoring
Validation : visualiser une métrique et expliquer une panne.

## Jour 8 — Projet final : API, CI et Kubernetes  [ ]
- [ ] API FastAPI : /health et /metrics
- [ ] Tests unitaires et README initial
- [ ] Pipeline GitHub Actions
- [ ] Déployer sur Minikube avec 3 replicas
- [ ] Service et probes
- [ ] Tests et corrections
Validation : API accessible sur Kubernetes, CI verte.

## Jour 9 — Projet final : AWS, monitoring, documentation  [ ]
- [ ] VM AWS avec Terraform
- [ ] Application sur EC2 avec Docker
- [ ] Monitoring local Prometheus/Grafana
- [ ] README : architecture, prérequis, exécution, captures
- [ ] Détruire les ressources cloud, contrôler les frais
- [ ] Démo de cinq minutes
Validation : dépôt reproductible, documenté et présentable.

## Jour 10 — Entretiens blancs et révisions  [ ]
- [ ] Linux, Git, Docker, Kubernetes : questions rapides
- [ ] GitHub Actions et CI/CD
- [ ] AWS et Terraform
- [ ] Dépannage chronométré
- [ ] Présentation du projet GitHub
- [ ] Présentation personnelle, motivation, questions RH
- [ ] Corriger les lacunes, ajuster le CV
Validation : défendre le projet et résoudre un incident simple.

## Questions d'entretien indispensables (suivi)
| # | Question | État |
|---|---|---|
| 1 | Différence entre Pod, ReplicaSet et Deployment ? | [x] (19/10) |
| 2 | Comment Kubernetes maintient-il trois replicas ? | [ ] |
| 3 | Comment diagnostiquer CrashLoopBackOff ? | [ ] |
| 4 | Comment fonctionne un pipeline GitHub Actions ? | [ ] |
| 5 | Comment sécuriser IAM, SSH et les secrets CI/CD ? | [ ] |
| 6 | Comment déployer sur EC2 ? | [ ] |
| 7 | Différence entre terraform plan et apply ? | [ ] |
| 8 | Comment surveiller une API avec Prometheus et Grafana ? | [ ] |
| 9 | Comment diagnostiquer un service inaccessible ? | [ ] |
| 10 | Comment présenter les compromis de ton architecture ? | [ ] |

## Journal
- 19/10 : bilan de départ, début Jour 1 → Ex. 1 ReplicaSet validé ; Ex. 2 validé ; Ex. 3 validé (intrus supprimé puis adopté) ; Ex. 4 validé. Question n°1 validée. web-rs nettoyé. Ex. 5 scale 3→5 vérifié (4/4 prédictions). Déclaratif vs impératif compris. Ex. 6 rolling update validé. Ex. 7 rollback validé (+ dérive corrigée, last-applied-configuration). Suite : commit Git, puis Services (ClusterIP, NodePort, port-forward).
