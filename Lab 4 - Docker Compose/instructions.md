# LAB 4 - Déployer une Application Multi-Conteneurs avec Docker Compose

Cet TP sera effectué sur la machine hôte. Il est recommendé d'avoir complété les Lab 1, 2 & 3.

Bienvenue dans le monde du développement moderne \! Fini la gestion manuelle des conteneurs un par un. En tant que développeur, vous allez assembler une application complète composée de plusieurs services et l'orchestrer avec une seule commande.

### Mise en Situation

Vous êtes chargé de développer un compteur de visites web. C'est une application simple mais qui illustre parfaitement une architecture micro-services :

1.  **Un service web** : Une application en Python (Flask) qui affiche le nombre de visiteurs.
2.  **Un service de base de données** : Une base de données Redis qui stocke le compteur.

Gérer les `docker run`, les réseaux et les volumes à la main pour ces deux services serait fastidieux. C'est là que **Docker Compose** entre en jeu : il va nous permettre de décrire toute notre application dans un seul fichier et de la gérer comme un seul bloc.

## Partie 1 : Préparation de l'Application

Commençons par créer les fichiers de notre application.

**1. Créez l'arborescence du projet**
Ouvrez votre terminal et créez la structure suivante :

```bash
mkdir lab3-compose && cd lab3-compose
touch app.py
touch requirements.txt
touch Dockerfile
touch docker-compose.yml
```

**2. Le code de l'application (app.py)**
Copiez ce code Python dans votre fichier `app.py`. C'est notre service web.

```python
# Contenu de app.py
import time
import redis
from flask import Flask

app = Flask(__name__)
# On se connecte au service 'redis' sur le port par défaut.
# Docker Compose va s'assurer que l'hostname 'redis' pointe vers le bon conteneur !
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            # On incrémente de 1 la valeur de la clé 'hits' et on la retourne
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return f'Bonjour ! Vous êtes le visiteur numéro {count}.'

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

**3. Les dépendances Python (requirements.txt)**

Notre application a besoin des librairies `flask` et `redis`. Ajoutez-les dans `requirements.txt`.

Contenu de `requirements.txt`

```
flask
redis
```

**4. Le Dockerfile pour notre service web**

Créez le `Dockerfile` qui va dockeriser notre application Python pour la rendre portable.
* Partez de l'image de base : `python:3.9-slim`
* Le répertoire de travail : `/app`

<details><summary>Contenu du `Dockerfile`</summary>

```dockerfile
# On part d'une image Python officielle et légère
FROM python:3.9-slim

# On définit le répertoire de travail dans l'image
WORKDIR /app

# On copie d'abord les dépendances pour optimiser le cache de build
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# On copie ensuite le reste de notre application
COPY . .

# Commande pour lancer l'application quand le conteneur démarre
CMD ["python", "app.py"]
```

</details>

## Partie 2 : Définir la Stack avec `docker-compose.yml`

Le fichier `docker-compose.yml` est le **plan de construction** de notre application multi-services. C'est ici que toute la magie opère.
Créez 2 services Composes :
* `Web` - Service construit à partir d'un build et non de l'image. 
  * Pour un dévéloppement rapide montez le contenu du répertoire courant dans le répertoire de travail `/app`
  * Mapping de port `8000` sur la machine hôte
  * Ce service devra dépendre du démarage de Redis
* `redis`- Service lancé à partir de l'image Redis `"redis:6-alpine"`
  * Persistance de volume sur un volume `redis-data`

**Remplissez votre fichier `docker-compose.yml` :**

<details><summary>Correction - Contenu de docker-compose.yml</summary>

```yaml
# Version de la syntaxe Docker Compose. Il est désormais déprécié, donc on peut ne pas le mettre
version: "3.8"

# Définition de nos services (nos conteneurs)
services:

  # Le premier service : notre application web
  web:
    # On dit à Compose de construire l'image à partir du Dockerfile dans le répertoire courant (.)
    build: .
    # On mappe le port 8000 de notre machine au port 5000 de notre conteneur (où Flask écoute)
    ports:
      - "8000:5000"
    # On monte notre répertoire local dans le conteneur pour le développement live.
    # Vous vous souvenez du Lab 3 ?
    volumes:
      - .:/app
    # Ce service dépend du service 'redis' pour démarrer correctement
    depends_on:
      - redis

  # Le second service : notre base de données Redis
  redis:
    # On utilise une image officielle depuis Docker Hub
    image: "redis:6-alpine"
    # On attache un volume nommé pour rendre les données persistantes
    volumes:
      - redis-data:/data

# Définition des volumes nommés gérés par Docker
volumes:
  redis-data:
```

</details>

## Partie 3 : Piloter l'Application avec une seule commande \!

Grâce à notre fichier `docker-compose.yml`, la gestion devient un jeu d'enfant.

**1. Lancez toute l'application**
Cette commande va lire votre `docker-compose.yml`, construire l'image `web` si nécessaire, créer un réseau pour les services, et démarrer les conteneurs.

```bash
docker compose up
```

*Pour lancer en arrière-plan, utilisez `docker compose up -d`.*

**2. Vérifiez le fonctionnement**
Ouvrez votre navigateur et allez sur **http://localhost:8000**. Rafraîchissez la page plusieurs fois. Le compteur doit s'incrémenter \!

**3. Visualisez les services**
Ouvrez un second terminal et listez les services gérés par Compose.

```bash
docker compose ps
```

**4. Consultez les logs agrégés**
Affichez les logs de *tous* les services en temps réel. Très pratique pour le débogage.

```bash
docker compose logs -f
```

**5. Arrêtez et nettoyez proprement l'application**
Cette commande arrête les conteneurs ET supprime les ressources associées (conteneurs, réseau par défaut).

```bash
docker compose down
```

*Note : Par défaut, cela ne supprime pas les volumes nommés pour protéger vos données.*

## Partie 4 : Démontrer la puissance de Compose

**1. La persistance des données**

  * Lancez l'application : `docker compose up -d`
  * Rafraîchissez plusieurs fois la page pour que le compteur atteigne, disons, "visiteur 10".
  * Arrêtez tout : `docker compose down`
  * Relancez l'application : `docker compose up -d`
  * Retournez sur `http://localhost:8000`. **Le compteur repart de 11 \!** Le volume nommé `redis-data` a parfaitement conservé l'état de notre base de données.

**2. Le développement en live (live reloading)**

  * Assurez-vous que l'application tourne (`docker compose up -d`).
  * Dans votre éditeur de code, ouvrez `app.py` et modifiez le message de retour :
    `return f'Bienvenue ! Vous êtes la {count}ème personne à visiter ce site.'`
  * Sauvegardez le fichier.
  * **Rafraîchissez simplement la page du navigateur.** Le nouveau message apparaît instantanément, sans avoir à reconstruire l'image ou à redémarrer le conteneur \! C'est le volume `.:/app` qui synchronise votre code en temps réel.

Félicitations \! Vous venez de monter et de piloter une application multi-conteneurs comme un vrai développeur, en utilisant la puissance et la simplicité de Docker Compose.