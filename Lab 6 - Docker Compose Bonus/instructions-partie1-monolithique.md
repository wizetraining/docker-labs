# LAB 6 BONUS - Orchestrer une Application Microservices Complète

### Mise en Situation

Vous rejoignez une nouvelle équipe \! Votre première mission est de faire tourner en local l'application de vote de l'entreprise. Votre objectif est de la conteneuriser entièrement pour que n'importe quel développeur puisse la lancer avec une seule commande.

Nous nous mettons dans le contexte où nous avons développé une application `Voting-app`.


### Déployer une application Monolithique N-Tiers (Voting App) 

Dans un premier temps, nous allons déployer la **Voting App** dans sa version monolithique. Bien que le code de l'application reste "monolithique" (tout le code Python est regroupé), nous basculons ici sur une **architecture N-tiers** (ou Client-Serveur).

![architecture monolotique de notre application](../images/architecture-app.png)

Nous avons désormais deux services distincts :

1.  **Le Serveur d'Application :** Le code Python (Flask) qui gère toute la logique (voter, traiter les données, afficher les résultats).
2.  **Le Serveur de Données :** Une base de données **PostgreSQL** robuste.

### Objectifs

L'objectif est de comprendre par la pratique :

1.  Les difficultés de lancer une application dépendante d'une BDD localement.
2.  La complexité de connecter manuellement deux conteneurs (`docker run` multiples, réseaux, variables d'environnement).
3.  La simplicité et la puissance de **Docker Compose** pour automatiser tout cela.

-----

## Partie 1 : Préparation de l'Application

Commençons par créer les fichiers de notre application.

**1. Créez l'arborescence du projet**
Ouvrez votre terminal et créez la structure suivante :

```bash
mkdir lab6-voting-postgres && cd lab6-voting-postgres
touch app.py
touch requirements.txt
touch Dockerfile
touch docker-compose.yml
```

**2. Le code de l'application (app.py)**
Copiez le code ci-dessous.
*Notez la logique de "Retry" (boucle while) : le conteneur Python démarre souvent plus vite que la base de données, il faut donc attendre que Postgres soit prêt.*

<details><summary>Contenu du fichier `app.py`</summary>

```python
import time
import os
import psycopg2
from psycopg2.extras import RealDictCursor
from flask import Flask, render_template_string, request, redirect, url_for

app = Flask(__name__)

# --- Configuration DB via Variables d'Environnement ---
# Par défaut, l'app cherche une DB sur localhost, ce qui échouera dans un conteneur sans config
DB_HOST = os.environ.get('DB_HOST', 'localhost') 
DB_NAME = os.environ.get('DB_NAME', 'postgres')
DB_USER = os.environ.get('DB_USER', 'postgres')
DB_PASS = os.environ.get('DB_PASS', 'password')

def get_db_connection():
    """Tente de se connecter à la DB avec un mécanisme de retry"""
    retries = 5
    while retries > 0:
        try:
            conn = psycopg2.connect(
                host=DB_HOST,
                database=DB_NAME,
                user=DB_USER,
                password=DB_PASS
            )
            return conn
        except psycopg2.OperationalError as e:
            print(f"La DB n'est pas encore prête... ({retries} essais restants)")
            time.sleep(2)
            retries -= 1
    raise Exception("Impossible de se connecter à la base de données Postgres")

def init_db():
    """Initialise la table dans Postgres"""
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('''
        CREATE TABLE IF NOT EXISTS votes (
            id SERIAL PRIMARY KEY,
            vote TEXT NOT NULL
        )
    ''')
    conn.commit()
    cur.close()
    conn.close()

# --- Templates HTML/CSS ---
STYLE = """
<style>
    body { font-family: 'Helvetica', sans-serif; background-color: #f4f4f4; text-align: center; padding: 50px; }
    .container { background: white; max-width: 500px; margin: 0 auto; padding: 30px; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
    h1 { color: #333; }
    button { width: 45%; padding: 15px; font-size: 18px; border: none; cursor: pointer; color: white; border-radius: 5px; margin: 2%; }
    .btn-a { background-color: #1abc9c; } 
    .btn-b { background-color: #3498db; }
    .bar-container { background-color: #ddd; border-radius: 5px; margin: 10px 0; text-align: left; overflow: hidden; }
    .bar { height: 30px; line-height: 30px; color: white; text-align: center; }
    .link { display: block; margin-top: 20px; color: #555; text-decoration: none; }
</style>
"""

VOTE_TEMPLATE = """<!DOCTYPE html><html><head><title>Vote</title>""" + STYLE + """</head><body>
    <div class="container">
        <h1>Cats vs Dogs! (Postgres Edition)</h1>
        <form action="/vote" method="post">
            <button type="submit" name="vote" value="Cats" class="btn-a">🐱 Cats</button>
            <button type="submit" name="vote" value="Dogs" class="btn-b">🐶 Dogs</button>
        </form>
        <a href="/results" class="link">Voir les résultats</a>
    </div>
</body></html>"""

RESULT_TEMPLATE = """<!DOCTYPE html><html><head><title>Résultats</title>""" + STYLE + """</head><body>
    <div class="container">
        <h1>Résultats</h1>
        <h3>🐱 Cats : {{ cats }} ({{ cats_pct }}%)</h3>
        <div class="bar-container"><div class="bar btn-a" style="width: {{ cats_pct }}%;"></div></div>
        <h3>🐶 Dogs : {{ dogs }} ({{ dogs_pct }}%)</h3>
        <div class="bar-container"><div class="bar btn-b" style="width: {{ dogs_pct }}%;"></div></div>
        <p>Total: {{ total }}</p>
        <a href="/" class="link">Retour</a>
    </div>
</body></html>"""

# --- Routes ---

@app.route('/')
def home():
    return render_template_string(VOTE_TEMPLATE)

@app.route('/vote', methods=['POST'])
def vote():
    vote_val = request.form['vote']
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("INSERT INTO votes (vote) VALUES (%s)", (vote_val,))
    conn.commit()
    cur.close()
    conn.close()
    return redirect(url_for('results'))

@app.route('/results')
def results():
    conn = get_db_connection()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    
    cur.execute("SELECT count(*) as count FROM votes WHERE vote='Cats'")
    cats_count = cur.fetchone()['count']
    
    cur.execute("SELECT count(*) as count FROM votes WHERE vote='Dogs'")
    dogs_count = cur.fetchone()['count']
    
    cur.close()
    conn.close()
    
    total = cats_count + dogs_count
    cats_pct = round((cats_count / total * 100), 1) if total > 0 else 0
    dogs_pct = round((dogs_count / total * 100), 1) if total > 0 else 0

    return render_template_string(RESULT_TEMPLATE, cats=cats_count, cats_pct=cats_pct, dogs=dogs_count, dogs_pct=dogs_pct, total=total)

if __name__ == '__main__':
    # Initialisation de la DB au lancement
    init_db()
    app.run(debug=True, host='0.0.0.0', port=5000)
```

</details>

**3. Les dépendances Python (requirements.txt)**

Contenu de `requirements.txt` :

```
flask
psycopg2-binary
```

-----

## Partie 2 : Le problème du déploiement manuel (Sans Docker)

Avant de conteneuriser, essayons de lancer l'application comme un développeur le ferait sur sa machine ("The old way").

1.  **Installez les dépendances** (si vous avez Python installé) :

    ```bash
    # installer le module venv si nécessaire
    sudo apt install python3-pip python3-venv 

    # créer l'environnement
    python3 -m venv .venv

    # l'activer
    source .venv/bin/activate

    # installer les dépendances
    pip install -r requirements.txt
    ```

2.  **Lancez l'application** :
    ```bash
    python3 app.py
    ```

**Que se passe-t-il ?**

L'application va probablement **crasher** ou afficher une erreur en boucle ci-dessous: `Exception: Impossible de se connecter à la base de données Postgres`.

```bash
La DB n'est pas encore prête... (5 essais restants)
La DB n'est pas encore prête... (4 essais restants)
La DB n'est pas encore prête... (3 essais restants)
La DB n'est pas encore prête... (2 essais restants)
La DB n'est pas encore prête... (1 essais restants)
Traceback (most recent call last):
  File "/home/plb/lab4-voting-postgres/app.py", line 125, in <module>
    init_db()
  File "/home/plb/lab4-voting-postgres/app.py", line 36, in init_db
    conn = get_db_connection()
  File "/home/plb/lab4-voting-postgres/app.py", line 32, in get_db_connection
    raise Exception("Impossible de se connecter à la base de données Postgres")
Exception: Impossible de se connecter à la base de données Postgres
```

**Pourquoi ?**
Parce que votre code attend une base de données PostgreSQL sur votre machine, mais vous n'en avez pas installé \!

> **Leçon n°1 :** Sans conteneurs, vous devez installer et configurer manuellement chaque service (DB, Cache, Serveur Web) sur votre machine. C'est long et source d'erreurs.

Ouvrez un second terminal pour lancer une base de donnée localement via docker :

```bash
docker run -d --name pg-manual -p5432:5432 -e POSTGRES_PASSWORD=password postgres:13-alpine
```

Une fois que la base de donnée est prête, revenez dans le précédant terminal et relancer l'application

3.  **Relancez l'application** :
    ```bash
    python app.py
    ```
-----

## Partie 3 : Le problème du déploiement conteneurisé manuel (Sans Compose)

Réparons le problème précédent avec Docker, mais **sans** Docker Compose, pour voir la difficulté de connecter les choses "à la main".

**1. Le Dockerfile**
Créez le fichier `Dockerfile` pour rendre notre application portable.

<details><summary>Contenu du `Dockerfile`</summary>

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

</details>

**2. Construisez l'image**

```bash
docker build -t voting-app-manual .
```

**3. Tentez de faire fonctionner l'ensemble manuellement**

Pour que cela fonctionne, il faut :

1.  Créer un **Réseau Docker** (pour qu'ils se voient).
2.  Lancer **Postgres** sur ce réseau avec un nom spécifique.
3.  Lancer **l'App** sur ce même réseau en lui donnant les variables d'environnement.

Essayez de lancer ces commandes une par une :

```bash
# 1. Création du réseau
docker network create reseau-vote

# 2. Lancement de la DB (en arrière plan)
docker run -d --name pg-manual --network reseau-vote -e POSTGRES_PASSWORD=password postgres:13-alpine

# 3. Lancement de l'App (connectée au réseau + variables d'env pour trouver la DB)
docker run -p 5000:5000 --network reseau-vote -e DB_HOST=pg-manual -e DB_PASS=password voting-app-manual
```

Allez sur `http://localhost:5000`. Cela fonctionne \!

**Mais quel effort \!**
Regardez la longueur des commandes. Imaginez devoir faire cela chaque matin, ou pire, avec 10 micro-services différents.

> **Leçon n°2 :** Gérer manuellement le réseau (`--network`), les noms DNS (`--name`) et les variables (`-e`) devient vite ingérable.

**Nettoyage avant la suite :**
Supprimez ces conteneurs manuels pour laisser la place propre à Docker Compose.

```bash
docker rm -f pg-manual 
docker rm -f voting-app-manual
docker network rm reseau-vote
```

-----

## Partie 4 : La Solution - Définir la Stack avec `docker-compose.yml`

Nous allons maintenant automatiser tout ce que nous venons de faire péniblement.

Créez le fichier `docker-compose.yml`. Il va orchestrer nos deux services :

  * `voting-app` : Construit l'image et injecte automatiquement les variables.
  * `db` : Lance Postgres.
  * **Magie :** Compose crée automatiquement le réseau entre eux \!

**Remplissez votre fichier `docker-compose.yml` :**

<details><summary>Correction - Contenu de docker-compose.yml</summary>

```yaml
services:
  
  # Notre application Python
  voting-app:
    build: .
    ports:
      - "5000:5000"
    environment:
      # On dit à l'app que le nom d'hôte de la DB est le nom du service ci-dessous : 'db'
      - DB_HOST=db
      - DB_NAME=postgres
      - DB_USER=postgres
      - DB_PASS=password
    volumes:
      - .:/app
    depends_on:
      - db

  # Notre base de données PostgreSQL
  db:
    image: postgres:13-alpine
    # Variables obligatoires pour l'image Postgres officielle
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=postgres
    # Persistance des données (Volume)
    volumes:
      - pg-data:/var/lib/postgresql/data

# Volume géré par Docker pour stocker les fichiers de la BDD
volumes:
  pg-data:
```

</details>

-----

## Partie 5 : Piloter l'Application avec une seule commande \!

**1. Lancez toute l'application**
Plus besoin de commandes complexes. Docker Compose lit le fichier, construit l'app, télécharge Postgres, crée le réseau et connecte tout.

```bash
docker compose up -d
```

*Observez les logs : vous verrez l'application Python et la base de données démarrer ensemble.*

**2. Vérifiez le fonctionnement**
Ouvrez votre navigateur sur **http://localhost:5000**.
Votez pour "Cats" ou "Dogs".

**3. Visualisez l'état du système**
Dans un nouveau terminal, lancez :

```bash
docker compose ps
```

Vous verrez vos deux conteneurs proprement nommés et actifs.

**4. Test de Persistance (La puissance des Volumes)**

  * Faites quelques votes sur l'interface.
  * Arrêtez tout (simulons un crash ou une mise à jour) :
    ```bash
    docker compose down
    ```
  * Relancez l'application :
    ```bash
    docker compose up -d
    ```
  * Retournez sur `http://localhost:5000/results`. **Les votes sont toujours là \!** Le volume `pg-data` défini dans le Compose a protégé vos données.

**5. Debugging : Entrer dans la base**
Vous pouvez utiliser Compose pour exécuter des commandes dans vos services facilement. Vérifions les votes en SQL direct :

```bash
# Exécuter une commande psql à l'intérieur du service 'db'
docker compose exec db psql -U postgres -c "SELECT * FROM votes;"
```

Félicitations \! Vous avez non seulement déployé une architecture N-Tiers, mais vous avez surtout compris **pourquoi** Docker Compose est indispensable pour gérer la complexité des applications modernes.