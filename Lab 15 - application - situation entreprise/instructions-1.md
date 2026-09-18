# LAB 15.1 - Déployer une Application Sécurisée avec Secrets et StatefulSet

Ce lab met en pratique la gestion des secrets, le déploiement d'une application sans état (stateless)  et l'utilisation d'un **Opérateur** pour gérer une base de données PostgreSQL de production.

**Concepts couverts :**
* **Secret** : Pour gérer les mots de passe de manière sécurisée.
* **Deployment** : Pour gérer notre application web sans état.
* **Service (ClusterIP & LoadBalancer)** : Pour la communication interne et l'exposition externe.
* **StatefulSet** : Pour déployer des applications avec état comme PostgreSQL.
* **La notion d'Operator Kubernetes** : Introduction aux Opérateurs pour palier les limitations du StatefulSet
  * **CloudNativePG (CNPG)** : Utilisation d'un opérateur pour automatiser le cycle de vie de PostgreSQL.
  * **Services RW/RO** : Comprendre comment l'opérateur sépare le trafic d'écriture et de lecture.
  * **Persistence** : Gestion automatique des volumes par l'opérateur.
  * **Haute Disponibilité (HA)** : Mise en place d'une réplication primaire/réplique.

**Prérequis :**

L'image de l'application dans sa version Entreprise est : `public.ecr.aws/wizetraining/webapp-count-secure:v1` (disponible sur un registre d'images public AWS ECR)

<details>
<summary>Voir le code de l'application (Version Entreprise) Ici : </summary>

```python
import time
import socket
import logging
import json
import os
import psycopg2
from flask import Flask, Response, jsonify, request

# --------------------------------------------------
# Configuration du logging (Format standard Entreprise)
# --------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger("app")

app = Flask(__name__)

# --------------------------------------------------
# Configuration PostgreSQL dynamique (12-factor compliant)
# --------------------------------------------------
PG_HOST = os.getenv('PG_HOST', 'postgres')
PG_PORT = os.getenv('PG_PORT', '5432')
PG_USER = os.getenv('PG_USER', 'postgres')
PG_DB = os.getenv('PG_DB', 'postgres')

logger.info(f"Configuration PostgreSQL: {PG_HOST}:{PG_PORT} (User: {PG_USER}, DB: {PG_DB})")

# --------------------------------------------------
# Gestion du mot de passe via Docker/K8s Secret
# --------------------------------------------------
def get_pg_password():
    secret_path = os.getenv("PG_PASSWORD_FILE", "/etc/app/secrets/pg_password")
    try:
        with open(secret_path, 'r') as secret_file:
            password = secret_file.read().strip()
            # On ne logue JAMAIS le mot de passe, juste le succès
            logger.info(f"Mot de passe PostgreSQL chargé avec succès depuis {secret_path}.")
            return password
    except IOError:
        logger.warning(
            f"Secret introuvable dans {secret_path}. "
            "Tentative de connexion sans mot de passe."
        )
        return ""

# --------------------------------------------------
# Sondes Kubernetes (Liveness & Readiness)
# --------------------------------------------------
@app.route('/healthz')
def healthz():
    """Liveness Probe: Vérifie si le conteneur tourne."""
    return jsonify({"status": "alive"}), 200

@app.route('/readyz')
def readyz():
    """
    Readiness Probe: Normalement on teste la DB ici. 
    Mais pour ton lab (où on veut voir l'UI chercher la DB en temps réel), 
    on doit dire à K8s que l'UI est prête à être affichée.
    """
    return jsonify({"status": "ready", "ui": "available"}), 200

# --------------------------------------------------
# Fonction métier : Connexion et incrément (SSE)
# --------------------------------------------------
def stream_connection_and_count():
    """Générateur Server-Sent Events (SSE)."""
    password = get_pg_password()
    retries = 5
    
    while True:
        logger.info(f"Tentative de connexion à PostgreSQL (Essais restants : {retries})...")
        yield f"data: {json.dumps({'status': 'loading', 'message': f'Recherche de la base de données... ({retries} essais restants)'})}\n\n"
        
        try:
            conn = psycopg2.connect(
                host=PG_HOST, port=PG_PORT, user=PG_USER, password=password, dbname=PG_DB,
                connect_timeout=2
            )
            conn.autocommit = True
            with conn.cursor() as cur:
                cur.execute("CREATE TABLE IF NOT EXISTS hits (id serial PRIMARY KEY, count integer);")
                cur.execute("INSERT INTO hits (id, count) VALUES (1, 1) ON CONFLICT (id) DO UPDATE SET count = hits.count + 1 RETURNING count;")
                count = cur.fetchone()[0]
            conn.close()

            logger.info(f"Connexion réussie ! Visiteur n°{count}")
            yield f"data: {json.dumps({'status': 'success', 'count': count, 'message': 'Connecté à PostgreSQL (Sécurisé)'})}\n\n"
            break

        except psycopg2.OperationalError as exc:
            if retries == 0:
                logger.error(f"Échec critique PostgreSQL : {exc}")
                yield f"data: {json.dumps({'status': 'error', 'message': 'Echec de connexion à PostgreSQL'})}\n\n"
                break
            retries -= 1
            time.sleep(1.5)

@app.route('/stream')
def stream():
    """Route asynchrone pour les événements en temps réel."""
    return Response(stream_connection_and_count(), mimetype='text/event-stream')

# --------------------------------------------------
# Route Principale (Chargement immédiat)
# --------------------------------------------------
@app.route('/')
def hello():
    hostname = socket.gethostname()

    html_response = f"""
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Compteur Sécurisé (PostgreSQL)</title>
        <style>
            body {{ font-family: -apple-system, sans-serif; text-align: center; margin-top: 50px; background: #f8f9fa; }}
            .container {{ padding: 30px; border: 1px solid #e9ecef; border-radius: 12px; display: inline-block; background: #fff; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }}
            .hostname {{ font-size: 0.9em; color: #6c757d; margin-top: 25px; padding: 10px; background: #f8f9fa; border-radius: 6px; }}
            .db-status {{ margin-top: 20px; font-size: 1.1em; }}
            .blinking {{ animation: blinker 1s linear infinite; color: #ff9800; font-weight: bold; }}
            @keyframes blinker {{ 50% {{ opacity: 0; }} }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Application d'Entreprise 🚀</h1>
            <p>Vous êtes le visiteur numéro <strong id="count-display" style="font-size: 1.2em;">...</strong>.</p>
            <div class="hostname">Instance : <code>{hostname}</code></div>
            <div class="db-status" id="db-status-display"><span class="blinking">Initialisation de la connexion... 📡</span></div>
        </div>

        <script>
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
                    source.close();
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
    app.run(host="0.0.0.0", port=5000)
```

</details>

Un fichier local contenant le mot de passe de votre base de données, par exemple `secrets/pg_password.txt`.

---

## Partie 1 : Création du Secret Kubernetes

Tout comme avec un Docker Secrets, nous devons d'abord stocker le mot de passe de Postgresql de manière sécurisée. Kubernetes utilise un objet Secret pour cela.

1. Préparer le fichier de mot de passe :

Assurez-vous d'avoir un fichier local. Par exemple, créez le répertoire et le fichier :

```bash
mkdir secrets
echo -n "PostgresSuperPassword123" > secrets/pg_password.txt
```

2. Créer le Secret dans Kubernetes :
* nom du secret : `pg-secret` à partir du fichier précédent.
* La clé à l'intérieur du secret (`password`) correspondra au nom du fichier.
* Quel commande pour vérifier le contenu ?

<details>
<summary>Correction : Commande Secret</summary>

```bash
# Nous créons un secret nommé 'pg-secret'
# La clé 'password' est celle attendue par PostgreSQL et plus tard par CNPG par défaut
kubectl create secret generic pg-secret --from-file=password=./secrets/pg_password.txt
```

```bash
kubectl get secrets
NAME           TYPE     DATA   AGE
pg-secret   Opaque   1      10s
```

Vérifiez le contenu (encodé en base64) :

```bash
kubectl get secret pg-secret -o jsonpath='{.data.password}' | base64 -d
```
</details>

---

## Partie 2 : Re-Déploiement de Postgres avec un StatefulSet et notre Secret

1. Si vous deviez redéployer la base de données PostgreSQL du lab précédent en utilisant le nouveau Secret, comment auriez-vous modifié le fichier `postgres-statefulset.yaml` ? 

*(Note : Dans un StatefulSet, il est de bonne pratique de l'associer à un Service "Headless" pour la résolution DNS individuelle des pods).*

<details>
<summary>Correction YAML - postgres-statefulset.yaml </summary>

```yaml
# Service "Headless" très utilisé pour un StatefulSet propre
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  clusterIP: None # C'est ce qui le rend "Headless"
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: "postgres-service" # Lien vers le service ci-dessus
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
        # AU LIEU DE CODER EN DUR, ON UTILISE LE SECRET
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: postgres
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```
</details>

2. Appliquez la configuration et mettez à jour le déploiement `webapp-deployment`. 
* Changez l'image pour utiliser la nouvelle version `public.ecr.aws/wizetraining/webapp-count-secure:v1` 
* chargez les variables d'environnement ainsi que le volume nécessaires pour que l'application se connecte bien à notre nouvelle base de données.

_**Astuce :** Analysez le code `app.py` et voyez les variables d'environnements attendues (`PG_HOST`, `PG_PORT`, `PG_USER`, `PG_DB`) et surtout où l'application cherche le fichier contenant le mot de passe (`PG_PASSWORD_FILE`)._

<details>
<summary>Correction : Déploiement webapp et explication </summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: public.ecr.aws/wizetraining/webapp-count-secure:v1
        ports:
        - containerPort: 5000
        env:
        - name: PG_HOST
          value: "postgres-service" # Utilise le nom du service headless créé plus haut
        - name: PG_USER
          value: "postgres"
        - name: PG_DB
          value: "postgres"
        - name: PG_PASSWORD_FILE
          value: "/etc/app/secrets/pg_password" # Chemin lu par le code Python
        volumeMounts:
        # On monte le secret comme un fichier
        - name: secret-vol
          mountPath: "/etc/app/secrets"
          readOnly: true
      volumes:
      - name: secret-vol
        secret:
          secretName: pg-secret
          items:
          - key: password
            path: pg_password # Transforme la clé 'password' en un fichier nommé 'pg_password'
```

Appliquez les fichiers :
```bash
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f webapp-deployment.yaml
```

**À propos du Service Headless créé pour PostgreSQL :**
Le Service "Headless" (avec `clusterIP: None`) pour le StatefulSet ne fait pas de load balancing global comme un service ClusterIP classique. Il donne une entrée DNS stable et prédictible à **chaque** pod de manière individuelle (ex: `postgres-statefulset-0.postgres-service`). C'est crucial pour les bases de données distribuées où chaque nœud a un rôle spécifique.

</details>

3. Testez que vous avez bien dans votre navigateur la nouvelle application Sécurisée qui s'affiche.

<details>
<summary>Correction : Vérification</summary>

Si vous n'avez pas encore de Service de type NodePort ou LoadBalancer pour votre webapp, exposez-la :

```bash
kubectl expose deployment webapp-deployment --name=webapp-service --type=LoadBalancer --port=80 --targetPort=5000
```

Récupérez l'accès (via `minikube service webapp-service` ou en cherchant l'IP de votre nœud et le NodePort).

Dans votre navigateur, vous devriez maintenant voir le titre mis à jour : **"Application d'Entreprise 🚀"** (ou "Connexion Sécurisée à la Base de Données!"), avec le statut indiquant que la connexion à PostgreSQL s'est faite avec succès !

</details>

--- 

## Partie 3 : Installation de l'Opérateur et Déploiement d'un cluster de base de données PostgreSQL de type Entreprise

Contrairement au StatefulSet manuel qui est limité comme nous l'avons vu dans le lab précédent, nous allons ici utiliser un **Opérateur**. C'est un agent logiciel qui tourne dans votre cluster et qui agit comme un "administrateur de base de données virtuel".

### 1. Installation de l'Opérateur CloudNativePG avec Helm

Avant de pouvoir demander un cluster PostgreSQL, nous devons installer l'intelligence qui va le gérer.
Nous utiliserons Helm pour l'installation.
Voir la documentation ici : [https://cloudnative-pg.io/docs](https://cloudnative-pg.io/docs)

Installation avec Helm : https://github.com/cloudnative-pg/charts

```bash
# 1. Ajouter le dépôt officiel de CloudNativePG
helm repo add cnpg https://cloudnative-pg.github.io/charts

# 2. Mettre à jour les dépôts locaux
helm repo update

# 3. Installer l'opérateur dans un namespace dédié
# L'option --create-namespace crée 'cnpg-system' s'il n'existe pas
helm upgrade --install cnpg-operator cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace
```

**Vérification :** Attendez que le pod de l'opérateur soit en état `Running`.
```bash
kubectl get pods -n cnpg-system
```

### 2. Déploiement du Cluster PostgreSQL (Custom Resource)

Maintenant que Kubernetes "comprend" ce qu'est un cluster PostgreSQL grâce à l'opérateur, nous n'avons plus besoin de gérer des StatefulSets complexes. Nous déclarons simplement notre besoin via l'objet (un Custom Ressource) : `Cluster`.

Vous pouvez consulter la Doc pour voir comment on bootstrap un nouveau cluster : https://cloudnative-pg.io/documentation/1.19/bootstrap/

Bootstraper un Cluster PostgreSQL avec les éléments suivants :
* Nom du cluster : `pg-database`
* Instances : 3 (1 Primaire et 2 Répliques pour la Haute Disponibilité)
* Image : `ghcr.io/cloudnative-pg/postgresql:16`
* Stockage : `1Gi`
* Sécurité : Utilisation du secret `pg-secret` créé en Partie 1.

Comme vous pouvez le lire dans la documentation, l'operator a besoin d'un secret de type `kubernetes.io/basic-auth`avec ces deux clés : `username` et `password`. Pour ce faire, vous pouvez créer un nouveau secret ou mettre à jour le secret précédent

```bash
# Création d'un nouveau
kubectl create secret generic pg-secret-2 \
  --from-literal=username=app_user \
  --from-file=password=./secrets/pg_password.txt
```

<details>
<summary>Correction : Manifeste postgres-cluster.yaml</summary>

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-database
spec:
  instances: 3 # L'opérateur va créer 3 pods et gérer la réplication
  imageName: ghcr.io/cloudnative-pg/postgresql:16
  
  # Configuration du stockage (PVC gérés automatiquement)
  storage:
    size: 1Gi
  
  # Initialisation de la base de données
  bootstrap:
    initdb:
      database: app_db
      owner: app_user
      secret:
        name: pg-secret-2 # Le secret créé en Partie 1 mis à jour ou le nouveau secret 
```
</details>

<details>
<summary>Correction : Application et observation</summary>

```bash
# Appliquer le cluster
kubectl apply -f postgres-cluster.yaml

# Observer l'opérateur travailler (il va créer les PVC, puis les Pods un par un)
kubectl get pod -w

# Observer le Custom Resource Cluster
kubectl get cluster
```
</details>

### 3. Comprendre les Services générés et mettez à jour l'application Web

L'opérateur a créé automatiquement plusieurs Services. Tapez `kubectl get svc`. Vous verrez :
* `pg-database-rw` : **Read-Write**. C'est l'adresse que l'application utilise pour écrire des données. Elle pointe toujours vers le "Maître".
* `pg-database-ro` : **Read-Only**. Pour envoyer les requêtes de lecture vers les répliques et soulager le maître.

**Question :** Selon vous, quel hôte (`PG_HOST`) devrons-nous configurer dans notre application Web pour que le compteur fonctionne ?

<details>
<summary>Réponse</summary>
Nous devons utiliser `pg-database-rw`. Si le maître tombe, l'opérateur promeut une réplique et met à jour ce service automatiquement pour qu'il pointe vers le nouveau maître. C'est la magie de l'opérateur !
</details>

Mettez à jour l'application Web avec les bonnes information et augmentez le nombre de replicas si ce n'est pas fait
* Replicas : 2
* Variables d'env : `PG_HOST=pg-database-rw`, `PG_USER=app_user`, `PG_DB=app_db`
* Montage du Secret **(Inchangé)** : Montez `pg-secret` dans `/etc/app/secrets` et renommez la clé en `pg_password`.

---

## Partie 4 : Tester la Haute Disponibilité (HA)

Contrairement au StatefulSet manuel du Lab 6, l'Opérateur CNPG gère la réplication des données entre les instances PostgreSQL.

1. **Accédez à l'application :**
Récupérez l'IP du LoadBalancer et rafraîchissez la page.
```bash
kubectl get svc webapp-service
```

2. **Simuler une panne majeure :**
Supprimez le pod qui fait office de "Primaire" (celui qui a le rôle Primary/Master).

```bash
# Identifier le primaire
kubectl get pods -l cnpg.io/cluster=pg-database --show-labels

# pour le lister lui seul
kubectl get pods -l cnpg.io/instanceRole=primary

# Supprimer le pod primaire
kubectl delete pods -l cnpg.io/instanceRole=primary --grace-period 0 --force

# Ou simplement
kubectl delete pod pg-database-1 # (remplacez par le nom du primaire actuel)
```

3. **Observer le "Failover" :**
Pendant que le pod redémarre, l'opérateur va promouvoir une réplique en nouveau Primaire.

<details>
<summary>Résultat attendu</summary>
* L'application web peut afficher une erreur pendant quelques secondes (le temps de l'élection).
* Très vite, le compteur reprend **exactement là où il était**.
* Les données n'ont pas été perdues car l'opérateur gère la synchronisation continue entre les 3 nœuds.
</details>

---

## Partie 5 : Pourquoi l'Opérateur gagne face au StatefulSet ?

Dans le lab précédent, nous avons vu qu'un StatefulSet classique avec 3 réplicas créait 3 bases de données vides et indépendantes.

**Comparaison avec CloudNativePG :**

| Fonctionnalité | StatefulSet Manuel | Opérateur (CNPG) |
| :--- | :--- | :--- |
| **Réplication** | Aucune (chaque pod est isolé) | Automatique (Streaming Replication) |
| **Failover** | Manuel (le service casse) | Automatique (Élection d'un nouveau maître) |
| **Backups** | Difficile à scripter | Natif vers S3/Azure/Google Cloud |
| **Mises à jour** | Risque de perte de données | Rolling updates sans coupure (Switchover) |

**Conclusion :** L'Opérateur est une "intelligence" ajoutée à Kubernetes qui connaît les spécificités de PostgreSQL pour garantir que vos données de compteur restent cohérentes et hautement disponibles.