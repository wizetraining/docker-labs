# LAB 15.2 (Suite) Ingress Controller

Ce lab marque une évolution majeure dans l'exposition de notre application. Nous allons la rendre accessible de manière professionnelle en gérant le routage du trafic entrant et en sécurisant les accès via HTTPS grâce à un objet Ingress.

**Concepts couverts :**

* **Ingress Controller (Nginx)** : Pour gérer le routage du trafic entrant (L7) et la terminaison TLS (HTTPS).
* **Règles de routage avancées** : Utilisation des chemins (paths) et des annotations pour réécrire les URL.
* **Terminaison TLS** : Sécurisation des flux Web à l'aide de certificats.

**Prérequis :**

* Un cluster Kubernetes fonctionnel avec `kubectl` configuré.
* L'application `webapp` (déployée dans les labs précédents) doit être fonctionnelle.

-----

## Partie 1 : Ingress Controller et Exposition Sécurisée

Pour accéder à l'application via un nom de domaine et en HTTPS, nous allons utiliser un Ingress Controller. 
Bien que certains fournisseurs Cloud (comme GKE) proposent leur propre Ingress Controller par défaut, nous utiliserons ici **Nginx Ingress Controller** (déjà pré-installé sur votre cluster de lab) car il offre une excellente gestion des annotations comme `rewrite-target` (nécessaire pour réécrire les chemins HTTP comme `/result` vers `/`).

Nous allons exposer 2 applications :
1. L'application **Countvisit** (`webapp`), déjà installée dans les parties précédentes.
2. L'application **Voting App**, que nous allons déployer.

1.  **Installation de Nginx Ingress Controller (Déjà installer)** :
    Installez le contrôleur Nginx via Helm.

<details>
<summary>Correction - Installation Nginx</summary>

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

</details>

Vérifiez que le service du contrôleur est actif (`Running`)

<details>
<summary>Correction - Vérification Nginx</summary>

```bash
# Vérifier les pods du contrôleur
kubectl get pods -n ingress-nginx

# Vérifier le service (Notez si une IP Externe ou un NodePort est attribué)
kubectl get svc -n ingress-nginx
```

Sur un cluster On-Premise sans LoadBalancer Cloud, notez le **NodePort** attribué au service HTTP/HTTPS. Pour un cluster Cloud, attendez qu'une adresse `EXTERNAL-IP` soit disponible (Dans notre Cluster, vous devriez voir un `EXTERNAL-IP` ).

</details>

2. **Installation de l'application Voting App** : 

    Récupérez les fichiers YAML Kubernetes de la **VotingApp** (Vote, Result, Worker, Redis, DB) situés ici : `https://github.com/dockersamples/example-voting-app/tree/main/k8s-specifications`.
    Installez l'application et testez son bon fonctionnement en interne.

3. **Création des ressources Ingress** :

    Créez les manifestes Ingress pour exposer les deux applications :
    * L'application `countvisit` sera exposée sur `countvisit.wizetraining.com`
    * L'application `VotingApp` sera exposée via `votingapp.wizetraining.com` de sorte à ce que :
      * Par défaut (racine `/`), ce soit le service `vote` qui s'affiche.
      * Les chemins `/vote` et `/result` conduisent respectivement aux services `vote` et `result`. 

<details>
<summary>Correction YAML - Countvisit ingress.yaml</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: countvisit-ingress
  namespace: <votre namespace>  # Remplacez par votre namespace
spec:
  ingressClassName: <?? Que mettrez vous ici. Et pourquoi ?>       # On force l'utilisation de Nginx Ingress Controller
  rules:
  - host: countvisit.wizetraining.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service # Vérifiez le nom exact de votre service
            port:
              number: 80         # Vérifiez l'exactitude du port du Service
```

</details>

<details>
<summary>Correction YAML - VotingApp ingress.yaml</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: votingapp-ingress
  namespace: default # Ajustez si VotingApp est dans un autre namespace
  annotations:
    # Indispensable pour que le chemin /result fonctionne sur le conteneur Result qui attend /
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: <?? Que mettrez vous ici. Et pourquoi ?>          # On force l'utilisation de Nginx Ingress Controller
  rules:
  - host: votingapp.wizetraining.com
    http:
      paths:
      # Règle 1 : La racine pointe vers Vote
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vote
            port:
              number: 5000  # Vérifiez l'exactitude du port
      # Règle 2 : /vote pointe vers Vote
      - path: /vote
        pathType: Prefix
        backend:
          service:
            name: vote
            port:
              number: 5000  
      # Règle 3 : /result pointe vers Result (avec réécriture d'URL via l'annotation)
      - path: /result
        pathType: Prefix
        backend:
          service:
            name: result
            port:
              number: 5001  
```

</details>

Appliquez les fichiers :

```bash
kubectl apply -f countvisit-ingress.yaml
kubectl apply -f votingapp-ingress.yaml
```

**Préparation pour les tests locaux :**
Pour un déploiement en production, on créerait des enregistrements DNS (Type A) pour rediriger `votingapp.wizetraining.com` et `countvisit.wizetraining.com` vers l'IP Loadbalancer du contrôleur Nginx.
Pour faire un test rapide depuis votre poste de travail, modifiez votre fichier `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows) pour y faire une résolution statique locale vers l'IP de votre contrôleur Ingress (ou l'IP de votre nœud Minikube/On-Premise).

4. **Terminaison TLS (HTTPS)** :

L'objet Ingress permet d'exposer des applications Web en HTTPS par une simple référence à un Secret Kubernetes de type `tls` que vous devez créer à l'avance.
Nous allons configurer le HTTPS pour l'application Countvisit.

a. Générez un certificat auto-signé pour le domaine `countvisit.wizetraining.com` et stockez-le dans un Secret Kubernetes nommé `countvisit-tls`.

<details>
<summary>Correction - OpenSSL et Secret</summary>

```bash
# Génération du certificat et de la clé privée
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=countvisit.wizetraining.com"

# Création du secret de type TLS dans Kubernetes
kubectl create secret tls countvisit-tls --key tls.key --cert tls.crt
```

</details>

b. Mettez à jour votre manifeste Ingress pour exposer l'application en HTTPS.

<details>
<summary>Correction YAML - Countvisit HTTPS ingress.yaml</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: countvisit-ingress
  namespace: <votre namespace>
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - countvisit.wizetraining.com
    secretName: countvisit-tls # Le nom du secret créé à l'étape précédente
  rules:
  - host: countvisit.wizetraining.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service 
            port:
              number: 80    
```

</details>

Appliquez la mise à jour :
```bash
kubectl apply -f countvisit-ingress.yaml
```

5. **Test final** :

Accédez à `https://countvisit.wizetraining.com` depuis votre navigateur. Vous devriez avoir une alerte de sécurité (car le certificat est auto-signé et non reconnu par une autorité publique). Acceptez le risque, et vous devriez voir votre application s'afficher de manière sécurisée !