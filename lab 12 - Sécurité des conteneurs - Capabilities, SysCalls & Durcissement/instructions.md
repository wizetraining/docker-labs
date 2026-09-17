# Lab 12 : Sécurité des conteneurs (Capabilities, SysCalls & Durcissement)

**Objectif :** Comprendre concrètement pourquoi un conteneur n'est pas une frontière de sécurité par défaut, puis construire cette frontière couche par couche : capabilities Linux, filtrage des appels système (seccomp), contrôle d'accès obligatoire (AppArmor), et durcissement du système de fichiers et des ressources.

**Prérequis :**
  * Une **VM jetable** sous Linux (Ubuntu 22.04+ / Debian 12+), avec **cgroups v2**.
  * Docker Engine 24+ et accès `sudo`.
  * Paquets utiles : `libcap2-bin` (pour `capsh`), `apparmor-utils`, `strace`, `jq`.
  * **Avertissement :** la Partie 2 réalise de véritables évasions de conteneur. Elle ne doit être exécutée **que** sur une VM jetable, jamais sur un poste de travail ni sur un serveur partagé.

```bash
sudo apt update
sudo apt install -y libcap2-bin apparmor-utils strace jq
```

<details><summary>Réinitialisation de l'environnement du lab</summary>

```bash
cat > /tmp/reset-container-lab.sh <<'EOF'
#!/usr/bin/env bash

# ============================================================
# RESET COMPLET DOCKER / PODMAN / CONTAINERD
# ============================================================

set -u

export DEBIAN_FRONTEND=noninteractive

echo
echo "============================================================"
echo "       RESET COMPLET DOCKER / PODMAN / CONTAINERD"
echo "============================================================"
echo
echo "Ce script va supprimer :"
echo
echo "  - Docker"
echo "  - Podman"
echo "  - containerd"
echo "  - Buildah / Skopeo"
echo "  - conteneurs"
echo "  - images"
echo "  - volumes"
echo "  - réseaux"
echo "  - caches"
echo "  - credentials Docker"
echo "  - configuration Docker"
echo "  - configuration Podman"
echo "  - stockage rootless"
echo "  - services systemd"
echo "  - installations Snap"
echo
echo "AUCUNE DONNÉE DOCKER NE SERA CONSERVÉE."
echo
read -r -p "Tape exactement RESET pour continuer : " CONFIRM

if [[ "$CONFIRM" != "RESET" ]]; then
    echo
    echo "Annulation."
    exit 1
fi

echo
echo "============================================================"
echo "DÉBUT DU NETTOYAGE"
echo "============================================================"
echo


# ============================================================
# 1. SERVICES
# ============================================================

echo "[1/12] Arrêt des services..."

sudo systemctl stop docker.service 2>/dev/null || true
sudo systemctl stop docker.socket 2>/dev/null || true
sudo systemctl stop containerd.service 2>/dev/null || true
sudo systemctl stop podman.service 2>/dev/null || true
sudo systemctl stop podman.socket 2>/dev/null || true

systemctl --user stop docker.service 2>/dev/null || true
systemctl --user stop docker.socket 2>/dev/null || true
systemctl --user stop podman.service 2>/dev/null || true
systemctl --user stop podman.socket 2>/dev/null || true


# ============================================================
# 2. SUPPRESSION DES CONTENEURS DOCKER
# ============================================================

echo "[2/12] Suppression des conteneurs Docker..."

if command -v docker >/dev/null 2>&1; then
    docker ps -aq 2>/dev/null | xargs -r docker rm -f 2>/dev/null || true
fi

if sudo docker version >/dev/null 2>&1; then
    sudo docker ps -aq 2>/dev/null | xargs -r sudo docker rm -f 2>/dev/null || true
fi


# ============================================================
# 3. SUPPRESSION PODMAN
# ============================================================

echo "[3/12] Suppression des conteneurs/images/volumes Podman..."

if command -v podman >/dev/null 2>&1; then
    podman stop -a -t 0 2>/dev/null || true
    podman rm -a -f 2>/dev/null || true
    podman system reset --force 2>/dev/null || true
fi

if sudo podman version >/dev/null 2>&1; then
    sudo podman stop -a -t 0 2>/dev/null || true
    sudo podman rm -a -f 2>/dev/null || true
    sudo podman system reset --force 2>/dev/null || true
fi


# ============================================================
# 4. KILL DES PROCESSUS RESTANTS
# ============================================================

echo "[4/12] Arrêt forcé des processus restants..."

sudo pkill -9 dockerd 2>/dev/null || true
sudo pkill -9 containerd 2>/dev/null || true
sudo pkill -9 containerd-shim 2>/dev/null || true
sudo pkill -9 docker-proxy 2>/dev/null || true
sudo pkill -9 podman 2>/dev/null || true
sudo pkill -9 conmon 2>/dev/null || true
sudo pkill -9 buildah 2>/dev/null || true
sudo pkill -9 pasta 2>/dev/null || true
sudo pkill -9 slirp4netns 2>/dev/null || true


# ============================================================
# 5. SUPPRESSION DES DONNÉES DOCKER
# ============================================================

echo "[5/12] Suppression complète du stockage Docker..."

sudo rm -rf \
    /var/lib/docker \
    /var/lib/containerd \
    /var/cache/docker \
    /etc/docker \
    /run/docker \
    /run/containerd

rm -rf \
    "$HOME/.docker" \
    "$HOME/.config/docker" \
    "$HOME/.cache/docker" \
    "$HOME/.local/share/docker"


# ============================================================
# 6. SUPPRESSION DES DONNÉES PODMAN / CONTAINERS
# ============================================================

echo "[6/12] Suppression complète du stockage Podman..."

rm -rf \
    "$HOME/.config/containers" \
    "$HOME/.local/share/containers" \
    "$HOME/.cache/containers" \
    "$HOME/.local/share/podman" \
    "$HOME/.config/podman" \
    "$HOME/.cache/podman"

sudo rm -rf \
    /etc/containers \
    /var/lib/containers \
    /var/cache/containers \
    /run/containers


# ============================================================
# 7. BUILDAH / SKOPEO / CRI-O
# ============================================================

echo "[7/12] Suppression Buildah / Skopeo / CRI-O..."

rm -rf \
    "$HOME/.config/buildah" \
    "$HOME/.local/share/buildah" \
    "$HOME/.config/skopeo"

sudo rm -rf \
    /etc/crio \
    /var/lib/crio \
    /var/log/crio


# ============================================================
# 8. PAQUETS APT
# ============================================================

echo "[8/12] Suppression des paquets APT..."

if command -v apt-get >/dev/null 2>&1; then

    sudo apt-get purge -y \
        docker-ce \
        docker-ce-cli \
        docker-ce-rootless-extras \
        docker-buildx-plugin \
        docker-compose-plugin \
        docker-compose \
        docker.io \
        docker-doc \
        docker-registry \
        containerd \
        containerd.io \
        runc \
        podman \
        podman-docker \
        buildah \
        skopeo \
        cri-o \
        cri-o-runc \
        2>/dev/null || true

    sudo apt-get autoremove -y 2>/dev/null || true
    sudo apt-get autoclean -y 2>/dev/null || true
fi


# ============================================================
# 9. SNAP — DOCKER
# ============================================================

echo "[9/12] Suppression Docker Snap..."

if command -v snap >/dev/null 2>&1; then

    # Suppression avec --purge pour empêcher la création
    # d'un snapshot automatique des données Docker.
    sudo snap remove docker --purge 2>/dev/null || true

    # Recherche et suppression des snapshots Docker existants.
    SNAP_IDS=$(sudo snap saved 2>/dev/null | awk '$2 == "docker" {print $1}')

    if [[ -n "${SNAP_IDS:-}" ]]; then
        for ID in $SNAP_IDS; do
            echo "Suppression du snapshot Snap Docker #$ID..."
            sudo snap forget "$ID" 2>/dev/null || true
        done
    fi

    # Au cas où un ancien paquet Docker Snap aurait laissé
    # des données résiduelles.
    sudo rm -rf \
        /var/snap/docker \
        /snap/docker \
        /var/lib/snapd/snap/docker
fi


# ============================================================
# 10. SERVICES SYSTEMD / CONFIGURATION
# ============================================================

echo "[10/12] Nettoyage systemd..."

sudo rm -f \
    /etc/systemd/system/docker.service \
    /etc/systemd/system/docker.socket \
    /etc/systemd/system/containerd.service \
    /etc/systemd/system/podman.service \
    /etc/systemd/system/podman.socket

rm -f \
    "$HOME/.config/systemd/user/docker.service" \
    "$HOME/.config/systemd/user/docker.socket" \
    "$HOME/.config/systemd/user/podman.service" \
    "$HOME/.config/systemd/user/podman.socket"

sudo systemctl daemon-reload 2>/dev/null || true
systemctl --user daemon-reload 2>/dev/null || true


# ============================================================
# 11. GROUPE DOCKER + CONFIGURATION RÉSEAU
# ============================================================

echo "[11/12] Nettoyage groupe Docker et interfaces réseau..."

if getent group docker >/dev/null 2>&1; then
    sudo groupdel docker 2>/dev/null || true
fi

# Suppression des interfaces réseau Docker/Podman restantes.
for IFACE in docker0 cni0 podman0; do
    if ip link show "$IFACE" >/dev/null 2>&1; then
        sudo ip link delete "$IFACE" 2>/dev/null || true
    fi
done

# Nettoyage éventuel des fichiers CNI.
sudo rm -rf \
    /etc/cni \
    /var/lib/cni \
    "$HOME/.config/cni" \
    "$HOME/.local/share/cni"


# ============================================================
# 12. NETTOYAGE FINAL
# ============================================================

echo "[12/12] Nettoyage final..."

rm -rf \
    "$HOME/.docker" \
    "$HOME/.config/docker" \
    "$HOME/.cache/docker" \
    "$HOME/.local/share/docker" \
    "$HOME/.config/containers" \
    "$HOME/.local/share/containers" \
    "$HOME/.cache/containers" \
    "$HOME/.config/podman" \
    "$HOME/.local/share/podman" \
    "$HOME/.cache/podman"

sudo rm -rf \
    /var/lib/docker \
    /var/lib/containerd \
    /var/lib/containers \
    /etc/docker \
    /etc/containers \
    /etc/cni \
    /var/lib/cni \
    /run/docker \
    /run/containerd \
    /run/containers

sudo systemctl daemon-reload 2>/dev/null || true


# ============================================================
# VÉRIFICATION
# ============================================================

echo
echo "============================================================"
echo "VÉRIFICATION FINALE"
echo "============================================================"
echo

echo "=== COMMANDES ==="

command -v docker >/dev/null 2>&1 \
    && echo "ATTENTION : docker existe encore" \
    || echo "OK : docker absent"

command -v dockerd >/dev/null 2>&1 \
    && echo "ATTENTION : dockerd existe encore" \
    || echo "OK : dockerd absent"

command -v docker-compose >/dev/null 2>&1 \
    && echo "ATTENTION : docker-compose existe encore" \
    || echo "OK : docker-compose absent"

command -v podman >/dev/null 2>&1 \
    && echo "ATTENTION : podman existe encore" \
    || echo "OK : podman absent"

command -v buildah >/dev/null 2>&1 \
    && echo "ATTENTION : buildah existe encore" \
    || echo "OK : buildah absent"

command -v skopeo >/dev/null 2>&1 \
    && echo "ATTENTION : skopeo existe encore" \
    || echo "OK : skopeo absent"

echo
echo "=== SNAP ==="

if command -v snap >/dev/null 2>&1; then
    if snap list 2>/dev/null | grep -Eiq 'docker|podman'; then
        echo "ATTENTION : Docker/Podman présent dans Snap"
        snap list 2>/dev/null | grep -Ei 'docker|podman' || true
    else
        echo "OK : aucun Docker/Podman Snap"
    fi
else
    echo "OK : Snap non installé"
fi

echo
echo "=== SERVICES ==="

SERVICES=$(systemctl list-unit-files 2>/dev/null | grep -Ei 'docker|containerd|podman' || true)

if [[ -n "$SERVICES" ]]; then
    echo "ATTENTION : services trouvés :"
    echo "$SERVICES"
else
    echo "OK : aucun service Docker/Podman/containerd"
fi

echo
echo "=== PROCESSUS ==="

PROCS=$(ps aux | grep -E '[d]ockerd|[c]ontainerd|[p]odman|[c]onmon' || true)

if [[ -n "$PROCS" ]]; then
    echo "ATTENTION : processus encore actifs :"
    echo "$PROCS"
else
    echo "OK : aucun processus Docker/Podman"
fi

echo
echo "=== RÉPERTOIRES ==="

DIRS=(
    "$HOME/.docker"
    "$HOME/.config/docker"
    "$HOME/.config/containers"
    "$HOME/.local/share/containers"
    "$HOME/.config/podman"
    "$HOME/.local/share/podman"
    /etc/docker
    /etc/containers
    /var/lib/docker
    /var/lib/containerd
    /var/lib/containers
    /var/snap/docker
)

ALL_CLEAN=true

for DIR in "${DIRS[@]}"; do
    if [[ -e "$DIR" ]]; then
        echo "ATTENTION : $DIR existe encore"
        ALL_CLEAN=false
    else
        echo "OK : $DIR"
    fi
done

echo
echo "============================================================"

if [[ "$ALL_CLEAN" == true ]]; then
    echo "        RESET TERMINÉ — ENVIRONNEMENT PROPRE"
else
    echo "        RESET TERMINÉ AVEC QUELQUES RESTES"
fi

echo "============================================================"
echo
echo "Docker / Podman / containerd ont été nettoyés."
echo
echo "IMPORTANT : les credentials Docker/Harbor qui étaient"
echo "présents dans ~/.docker/config.json doivent être considérés"
echo "comme compromis. Révoque/renouvelle les tokens concernés."
echo
EOF

chmod +x /tmp/reset-container-lab.sh
/tmp/reset-container-lab.sh
```

</details>

-----

## PARTIE 1 : Cartographier la surface d'attaque

Avant de durcir, il faut mesurer. Nous allons observer ce qu'un conteneur reçoit **par défaut**.

### Exercice 1 : Quelles capabilities mon conteneur possède-t-il ?

**Concept :** Docker n'accorde pas tous les privilèges de root, mais il en accorde 14. Voyons lesquelles.

1. Lancez un conteneur et affichez ses capabilities :

```bash
docker run --rm -it --name cap-test alpine sh -c \
  'apk add -q libcap; capsh --print | head -20'
```

2. Observez la ligne `Current:` : elle liste les capabilities effectivement détenues.

3. Comparez avec la vue brute du noyau, en hexadécimal :

```bash
docker run --rm alpine grep -E 'CapPrm|CapEff|CapBnd' /proc/self/status
```

4. Décodez ce masque de bits — on lit la valeur réelle plutôt que de la coder en dur, car elle varie selon la version de Docker :

```bash
MASQUE=$(docker run --rm alpine sh -c \
  'grep CapEff /proc/self/status | awk "{print \$2}"')
echo "Masque effectif : $MASQUE"

docker run --rm alpine sh -c \
  "apk add -q libcap; capsh --decode=${MASQUE}"
```

*Observation :* vous devez retrouver les 14 capabilities par défaut, dont `cap_net_raw`, `cap_chown`, `cap_setuid`, `cap_dac_override`.

### Exercice 2 : Et avec `--privileged` ?

1. Relancez la même commande, en mode privilégié :

```bash
docker run --rm --privileged alpine sh -c \
  'apk add -q libcap; capsh --print | grep Current'
```

2. Comparez les devices visibles dans les deux cas :

```bash
echo "--- Conteneur normal ---"
docker run --rm alpine sh -c 'ls /dev | wc -l'

echo "--- Conteneur privilégié ---"
docker run --rm --privileged alpine sh -c 'ls /dev | wc -l'
```

3. Vérifiez si les disques de l'hôte sont visibles depuis le conteneur privilégié :

```bash
docker run --rm --privileged alpine sh -c 'ls -l /dev/sd* /dev/vd* /dev/nvme* 2>/dev/null'
```

*Observation :* en mode privilégié, **toutes** les capabilities sont accordées et les disques de l'hôte sont directement accessibles. Un attaquant peut monter le disque système et lire `/etc/shadow`.

### Exercice 3 : L'état de seccomp et d'AppArmor

1. Vérifiez que seccomp est actif dans un conteneur normal :

```bash
docker run --rm alpine grep Seccomp /proc/self/status
```

Les valeurs possibles sont : `0` (désactivé), `1` (strict), **`2` (mode filtre — c'est ce que nous attendons)**.

2. Vérifiez maintenant en mode privilégié :

```bash
docker run --rm --privileged alpine grep Seccomp /proc/self/status
```

3. Regardez ce que le daemon annonce comme options de sécurité :

```bash
docker info -f '{{.SecurityOptions}}'
```

4. Inspectez le profil AppArmor appliqué à un conteneur en cours d'exécution :

```bash
docker run -d --name cap-test alpine sleep 600
docker inspect cap-test -f 'AppArmor={{.AppArmorProfile}}'
docker inspect cap-test -f 'SecurityOpt={{.HostConfig.SecurityOpt}}'
docker rm -f cap-test
```

*Conclusion de la partie 1 :* **`--privileged` désactive simultanément les capabilities restreintes, seccomp et AppArmor.** C'est la première chose à bannir en production.

-----

## PARTIE 2 : Les évasions (à exécuter uniquement sur une VM jetable)

Cette partie démontre que certaines options courantes donnent un accès **root complet à l'hôte**. L'objectif n'est pas d'apprendre à attaquer, mais de comprendre **pourquoi** ces options sont interdites en production.

### Exercice 1 : Évasion par montage de la racine de l'hôte

1. Déposez d'abord un fichier témoin sur l'hôte :

```bash
echo "fichier de l'hote" | sudo tee /root/temoin-hote.txt
```

2. Lancez un conteneur en montant la racine de l'hôte :

```bash
docker run --rm -it --name evasion -v /:/host alpine sh
```

3. **À l'intérieur du conteneur**, changez de racine et regardez :

```bash
chroot /host /bin/sh

# Vous êtes maintenant root sur l'hôte
cat /root/temoin-hote.txt
head -3 /etc/shadow
hostname
exit
exit
```

*Observation :* un simple `-v /:/host` suffit. Aucune faille, aucun exploit : c'est le comportement attendu de Docker.

### Exercice 2 : Évasion par le socket Docker

1. Montez le socket Docker dans un conteneur :

```bash
docker run --rm -it --name sock-evasion \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli sh
```

2. **À l'intérieur**, vous pilotez le daemon de l'hôte :

```bash
docker ps
docker images

# Et donc vous pouvez lancer un conteneur privilégié
docker run --rm -v /:/host alpine cat /host/root/temoin-hote.txt
exit
```

*Observation :* **accès au socket = root sur l'hôte.** C'est pour cela qu'appartenir au groupe `docker` équivaut à `sudo` sans mot de passe.

3. Vérifiez qui est concerné sur votre machine :

```bash
getent group docker
ls -l /var/run/docker.sock
```

### Exercice 3 : Les namespaces de l'hôte

1. Comparez la vue des processus :

```bash
echo "--- Isolation normale ---"
docker run --rm alpine ps aux | wc -l

echo "--- Avec --pid=host ---"
docker run --rm --pid=host alpine ps aux | wc -l
```

2. Avec `--pid=host`, le conteneur voit — et peut signaler — les processus de l'hôte :

```bash
docker run --rm --pid=host alpine ps aux | grep -E 'dockerd|systemd' | head -5
```

3. Et avec `--net=host`, il partage la pile réseau :

```bash
docker run --rm --net=host alpine ip addr | grep -E '^[0-9]+:'
```

*Conclusion de la partie 2 :* quatre options à traiter comme des interdictions par défaut : **`--privileged`**, **`-v /:/...`**, **`-v /var/run/docker.sock`**, **`--pid=host` / `--net=host` / `--ipc=host`**.

Nettoyez le fichier témoin :

```bash
sudo rm -f /root/temoin-hote.txt
```

-----

## PARTIE 3 : Les Capabilities Linux

Nous passons à la construction de la frontière. Première couche : ne donner que les privilèges nécessaires.

### Exercice 1 : `CAP_NET_RAW` et le `arping`

**Concept :** `arping` a besoin de créer une socket RAW. C'est un bon révélateur de `CAP_NET_RAW`, accordée par défaut — et qui permet aussi le spoofing ARP/IP.

1. Par défaut, le `arping` fonctionne :

```bash
docker run --rm alpine sh -c "apk add --no-cache iputils-arping && arping -c 2 172.17.0.1"
```

2. Retirez la capability :

```bash
docker run --rm --cap-drop=NET_RAW alpine sh -c "apk add --no-cache iputils-arping && arping -c 2 172.17.0.1"
```

*Résultat attendu :* 
```
(1/2) Installing libcap2 (2.78-r0)
(2/2) Installing iputils-arping (20250605-r2)
Executing busybox-1.37.0-r31.trigger
OK: 8269 KiB in 18 packages
arping: socket: Operation not permitted             <------------ Permission non accordée
```

3. Le CIS recommande de retirer `NET_RAW` : votre application web n'en a aucun besoin.

### Exercice 2 : `CAP_NET_BIND_SERVICE` et les ports privilégiés

**Concept :** écouter sous le port 1024 demande un privilège. C'est le seul dont un serveur web a réellement besoin.

1. Préparez un répertoire de travail :

```bash
mkdir -p ~/lab-a1 && cd ~/lab-a1
```

2. Tentez d'écouter sur le port 80 en tant qu'utilisateur non privilégié, **sans** la capability :

```bash
docker run --rm -u 10001 --cap-drop=ALL alpine sh -c \
  'nc -l -p 80 & sleep 1; wait' 2>&1 | head -3
```

*Résultat attendu :* un échec (`Permission denied`).

3. Rajoutez uniquement la capability nécessaire :

```bash
docker run --rm -u 10001 --cap-drop=ALL --cap-add=NET_BIND_SERVICE alpine sh -c \
  'timeout 2 nc -l -p 80; echo "ecoute sur le port 80 acceptee"'
```

*Observation :* c'est exactement le principe de moindre privilège — **tout retirer, puis rajouter une seule chose**.

### Exercice 3 : `CAP_CHOWN` et `CAP_DAC_OVERRIDE`

1. Dans un conteneur par défaut, root peut changer le propriétaire de n'importe quel fichier :

```bash
docker run --rm alpine sh -c \
  'touch /tmp/f && chown 1234:1234 /tmp/f && ls -l /tmp/f'
```

2. Retirez `CHOWN` :

```bash
docker run --rm --cap-drop=CHOWN alpine sh -c \
  'touch /tmp/f && chown 1234:1234 /tmp/f' 2>&1 | head -2
```

*Résultat attendu :* `chown: /tmp/f: Operation not permitted`.

3. `DAC_OVERRIDE` permet à root de contourner les permissions de fichiers. Observez la différence :

```bash
# Avec DAC_OVERRIDE (défaut) : root lit un fichier en mode 000
docker run --rm alpine sh -c \
  'echo secret > /tmp/f && chmod 000 /tmp/f && cat /tmp/f'

# Sans DAC_OVERRIDE
docker run --rm --cap-drop=DAC_OVERRIDE alpine sh -c \
  'echo secret > /tmp/f && chmod 000 /tmp/f && cat /tmp/f' 2>&1 | head -2
```

### Exercice 4 : À vous — le conteneur nginx au minimum de privilèges

**Énoncé :** lancez l'image `nginx:alpine` en ne lui accordant **aucune** capability, sauf celles strictement nécessaires pour qu'elle démarre et serve une page sur le port 8080 de l'hôte.

Indices :
* L'image officielle nginx écoute sur le port 80 **dans** le conteneur.
* Elle a besoin de changer d'identité pour ses processus *worker*.
* Testez avec `curl http://localhost:8080` puis vérifiez `docker logs`.

<details><summary>Correction</summary>

```bash
docker run -d --name durci \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=SETUID \
  --cap-add=SETGID \
  --cap-add=CHOWN \
  -p 8080:80 \
  nginx:alpine

# Validation
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080
docker logs durci | tail -5
docker inspect durci -f '{{.HostConfig.CapDrop}} / {{.HostConfig.CapAdd}}'
docker rm -f durci
```

**Explication :**
* `NET_BIND_SERVICE` : le processus maître écoute sur le port 80.
* `SETUID` / `SETGID` : le maître (root) démarre les *workers* sous l'utilisateur `nginx`.
* `CHOWN` : l'entrypoint ajuste les droits des répertoires de cache au démarrage.

Une image mieux conçue pour le moindre privilège (`nginxinc/nginx-unprivileged`) écoute sur le port 8080 en tant qu'utilisateur non root et fonctionne avec `--cap-drop=ALL` seul :

```bash
docker run -d --name durci --cap-drop=ALL -p 8080:8080 \
  nginxinc/nginx-unprivileged:alpine
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080
docker rm -f durci
```

</details>

-----

## PARTIE 4 : Les SysCalls et seccomp

Deuxième couche : limiter **ce que** le conteneur peut demander au noyau.

### Exercice 1 : Le profil par défaut de Docker en action

**Concept :** Docker bloque une quarantaine d'appels système par défaut. Vérifions-le.

1. `mount()` est bloqué par le profil par défaut — même avec la capability :

```bash
docker run --rm --cap-add=SYS_ADMIN alpine sh -c \
  'mkdir -p /mnt/t && mount -t tmpfs none /mnt/t && echo "montage reussi"' 2>&1 | head -3
```

*Résultat attendu :* `Operation not permitted` — c'est seccomp, pas la capability.

2. La preuve : désactivez seccomp et réessayez (**à ne jamais faire en production**) :

```bash
docker run --rm --cap-add=SYS_ADMIN --security-opt seccomp=unconfined alpine sh -c \
  'mkdir -p /mnt/t && mount -t tmpfs none /mnt/t && echo "montage reussi"'
```

*Observation :* le montage réussit. Le blocage venait bien de **seccomp**, qui vient en complément des capabilities.

3. Autre exemple : la création de namespaces par `unshare()` :

```bash
docker run --rm alpine unshare --mount sh -c 'echo ok' 2>&1 | head -2
docker run --rm --security-opt seccomp=unconfined --cap-add=SYS_ADMIN \
  alpine unshare --mount sh -c 'echo ok'
```

### Exercice 2 : Écrire un profil seccomp sur mesure

**Concept :** nous allons bloquer précisément `chmod` et `chown`, tout en laissant le reste fonctionner.

1. Créez le profil (liste de refus explicite, tout le reste autorisé) :

```bash
cat > /tmp/seccomp-nochmod.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "chmod",
        "fchmod",
        "fchmodat",
        "chown",
        "fchown",
        "fchownat",
        "lchown"
      ],
      "action": "SCMP_ACT_ERRNO",
      "errnoRet": 1
    }
  ]
}
EOF
```

2. Appliquez-le :

```bash
docker run --rm --security-opt seccomp=/tmp/seccomp-nochmod.json alpine sh -c \
  'touch /tmp/f && echo "creation OK" && chmod 777 /tmp/f' 2>&1 | head -3
```

*Résultat attendu :* la création réussit, le `chmod` échoue avec `Operation not permitted`.

3. Vérifiez que le reste fonctionne normalement :

```bash
docker run --rm --security-opt seccomp=/tmp/seccomp-nochmod.json alpine sh -c \
  'echo bonjour > /tmp/f && cat /tmp/f && ls -l /tmp/f'
```

### Exercice 3 : À vous — un profil en liste blanche

**Énoncé :** écrivez un profil seccomp en **liste blanche** (`defaultAction: SCMP_ACT_ERRNO`) qui autorise juste assez d'appels système pour que `alpine` exécute `/bin/true`. Tout le reste doit être refusé.

Indice : commencez par tracer les appels système réellement utilisés.

```bash
strace -c -f /bin/true 2>&1 | tail -25
```

<details><summary>Correction</summary>

```bash
cat > /tmp/seccomp-whitelist.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "execve",
        "arch_prctl",
        "brk",
        "access",
        "openat",
        "open",
        "newfstatat",
        "fstat",
        "close",
        "read",
        "pread64",
        "mmap",
        "mprotect",
        "munmap",
        "set_tid_address",
        "set_robust_list",
        "rseq",
        "prlimit64",
        "readlink",
        "getrandom",
        "exit_group",
        "exit"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF

# /bin/true doit fonctionner
docker run --rm --security-opt seccomp=/tmp/seccomp-whitelist.json \
  alpine /bin/true && echo "OK : /bin/true autorise"

# Mais un shell complet ne passe pas : il lui faut bien plus de syscalls
docker run --rm --security-opt seccomp=/tmp/seccomp-whitelist.json \
  alpine sh -c 'ls /' 2>&1 | head -3
```

**À retenir :** un profil en liste blanche est très efficace mais **coûteux à maintenir** : la moindre mise à jour de la libc ou de l'application peut ajouter un appel système. En pratique, on part du profil par défaut de Docker et on **retire** ce dont on n'a pas besoin, plutôt que de repartir de zéro.

La liste exacte dépend de la version de la libc musl : si la commande échoue, relevez l'appel manquant avec `strace` et ajoutez-le.

</details>

-----

## PARTIE 5 : `no-new-privileges` et les binaires setuid

Troisième couche : empêcher l'escalade de privilèges **à l'intérieur** du conteneur.

### Exercice 1 : Construire une image vulnérable

**Concept :** un binaire portant le bit setuid s'exécute avec les droits de son propriétaire. Si ce propriétaire est root, un utilisateur non privilégié peut devenir root.

1. Préparez les fichiers :

```bash
mkdir -p ~/lab-a1/setuid && cd ~/lab-a1/setuid

cat > escalate.c <<'EOF'
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("Avant : uid=%d euid=%d\n", getuid(), geteuid());
    if (setuid(0) == 0) {
        printf("ESCALADE REUSSIE : uid=%d\n", getuid());
    } else {
        perror("setuid a echoue");
    }
    return 0;
}
EOF

cat > Dockerfile <<'EOF'
FROM gcc:13 AS build
COPY escalate.c /tmp/escalate.c
RUN gcc -static -o /tmp/escalate /tmp/escalate.c

FROM debian:12-slim
COPY --from=build /tmp/escalate /usr/local/bin/escalate
RUN chown root:root /usr/local/bin/escalate \
 && chmod u+s /usr/local/bin/escalate \
 && useradd -u 10001 -m appuser
USER 10001
CMD ["/usr/local/bin/escalate"]
EOF
```

2. Construisez l'image :

```bash
docker build -t lab-a1/setuid-demo:1.0 .
```

### Exercice 2 : L'escalade, avec et sans protection

1. Sans protection, l'escalade réussit :

```bash
docker run --rm --name setuid-demo lab-a1/setuid-demo:1.0
```

*Résultat attendu :*
```
Avant : uid=10001 euid=0
ESCALADE REUSSIE : uid=0
```

Notez `euid=0` : le bit setuid a déjà fait son effet avant même l'appel à `setuid()`.

2. Activez `no-new-privileges` :

```bash
docker run --rm --security-opt no-new-privileges lab-a1/setuid-demo:1.0
```

*Résultat attendu :*
```
Avant : uid=10001 euid=10001
setuid a echoue: Operation not permitted
```

3. La même protection, en retirant simplement les capabilities concernées :

```bash
docker run --rm --cap-drop=SETUID --cap-drop=SETGID lab-a1/setuid-demo:1.0
```

*Observation :* `no-new-privileges` est **une ligne de configuration** et neutralise toute une classe d'attaques. Il n'y a aucune raison de ne pas l'activer, y compris par défaut dans `/etc/docker/daemon.json`.

-----

## PARTIE 6 : AppArmor (optionnel — Debian / Ubuntu)

Quatrième couche : le contrôle d'accès obligatoire, qui décide **quel fichier** le processus peut toucher.

### Exercice 1 : Le profil par défaut

1. Vérifiez qu'AppArmor est actif :

```bash
sudo aa-status | head -5
sudo aa-status | grep docker
```

2. Observez le profil appliqué automatiquement :

```bash
docker run -d --name apparmor-test alpine sleep 600
docker inspect apparmor-test -f '{{.AppArmorProfile}}'
```

3. Le profil `docker-default` protège déjà certains chemins sensibles :

```bash
docker exec apparmor-test sh -c 'cat /proc/sys/kernel/hostname' 2>&1 | head -2
docker exec apparmor-test sh -c 'echo test > /proc/sys/kernel/hostname' 2>&1 | head -2
docker rm -f apparmor-test
```

### Exercice 2 : Un profil sur mesure

1. Créez un profil qui interdit toute écriture dans `/etc` :

```bash
sudo tee /etc/apparmor.d/docker-lab-a1 > /dev/null <<'EOF'
#include <tunables/global>

profile docker-lab-a1 flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  network,
  capability,
  file,
  umount,

  # Interdiction explicite d'écrire dans /etc
  deny /etc/** w,
  deny /etc/** l,
}
EOF
```

2. Chargez-le :

```bash
sudo apparmor_parser -r -W /etc/apparmor.d/docker-lab-a1
sudo aa-status | grep docker-lab-a1
```

3. Testez :

```bash
# La lecture fonctionne
docker run --rm --security-opt apparmor=docker-lab-a1 alpine \
  sh -c 'head -2 /etc/hosts'

# L'écriture est refusée
docker run --rm --security-opt apparmor=docker-lab-a1 alpine \
  sh -c 'echo test > /etc/nouveau-fichier' 2>&1 | head -2
```

4. Consultez les refus dans le journal de l'hôte :

```bash
sudo dmesg | grep -i apparmor | tail -5
# ou
sudo journalctl -k | grep -i 'apparmor.*DENIED' | tail -5
```

5. Déchargez le profil :

```bash
sudo apparmor_parser -R /etc/apparmor.d/docker-lab-a1
sudo rm -f /etc/apparmor.d/docker-lab-a1
```

*Note :* sur RHEL / Fedora / CentOS, c'est **SELinux** qui joue ce rôle. L'équivalent se fait avec `--security-opt label=...` et l'option `:Z` sur les volumes.

-----

## PARTIE 7 : Durcir le système de fichiers et les ressources

### Exercice 1 : La racine en lecture seule

1. Par défaut, un conteneur peut écrire partout dans son système de fichiers :

```bash
docker run --rm alpine sh -c 'touch /fichier-a-la-racine && echo "ecriture OK"'
```

2. Passez la racine en lecture seule :

```bash
docker run --rm --read-only alpine sh -c \
  'touch /fichier-a-la-racine' 2>&1 | head -2
```

3. La plupart des applications ont besoin d'écrire **quelque part** (`/tmp`, caches, fichiers PID). Donnez-leur un `tmpfs` :

```bash
docker run --rm --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  alpine sh -c 'touch /tmp/ok && echo "ecriture dans /tmp OK" && df -h /tmp | tail -1'
```

4. Notez l'option **`noexec`** : même en écrivant un binaire dans `/tmp`, l'attaquant ne pourra pas l'exécuter.

```bash
docker run --rm --read-only --tmpfs /tmp:rw,noexec,nosuid alpine sh -c \
  'cp /bin/busybox /tmp/b && chmod +x /tmp/b && /tmp/b echo test' 2>&1 | head -3
```

### Exercice 2 : Plafonner les ressources

1. Sans limite, un conteneur peut consommer toute la mémoire de l'hôte. Fixez un plafond :

```bash
docker run --rm --memory=64m --memory-swap=64m alpine sh -c \
  'dd if=/dev/zero of=/dev/null bs=1M count=100; echo termine'
```

2. Vérifiez la limite appliquée par les cgroups :

```bash
docker run --rm --memory=64m alpine cat /sys/fs/cgroup/memory.max
```

3. Provoquez volontairement l'intervention du *OOM killer* :

```bash
docker run --rm --memory=32m --memory-swap=32m python:3.12-alpine python -c \
  "a = 'x' * (64 * 1024 * 1024); print('alloue')"
echo "Code de sortie : $?"
```

*Résultat attendu :* le code de sortie **137** (128 + SIGKILL) : le noyau a tué le processus.

4. La protection la plus simple contre une *fork bomb* :

```bash
docker run --rm --pids-limit=20 alpine sh -c \
  'for i in $(seq 1 50); do sleep 30 & done; echo "processus lances : $(ls /proc | grep -c "^[0-9]")"' 2>&1 | tail -3
```

-----

## PARTIE 8 : Le conteneur durci de bout en bout

### Exercice final : assembler toutes les couches

**Énoncé :** déployez `nginxinc/nginx-unprivileged:alpine` en appliquant **toutes** les protections vues dans ce lab :

1. Un réseau dédié (pas le bridge par défaut).
2. Aucune capability.
3. `no-new-privileges` actif.
4. Racine en lecture seule, avec les `tmpfs` nécessaires.
5. Mémoire, CPU et nombre de processus plafonnés.
6. Le port publié **uniquement sur la boucle locale** de l'hôte.
7. Une politique de redémarrage.

Puis validez que le service répond, et prouvez que chaque protection est bien active.

<details><summary>Correction</summary>

```bash
# 1. Le réseau dédié
docker network create lab-a1-net

# 2. Le conteneur durci
docker run -d --name durci \
  --network lab-a1-net \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --tmpfs /var/cache/nginx:rw,noexec,nosuid,size=32m \
  --tmpfs /var/run:rw,noexec,nosuid,size=4m \
  --memory=128m --memory-swap=128m \
  --cpus=0.5 \
  --pids-limit=50 \
  -p 127.0.0.1:8080:8080 \
  --restart=on-failure:3 \
  nginxinc/nginx-unprivileged:alpine

# 3. Le service répond-il ?
sleep 2
curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:8080
```

**Validation, protection par protection :**

```bash
echo "=== Utilisateur non root ==="
docker exec durci id

echo "=== Aucune capability ==="
docker exec durci grep CapEff /proc/self/status
# 0000000000000000 attendu

echo "=== Racine en lecture seule ==="
docker exec durci sh -c 'touch /test' 2>&1 | head -1

echo "=== tmpfs accessible en écriture ==="
docker exec durci sh -c 'touch /tmp/ok && echo OK'

echo "=== no-new-privileges ==="
docker inspect durci -f '{{.HostConfig.SecurityOpt}}'

echo "=== Limites de ressources ==="
docker inspect durci \
  -f 'Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} Pids={{.HostConfig.PidsLimit}}'
docker exec durci cat /sys/fs/cgroup/memory.max
docker exec durci cat /sys/fs/cgroup/pids.max

echo "=== seccomp actif ==="
docker exec durci grep Seccomp /proc/self/status

echo "=== Publication limitée à la boucle locale ==="
docker port durci
ss -ltnp 2>/dev/null | grep 8080

echo "=== Vue d'ensemble ==="
docker ps --filter name=durci \
  --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

**Le résultat en une commande, sous forme de fichier Compose :**

```yaml
services:
  web:
    image: nginxinc/nginx-unprivileged:alpine
    restart: on-failure:3
    networks: [lab-a1-net]
    ports:
      - "127.0.0.1:8080:8080"
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=16m
      - /var/cache/nginx:rw,noexec,nosuid,size=32m
      - /var/run:rw,noexec,nosuid,size=4m
    pids_limit: 50
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 128M

networks:
  lab-a1-net:
```

</details>

### Exercice bonus : auditer l'hôte avec le benchmark CIS

1. Lancez l'audit automatisé :

```bash
docker run --rm --net host --pid host --userns host \
  --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST="$DOCKER_CONTENT_TRUST" \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security 2>&1 | tee ~/lab-a1/cis-audit.txt
```

2. Comptez les résultats :

```bash
grep -c '^\[WARN\]' ~/lab-a1/cis-audit.txt
grep -c '^\[PASS\]' ~/lab-a1/cis-audit.txt
```

3. Relevez les avertissements concernant la configuration du daemon :

```bash
grep '^\[WARN\]' ~/lab-a1/cis-audit.txt | grep -E '^\[WARN\] 2\.' | head -10
```

4. **Question :** appliquez une correction de la section 2 dans `/etc/docker/daemon.json`, redémarrez le daemon, et relancez l'audit pour vérifier que l'avertissement a disparu.

<details><summary>Piste de correction</summary>

Par exemple, pour la recommandation « *Ensure the default ulimit is configured appropriately* » et « *Ensure containers are restricted from acquiring new privileges* » :

```bash
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak 2>/dev/null || true

sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "icc": false,
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 2048,
      "Soft": 1024
    }
  }
}
EOF

sudo systemctl restart docker
docker info -f '{{.SecurityOptions}}'
```

Vérifiez que `no-new-privileges` s'applique désormais **par défaut**, sans option sur la ligne de commande :

```bash
docker run --rm lab-a1/setuid-demo:1.0
# L'escalade doit maintenant échouer sans avoir rien précisé
```

**Attention :** `icc: false` coupe la communication entre conteneurs sur le bridge par défaut. Les conteneurs d'un réseau **personnalisé** continuent de se parler normalement — c'est une raison de plus de toujours créer ses propres réseaux.

</details>

-----

## Conclusion du Lab

Vous avez construit une frontière de sécurité, couche par couche :

1. **Constat :** un conteneur mal lancé (`--privileged`, socket monté, racine montée) donne un accès root immédiat à l'hôte. Ce n'est pas une faille : c'est le comportement normal de Docker.
2. **Capabilities :** le privilège de root est découpé en droits unitaires. `--cap-drop=ALL` puis quelques `--cap-add` ciblés couvrent la quasi-totalité des besoins réels.
3. **seccomp :** les capabilities disent *qui a le droit*, seccomp dit *quel appel système est possible*. Le profil par défaut de Docker est un bon filet de sécurité — ne le désactivez jamais.
4. **`no-new-privileges` :** une ligne de configuration qui neutralise toute la classe des escalades par binaire setuid.
5. **AppArmor / SELinux :** le contrôle d'accès obligatoire encadre l'accès aux objets du système.
6. **`--read-only`, `tmpfs`, cgroups :** un système de fichiers immuable et des ressources plafonnées transforment une compromission en simple incident.

**L'enseignement central : aucune de ces protections ne coûte en performance, et chacune retire une possibilité à un attaquant. Leur seul coût est de les connaître — et de les écrire.**

En production, ne les répétez pas sur la ligne de commande : figez-les dans votre fichier Compose, dans `/etc/docker/daemon.json`, ou dans les `securityContext` de vos manifestes Kubernetes.