# Lab 0 : Anatomie des Conteneurs (Namespaces & Cgroups)

**Objectif :** Comprendre comment Linux isole les processus (Namespaces) et limite leurs ressources (Cgroups). Vous allez construire manuellement les briques qui composent des outils comme Docker.

**Prérequis :**
  * Une distribution Linux moderne avec **Cgroups v2** (Ubuntu 22.04+, Debian 11+).
  * Accès `root` ou `sudo`.
  * Paquets recommandés : `iproute2`, `python3`, `stress` (optionnel), `curl`.
  * **Attention :** Ce lab manipule des paramètres noyau. Il est fortement recommandé de l'utiliser dans une VM jetable.

<details><summary>Réinisitalisation</summary>

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

## PARTIE 1 : Les Namespaces (L'Isolation)

Les namespaces (espaces de noms) modifient la "vision" qu'un processus a du système. Nous allons explorer les namespaces disponibles sous Linux.

### Exercice 1 : UTS Namespace (Hostname)

**Concept :** Isoler le nom d'hôte et de domaine.

1.  Ouvrez un terminal. Vérifiez votre hostname actuel :

```bash
hostname
```

2.  Créez un nouveau processus shell (`bash`) dans un nouvel espace de noms UTS :

```bash
sudo unshare --uts
```

3.  Changez le hostname dans ce nouveau shell :

```bash
hostname container-test
hostname
# Doit afficher "container-test"
```

4.  Ouvrez un **second terminal** sur l'hôte et vérifiez le hostname :

```bash
hostname
# Doit afficher le nom d'origine (isolation réussie !)
```

5.  Quittez le namespace dans le premier terminal :

```bash
exit
```

### Exercice 2 : Network Namespace (Réseau)

**Concept :** Avoir sa propre pile réseau (interfaces, routes, iptables).

1.  Créez un shell isolé du réseau :

```bash
sudo unshare --net
```

2.  Vérifiez l'état du réseau (il doit être vide) :

```bash
ip link
# Seule l'interface 'lo' est visible et DOWN
ping 8.8.8.8
# Erreur : Network is unreachable
```

3.  Récupérez le PID de ce shell (depuis l'intérieur du shell) :

```bash
echo $$
# Notez ce numéro (ex: 1234)
```

4.  **Depuis un second terminal (sur l'hôte)**, nous allons créer un "câble virtuel" (veth pair) :

```bash
# Remplacez PID_DU_SHELL par le numéro noté ci-dessus
PID_NS=1234

# Création de la paire de câbles
sudo ip link add veth_host type veth peer name veth_container

# Brancher un bout sur l'hôte (assignation d'IP directe)
sudo ip addr add 10.0.0.1/24 dev veth_host
sudo ip link set veth_host up

# Envoyer l'autre bout dans le namespace du conteneur
sudo ip link set veth_container netns $PID_NS
```

5.  **Retournez dans le premier terminal (le namespace)** et configurez l'interface reçue :

```bash
# Renommer l'interface (cosmétique standard)
ip link set veth_container name eth0
ip link set lo up
ip link set eth0 up
ip addr add 10.0.0.2/24 dev eth0
ip route add default via 10.0.0.1
```

6.  Testez la connectivité :

```bash
ping 10.0.0.1
# Ça marche ! Vous communiquez avec l'hôte via votre réseau privé.
```

### Exercice 3 : Mount Namespace (Fichiers)

**Concept :** Isoler les points de montage pour que le conteneur ne modifie pas les dossiers de l'hôte.

1.  Créez un namespace Mount :

```bash
sudo unshare --mount
```

2.  Créez un montage privé en mémoire vive pour `/tmp` (pour cacher les fichiers temporaires de l'hôte) :

```bash
mount -t tmpfs none /tmp
```

3.  Créez un fichier dans ce `/tmp` :

```bash
touch /tmp/fichier_secret_container
ls /tmp
```

4.  Vérifiez depuis un autre terminal sur l'hôte :

```bash
ls /tmp
# Le fichier n'apparaît pas. L'isolation est active.
```

### Exercice 4 : PID Namespace (Processus)

**Concept :** Avoir son propre PID 1 (comme init ou systemd).

1.  Lancez `unshare`. Notez les options `--fork` (pour que le processus enfant devienne PID 1) et `--mount-proc` (crucial pour que `ps` lise les bons processus) :

```bash
sudo unshare --pid --fork --mount-proc
```

2.  Vérifiez les processus :

```bash
sleep 1000 &
ps aux
```

*Observation :* - Vous ne voyez que vos processus internes. Votre shell `bash` a le PID 1.
- Vous ne voyez plus les processus de l'hôte.
- Sur un autre terminal (l'hôte), la commande `ps aux | grep sleep` montre le processus `sleep`, mais avec un PID réel système (ex: 4567), illustrant que le PID est "traduit".

### Exercice 5 : IPC, User & Time Namespaces (Pour aller plus loin)

* **IPC Namespace (`--ipc`) :** Isole la mémoire partagée et les files de messages entre les processus.
* **User Namespace (`--user`) :** Permet d'être `root` (UID 0) à l'intérieur du conteneur, mais d'être un utilisateur standard non privilégié sur la machine hôte. (Essentiel pour la sécurité).
* **Time Namespace (`--time`) :** Permet au conteneur d'avoir une horloge différente de l'hôte.

-----

## PARTIE 2 : Les Control Groups (Les Quotas - Cgroups v2)

Les Cgroups limitent la consommation ("Combien tu peux manger"). Les distributions modernes utilisent la hiérarchie unifiée **Cgroups v2**, située dans `/sys/fs/cgroup/`.

### Exercice 1 : Memory Cgroup (Limiter la RAM)

**Objectif :** Tuer un processus qui consomme trop de mémoire (OOM Killer).

1.  Définissez une variable pour le dossier cgroup et créez-le :

```bash
CG=/sys/fs/cgroup/lab_memory
sudo mkdir $CG
```

2.  Fixez une limite stricte à 100 MB et bloquez l'usage du SWAP :

```bash
# 100 MB = 100 * 1024 * 1024 = 104857600 octets
echo 104857600 | sudo tee $CG/memory.max
# Interdire le swap pour forcer le kill immédiat
echo 0 | sudo tee $CG/memory.swap.max
```

3.  Ouvrez un shell Python et ajoutez-le au Cgroup (dans Cgroups v2, on utilise `cgroup.procs`) :

```bash
# Ajoutez le PID du shell actuel au groupe
echo $$ | sudo tee $CG/cgroup.procs

# Lancez python
python3
```

4.  **Le Crash Test :** Allouez de la mémoire progressivement.

```python
# Allouer 80MB (Ça passe)
a = "a" * 1024 * 1024 * 80

# Allouer 40MB de plus (Total 120MB > 100MB Limite)
b = "b" * 1024 * 1024 * 40
```

*Résultat :* Votre processus Python doit être brutalement tué par le noyau avec le message `Killed`.

5.  Nettoyage (depuis un autre terminal) :

```bash
sudo rmdir /sys/fs/cgroup/lab_memory
```

### Exercice 2 : CPU Cgroup (Limiter le Processeur)

**Objectif :** Restreindre le temps processeur alloué à un groupe.

1.  Créez un Cgroup CPU :

```bash
# Nous activons le contrôleur cpu depuis la racine des cgroups
echo "+cpu" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```

```bash
# On crée un Cgroup CPU
CG_CPU=/sys/fs/cgroup/lab_cpu
sudo mkdir $CG_CPU
```

2.  Configurez le quota CPU (Cgroups v2 utilise `cpu.max` avec le format `QUOTA PÉRIODE`) :

```bash
# Limite à 100 000 microsecondes par période de 100 000 (Équivaut à 1 CPU à 100% maximum)
echo "100000 100000" | sudo tee $CG_CPU/cpu.max
```

3.  Lancez un processus `stress` dans ce groupe :

```bash
# Si besoin installer la commande stress
sudo apt install stress
```

```bash
# On lance stress sur 4 CPU virtuels, mais on le confine dans le cgroup
sudo sh -c "echo \$$ > $CG_CPU/cgroup.procs && exec stress --cpu 4"
```


4.  Dans un autre terminal, lancez `htop` ou `top`.

*Observation :* Même si `stress` lance 4 threads essayant de consommer 400% de CPU, le total cumulé de ces 4 processus ne dépassera jamais **100%** (soit l'équivalent d'un seul cœur complet).

### Exercice 3 : IO Cgroup (Disque - Optionnel)

**Objectif :** Limiter la vitesse de lecture (io.max en v2).

1.  Repérez votre disque principal (ex: 8:0 pour sda, utilisez `ls -l /dev/sda` pour voir Major:Minor).
2.  Créez le cgroup et limitez la lecture à 1MB/s :

```bash
# Nous activons le contrôleur IO depuis la racine des cgroups
echo "+io" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
```

```bash
CG_IO=/sys/fs/cgroup/lab_io
sudo mkdir $CG_IO
# Format Cgroups v2: "MAJOR:MINOR rbps=BYTES_PER_SECOND"
echo "8:0 rbps=1048576" | sudo tee $CG_IO/io.max
```

3.  Testez la vitesse :

```bash
sudo sh -c "echo \$$ > $CG_IO/cgroup.procs && dd if=/dev/sda of=/dev/null bs=1M count=10 iflag=direct"
```

*Résultat :* La vitesse affichée par `dd` devrait être bridée autour de `1.0 MB/s`. Pour observer cela, patientez que la commande précédente finisse son exécution pour voir le rapport.

-----

## PARTIE 3 : Le "Real Deal" (Namespaces + Cgroups + RootFS)

Pour finir, nous allons créer un conteneur complet. Nous n'allons pas seulement isoler les processus, nous allons lui donner **son propre système de fichiers** (une image Alpine Linux vierge) grâce à `chroot`.

### Exercice Final : Création d'un vrai conteneur "from scratch"

1. Préparation du système de fichiers (Image de base)

Sur l'hôte, téléchargez une image "mini-rootfs" d'Alpine Linux et extrayez-la dans un dossier :

```bash
mkdir -p ~/mon_vrai_conteneur/rootfs
cd ~/mon_vrai_conteneur/rootfs

# Téléchargement des dossiers vitaux d'Alpine (/bin, /etc, /lib...)
curl -O https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-minirootfs-3.19.1-x86_64.tar.gz
tar xf alpine-minirootfs-3.19.1-x86_64.tar.gz
rm alpine-minirootfs-3.19.1-x86_64.tar.gz
```

2. Préparation des Cgroups (Sur l'hôte)

```bash
CG_CONT=/sys/fs/cgroup/mon_vrai_conteneur
sudo mkdir -p $CG_CONT

# Limite RAM : 200MB max
echo 209715200 | sudo tee $CG_CONT/memory.max
# Limite CPU : Equivalent 1 coeur max
echo "100000 100000" | sudo tee $CG_CONT/cpu.max
```

3. Lancement du Conteneur avec `chroot`

Nous allons combiner `unshare` (pour l'isolation) avec `chroot` (pour changer le dossier racine / vers notre dossier Alpine). 

```bash
cd ~/mon_vrai_conteneur

# Création du conteneur. Notez l'appel à chroot à la fin !
sudo unshare --uts --net --mount --ipc --pid --fork chroot rootfs /bin/sh
```

Vous êtes maintenant à l'intérieur de votre conteneur ! Mais attendez, il manque une chose pour que des outils comme `ps` ou `top` fonctionnent, il faut monter un faux `/proc` propre à ce conteneur :

```bash
# À exécuter À L'INTÉRIEUR du conteneur
mount -t proc proc /proc
```

**Testez votre conteneur :**
* Tapez `ls /` : Vous voyez les fichiers d'Alpine, pas ceux de votre hôte. L'hôte est invisible !
* Tapez `cat /etc/os-release` : Vous êtes sous Alpine Linux.
* Tapez `ps aux` : Vous êtes bien le PID 1.

4. Application des Limites (Depuis l'Hôte)

Gardez le terminal du conteneur ouvert. Dans un **autre terminal** sur l'hôte, trouvez le PID du `sh` de votre conteneur et ajoutez-le au Cgroup :

```bash
# Trouver le PID de 'unshare' puis prendre son enfant direct
ps axf | grep unshare -A 1

PID_CIBLE=12345  # Remplacez par le vrai PID trouvé !
echo $PID_CIBLE | sudo tee $CG_CONT/cgroup.procs
```

Votre processus est maintenant enfermé dans son propre système de fichiers (Alpine), isolé du réseau, seul dans son arbre de processus, limité à 200MB de RAM et 1 CPU. **Félicitations, vous venez de recoder Docker !**

5. Nettoyage

Quittez le conteneur (`exit`) et nettoyez sur l'hôte :

```bash
sudo rmdir /sys/fs/cgroup/mon_vrai_conteneur
sudo rm -rf ~/mon_vrai_conteneur
```

## Conclusion du Lab

Vous venez de manipuler les fondamentaux :

1.  **Namespaces :** Vous avez créé des "boîtes" invisibles pour isoler le réseau, les fichiers, les PIDs, etc.
2.  **Cgroups :** Vous avez appliqué des menottes aux processus pour contrôler leur consommation de RAM et CPU.
3.  **Rootfs (Root Filesystem) :** Vous avez un arborescence de base d'un système, montée à la racine (`/`), qui contient l'ensemble des dossiers, bibliothèques et exécutables vitaux dont un système d'exploitation ou un conteneur a besoin pour démarrer et fonctionner.

**En conclusion, la magie des conteneurs n'existe pas : c'est exactement ce que fait le Docker Engine au quotidien en automatisant les appels aux Namespaces pour l'isolation, aux Cgroups pour les quotas, et au `chroot` pour le système de fichiers, créant ainsi des environnements d'exécution sûrs et isolés.**