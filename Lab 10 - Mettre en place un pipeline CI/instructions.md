# Lab 10 : Intégration CI/CD GitLab & Harbor pour le Build d'Images Docker

## Objectif

Mettre en place un pipeline GitLab CI qui construit automatiquement une image Docker et la pousse vers un registre privé sécurisé (Harbor) à chaque *push* sur la branche principale (`main`).

## Partie 1 : Introduction et Test du Registre Docker Standard

Cette partie permet de comprendre le concept d'un registre sans les complexités de la sécurité de production.

1.  **Démarrer un Registre Privé Local (`registry:2`) :**
    Exécutez cette commande sur votre machine locale pour lancer un registre Docker non sécurisé (HTTP) sur le port 5000 :

    ```bash
    docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
    ```

2.  **Activer l'Accès HTTP pour le Test :**
    Pour que votre Docker CLI puisse se connecter à un registre non sécurisé, vous devez temporairement l'ajouter à la liste d'accès (édition de `/etc/docker/daemon.json` ou `~/.docker/daemon.json` sur Linux/macOS, ou via l'interface de Docker Desktop sur Windows). 
    
    ```json
      {
        "insecure-registries": [
          "localhost:5000"
        ]
      }
    ```
    
    Redémarrez le démon Docker après modification.

    ```bash
    sudo systemctl restart docker
    ```

3.  **Test Manuel (Build & Push) :**

      * Créez un simple `Dockerfile` dans un répertoire.
      * Construisez et poussez une image :
        ```bash
        docker build -t localhost:5000/test-app:v1 .
        docker push localhost:5000/test-app:v1
        ```

4.  **🔍 Exploration et Vérification (Accès Anonyme) :**

      * **Via le Navigateur :** Ouvrez votre navigateur et allez à l'adresse suivante : **`http://localhost:5000/v2/_catalog`**.
          * **Observation :** Vous devriez voir une réponse JSON listant le dépôt `test-app` que vous venez de pousser.
      * **Via `curl` (API) :** Exécutez cette commande dans votre terminal :
        ```bash
        curl http://localhost:5000/v2/_catalog
        ```
          * **Observation :** La commande réussit sans aucune authentification.

5.  **Conclusion :** Félicitations, vous avez un registre. Il est simple et fonctionnel, mais **l'accès anonyme en lecture et en écriture n'est pas sécurisé ni idéal pour la production**. Ce manque de contrôle justifie l'utilisation de **Harbor**.

-----

## Partie 2 : Exploration et Utilisation du Registre de Production (Harbor)

En production, nous utilisons des solutions robustes comme **Harbor** pour la sécurité (HTTPS, Scan de Vulnérabilité, Governance).

1.  **Accès à Harbor :**
    Ouvrez l'interface web de votre registre privé de production : **`https://harbor.mpakoupete.com`**.

      * **Login :** Utilisez le `login` fourni dans le fichier partagé.
      * **Mot de passe :** Utilisez le `mot de passe` fourni dans le fichier partagé.

2.  **Créer un Projet Privé :**

      * Allez dans la section **Projects** (Projets).
      * Cliquez sur **New Project**.
      * **Nom du Projet :** `votre-login-images` (ex: `mpakoupete-images`).
      * **Access Level :** Cochez **Private** (Privé).
      * Cliquez sur **OK**.

3.  **Test Manuel (Build & Push vers Harbor) :**

      * Réutilisez votre `Dockerfile` local.
      * Connectez-vous à Harbor :
        ```bash
        docker login harbor.mpakoupete.com
        # (Utilisez votre login/mot de passe d'utilisateur Harbor)
        ```
      * Taguage et Push :
        ```bash
        docker tag local-app:latest harbor.mpakoupete.com/votre-login-images/mon-app:v1
        docker push harbor.mpakoupete.com/votre-login-images/mon-app:v1
        ```

4.  **Explorer Harbor et Activer le Scan :**

      * Dans l'interface Harbor, allez dans votre nouveau projet.
      * Cliquez sur le dépôt `mon-app`.
      * **Observation :** Harbor démarre automatiquement le **scan de vulnérabilités (Trivy)**. Observez le résultat (Common Vulnerabilities and Exposures - CVEs).

-----

## Partie 3 : Préparation de l'Environnement GitLab et CI/CD

Nous allons automatiser le *build* et le *push* vers Harbor via GitLab CI.

1.  **Accès et Compte GitLab :**

      * Allez sur **`http://gitlab.mpakoupete.com/`** et créez votre compte en utilisant le même `login` que celui du fichier partagé.

2.  **Création du Projet et Clé SSH :**

      * Créez un nouveau projet privé dans GitLab (ex: `docker-builder-lab`).
      * **Générez et ajoutez votre clé SSH** :
        ```bash
        ssh-keygen -t ed25519 -C "votre_prenom@gitlab.mpakoupete.com"
        cat ~/.ssh/id_ed25519.pub
        ```
      * Collez la clé dans **GitLab $\rightarrow$ Profil $\rightarrow$ SSH Keys**.

3.  **Configuration Locale et Code :**

      * Clonez le projet via SSH :
        ```bash
        git clone git@gitlab.mpakoupete.com:votre_login/docker-builder-lab.git
        cd docker-builder-lab
        ```
      * Créez un fichier **`Dockerfile`** (même le plus simple) et un fichier **`.gitlab-ci.yml`**.

4.  **Configuration des Variables Secrètes (CRITIQUE) :**
    Pour permettre à votre pipeline de s'authentifier auprès de Harbor, vous devez définir les variables CI/CD.

      * Allez dans votre projet GitLab $\rightarrow$ **Settings** (Paramètres) $\rightarrow$ **CI/CD**.
      * Développez **Variables**.
      * Ajoutez les variables suivantes (marquées **Protected** et **Masked**) :
        | Clé | Valeur | Rôle |
        | :--- | :--- | :--- |
        | `DOCKER_USER` | Votre login Harbor (ex: `mpakoupete`) | Nom d'utilisateur pour le `docker login`. |
        | `DOCKER_PASSWORD` | Votre mot de passe Harbor | Mot de passe/Jeton pour le `docker login`. |

5.  **Fichier `.gitlab-ci.yml` (Code Fourni) :**
    Créez ce fichier dans votre répertoire local :

    Vous pouvez consulter sur ce lien la liste des Variable CI/CD prédéfinies : https://docs.gitlab.com/ci/variables/predefined_variables/#predefined-variables

    <details><summary>Fichier .gitlab-ci.yml</summary>

    ```yaml
    # .gitlab-ci.yml
    stages:
      - build
      - push

    variables:
      DOCKER_REGISTRY_URL: "harbor.mpakoupete.com" 
      # Utilise le hachage pour le tag (sera changé en Partie 4)
      # IMAGE_NAME: "$DOCKER_REGISTRY_URL/$CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA"
      IMAGE_NAME: "$DOCKER_REGISTRY_URL/<votre-registre>/<nom-image>:$CI_COMMIT_SHORT_SHA"
      # Example : "$DOCKER_REGISTRY_URL/jdupond-lab10/app-countvisit:$CI_COMMIT_SHORT_SHA"
      # Assurez vous d'avoir créé dans le Repository Harbor le projet (ex : jdupond-lab10)

    # Job 1: Build de l'Image Docker
    build_image:
      stage: build
      tags:
        - docker 
      script:
        - echo "Building Docker image..."
        - docker build -t $IMAGE_NAME .

    # Job 2: Push de l'Image vers le Registre Privé
    push_image:
      stage: push
      tags:
        - docker
      script:
        - echo "Logging into Docker Registry..."
        - docker login $DOCKER_REGISTRY_URL -u "$DOCKER_USER" -p "$DOCKER_PASSWORD"
        
        - echo "Pushing image to $IMAGE_NAME"
        - docker push $IMAGE_NAME

      rules:
        # Pousser uniquement si on est sur la branche par défaut (main)
        - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    ```

    </details>

6.  **Test Initial du Pipeline :**

      * Ajoutez et poussez votre code :
        ```bash
        git add .
        git commit -m "Initial pipeline and Dockerfile"
        git push origin main
        ```
      * **Vérifiez le Pipeline :** Allez dans GitLab $\rightarrow$ **CI/CD** $\rightarrow$ **Pipelines**.
      * **Le pipeline marche-t-il ?** Si vous avez géré les problèmes de certificat (Harbor) et de tags (`docker`) sur le Runner, le pipeline devrait réussir.
      * **Vérifiez votre repo Harbor :** Après succès, l'image sera visible dans votre projet Harbor `votre-login-images` avec un tag **hash** (ex: `b5cdc5d3`).

-----

## Partie 4 : Mise à Jour et Amélioration (Tags Sémantiques)

### 1\. Pourquoi le Tag est un Hash ?

  * **Explication :** Le tag est un hash (`$CI_COMMIT_SHORT_SHA`) car la variable `IMAGE_NAME` dans le pipeline est configurée pour utiliser cet identifiant court du commit Git. Il est utilisé pour la traçabilité.

### 2\. Modification pour le Tag Sémantique (`1.2.0`)

Nous allons utiliser la variable **`$CI_COMMIT_TAG`** qui ne s'active que lorsque vous créez un tag Git.

1.  **Modifiez le `.gitlab-ci.yml`** (changez la variable et ajoutez une règle plus stricte dans les deux Jobs) :

    ```yaml
    # .gitlab-ci.yml

    # ⬇️ CHANGEMENT : Utiliser la variable $CI_COMMIT_TAG
    variables:
      DOCKER_REGISTRY_URL: "harbor.mpakoupete.com" 
      IMAGE_NAME: "$DOCKER_REGISTRY_URL/$CI_PROJECT_PATH:$CI_COMMIT_TAG" # ⬅️ MODIFIÉ

    # ...

    push_image:
      # ... (reste du job)
      rules:
        # ⬇️ MODIFIÉ : Pousser UNIQUEMENT quand un tag Git est créé
        - if: $CI_COMMIT_TAG
          when: on_success
        - when: never # Évite l'exécution sur les branches normales
    ```

2.  **Test du Bon Fonctionnement (Workflow de Versionnement) :**

      * Poussez la modification du pipeline :

        ```bash
        git add .gitlab-ci.yml
        git commit -m "Feature: Use semantic tagging"
        git push origin main
        ```

        *Observation : Ce push ne fait rien (car la règle `when: never` bloque).*

      * **Créez et poussez un Tag de Version :**

        ```bash
        git tag 1.2.0
        git push origin 1.2.0 # ⬅️ DÉCLENCHE LE PIPELINE
        ```

      * **Vérifiez :** Le pipeline s'exécute. L'image est poussée vers Harbor avec le tag **`harbor.mpakoupete.com/votre_login/mon-app:1.2.0`**.

      Si vous rencontrez cette erreur ci-dessous, c'est parce que vos variables `DOCKER_USER` et `DOCKER_PASSWORD` sont des variables protégées. Dans GitLab ne sont disponibles que pour les pipelines lancés sur des branches protégées (typiquement main, master, ou des tags protégés).

      ```
      (...)
      Logging into Docker Registry...
      $ docker login $DOCKER_REGISTRY_URL -u "$DOCKER_USER" -p "$DOCKER_PASSWORD"
      username is empty
      Cleaning up project directory and file based variables 00:01
      ERROR: Job failed: exit code 1
      ```

      **Solutions**
      * Décoche l’option “Protected”
      * Ou protéger la branche tag : Aller dans `Settings` > `Repository` > `Protected Branches` et ajouter la branche à la Liste



