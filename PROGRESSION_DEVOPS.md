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

## Jour 1 — Kubernetes : fondamentaux  [x] (validé le 26/10/2026)
Objectif : déployer une application, la rendre accessible et gérer ses mises à jour.
- [x] Révision Pods, ReplicaSets, Deployments, YAML (partie ReplicaSet)
  - [x] Ex. 1 : observer le ReplicaSet créé par un Deployment (describe, Controlled By)
  - [x] Ex. 2 : écrire et appliquer un ReplicaSet à la main
  - [x] Ex. 3 : ReplicaSet et labels (suppression de l intrus, adoption prouvée)
  - [x] Ex. 4 : pourquoi un Deployment plutôt qu'un ReplicaSet (mise à jour manuelle, 2 versions en même temps)
- [x] Déployer Nginx avec 3 replicas et vérifier les Pods (sans tutoriel) — scale 3→5 vérifié ; défi boutique 3/3 sans aide
- [x] Tester ClusterIP, NodePort et kubectl port-forward (Ex. 8–10, 06-services/)
- [x] Réaliser un rolling update puis un rollback (rollout status / history / undo, dérive corrigée)
- [x] Supprimer un Pod et observer sa recréation (Endpoints mis à jour, port-forward cassé)
- [x] Questions d'entretien Kubernetes (5 questions + 2 cas concrets, 06-services/README.md)
Validation : déploiement, exposition et rollback sans tutoriel. ✅ Défi final réussi (08-defi-jour1/).

## Jour 2 — Kubernetes : configuration et dépannage  [x] (validé le 27/10/2026)
- [x] ConfigMaps, Secrets, variables d'environnement (09-config/)
- [x] Readiness, liveness, startup probes (10-probes/)
- [x] Requests, limits, namespaces (11-ressources/, 12-namespaces/)
- [x] Volumes, PVC et bases d'Ingress (13-volumes/, 14-ingress/)
- [x] Simuler CrashLoopBackOff et ImagePullBackOff ; analyser describe, logs, events (15-pannes/)
- [x] Mini-entretien Kubernetes (15-pannes/README.md)
Validation : identifier une panne et justifier sa correction. ✅ Défi réussi (16-defi-jour2/ : 4 pannes).

> Les Jours 3 à 10 continuent dans un autre dossier (CI/CD et suite DevOps). Dans ce dépôt, la suite
> Kubernetes avancé est suivie dans `PROGRESSION_K8S_AVANCE.md`.

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
| 2 | Comment Kubernetes maintient-il trois replicas ? | [x] (20/10) |
| 3 | Comment diagnostiquer CrashLoopBackOff ? | [x] (27/10) |
| 4 | Comment fonctionne un pipeline GitHub Actions ? | [ ] |
| 5 | Comment sécuriser IAM, SSH et les secrets CI/CD ? | [ ] |
| 6 | Comment déployer sur EC2 ? | [ ] |
| 7 | Différence entre terraform plan et apply ? | [ ] |
| 8 | Comment surveiller une API avec Prometheus et Grafana ? | [ ] |
| 9 | Comment diagnostiquer un service inaccessible ? | [ ] |
| 10 | Comment présenter les compromis de ton architecture ? | [ ] |

## Journal
- 19/10 : bilan de départ, début Jour 1 → Ex. 1 ReplicaSet validé ; Ex. 2 validé ; Ex. 3 validé (intrus supprimé puis adopté) ; Ex. 4 validé. Question n°1 validée. web-rs nettoyé. Ex. 5 scale 3→5 vérifié (4/4 prédictions). Déclaratif vs impératif compris. Ex. 6 rolling update validé. Ex. 7 rollback validé (+ dérive corrigée, last-applied-configuration). Commit 4aae261 poussé. Révision : notes.md, commandes.md, revision/resume-kubernetes.pdf mis à jour. REPRISE : Ex. 8 Service ClusterIP (06-services/service-clusterip.yaml, monsite-svc, selector app: monsite, port 80 → targetPort 80).
- 20/10 : Ex. 8 ClusterIP monsite-svc validé (5 Endpoints, test busybox + DNS). Ex. 9 NodePort 30080 validé (curl depuis le PC). Ex. 10 port-forward + panne d'un Pod validé (Endpoints .19→.25, port-forward lié à un seul Pod). Questions d'entretien Jour 1 validées (Q1–Q5 + 2 cas concrets). REPRISE 21/10 : défi final sans aide (07-defi-jour1/ : Deployment boutique httpd:2.4 ×3, Service NodePort boutique-svc 30090, rolling update httpd:2.4-alpine, rollback, fichier remis d'accord, site testé).
- 24/10 : décision de REFAIRE le Jour 1 complet de zéro avant le défi final (dossier 07-revision-jour1/, appli « vitrine », solutions expliquées commande par commande, cluster nettoyé). Défi final ensuite dans 08-defi-jour1/, sans aide.
- 25/10 : révision étape 0 validée (minikube redémarré, monsite + 2 Services supprimés, il ne reste que service/kubernetes ; 3 questions : etcd/réconciliation, cascade ownerReferences, Service kubernetes). Étape 1 validée (Pod seul vitrine : pas de Controlled By, non recréé ; scheduler vs kubelet ; IP de Pod temporaire). Étape 2 validée (Deployment vitrine ×3, Pod supprimé remplacé par un nouveau, selector ≠ labels refusé, testé avec --dry-run=server). Étape 3 validée (Controlled By sur Pod et RS, pod-template-hash, -l). Étape 4 validée (scale 3→5 déclaratif, kubectl diff, dérive, hash = template seulement). Étape 5 validée (rolling update 1.27 → 1.28, rollout status, 2 ReplicaSets, maxSurge/maxUnavailable, image inexistante). Étape 6 validée (rollout history/undo, révision 1 → 3, dérive corrigée : fichier + last-applied-configuration vérifiée en jsonpath). Étape 7 validée (ClusterIP vitrine-svc, Endpoints, test busybox par le nom DNS). Étape 8 validée (NodePort vitrine-nodeport 30080, curl depuis le PC, limites du NodePort en prod). Étape 9 validée (port-forward cassé par la suppression du Pod, NodePort toujours OK, Endpoints .21 → .27). Étape 10 validée (mini-entretien 5 questions). RÉVISION JOUR 1 TERMINÉE. Défi final commencé (08-defi-jour1/README.md). Défi étape 0 OK (delete -f 07-revision-jour1/). commandes.md complété avec les commandes de la révision. Pause. 26/10 : défi étape 1 OK (Deployment boutique httpd:2.4 3/3, RS boutique-68f55567b7). Défi étape 2 OK (service-boutique.yaml, NodePort 30090, 3 Endpoints). Défi étape 3 OK (It works! sur :30090). Défi étape 4 OK (rolling update → RS 6dc98d9795 httpd:2.4-alpine 3/3, ancien 68f55567b7 à 0). Défi étape 5 OK (rollback → RS 68f55567b7 httpd:2.4 3/3). Défi étape 6 OK (fichier remis en httpd:2.4, diff vide, apply, last-applied vérifiée). Défi étape 7 OK (It works! après rollback). 3 questions de fin réussies. ✅ JOUR 1 VALIDÉ. REPRISE : Jour 2.
- 26/10 (suite) : Jour 2 commencé. Cluster nettoyé (boutique supprimé). Étape 1 variables d'environnement validée (Pod env-demo, logs, exec, Pod immuable → delete + apply, kubectl wait). Étape 2 validée (ConfigMap app-config, envFrom, CreateContainerConfigError réparée, variables figées → rollout restart). Étape 3 validée (Secret db-secret, base64 décodé, secretKeyRef, RBAC/etcd/Vault). Étape 4 validée (readiness 404 → rolling update bloqué, Pod hors Endpoints ; liveness 404 → tous les anciens remplacés, CrashLoopBackOff, Killing/BackOff ; réparation déclarative, RS réutilisé). Étape 5 validée (deployment-lent : sleep 30 + liveness stricte → CrashLoopBackOff ; startupProbe 12 × 5 s → 1/1 sans restart ; marge du budget). Fichier deployment-probes.yaml écrasé par erreur (édition nano du mauvais fichier) puis reconstruit depuis last-applied-configuration. Étape 6 validée (res-demo Burstable, Allocated resources, oom-demo OOMKilled Exit 137, trop-gros Pending Insufficient cpu). REPRISE : étape 7, namespaces.
- 27/10 : étape 7 validée (namespaces système, dev/prod en YAML avec ---, même Deployment dans 2 namespaces, -n / -A, delete namespace en cascade, DNS vitrine-svc.prod). Étape 8 validée (sans volume : fichier perdu ; PVC data-pvc Bound, PV dynamique, fichier conservé ; PVC Terminating tant qu'utilisé ; Delete → PV supprimé). Étape 9 validée (addon ingress, ingress-nginx, Ingress vitrine.local/boutique.local, curl -H Host, 404 host inconnu). Étape 10 validée (ImagePullBackOff nginx:9.99 not found → set image ; panne-app Exit 1, logs vs logs --previous). Étape 11 validée (mini-entretien 5 questions). Défi Jour 2 réussi sans aide : 4 pannes (image alpnie, Secret manquant, readiness port 8080, selector magasin-web), diff vide, curl OK. ✅ JOUR 2 VALIDÉ. Suite : Jours 3 à 10 dans un autre dossier ; ici, Kubernetes avancé (PROGRESSION_K8S_AVANCE.md).
