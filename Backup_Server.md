# Documentation d'Exploitation : Configuration du Serveur de Sauvegarde Déporté (vm-backup)

Ce document décrit pas à pas la procédure d'installation, de sécurisation, de configuration et d'automatisation du serveur de sauvegarde dédié (`vm-backup`). Ce serveur implémente une politique de sauvegarde de type **Pull** (aspiration) sécurisée via SSH et optimisée pour préserver l'intégrité des ACL (droits d'accès) d'un environnement Active Directory Windows émulé par Samba / Winbind.

---

## 1. Architecture et Prérequis

### Architecture des flux
* **Serveur de Fichiers (Source) :** `192.168.10.11` (Ubuntu Server + Samba lié à `entreprise.local`)
* **Serveur de Sauvegarde (Destination) :** IP Statique dédiée (ex: `192.168.10.12` ou similaire)
* **Flux réseau requis :** Port TCP/22 (SSH) ouvert de la VM Sauvegarde vers la VM Fichiers.

### Spécificités de l'infrastructure
La VM de sauvegarde n'étant pas membre du domaine Active Directory, elle ne résout pas nativement les noms d'utilisateurs et de groupes Windows. La réplication doit impérativement s'appuyer sur les identifiants numériques bruts (**UID/GID**) afin de garantir qu'une restauration future réapplique exactement les mêmes permissions aux clients du domaine.

---

## Étape 1 : Configuration Réseau Statique (Netplan)

Pour s'assurer que le serveur conserve son adressage, l'IP doit être fixée via Netplan.

1. Éditer le fichier de configuration réseau :
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
2. Adapter la configuration suivante (ajuster le nom de l'interface, ex: eth0 ou ens18) :
```Yaml
   network:
      version: 2
      renderer: networkd
      ethernets:
        ens18:
          dhcp4: no
          addresses:
              - 192.168.10.12/24
          nameservers:
              addresses:
                - 192.168.10.10  # IP du Contrôleur de Domaine Active Directory
          routes:
            - to: default
             via: 192.168.10.1
   ```
## Étape 2 : Établissement de la Liaison de Confiance SSH (Root-to-Root)
Pour permettre au script automatisé de s'exécuter la nuit sans interruption ni demande de mot de passe, une authentification par clé asymétrique non chiffrée est requise.

### 2.1. Génération de la paire de clés sur la VM Sauvegarde
  1.Basculer sous le compte de l'administrateur système suprême :
```
sudo -i
```
  2. Générer une paire de clés sécurisée de type Ed25519:
```
ssh-keygen -t ed25519 -C "sauvegarde@entreprise.local"
```
  3. Instructions de validation :

. Location (Emplacement) : Appuyer sur Entrée (conserver /root/.ssh/id_ed25519).

. Passphrase (Mot de passe de clé) : Laisser vide (Appuyer deux fois sur Entrée). Une passphrase bloquerait l'automatisation nocturne.

### 2.2. Injection et migration de la clé vers le profil Root cible
1. Envoyer la clé publique vers le compte d'administration intermédiaire du serveur de fichiers :
```
ssh-copy-id votre_user_admin@192.168.10.11
```
(Saisir le mot de passe de l'utilisateur du serveur de fichiers lorsque demandé).
2. Se connecter au serveur de fichiers pour transférer l'autorisation au compte `root`
```Bash
ssh votre_user_admin@192.168.10.11µ
```
3. Exécuter le bloc de commandes suivant sur le serveur de fichiers pour isoler et sécuriser la clé au niveau du profil root :
```Bash
sudo mkdir -p /root/.ssh
sudo cat ~/.ssh/authorized_keys | sudo tee -a /root/.ssh/authorized_keys
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
```
4. Quitter le serveur de fichiers
   ```Bash
   exit
   ```
### 2.3. Validation du fonctionnement transparent
Depuis la VM Sauvegarde (en root), exécuter la commande de test :
```
ssh root@192.168.10.11
```
Résultat obligatoire : L'accès au terminal du serveur de fichiers doit se faire instantanément, sans aucune invite de mot de passe. Taper exit pour revenir sur la VM Sauvegarde.

## Etape 3 : Initialisation de l'Espace de Stockage Local
Sur la VM Sauvegarde, créer l'arborescence standardisée qui accueillera les miroirs des partages réseau Samba :

```Bash
mkdir -p /srv/backup/samba/
```
## Étape 4 : Déploiement du Script d'Aspiration Rsync
Créer un nouveau fichier script à la racine de l'espace root :

```Bash
nano /root/backup_samba.sh
```
Intégrer l'intégralité du code de production ci-dessous :

```Bash
#!/bin/bash
# ==============================================================================
# Script de sauvegarde incrémentielle des partages Samba
# Emplacement recommandé : /root/backup_samba.sh
# ==============================================================================

# 1. Configuration des variables d'environnement
SERVER_FICHIERS="192.168.10.11"
SOURCE="root@${SERVER_FICHIERS}:/srv/samba/"
DESTINATION="/srv/backup/samba/"
LOG_FILE="/var/log/backup_samba.log"
DATE_EXEC=$(date +"%Y-%m-%d %H:%M:%S")

# 2. Initialisation de l'en-tête du journal
echo "==================================================" >> $LOG_FILE
echo "DÉBUT DE LA SAUVEGARDE : ${DATE_EXEC}" >> $LOG_FILE
echo "==================================================" >> $LOG_FILE

# 3. Exécution de la réplication rsync ordonnée
# Explication des options critiques :
# -a            : Mode archive (conserve permissions, dates de modification, liens symboliques)
# -H            : Préserve les liens matériels (Hard links)
# -X            : Transfère les attributs étendus (crucial pour la cartographie des ACLs Samba)
# --numeric-ids : Force le transfert des UID/GID chiffrés sans résolution de nom AD locale
# --delete      : Supprime côté backup les éléments purgés sur le serveur de production
# -e ssh        : Sécurise le transit des données au travers du tunnel SSH chiffré
rsync -aHX --numeric-ids --delete -e ssh "$SOURCE" "$DESTINATION" >> $LOG_FILE 2>&1

# 4. Traitement et vérification du code de retour rsync
if [ $? -eq 0 ]; then
    echo "RÉSULTAT : Sauvegarde réussie avec succès le $(date +'%Y-%m-%d à %H:%M:%S')" >> $LOG_FILE
else
    echo "ATTENTION : Des erreurs ou anomalies ont été détectées pendant le traitement !" >> $LOG_FILE
fi

echo "FIN DE LA SÉANCE DE SAUVEGARDE" >> $LOG_FILE
echo "" >> $LOG_FILE
```
Enregistrer le fichier (`Ctrl+O`, `Entrée`, puis `Ctrl+X`).

Élever les privilèges du fichier pour le rendre exécutable :

```Bash
chmod +x /root/backup_samba.sh
```
## Étape 5 : Automatisation Temporelle (Cron)
Pour garantir une exécution récurrente et totalement autonome au milieu de la nuit (période de basse activité pour le domaine) :

Ouvrir le planificateur de tâches du compte root :

```Bash
crontab -e
```
Ajouter la directive de planification suivante à l'extrême fin du fichier (planification quotidienne à 2h00 du matin) :

```Plaintext
0 2 * * * /bin/bash /root/backup_samba.sh
```
Sauvegarder et quitter. Le système valide l'opération par le message : crontab: installing new crontab.

## Étape 6 : Plan de Reprise d'Activité (Procédures de Restauration)
Toutes les commandes de restauration doivent être impérativement exécutées depuis le serveur de sauvegarde.

### Scénario A : Restauration ciblée / granulaire (Erreur utilisateur)
Lorsqu'un utilisateur supprime ou altère accidentellement un fichier dans son dossier personnel (ex: utilisateur samir), il convient d'injecter uniquement la source concernée sans impacter le reste du serveur.

```Bash
rsync -aHX --numeric-ids /srv/backup/samba/privateshares/samir/ root@192.168.10.11:/srv/samba/privateshares/samir/
```
*Avertissement de sécurité* : N'utilisez jamais l'option `--delete` dans ce scénario, sous peine d'effacer les nouveaux documents légitimes créés par l'utilisateur dans ses autres répertoires depuis la dernière sauvegarde nocturne.

### Scénario B : Restauration totale (Crash matériel ou perte système)
En cas de sinistre total majeur sur le serveur de fichiers, la procédure suit un ordonnancement strict.

Sur le Serveur de Fichiers (neuf ou réinstallé) : Réinstaller le système, lier la machine au domaine AD via Winbind, puis figer impérativement les services Samba pour prévenir les accès concurrents :

```Bash
sudo systemctl stop smbd nmbd
```
Sur le Serveur de Sauvegarde - Test de simulation à blanc : Valider l'intégrité de la commande de restauration globale sans réaliser d'écriture physique via le commutateur -n (--dry-run) :

```Bash
rsync -aHX --numeric-ids -n --delete /srv/backup/samba/ root@192.168.10.11:/srv/samba/
```
*Analyser la sortie standard. Si aucune erreur n'est remontée, passer à l'application réelle.*

Sur le Serveur de Sauvegarde - Restauration réelle : Déclencher l'écriture définitive des données :

```Bash
rsync -aHX --numeric-ids --delete /srv/backup/samba/ root@192.168.10.11:/srv/samba/
```

Sur le Serveur de Fichiers : Relancer les démons de partage réseau :

```Bash
sudo systemctl start smbd nmbd
```
Les partages réseau Windows remontent instantanément et les ACL de sécurité d'origine sont immédiatement opérationnels pour l'ensemble des groupes et utilisateurs Active Directory.

8. Supervision et Vérification quotidienne
Pour s'assurer du maintien en condition opérationnelle du système, les administrateurs peuvent suivre les deux indicateurs de santé suivants :

Lecture du journal d'activité :

```Bash
cat /var/log/backup_samba.log
```
Contrôle de la persistance des UID/GID natifs :

```Bash
ls -ln /srv/backup/samba/
```
