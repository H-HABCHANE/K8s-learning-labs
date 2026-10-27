# Progression — Kubernetes avancé

Suite des Jours 1 et 2 (fondamentaux, configuration, dépannage : voir `PROGRESSION_DEVOPS.md`).
Objectif : passer d'un niveau intermédiaire à avancé, puis préparer la certification CKAD.
Méthode : explication → prédiction → test → questions → défi sans aide à la fin de chaque bloc.
Légende : [ ] à faire · [~] en cours · [x] validé

## Bloc 1 — Autres contrôleurs  [ ]
- [ ] StatefulSet (identité stable, un PVC par Pod, bases de données)
- [ ] DaemonSet (un Pod par nœud : logs, monitoring)
- [ ] Job et CronJob (tâches ponctuelles et planifiées)
- [ ] Défi du bloc

## Bloc 2 — Mise à l'échelle  [ ]
- [ ] metrics-server et `kubectl top`
- [ ] HPA (autoscaling selon le CPU)
- [ ] Défi du bloc

## Bloc 3 — Sécurité  [ ]
- [ ] ServiceAccount, Role, RoleBinding (RBAC), `kubectl auth can-i`
- [ ] NetworkPolicy (pare-feu entre Pods)
- [ ] securityContext (non-root, lecture seule)
- [ ] Défi du bloc

## Bloc 4 — Packaging  [ ]
- [ ] Kustomize (base + overlays dev/prod)
- [ ] Helm (charts, values, install, upgrade, rollback)
- [ ] Défi du bloc

## Bloc 5 — Scheduling avancé  [ ]
- [ ] Cluster minikube multi-nœuds
- [ ] nodeSelector, affinity et anti-affinity
- [ ] Taints et tolerations
- [ ] PodDisruptionBudget, drain et cordon
- [ ] Défi du bloc

## Bloc 6 — Patterns de Pods  [ ]
- [ ] Init containers
- [ ] Sidecars (plusieurs conteneurs dans un Pod, volume partagé emptyDir)
- [ ] ConfigMap et Secret montés en fichiers
- [ ] Défi du bloc

## Bloc 7 — Dépannage avancé et CKAD  [ ]
- [ ] Nœud NotReady, DNS, réseau entre Pods
- [ ] Exercices chronométrés style CKAD (impératif rapide : `kubectl create ... --dry-run=client -o yaml`)
- [ ] Examen blanc

## Journal
- 27/10/2026 : Jour 2 validé. Décision : ce dépôt continue sur Kubernetes avancé ; le CI/CD et la suite du
  programme DevOps (Jours 3 à 10) continuent dans un autre dossier. REPRISE : Bloc 1, StatefulSet.
