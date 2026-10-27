# Jour 2 — Volumes et PVC

## Étape 8 : volumes et PVC
Le système de fichiers d'un conteneur est éphémère : tout ce qui y est écrit disparaît quand le conteneur
(ou le Pod) est recréé.

| Type     | Durée de vie                                   | Usage                                              |
|----------|------------------------------------------------|----------------------------------------------------|
| emptyDir | vit avec le Pod, effacé quand le Pod est supprimé | partage entre conteneurs d'un Pod, cache temporaire |
| PVC      | indépendant du Pod, survit à sa suppression    | données à garder (base de données, fichiers)       |

Sans volume : fichier `pod-sans-volume.yaml` (Pod sans-volume, busybox:1.36, sleep 3600)
kubectl apply -f 13-volumes/pod-sans-volume.yaml
kubectl wait --for=condition=Ready pod/sans-volume --timeout=60s && kubectl exec sans-volume -- sh -c "echo 'mes donnees importantes' > /tmp/note.txt" && kubectl exec sans-volume -- cat /tmp/note.txt
    # mes donnees importantes
kubectl delete pod sans-volume
kubectl apply -f 13-volumes/pod-sans-volume.yaml && kubectl wait ... && kubectl exec sans-volume -- cat /tmp/note.txt
    # cat: can't open '/tmp/note.txt': No such file or directory → les données ont disparu

Avec PVC :
| Objet        | Rôle                                                  |
|--------------|-------------------------------------------------------|
| PVC          | la demande (« 100 Mi de disque »), dans un namespace  |
| PV           | le disque réel, ressource du cluster (sans namespace) |
| StorageClass | crée les PV automatiquement (provisionnement dynamique) ; ici `standard (default)` |

Fichiers : `pvc.yaml` (PVC data-pvc, ReadWriteOnce, 100Mi)
           `pod-avec-pvc.yaml` (Pod avec-pvc, busybox, volume data-pvc monté sur /data)
    spec:
      volumes:                      # niveau Pod : quel disque
      - name: stockage-data
        persistentVolumeClaim:
          claimName: data-pvc
      containers:
      - ...
        volumeMounts:               # niveau conteneur : où le brancher
        - name: stockage-data       # même nom que dans volumes
          mountPath: /data

kubectl get storageclass                               # standard (default), provisioner k8s.io/minikube-hostpath
kubectl apply -f 13-volumes/pvc.yaml -f 13-volumes/pod-avec-pvc.yaml && kubectl get pvc,pv
    # PVC data-pvc Bound, 100Mi, RWO ; PV pvc-3cb3b71e-... créé automatiquement, RECLAIM POLICY Delete, CLAIM default/data-pvc
kubectl exec avec-pvc -- sh -c "echo 'mes donnees importantes' > /data/note.txt"
kubectl delete pod avec-pvc
kubectl apply -f 13-volumes/pod-avec-pvc.yaml && kubectl wait ... && kubectl exec avec-pvc -- cat /data/note.txt
    # mes donnees importantes → les données ont survécu à la suppression du Pod

À retenir :
- On n'écrit que le PVC ; la StorageClass crée le PV. Bound = PVC lié à un PV.
- RECLAIM POLICY Delete : supprimer le PVC supprime le PV et les données. Retain : le disque est gardé.
- ReadWriteOnce (RWO) : monté en lecture/écriture par un seul nœud à la fois.

Supprimer le PVC :
kubectl delete pvc data-pvc --wait=false && sleep 2 && kubectl get pvc,pv
    # PVC Terminating, PV toujours Bound : protection (finalizer kubernetes.io/pvc-protection) tant qu'un Pod l'utilise
kubectl delete pod avec-pvc && sleep 3 && kubectl get pvc,pv
    # No resources found : PVC supprimé, puis PV supprimé (RECLAIM POLICY Delete) → données perdues

### Questions
1. Fichier dans le conteneur, dans un emptyDir, dans un PVC : que devient-il quand le Pod est supprimé ?
   → Conteneur : perdu à chaque recréation du conteneur. emptyDir : vit avec le Pod, effacé quand le Pod est
     supprimé. PVC : indépendant du Pod, conservé.
2. Rôle du PVC, du PV, de la StorageClass ?
   → J'écris un PVC pour demander du disque ; la StorageClass crée automatiquement le PV, le disque réel, et
     le lie au PVC (Bound).
3. Supprimer le PVC avec RECLAIM POLICY Delete ?
   → Le PV est détruit et les données sont perdues. Tant qu'un Pod utilise le PVC, il reste en Terminating
     (pvc-protection) ; il disparaît dès que le Pod est supprimé. Pour des données précieuses : Retain + sauvegardes.

✅ Étape 8 validée : volumes et PVC.
