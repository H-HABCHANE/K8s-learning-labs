# Jour 2 — Ressources : requests, limits, QoS

## Étape 6 : requests et limits
|                  | requests (minimum réservé)                 | limits (maximum autorisé)                     |
|------------------|--------------------------------------------|-----------------------------------------------|
| Rôle             | ce que le conteneur est sûr d'avoir        | ce qu'il ne peut jamais dépasser              |
| Qui s'en sert    | le scheduler (choisir un nœud avec de la place) | le kubelet / le noyau Linux                |
| Dépassement CPU  |                                            | ralenti (throttling), pas tué                 |
| Dépassement mémoire |                                         | tué : OOMKilled (Exit Code 137)               |

Unités : CPU `1` = 1 cœur, `500m` = 0,5 cœur, `100m` = 10 % d'un cœur. Mémoire `Mi`, `Gi` (pas `M`).
Syntaxe (dans le conteneur) :
    resources:
      requests: { cpu: "100m", memory: "64Mi" }
      limits:   { cpu: "200m", memory: "128Mi" }

Fichier : `pod-ressources.yaml` (Pod res-demo, nginx:1.27, requests 100m/64Mi, limits 200m/128Mi)
kubectl apply -f 11-ressources/pod-ressources.yaml
kubectl describe pod res-demo | grep -E -A 3 "Limits|Requests|QoS"    # QoS Class: Burstable
kubectl describe node minikube | grep -A 8 "Allocated resources"     # cpu 950m (11 %), memory 284Mi (1 %) réservés

Classes QoS (automatiques) :
- BestEffort : aucune request ni limit → expulsé en premier si le nœud manque de mémoire
- Burstable : requests < limits → ensuite
- Guaranteed : requests = limits (CPU et mémoire) → en dernier

Fichier : `pod-oom.yaml` (Pod oom-demo, polinux/stress, requests 50Mi, limits 100Mi, stress --vm-bytes 250M)
kubectl apply -f 11-ressources/pod-oom.yaml && kubectl get pods
kubectl get pods                                          # CrashLoopBackOff puis OOMKilled, RESTARTS 1 → 3 en 1 min
kubectl describe pod oom-demo | grep -A 5 "Last State"   # Terminated, Reason: OOMKilled, Exit Code: 137

À retenir :
- Le scheduler ne regarde que les requests pour placer un Pod ; la somme des limits peut dépasser 100 % (overcommit).
- OOMKilled = le conteneur a dépassé sa limite mémoire : tué net par le noyau Linux.
- Exit Code 137 = 128 + 9 (signal 9, SIGKILL) : tué de force.
- Le statut OOMKilled ne s'affiche qu'un instant dans get pods : la preuve fiable est dans describe → Last State.

Pod impossible à placer (requests trop grandes) :
printf 'apiVersion: v1\nkind: Pod\nmetadata:\n  name: trop-gros\n...    requests:\n        cpu: "20"\n' | kubectl apply -f -
kubectl get pod trop-gros                               # Pending
kubectl describe pod trop-gros | grep -A 5 Events
    # Warning FailedScheduling  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.

À retenir :
- Pending + FailedScheduling = le scheduler ne trouve aucun nœud qui convient (ici : Insufficient cpu).
  Le conteneur n'est jamais créé (≠ CrashLoopBackOff, où il démarre puis s'arrête).
- Correction : baisser les requests, ou ajouter des nœuds / un nœud plus gros.

### Questions
1. Différence requests / limits ? Qui les utilise ?
   → requests : minimum réservé, utilisé par le scheduler pour choisir un nœud avec assez de place.
     limits : plafond, que le kubelet et le noyau Linux font respecter pendant que le conteneur tourne.
2. Dépassement de la limite CPU ? de la limite mémoire ?
   → CPU : le conteneur est ralenti (throttling), il continue de tourner.
     Mémoire : il est tué, OOMKilled (Exit Code 137).
3. Pod avec requests cpu: "20" sur un nœud de 8 cœurs ?
   → Pending : le scheduler ne trouve aucun nœud avec 20 cœurs libres
     (Events : FailedScheduling, Insufficient cpu).

✅ Étape 6 validée : requests, limits, QoS, OOMKilled, Pending (Insufficient cpu).
