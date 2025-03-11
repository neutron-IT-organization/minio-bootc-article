---
title: Home
---
![](images/banner.png)

# Déployer MinIO avec BootC

## 1. Introduction

### Présentation de MinIO et BootC

MinIO est une solution de stockage d'objets open-source qui permet de créer des infrastructures de stockage évolutives et résilientes. BootC, quant à lui, est un outil de gestion de conteneurs qui facilite le déploiement et la gestion des applications conteneurisées.

### Importance du stockage d'objets et des conteneurs

Le stockage d'objets est essentiel pour gérer de grandes quantités de données non structurées, tandis que les conteneurs permettent de déployer des applications de manière cohérente et isolée. La combinaison de MinIO et BootC offre une solution puissante pour le stockage et la gestion des données dans des environnements modernes.

## 2. Qu'est-ce que MinIO ?

### Description de MinIO

MinIO est un serveur de stockage d'objets compatible avec le protocole Amazon S3. Il est conçu pour être léger, rapide et facile à déployer. MinIO peut être utilisé pour stocker des photos, des vidéos, des sauvegardes et d'autres types de données non structurées.

### Avantages et caractéristiques principales

* **Compatibilité S3** : MinIO est entièrement compatible avec le protocole Amazon S3, ce qui permet une intégration facile avec les outils et applications existants.
* **Haute performance** : MinIO est optimisé pour offrir des performances élevées en termes de débit et de latence.
* **Évolutivité** : MinIO peut être déployé sur un seul nœud ou sur plusieurs nœuds pour répondre aux besoins de stockage croissants.
* **Sécurité** : MinIO offre des fonctionnalités de sécurité robustes, telles que le chiffrement des données et l'authentification.

## 3. Qu'est-ce que BootC ?

### Description de BootC

BootC est un outil de gestion de conteneurs qui simplifie le déploiement, la gestion et l'orchestration des applications conteneurisées. Il est conçu pour être léger et facile à utiliser, tout en offrant des fonctionnalités puissantes pour les environnements de production. BootC est au cœur des conteneurs amorçables, qui permettent de gérer des systèmes d'exploitation immuables via des images de conteneurs OCI/Docker.

### Avantages et caractéristiques principales

* **Approche unifiée pour DevOps** : BootC permet de gérer l'ensemble du système d'exploitation via des outils et concepts basés sur les conteneurs, y compris GitOps et CI/CD.
* **Sécurité simplifiée** : Les mises à jour de sécurité peuvent être appliquées à l'ensemble du système d'exploitation, y compris le noyau, les pilotes et le chargeur d'amorçage, en utilisant des outils de sécurité avancés pour les conteneurs.
* **Intégration rapide et écosystème** : BootC s'intègre dans un vaste écosystème d'outils et de technologies autour des conteneurs, permettant de déployer et de gérer des systèmes Linux à grande échelle.

## 4. Prérequis

### Liste des éléments nécessaires avant de commencer

- **Podman** installé sur votre système
- Espace disque suffisant pour le stockage des conteneurs et des fichiers de sortie

## 5. Étapes de déploiement

### Étape 1 : Créer le `Containerfile`
Adaptez le `Containerfile` suivant à vos besoins :
```dockerfile
FROM quay.io/centos/centos:stream9 as builder

RUN dnf install -y wget
RUN wget https://dl.min.io/server/minio/release/linux-amd64/minio

FROM quay.io/centos-bootc/centos-bootc:stream9

COPY --from=builder --chmod=755 /minio /usr/local/bin/minio

RUN cat <<'EOF' > /etc/systemd/system/minio.service
[Unit]
Description=MinIO
Documentation=https://min.io/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local

User=minio
Group=minio
ProtectProc=invisible

EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"$MINIO_VOLUMES\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# MinIO RELEASE.2023-05-04T21-44-30Z adds support for Type=notify (https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=)
# This may improve systemctl setups where other services use `After=minio.server`
# Uncomment the line to enable the functionality
# Type=notify

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target

EOF

RUN cat <<'EOF' > /etc/default/minio
# MINIO_ROOT_USER and MINIO_ROOT_PASSWORD sets the root account for the MinIO server.
# This user has unrestricted permissions to perform S3 and administrative API operations on any resource in the deployment.
# Omit to use the default values 'minioadmin:minioadmin'.
# MinIO recommends setting non-default values as a best practice, regardless of environment

MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin

# MINIO_VOLUMES sets the storage volume or path to use for the MinIO server.

MINIO_VOLUMES="/home/minio"

# MINIO_OPTS sets any additional commandline options to pass to the MinIO server.
# For example, `--console-address :9001` sets the MinIO Console listen port
MINIO_OPTS="--console-address :9001"
EOF

WORKDIR /home

RUN mkdir minio

RUN systemctl enable minio.service
```

### Étape 2 : Créer le fichier `config.toml`
Adaptez le fichier `config.toml` suivant à vos besoins : 
```toml
[customizations.installer.kickstart]
contents = """
# Non-interactive text mode installation
text --non-interactive
reboot --eject

# Set language and keyboard layout
lang en_US
keyboard --vckeymap=fr --xlayouts='fr'

# Network configuration
network --bootproto=dhcp --device=link --activate --onboot=on

zerombr
clearpart --all --initlabel --disklabel=gpt
autopart --noswap --type=lvm --fstype=xfs 

# Create a user with SSH key
user --name=minio --password=toto --shell=/bin/bash --groups=wheel

%post
# Configure sudo for the user
echo "minio ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/minio
chmod 440 /etc/sudoers.d/minio
%end
"""
```

### Étape 3 : Construire l'image Bootc locale pour MinIO

1. Définissez vos identifiants Red Hat et le nom de l'image :

   ```bash
   export IMAGE_NAME=minio-centos-bootc
   ```

2. Exécutez la commande `podman build` pour créer l'image Bootc pour MinIO :

   ```bash
   sudo podman build -t "${IMAGE_NAME}" -f Containerfile
   ```

   **Remarque** : Le `Containerfile` doit être présent dans le répertoire de travail.

### Étape 4 : Créer l'ISO pour MinIO

1. Définissez les variables nécessaires :

   ```bash
   export CONFIG_PATH=<votre_chemin>/config.toml
   export OUTPUT_DIR=<chemin_vers_répertoire_de_sortie>
   ```

2. Exécutez la commande `podman run` pour générer l'ISO :

   ```bash
   sudo podman run \
       --rm \
       -it \
       --privileged \
       --pull=newer \
       --security-opt label=type:unconfined_t \
       -v ${CONFIG_PATH}:/config.toml:ro \
       -v ${OUTPUT_DIR}:/output \
       -v /var/lib/containers/storage:/var/lib/containers/storage \
       quay.io/centos-bootc/bootc-image-builder \
       --type iso \
       --use-librepo \
       --local localhost/${IMAGE_NAME}
   ```

   **Remarques** :
   - Assurez-vous que `config.toml` est correctement configuré pour votre installation MinIO.
   - Le répertoire de sortie doit être accessible en écriture pour stocker l'ISO résultant.
   - L'option `--pull=newer` garantit l'utilisation de la dernière image `bootc-image-builder`.
   - L'option `--security-opt label=type:unconfined_t` aide à éviter les problèmes liés à SELinux.

### Étape 5 : Installer CentOS Stream 9 avec MinIO

1. **Créer un support d'installation amorçable** : Utilisez l'ISO généré pour créer un support d'installation amorçable (par exemple, une clé USB) à l'aide d'un outil comme Rufus (Windows) ou dd (Linux).

2. **Démarrer à partir du support** : Insérez le support d'installation dans votre système cible et démarrez à partir de celui-ci.

3. **Suivre les instructions d'installation** : Procédez à l'installation en suivant les instructions à l'écran.

4. **Vérifier l'installation** : Une fois l'installation terminée, vérifiez que MinIO est en cours d'exécution en contrôlant l'état du service :

   ```bash
   sudo systemctl status MinIO
   ```

## 6. Configuration avancée (optionnel)

### Options de configuration supplémentaires

* **Réplication** : Configurez la réplication des données entre plusieurs nœuds MinIO pour une meilleure résilience.
* **Sécurité** : Activez le chiffrement des données et configurez les politiques de sécurité pour protéger vos données.

### Sécurité et optimisation

* **Sauvegarde** : Mettez en place des sauvegardes régulières pour protéger vos données contre les pertes.
* **Surveillance** : Utilisez des outils de surveillance pour suivre les performances et la santé de votre déploiement MinIO.

## 7. Conclusion

### Récapitulatif des avantages de MinIO et BootC

MinIO et BootC offrent une solution puissante et flexible pour le stockage et la gestion des données dans des environnements modernes. La combinaison de ces deux outils permet de créer des infrastructures de stockage évolutives, sécurisées et faciles à gérer.

## 8. Ressources supplémentaires

### Liens vers les documentations

* [Documentation de MinIO](https://docs.min.io/)
* [Documentation de BootC](https://docs.fedoraproject.org/en-US/bootc/)
* [Documentation d'OSBuild](https://osbuild.org/docs/bootc/)