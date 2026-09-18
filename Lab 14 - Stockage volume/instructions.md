# LAB 14 - Gestion Volumes (PostgreSQL)

Cet atelier vous guidera à travers les concepts fondamentaux de Gestion de volumes statiques et dynamiques et leur utilisation dans le cas d'une application Statefull.

## Partie 1 : Gestion des Volumes (PV & PVC)

Les conteneurs sont éphémères, leurs données aussi. Pour la persistance, Kubernetes utilise les Volumes.

**PersistentVolume (PV)** : Une ressource de stockage dans le cluster, provisionnée par un administrateur. C'est le "disque dur physique".

**PersistentVolumeClaim (PVC)** : Une demande de stockage faite par un utilisateur (ou une application). C'est la "demande d'un morceau du disque dur".

---

1. Créer un PV statique `pv-static.yaml` :
* Type: `hostPath`, `/tmp`
* Nom : `<votre_prenom>-pv-static`
* Capacité : `1Gi`

<details>
<summary>Correction : Manifeste pv-static.yaml </summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <votre_prenom>-pv-static
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  hostPath:
    path: /tmp
```
</details>

<details>
<summary>Correction : Commande d'application </summary>

```bash
kubectl apply -f pv-static.yaml
```
</details>

---

2. Créer un PVC (`pvc-static.yaml`) qui réclame ce PV :

<details>
<summary>Correction : Manifeste pvc-static.yaml </summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
```
</details>

<details>
<summary>Correction : Commandes de vérification </summary>

```bash
kubectl apply -f pvc-static.yaml
kubectl get pv,pvc
```
</details>

---

3. Utilisation du PVC dans un Pod `pod-pv.yaml` :
* Image : `busybox`
* Command : `[ "sh", "-c", "echo Hello from PV mon nom >> /data/out.txt && sleep 3600" ]`
* Point de montage du PV sur `/data`

<details>
<summary>Correction : Manifeste pod-pv.yaml </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-static
spec:
  containers:
    - name: app
      image: busybox
      command: [ "sh", "-c", "echo Hello from PV mon nom >> /data/out.txt && sleep 3600" ]
      volumeMounts:
        - mountPath: "/data"
          name: vol-pv
  volumes:
    - name: vol-pv
      persistentVolumeClaim:
        claimName: pvc-static
```
</details>

<details>
<summary>Correction : Commande de test </summary>

```bash
kubectl apply -f pod-pv.yaml
kubectl exec pod-static -- cat /data/out.txt
```
</details>

---

## Partie 2 : StorageClass et provisioning dynamique

1. Créer un StorageClass `sc-demo.yaml` :
* Nom du SC `sc-demo-<votre nom>`.
* Provisionner : `longhorn` (ou celui de votre cluster).
* Reclaim Policy : `Delete`. Quel est l'impact ?
* Mode de Binding : `Immediate`. Quel est l'impact ?

<details>
<summary>Correction : Manifeste sc-demo.yaml </summary>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-demo-toto
provisioner: <que mettrez-vous ?> # Exemple pour Longhorn
reclaimPolicy: Delete
volumeBindingMode: Immediate
```
</details>

---

2. Créer un PVC dynamique `pvc-dynamic.yaml` :
* Nom : `pvc-dynamic`
* Request de stockage : `2Gi`

<details>
<summary>Correction : Manifeste pvc-dynamic.yaml </summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: sc-demo-toto
```

</details>

---

3. Créer un Pod utilisant le PVC dynamique `pod-dynamic.yaml` :
* Nom du pod `pod-dynamic`
* image `busybox`
* Le pod monte le volume précédent sur le point de montage `/data`
* Le pod execute la commande `[ "sh", "-c", "echo Persisted with SC >> /data/sc.txt && sleep 3600" ]` 

<details>
<summary>Correction : Manifeste pod-dynamic.yaml </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dynamic
spec:
  containers:
    - name: app
      image: busybox
      command: [ "sh", "-c", "echo Persisted with SC >> /data/sc.txt && sleep 3600" ]
      volumeMounts:
        - mountPath: "/data"
          name: vol-sc
  volumes:
    - name: vol-sc
      persistentVolumeClaim:
        claimName: pvc-dynamic
```
</details>

---

## Partie 3 : StatefulSets et persistence de la donnée

Pour les applications avec état comme les bases de données, un **StatefulSet** est indispensable. Il garantit que chaque Pod conserve son volume même s'il est redémarré ou déplacé sur un autre nœud.

**Objectif :** Déployer **PostgreSQL** avec un StatefulSet pour garantir que le compteur de visites ne revienne jamais à zéro.

1. Créez le manifeste `postgres-statefulset.yaml` :
* Nom : `postgres-statefulset`
* Image : `postgres:16-alpine`
* Port : `5432`
* Variable d'env obligatoire : `POSTGRES_PASSWORD=postgres`
* volumeClaimTemplates : `1GB`, `rwo`
* Chemin de données Postgres : `/var/lib/postgresql/data`

<details>
<summary>Correction : Manifeste postgres-statefulset.yaml </summary>

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "postgres"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: postgres
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "sc-demo-toto" # Utilisez votre SC créé précédemment
      resources:
        requests:
          storage: 1Gi
```
</details>

---

2. Déployez et testez la persistance :

<details>
<summary>Correction : Action de nettoyage </summary>

```bash
# Supprimez l'ancien déploiement sans état (s'il existe)
kubectl delete deployment postgres-deployment
```
</details>

<details>
<summary>Correction : Action de déploiement </summary>

```bash
# Appliquez le StatefulSet
kubectl apply -f postgres-statefulset.yaml

# Attendez que le pod postgres-statefulset-0 soit prêt
kubectl get pods -w
```
</details>

<details>
<summary>Correction : Test de persistance </summary>

1. Accédez à votre Webapp et incrémentez le compteur (ex: jusqu'à 10).
2. Supprimez le Pod PostgreSQL : `kubectl delete pod postgres-statefulset-0`.
3. Attendez que Kubernetes recrée le Pod automatiquement.
4. Rafraîchissez la page Web.
   
**Constat :** Le compteur reprend à 11. Contrairement à un Deployment classique sans volume persistant, le StatefulSet a rattaché le même disque au nouveau Pod, préservant ainsi la base de données.
</details>

---

## Partie 4 : Limitations du StatefulSet et Introduction aux Opérateurs

Jusqu'à présent, nous avions 1 seul réplica de PostgreSQL. Pour assurer la haute disponibilité en production et tolérer la panne d'un nœud, nous en voudrions plusieurs. Voyons ce qui se passe si nous utilisons simplement les fonctionnalités natives de Kubernetes pour cela.

### 1. Mise à l'échelle du StatefulSet

Modifions notre infrastructure pour passer de 1 à 3 bases de données PostgreSQL, ainsi que 3 réplicas de notre application Web.

Exécutez les commandes suivantes :

```bash
kubectl scale statefulset postgres-statefulset --replicas=3
kubectl scale deployment webapp-deployment --replicas=3
```

Attendez que les nouveaux pods PostgreSQL (`postgres-statefulset-1` et `postgres-statefulset-2`) soient à l'état `Running`.

```bash
kubectl get pods -l app=postgres --watch
```

### 2. Test et Constat (La limitation)

Retournez sur votre navigateur web (sur l'IP du LoadBalancer ou le NodePort de l'application web) et rafraîchissez la page frénétiquement plusieurs fois.

**Que constatez-vous au niveau du compteur de visites ?**

<details>
<summary>Réponse et Explication</summary>

**Constat :**
Le compteur devient totalement incohérent. Il peut afficher `15` (votre ancien record), puis au rafraîchissement suivant chuter à `1`, remonter à `16`, puis tomber encore à `1` ou `2`.

**Explication :**
Le Service Kubernetes `postgres` effectue un équilibrage de charge aléatoire (Round-Robin) entre les 3 pods (`postgres-statefulset-0`, `postgres-statefulset-1`, `postgres-statefulset-2`).
Cependant, ces 3 instances PostgreSQL sont totalement indépendantes (architecture "Shared Nothing"). Elles ne se connaissent pas et ne synchronisent pas leurs données.

* La requête A arrive sur `postgres-statefulset-0` : la base contient déjà les anciennes données, il incrémente son compteur local (ex: passe à 16).
* La requête B arrive sur `postgres-statefulset-1` : grâce au `volumeClaimTemplates`, Kubernetes lui a attaché un disque (PVC) **totalement vierge**. Il crée sa propre table locale et incrémente son compteur qui commence à 1.
* La requête C arrive sur `postgres-statefulset-2` : lui aussi a son propre disque vierge, son compteur est à 1.

Le `StatefulSet` a correctement accompli son travail d'infrastructure : il a créé 3 pods avec 3 identités réseau stables et 3 disques persistants distincts. **Mais il n'a pas configuré PostgreSQL.**

Le `StatefulSet` ne connaît pas le métier de la base de données. Il ne sait pas configurer la "Streaming Replication" interne à PostgreSQL, ni dire quel nœud a le droit d'écrire et quels nœuds doivent se contenter de lire.

</details>

-----

### 3. La Solution : Les Opérateurs Kubernetes

Pour avoir un véritable cluster PostgreSQL de production (avec un **Primaire** qui reçoit les écritures et des **Répliques** qui synchronisent les données en temps réel), nous avons besoin d'une intelligence opérationnelle capable de configurer le moteur de la base de données. 

C'est le rôle de l'**Opérateur**. 

Un Opérateur comme **CloudNativePG (CNPG)** va non seulement créer les pods et les volumes (comme le fait le StatefulSet), mais il va surtout injecter la configuration PostgreSQL nécessaire pour lier ces instances entre elles, gérer le basculement (failover) automatique en cas de crash du nœud Primaire, et aiguiller le trafic proprement.