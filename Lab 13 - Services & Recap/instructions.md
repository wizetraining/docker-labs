# LAB 13 - Récap (Pod, ReplicaSet, Deployment) et pratique des Services : Déploiement d'une Application Web avec PostgreSQL

Cet atelier vous guidera à travers les concepts fondamentaux de Kubernetes, de la création d'un simple Pod au déploiement d'une application multi-services, persistante et hautement disponible.

**Prérequis :**
* Un cluster Kubernetes fonctionnel
* `kubectl` configuré pour communiquer avec votre cluster.

L'image Docker `wizetraining/webapp-count:v1` est disponible sur Docker Hub. 
* Si vous rencontrez des limitations, utilisez `public.ecr.aws/wizetraining/webapp-count:v1` ou `public.ecr.aws/wizetraining/postgres:16-alpine`, ou `public.ecr.aws/wizetraining/webapp-count:v2`

Le code de l'application est ci-dessous :

<details><summary>Code Webapp-Count v1</summary>

```python
import time
import socket
import logging
import json
import os
import psycopg2
from flask import Flask, Response

app = Flask(__name__)

# Configuration des logs pour voir l'activité en temps réel
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')

# Paramètres de connexion PostgreSQL (Docker Compose résoudra le hostname 'postgres')
PG_HOST = os.getenv('PG_HOST', 'postgres')
PG_PORT = os.getenv('PG_PORT', '5432')
PG_USER = os.getenv('PG_USER', 'postgres')
PG_PASSWORD = os.getenv('PG_PASSWORD', 'postgres')
PG_DB = os.getenv('PG_DB', 'postgres')

def stream_connection_and_count():
    """
    Générateur Server-Sent Events (SSE).
    Tente de se connecter à PostgreSQL, incrémente le compteur, et envoie 
    l'état en temps réel au navigateur ainsi qu'aux logs.
    """
    retries = 5
    while True:
        # 1. On log en console et on notifie le navigateur de la tentative
        logging.info(f"Tentative de connexion à PostgreSQL (Essais restants : {retries})...")
        yield f"data: {json.dumps({'status': 'loading', 'message': f'Recherche de la base de données... ({retries} essais restants)'})}\n\n"
        
        try:
            # 2. Tentative de connexion
            conn = psycopg2.connect(
                host=PG_HOST, port=PG_PORT, user=PG_USER, password=PG_PASSWORD, dbname=PG_DB,
                connect_timeout=2
            )
            conn.autocommit = True
            with conn.cursor() as cur:
                # On s'assure que la table existe
                cur.execute("CREATE TABLE IF NOT EXISTS hits (id serial PRIMARY KEY, count integer);")
                # On insère la ligne si elle n'existe pas, sinon on l'incrémente
                cur.execute("INSERT INTO hits (id, count) VALUES (1, 1) ON CONFLICT (id) DO UPDATE SET count = hits.count + 1 RETURNING count;")
                count = cur.fetchone()[0]
            conn.close()

            # 3. Succès ! On notifie les logs et le navigateur
            logging.info(f"Connexion réussie ! Visiteur n°{count}")
            yield f"data: {json.dumps({'status': 'success', 'count': count, 'message': 'Connecté à la base de données'})}\n\n"
            break

        except psycopg2.OperationalError as exc:
            # 4. Echec de la tentative
            if retries == 0:
                logging.error("Échec critique : Impossible de joindre PostgreSQL.")
                yield f"data: {json.dumps({'status': 'error', 'message': 'Echec de connexion à la base de données'})}\n\n"
                break
            retries -= 1
            # Pause pour éviter de spammer et pour laisser le temps de voir l'animation côté navigateur
            time.sleep(1.5)

@app.route('/stream')
def stream():
    """Route asynchrone qui envoie les événements au navigateur en temps réel."""
    return Response(stream_connection_and_count(), mimetype='text/event-stream')

@app.route('/')
def hello():
    # L'affichage du HTML est IMMÉDIAT. Il n'y a plus de code bloquant ici.
    hostname = socket.gethostname()

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
            /* Effet visuel clignotant pour le fun */
            .blinking {{ animation: blinker 1s linear infinite; color: #ff9800; font-weight: bold; }}
            @keyframes blinker {{ 50% {{ opacity: 0; }} }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Bonjour !</h1>
            
            <p>Vous êtes le visiteur numéro <strong id="count-display">...</strong>.</p>
            <div class="hostname">Vous êtes sur le conteneur : {hostname}</div>
            <div class="db-status" id="db-status-display"><span class="blinking">Initialisation de la connexion... 📡</span></div>
        </div>

        <script>
            // Connexion à la route SSE pour recevoir les événements Python en direct
            const source = new EventSource('/stream');
            
            source.onmessage = function(event) {{
                const data = JSON.parse(event.data);
                const statusDisplay = document.getElementById('db-status-display');
                const countDisplay = document.getElementById('count-display');
                
                if (data.status === 'loading') {{
                    statusDisplay.innerHTML = '<span class="blinking">' + data.message + ' 📡</span>';
                }} else if (data.status === 'success') {{
                    statusDisplay.innerHTML = '<strong style="color:green;">' + data.message + '</strong>';
                    countDisplay.innerText = data.count;
                    source.close(); // Le travail est terminé, on ferme la connexion
                }} else if (data.status === 'error') {{
                    statusDisplay.innerHTML = '<span style="color:red;">' + data.message + '</span>';
                    countDisplay.innerText = 'N/A';
                    source.close();
                }}
            }};
        </script>
    </body>
    </html>
    """
    return html_response

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

</details>

---

## Partie 1 : Les Pods
Un Pod est la plus petite unité de déploiement dans Kubernetes. Il représente un ou plusieurs conteneurs s'exécutant ensemble sur le même nœud, partageant le même réseau et le même stockage.

**Objectif** : Déployer un Pod unique exécutant notre application web.

1. Créez un manifeste `pod-webapp.yaml` pour créer un Pod :
* Dans votre namespace à vous
* Nom du Pod : `webapp-pod`
* Nom du conteneur : `webapp-container`
* Image : `wizetraining/webapp-count:v1`
* Port du conteneur : `5000`

<details>
<summary>Correction : pod-webapp.yaml</summary>

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
    image: wizetraining/webapp-count:v1
    ports:
    - containerPort: 5000
```
</details>

2. Appliquez le manifeste et testez l'application (Vous devriez voir "Echec de connexion à la base de données" car PostgreSQL n'est pas encore déployé).

<details>
<summary>Correction : Commandes Pod</summary>

```bash
# Appliquer le manifeste
kubectl apply -f pod-webapp.yaml

# Vérifier le statut (doit être "Running")
kubectl get pods

# Rediriger le port pour tester en local
kubectl port-forward pod/webapp-pod 8080:5000
```
*Ouvrez votre navigateur sur http://localhost:8080*
</details>

---

## Partie 2 : Les ReplicaSets

Un ReplicaSet garantit qu'un nombre spécifié de répliques de Pods (copies identiques) s'exécutent en permanence. Si un Pod tombe, le ReplicaSet en crée un autre pour le remplacer.

**Objectif** : Assurer la haute disponibilité de notre application avec 3 répliques.

1. Créez le manifeste `replicaset-webapp.yaml` pour un ReplicaSet :
* Nom : `webapp-replicaset`
* Replicas : 3
* Image : `wizetraining/webapp-count:v1`

<details>
<summary>Correction : replicaset-webapp.yaml</summary>

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
  template:
    metadata:
      labels:
        app: webapp-rs
    spec:
      containers:
      - name: webapp-container
        image: wizetraining/webapp-count:v1
        ports:
        - containerPort: 5000
``` 
</details>

2. Appliquez et testez la résilience en supprimant un pod manuellement.

<details>
<summary>Correction : Commandes ReplicaSet</summary>

```bash
# Appliquer le manifeste
kubectl apply -f replicaset-webapp.yaml

# Lister les pods (il doit y en avoir 3)
kubectl get pods -l app=webapp-rs

# Supprimer un pod pour tester la résilience
kubectl delete pod <nom-dun-des-pods>

# Relister immédiatement (un nouveau pod est en cours de création)
kubectl get pods -l app=webapp-rs
```
</details>

---

## Partie 3 : Deployments

Le Deployment est une abstraction au-dessus des ReplicaSets. Il permet de gérer les mises à jour (rolling updates) et les retours en arrière (rollbacks) de manière déclarative et contrôlée. C'est l'objet que l'on utilise le plus souvent pour déployer des applications stateless.

**Objectif** : Déployer notre application via un Deployment, la scaler, la mettre à jour vers la v2 et revenir en arrière.

1. Créez le manifeste `deployment-webapp.yaml` pour un Deployment :
* Nom : `webapp-deployment`
* Replicas : 2
* Image : `wizetraining/webapp-count:v1`

<details>
<summary>Correction : deployment-webapp.yaml</summary>

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
        image: wizetraining/webapp-count:v1
        ports:
        - containerPort: 5000
```
</details>

2. Déployez, scalez, mettez à jour et revenez en arrière :
* Faites un scale du nombre de replicas à 4
* Faites une montée de version vers l'application `v2` (image `wizetraining/webapp-count:v2`)
* Vérifiez le statut de la mise à jour
* Revenez à la version précédente 

<details>
<summary>Correction : Commandes Deployment</summary>

```bash
# Appliquer le manifeste
kubectl apply -f deployment-webapp.yaml

# Scaler de 2 à 4 répliques
kubectl scale deployment webapp-deployment --replicas=4

# Mettre à jour vers la v2 (Rolling Update)
kubectl set image deployment/webapp-deployment webapp-container=wizetraining/webapp-count:v2

# Vérifier la progression de la mise à jour
kubectl rollout status deployment/webapp-deployment

# Revenir en arrière (Rollback à la v1)
kubectl rollout undo deployment/webapp-deployment
```
</details>

---

## Partie 4 : Services

Un Service expose un ensemble de Pods (généralement gérés par un Deployment) sous une seule adresse IP et un seul nom DNS stables à l'intérieur du cluster. Il permet aux différentes parties de votre application de communiquer entre elles.

**Objectif** : Déployer PostgreSQL et exposer notre webapp pour qu'elle puisse s'y connecter et être accessible.

1. Créez un service de type `NodePort` qui expose le déploiement webapp précédent :
* Nom du service : `webapp-service`
* Port à exposer : `80`
* Utilisez `describe` pour visualiser les endpoints.

<details>
<summary>Correction : webapp-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp-deploy
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
```
</details>

2. Créez un service de type `LoadBalancer` qui expose le déploiement précédent :
* Nom du service : `webapp-service-lb`
* Port à exposer : `80`

<details>
<summary>Correction</summary>
*Pas de correction pour cette étape ;)*
</details>


3. Créez les manifestes pour PostgreSQL (`postgres-deployment.yaml`, `postgres-service.yaml`) :
* Un deployment PostgreSQL, `replicas` : 1 , image : `postgres:16-alpine`
* Attention : PostgreSQL nécessite la variable d'environnement `POSTGRES_PASSWORD` pour démarrer (donnez-lui la valeur `postgres` pour correspondre à notre code Python).
* Un service qui expose PostgreSQL sur le port du conteneur `5432`
* Quel nom avez-vous donné au Service ? Pourquoi ?
* De quel type de Service avons-nous besoin ? Pourquoi ?

<details>
<summary>Correction : postgres-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
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
          value: postgres
```
</details>

<details>
<summary>Correction : postgres-service.yaml</summary>

*Note : Le nom du service DOIT être `postgres` car l'application cherche par défaut l'hôte `postgres`. Le type est `ClusterIP` (par défaut) car seul le cluster interne a besoin de communiquer avec la base de données.*

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
```
</details>


4. Appliquez les manifestes et testez la connexion. Tentez de vous connecter au service Web via le NodePort. Vous devriez maintenant voir le compteur clignoter puis afficher "Connecté à la base de données".

<details>
<summary>Correction : Commandes Finales</summary>

```bash
# Appliquer les manifestes PostgreSQL
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml

# Appliquer le service de la webapp
kubectl apply -f webapp-service.yaml

# Redémarrer les pods webapp pour forcer la reconnexion si nécessaire
kubectl rollout restart deployment webapp-deployment

# Obtenir l'accès (Minikube)
minikube service webapp-service

# Obtenir l'accès (Standard K8s)
kubectl get service webapp-service
kubectl get nodes -o wide
# Naviguez vers http://<IP-DU-NOEUD>:<NODE-PORT>
```
</details>