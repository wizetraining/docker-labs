# LAB 11.2 - Récap (Pod, ReplicaSet, Deployment) et pratique des Services : Déploiement d'une Application Web avec Redis

Cet atelier vous guidera à travers les concepts fondamentaux de Kubernetes, de la création d'un simple Pod au déploiement d'une application multi-services, persistante et hautement disponible.

**Prérequis :**
* Un cluster Kubernetes fonctionnel
* kubectl configuré pour communiquer avec votre cluster.

L'image Docker ` mpakoupete/webapp-count:v1` est disponible sur Docker Hub. 
* Si vous rencontrez des limitations, utilisez `harbor.mpakoupete.com/docker/webapp-count:v1` ou `harbor.mpakoupete.com/docker/redis:6-alpine`, ou `harbor.mpakoupete.com/docker/webapp-count:v2`

Le code de l'application est ci-dessous:

<details><summary>Code Webapp-Count v1</summary>

```python
# Contenu de app.py
import time
import redis
import socket
from flask import Flask

app = Flask(__name__)
# On se connecte au service 'redis' sur le port par défaut.
# Docker Compose va s'assurer que l'hostname 'redis' pointe vers le bon conteneur !
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    """
    Tente de se connecter à Redis et d'incrémenter le compteur.
    Lève une ConnectionError si la connexion échoue après plusieurs tentatives.
    """
    retries = 5
    while True:
        try:
            # Tente de vérifier la connexion avant d'incrémenter
            cache.ping()
            # On incrémente de 1 la valeur de la clé 'hits' et on la retourne
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    db_status_message = ''
    count = 0
    try:
        count = get_hit_count()
        # Si get_hit_count réussit, la connexion est établie
        db_status_message = '<strong style="color:green;">Connecté à la base de donnée</strong>'
    except redis.exceptions.ConnectionError:
        count = 'N/A'
        db_status_message = '<span style="color:red;">Echec de connexion à la base de donnée</span>'

    # Récupérer le nom d'hôte du conteneur
    hostname = socket.gethostname()

    # Construire le HTML de la réponse
    html_response = f"""
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Compteur de Visites</title>
        <style>
            body {{ font-family: sans-serif; text-align: center; margin-top: 50px; }}
            .container {{ padding: 20px; border: 1px solid #ccc; border-radius: 8px; display: inline-block; }}
            .hostname {{ font-size: 0.9em; color: #555; margin-top: 20px; }}
            .db-status {{ margin-top: 15px; font-size: 1.1em; }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Bonjour !</h1>
            <p>Vous êtes le visiteur numéro <strong>{count}</strong>.</p>
            <div class="hostname">Vous êtes sur le conteneur : {hostname}</div>
            <div class="db-status">{db_status_message}</div>
        </div>
    </body>
    </html>
    """
    return html_response

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

</details>

## Partie 1 : Les Pods
Un Pod est la plus petite unité de déploiement dans Kubernetes. Il représente un ou plusieurs conteneurs s'exécutant ensemble sur le même nœud, partageant le même réseau et le même stockage.

**Objectif** : Déployer un Pod unique exécutant notre application web.

1. Créez un manifeste `pod-webapp.yaml`  pour créer un Pod :
* Dans votre namespace à vous
* nom du Pod `webapp-pod`
* nom du conteneur `webapp-container`
* image ` mpakoupete/webapp-count:v1`
* Port du conteneur `5000`

<details>
<summary>Correction YAML - pod-webapp.yaml </summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    app: webapp
spec:
  containers:
  - name: webapp-container
    image:  mpakoupete/webapp-count:v1
    ports:
    - containerPort: 5000
```

</details>

2. Appliquez le manifeste et vérifiez le Pod :

<details>
<summary>Correction </summary>

```
# Appliquer le manifeste pour créer le pod
kubectl apply -f pod-webapp.yaml

# Lister les pods pour vérifier son statut (doit être "Running")
kubectl get pods

# Pour tester l'application, on peut utiliser le port-forwarding
# Ouvrez un autre terminal et lancez cette commande :
kubectl port-forward pod/webapp-pod 8080:5000

# Ouvrez maintenant votre navigateur à l'adresse http://localhost:8080
# Vous devriez voir "Echec de connexion à la base de donnée" car Redis n'est pas là.
```

</details>

## Partie 2 : Les ReplicaSets

Un ReplicaSet garantit qu'un nombre spécifié de répliques de Pods (copies identiques) s'exécutent en permanence. Si un Pod tombe, le ReplicaSet en crée un autre pour le remplacer.

**Objectif** : Assurer la haute disponibilité de notre application avec 3 répliques.

1. Créez le manifeste `replicaset-webapp.yaml` pour un ReplicaSet :
* Nom: `webapp-replicaset`
* Replicas : 3
* image ` mpakoupete/webapp-count:v1`

<details>
<summary>Correction YAML - replicaset-webapp.yaml </summary>

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp-rs
  template: # Le modèle du Pod à créer
    metadata:
      labels:
        app: webapp-rs
    spec:
      containers:
      - name: webapp-container
        image:  mpakoupete/webapp-count:v1
        ports:
        - containerPort: 5000
``` 

</details>

2. Appliquez et testez la résilience :

<details>
<summary>Correction Correction </summary>

```
# Appliquer le manifeste
kubectl apply -f replicaset-webapp.yaml

# Lister les pods (vous devriez en voir 3)
kubectl get pods -l app=webapp-rs

# Supprimons un pod pour voir le ReplicaSet en action
# Notez le nom d'un des pods et remplacez-le ci-dessous
kubectl delete pod <nom-du-pod>

# Relistez les pods immédiatement. Vous verrez qu'un nouveau pod est créé !
kubectl get pods -l app=webapp-rs
```

</details>

## Partie 3 : Deployments

Le Deployment est une abstraction au-dessus des ReplicaSets. Il permet de gérer les mises à jour (rolling updates) et les retours en arrière (rollbacks) de manière déclarative et contrôlée. C'est l'objet que l'on utilise le plus souvent pour déployer des applications stateless.

**Objectif** : Déployer notre application via un Deployment, la scaler, la mettre à jour vers la v2 et revenir en arrière.

1. Créez le manifeste `deployment-webapp.yaml` pour un Deployment :
* Nom: `webapp-deployment`
* Replicas : 2
* image ` mpakoupete/webapp-count:v1`

<details>
<summary>Correction YAML - deployment-webapp.yaml </summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-deploy
  template:
    metadata:
      labels:
        app: webapp-deploy
    spec:
      containers:
      - name: webapp-container
        image:  mpakoupete/webapp-count:v1 # On commence avec la v1
        ports:
        - containerPort: 5000
```

</details>

2. Déployez, scalez, mettez à jour et revenez en arrière :
* Faites un scale du nombere de replicas à 4
* Faite une montée de version vers l'application `v2` (image `mpakoupete/webapp-count:v2`)
* Vérifier le statut de la mise à jour
* Revenir à la version précédente 

<details>
<summary>Correction </summary>

```
# 1. Appliquer le manifeste
kubectl apply -f deployment-webapp.yaml

# 2. Scaler le nombre de répliques de 2 à 4
kubectl scale deployment webapp-deployment --replicas=4
kubectl get pods # Vous devriez en voir 4

# 3. Mettre à jour l'image vers la v2 (Rolling Update)
# Kubernetes va remplacer les pods un par un, sans interruption de service.
kubectl set image deployment/webapp-deployment webapp-container=mpakoupete/webapp-count:v2

# Vérifier le statut de la mise à jour
kubectl rollout status deployment/webapp-deployment

# 4. Vérifier l'historique des révisions
kubectl rollout history deployment/webapp-deployment

# 5. Revenir à la version précédente (Rollback)
kubectl rollout undo deployment/webapp-deployment

# Vérifier que les pods utilisent à nouveau l'image v1
kubectl describe deployment webapp-deployment | grep Image
```

</details>

## Partie 4 : Services

Un Service expose un ensemble de Pods (généralement gérés par un Deployment) sous une seule adresse IP et un seul nom DNS stables à l'intérieur du cluster. Il permet aux différentes parties de votre application de communiquer entre elles.

**Objectif** : Déployer Redis et exposer notre webapp pour qu'elle puisse s'y connecter et être accessible.

1. Créez un service de type `NodePort` qui expose le déployement précédent
* Nom du service : `webapp-service`
* Port à exposé `80`
* utilisez `describe` pour décrire le service et le visualiser. Voyez-vous les endpoint ?

<details>
<summary>Correction YAML - Fichiers pour Service Webapp </summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp-deploy # Doit correspondre aux labels des pods du déploiement webapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
    # nodePort: 30007 # Optionnel, K8s en choisira un si non spécifié
```

</details>

2. Créez un service de type `LoadBalancer` qui expose le déployement précédent
* Nom du service : `webapp-service-lb`
* Port à exposé `80`
* utilisez `describe` pour décrire le service et le visualiser. Voyez-vous les endpoint ?

<details>
<summary>Correction YAML - Fichiers pour Service Webapp LoadBalancer </summary>

Pas de correction ;)

</details>


3. Créez les manifestes pour Redis (redis-deployment.yaml, redis-service.yaml) :
* Un deployment Redis , `replica` : 1 , image : `redis:6-alpine`
* Un service qui expose Redis, le port du conteneur `6379`
* Quel nom lui avez vous donné ? Pourquoi ?
* De quel type de serice avons nous besoin ? Pourquoi ?
tentez de vous connecter au service Web de deux manières : viab LoadBalancer et NodePort

<details>
<summary>Correction YAML - Fichiers Deploiement et Service pour Redis </summary>

`redis-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6-alpine
        ports:
        - containerPort: 6379
```

`redis-service.yaml`

Note : Le nom du service `redis` correspond au host que `app.py` essaie de contacter.
Service de type `ClusterIp` car le client est en interne. Pas besoin de l'exposer à l'extérieur

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
```

</details>

3. Appliquez les manifestes et testez la connexion. Vous devriez voir "Connecté à la base de donnée" 

<details>
<summary>Correction </summary>

```
# Appliquer les manifestes de Redis
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml

# Appliquer le service pour la webapp
kubectl apply -f webapp-service.yaml

# La webapp devrait maintenant pouvoir se connecter à Redis.
# Redémarrez les pods de la webapp pour qu'ils prennent en compte le service Redis
kubectl rollout restart deployment webapp-deployment

# Trouvez l'URL pour accéder à votre application
# Si vous utilisez Minikube :
minikube service webapp-service

# Sinon, trouvez le NodePort et l'IP de votre noeud :
kubectl get service webapp-service # Cherchez le port mappé (ex: 3xxxx)
kubectl get nodes -o wide # Cherchez l'IP externe de votre noeud

# Accédez à http://<IP-DU-NOEUD>:<NODE-PORT>
# Vous devriez voir "Connecté à la base de donnée" !
```

</details>