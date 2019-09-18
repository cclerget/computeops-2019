# Computeops 19/09/2019

## Installation de la release 3.4

Dépendances et installation de Go:

https://github.com/sylabs/singularity/blob/master/INSTALL.md#install-system-dependencies

Récupération de la 3.4:

```bash
git clone https://github.com/sylabs/singularity.git -b release-3.4
cd singularity
./mconfig
cd builddir
make && sudo make install
```

## Multi-stage build pour une image minimale

Création du fichier de définition:

```bash
cat > /tmp/multi.def <<EOF
bootstrap: docker
from: alpine:3.8
stage: one

%post
    apk update
    apk add bash

bootstrap: scratch
stage: two

%files from one
    /bin/sh /bin/sh
EOF
```

Construction de l'image:

```bash
sudo singularity build /tmp/multi.sif /tmp/multi.def
```

Naviguer un peu dans l'image (eg: `/bin`,`/etc`):

```bash
singularity exec /tmp/multi.sif ls -la /bin
singularity exec /tmp/multi.sif ls -la /etc
```

## Cryptage image SIF

Construction et cryptage de l'image:

```bash
sudo singularity build --passphrase /tmp/encrypted.sif docker://alpine
```

Exécution du conteneur crypté :

```bash
singularity shell --passphrase /tmp/encrypted.sif
```

## Manipulation/inspection SIF

Lister les descripteurs SIF:

```bash
singularity sif list /tmp/multi.sif
singularity sif list /tmp/encrypted.sif
```

Voir les informations détaillées:

```bash
singularity sif info 2 /tmp/multi.sif
```

Extraire le fichier de définition de l'image

```bash
singularity sif dump 1 /tmp/multi.sif
```

Extraire le système de fichier (descripteur 3):

```bash
singularity sif dump 3 /tmp/multi.sif > /tmp/rootfs.img
```

Ce fichier sera utilisé comme overlay pour la suite

## Overlays

```bash
mkdir /tmp/empty
mksquashfs /tmp/empty /tmp/empty.img
```

Une petite vérification:

```bash
singularity shell /tmp/empty.img
```

Avec l'overlay précédemment extrait:

```bash
singularity shell -o /tmp/rootfs.img:ro /tmp/empty.img
```

## Fakeroot

### Test du support des user namespaces :

```bash
singularity exec --userns /tmp/test.sif true
```

En cas d'échec vérifier:

* la valeur de `/proc/sys/kernel/unprivileged_userns_clone`, si 0 :

  ```bash
  echo 1 | sudo tee /proc/sys/kernel/unprivileged_userns_clone
  ```

* ou la value de `/proc/sys/user/max_user_namespaces`, si 0 :

  ```bash
  echo 10000 | sudo tee /proc/sys/user/max_user_namespaces
  ```

### Configuration préalable

Ajouter les range mapping pour votre utilisateur:

```bash
sudo /bin/sh -c "echo $(id -u):3000000:65536 >> /etc/subuid"
sudo /bin/sh -c "echo $(id -u):3000000:65536 >> /etc/subgid"
```

### Build avec un fichier de définition (sans sudo)

```bash
cat > /tmp/fakeroot.def <<EOF
bootstrap: docker
from: alpine

%post
    apk update
    chown bin:bin /etc/profile
EOF
```

```bash
singularity build --fakeroot /tmp/fakeroot.sif /tmp/fakeroot.def
```

Observer le propriétaire de `/etc/profile`:

```bash
singularity exec /tmp/fakeroot.sif ls -la /etc/profile
```

### Réseau

Tester le réseau avec/sans l'option `--net`:

```bash
singularity exec --fakeroot --net /tmp/fakeroot.sif ping -c 1 8.8.8.8
singularity exec --fakeroot /tmp/fakeroot.sif ping -c 1 8.8.8.8
```

Exécuter un serveur nginx:

```bash
singularity run -f --net --network-args="portmap=8080:80/tcp" docker://nginx
```

Et dans un autre terminal:

```bash
wget -qO - localhost:8080
```

## Sykube (Singularity CRI et Singularity OCI runtime)

Pour l'installer:

```bash
sudo singularity run library://sykube
```

Installer et démarrer un mini-cluster Kubernetes:

```bash
sykube init
```

Cette étape prends plusieurs minutes pour télécharger et lancer les services Kubernetes

Installer l'alias dans sa session:

```bash
sykube kubectl
```

Pour obtenir l'URL de la dashboard K8S:

```bash
sykube dashboard
```
