# LAB 3 - Dockerfile et Registre Docker

Cet TP sera effectué sur la machine hôte. Il est recommendé d'avoir complété le Lab 1 & 2.

## Partie 1 : Le Registre Docker et le Dockerfile

### Registre Docker

Un **Registre Docker** est un entrepôt de stockage et de distribution d'images Docker. C'est une sorte de "GitHub" pour les images. **Docker Hub** est le registre public par défaut, utilisé par la communauté mondiale.

* Explorez le Registre public Docker [https://hub.docker.com/](https://hub.docker.com/).
* Créez un compte personnel, vous en aurez besoin pour la suite.

### Dockerfile et la construction d'image

Sur votre machine, créez un répertoire pour ce lab (exemple `lab3-docker`), et à l'intérieur, créez un sous-répertoire `site-web` contenant un fichier `index.html` (**le répertoire `site-web` vous est fourni**).

Créez un fichier nommé `Dockerfile` (sans extension) à la racine de `lab3-docker` pour construire une image qui servira notre site web.

<details><summary>Correction</summary>

```dockerfile
# Étape 1: Partir d'une image de base officielle et légère
FROM nginx:1.25-alpine

# Étape 2: Définir le répertoire de travail à l'intérieur de l'image
WORKDIR /usr/share/nginx/html

# Étape 3: Copier le contenu de notre dossier local `site-web` dans le répertoire de travail de l'image
COPY ./site-web/ .

# Étape 4: Indiquer (pour information) que le conteneur écoutera sur les ports 80 et 443
EXPOSE 80 443
```

*Note : `COPY` est généralement préféré à `ADD` car il est plus explicite.*

</details>


#### Construire l'image de votre application - Build & Tag

Construisez l'image en lui donnant un tag. Remplacez `<votre_login_dockerhub>` par votre identifiant.

<details><summary>Correction</summary>

```bash
docker build -t <votre_login_dockerhub>/simple-app:1.0 .
```

</details>

Connectez-vous à Docker Hub et publiez votre image pour la rendre accessible partout.

<details><summary>Correction</summary>

```bash
# Connexion à Docker Hub
docker login

# Push de l'image
docker push <votre_login_dockerhub>/simple-app:1.0
```

</details>

#### Lancer le conteneur

Lancer à présent le conteneur en mode détaché avec comme nom `simpleapp`

<details><summary>Correction</summary>

```Bash
sudo docker run --name simpleapp -d mpakoupete/simple-app:1.0
```

</details>

Arrivez-vous à accéder aux site sur la machine host ?

<details><summary>Correction</summary>

Non, on ne peut pas y accéder car aucun port n'est mappé entre l'hôte et le conteneur.

</details>

Accéder à l'intérieur du conteneur et vérifier qu'avec le `curl localhost` le site est bien accessible.

<details><summary>Correction</summary>

```Bash
sudo docker ps
# identifiez l'ID de votre conteneur et accédez à l'intérieur du conteneur
sudo docker exec -it b0792f56c7e0 sh
```

</details>

#### Lancer un autre conteneur

Lancez un autre conteneur `simpleapp2` avec le mapping de ports sur le 8080 de votre host
Vérifiez que le site web s'affiche bien `http://localhost:8080`

<details><summary>Correction</summary>

```Bash
sudo docker run --name simpleapp2 -p8080:80 -d mpakoupete/simple-app:1.0
```

</details>

## Partie 2 : Apportez une modification à l'image et mise à jour de l'image - Rebuild après modification

**1. Le problème de la mise à jour**
Modifiez le fichier `site-web/index.html` sur votre machine. Est-ce que le changement est visible sur `http://localhost:8080` ?

<details><summary>Correction</summary>

Non. Le fichier a été copié dans l'image au moment du `build`. Le conteneur utilise sa copie interne, il n'est pas lié à votre fichier local. La solution classique serait de re-construire et re-lancer, ce qui est très lent pour le développement.
Mais pour les raisons du lab nous le ferons et plus tard voir la solution recommandée.

</details>

**2. Modification de l'image**

Faite une modification du fichier `index.html` et faites à nouveau un build de l'image avec pour tag :2
Lancez un autre conteneur `simpleapp2` à partir de cette nouvelle image avec le mapping de ports sur le 8081 de votre poste local.
Vérifiez que le site web s'affiche bien `http://localhost:8081`

<details><summary>Correction</summary>

```Bash
sudo docker build -t mpakoupete/simpleapp:2.0 .
sudo docker run --name simpleapp02 -p8081:80 -d mpakoupete/simpleapp:2.0
```

</details>

**3. La solution recommandée : les Volumes \!**

Lancez un nouveau conteneur en utilisant un **volume (bind mount)** pour synchroniser directement votre dossier local `site-web` avec le dossier servi par Nginx dans le conteneur.

<details><summary>Correction</summary>

```bash
docker run -d --name app-dev -p 8888:80 -v "$(pwd)/site-web":"/usr/share/nginx/html" nginx:1.25-alpine
```

*Note : Nous utilisons l'image `nginx` de base, car notre code est maintenant fourni par le volume, pas par l'image.*

</details>

**4. Vérifier la magie des volumes**

  * Accédez à `http://localhost:8888`. Vous devriez voir votre site.
  * Maintenant, **modifiez à nouveau le fichier `site-web/index.html`** sur votre machine.
  * Rafraîchissez la page de votre navigateur. Le changement apparaît **instantanément** \!


## Partie 3 : Sous le capot : `/var/lib/docker`

Pour finir, jetons un œil à l'endroit où Docker stocke physiquement ses objets (images, volumes, réseaux...) sur votre machine.

**Attention :** Cette partie est pour votre culture. **Ne modifiez JAMAIS manuellement le contenu de ce répertoire \!**

  * Où sont stockées les couches de vos images ?
  * Où sont stockées les données d'un volume nommé (que nous n'avons pas créé, mais c'est le même principe) ?

<details><summary>Correction</summary>

```bash
# Vous aurez besoin de sudo pour explorer ce répertoire système
sudo ls -l /var/lib/docker/overlay2  # Pour les couches d'images
sudo ls -l /var/lib/docker/volumes   # Pour les volumes (dans le lab prochain vous le verrai plus en action)
```

Cela vous montre que tout ce que vous manipulez avec la commande `docker` correspond à des fichiers et des dossiers bien réels sur votre système.

</details>

## Partie 4 : Manipulation Docker

* Lister les images
* Lister les conteneurs
* Stoper le premier conteneur `simpleapp`
* Lister à nouveau les conteneurs en cours d'exécution et ensuite lister tous les conteneurs y compris ceux qui sont stoppés
* supprimer la première image de votre host
* Visualisez les logs du conteneur `simpleapp2` lorsque vous accédez au site web. Ensuite inspectez le.

<details><summary>Correction</summary>

```Bash
sudo docker images
sudo docker image ls
sudo docker container ls
sudo docker stop simpleapp
sudo docker container ls -a
sudo docker logs simpleapp2
sudo docker logs -f  simpleapp2
sudo docker container inspect simpleapp2
```

</details>

## Partie 5 : Image Docker application Python & Registre privé

Vous êtes chargé de développer un compteur de visites web. C'est une application simple mais qui illustre parfaitement une architecture micro-services :

1.  **Un service web** : Une application en Python (Flask) qui affiche le nombre de visiteurs.
2.  **Un service de base de données** : Une base de données Redis qui stocke le compteur.

Gérer les `docker run`, les réseaux et les volumes à la main pour ces deux services serait fastidieux. C'est là que **Docker Compose** entre en jeu : il va nous permettre de décrire toute notre application dans un seul fichier et de la gérer comme un seul bloc.

### Préparation de l'Application

Commençons par créer les fichiers de notre application.

**1. Créez l'arborescence du projet**
Ouvrez votre terminal et créez la structure suivante :

```bash
mkdir lab3 && cd lab3
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

<details><summary>Correction - Contenu du `Dockerfile`</summary>

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

### Buildez l'application et la mettre dans le régistre privé Harbor

C'est la partie critique pour une entreprise. Nous ne voulons pas stocker notre code sur le Docker Hub public, mais sur notre propre registre sécurisé.

#### 3.1 Configuration du projet dans Harbor

1.  Connectez-vous à l'interface web : **https://harbor.mpakoupete.com**.
2.  Cliquez sur le bouton **"+ NOUVEAU PROJET"**.
3.  Remplissez le formulaire :
      * **Nom du projet :** `<votre prenom>-lab3-python` (ou votre nom d'utilisateur).
      * **Accès :** Public ou Privé (gardez *Privé* pour ce test).
      * **Configuration :** 
        * Cochez la case **"Scanner automatiquement les images à l'envoi"** (Automatically scan images on push).
        * Cochez également la case **SBOM**
4.  Validez la création.

#### 3.2 Build et Tag de l'image

Pour envoyer une image vers un registre privé, il faut respecter une convention de nommage stricte : `URL_REGISTRE/NOM_PROJET/NOM_IMAGE:TAG`.

1.  Construisez l'image localement :

    ```bash
    docker build -t mon-python-app:v1 .
    ```

2.  Appliquez le tag pour Harbor (si nécessaire remplacez `<votre prenom>-lab3-python` par le nom de projet que vous avez créé à l'étape 3.1)  :

    ```bash
    # Syntaxe : docker tag <image_locale> harbor.mpakoupete.com/<projet>/<image>:<version>
    docker tag mon-python-app:v1 harbor.mpakoupete.com/<votre prenom>-lab3-python/compteur-visite:v1
    ```

#### 3.3 Authentification et Push

1.  Connectez votre client Docker au registre distant :

    ```bash
    docker login harbor.mpakoupete.com
    ```

    *Entrez vos identifiants fournis par l'administrateur.*

2.  Envoyez (Push) l'image :

    ```bash
    docker push harbor.mpakoupete.com/<votre prenom>-lab3-python/compteur-visite:v1
    ```

#### 3.4 Analyse de vulnérabilités

Retournez sur l'interface web de Harbor :

1.  Entrez dans votre projet `<votre prenom>-lab3-python`.
2.  Cliquez sur le dépôt `compteur-visite`.
3.  Regardez la colonne **"Vulnerabilities"**.
      * Si le scan est "Queued" (en file d'attente), attendez quelques secondes et rafraîchissez.
      * Si Harbor détecte des failles critiques (Critical), il pourrait bloquer le téléchargement de cette image selon vos réglages \! C'est le principe du "Security Gate".


