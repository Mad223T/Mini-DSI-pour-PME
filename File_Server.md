# Guide d'Installation et de Configuration du Serveur de Fichiers (Ubuntu Server)

Ce guide détaille pas à pas la création, l'intégration réseau et la configuration de la couche de partage sécurisée Samba validées sur le serveur Linux Ubuntu.

---

## Étape 1 : Préparation de l'environnement virtuel dans VMware Workstation
**📍 Position de l'utilisateur :** Logiciel VMware Workstation sur votre machine physique.

1. Ouvrez VMware Workstation et cliquez sur **Create a New Virtual Machine**.
2. Choisissez le mode **Typical (recommended)** et cliquez sur **Next**.
3. Sélectionnez **Installer disc image file (iso)**, cliquez sur **Browse** et chargez votre fichier ISO officiel d'Ubuntu Server. Cliquez sur **Next**.
4. Renseignez les informations de profil initial (Nom, Identifiant, et Mot de passe pour le compte d'administration local obligatoire). Cliquez sur **Next**.
5. Nommez la machine virtuelle (ex: `Ubuntu-Samba-Server`) et spécifiez son dossier de stockage.
6. Définissez la taille maximale du disque (minimum conseillé : 20 GB à 40 GB), cochez **Store virtual disk as a single file** et cliquez sur **Next**.
7. Sur la page récapitulative, cliquez sur **Customize Hardware...** pour appliquer le raccordement réseau :
   * Sélectionnez la carte réseau par défaut (**Network Adapter**).
   * Basculez son mode de connexion sur **LAN Segment**.
   * Dans le menu déroulant, sélectionnez scrupuleusement le même segment privé créé pour votre infrastructure d'entreprise (ex: `LAN-Entreprise`). L'étanchéité et la communication directe avec le serveur Active Directory et le client dépendent de ce choix.
   * Assurez-vous que la case **Connect at power on** est cochée.
8. Cliquez sur **Close** puis sur **Finish** pour lancer la machine virtuelle.

---

## Étape 2 : Configuration de l'IP Statique et Alignement DNS
**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Pour maintenir une liaison d'interconnexion stable et être découvrable par l'Active Directory, l'interface réseau doit posséder des paramètres fixes.

1. Identifiez le nom de votre interface réseau active (ex: ens33 ou eth0) :
   ``` bash
   ip a
   ```

3. Éditez le fichier de configuration réseau Netplan (le nom du fichier .yaml peut varier) :
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```
4. Modifiez ou ajoutez la configuration suivante en adaptant les adresses à votre plan réseau. Important : Le serveur DNS doit pointer explicitement sur l'adresse IP fixe de votre contrôleur de domaine Windows Server :
   
  ```yaml
  network:
    version: 2
    ethernets:
      ens33:
        dhcp4: no
        addresses:
          - 192.168.10.11/24
        nameservers:
          addresses:
            - 192.168.10.10
  ```

4. Enregistrez (Ctrl+O puis Entrée) et quittez l'éditeur (Ctrl+X).
5. Appliquez immédiatement la nouvelle politique réseau :

   ``` bash
   sudo netplan apply
   ```
---

## Étape 3 : Configuration du Fichier Samba (smb.conf)
**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Le service Samba est configuré pour monter un partage masqué, accessible uniquement par les identifiants validés auprès de l'annuaire de l'organisation.

1. Ouvrez le fichier de configuration principal de Samba :

   ```bash
   sudo nano /etc/samba/smb.conf
   ```
2. Descendez tout en bas du fichier et insérez le bloc de déclaration du partage masqué. Le suffixe $ garantit l'invisibilité du dossier lors du scan réseau public. Les guillemets entourent le groupe Windows pour protéger les espaces systèmes :

  ```yaml
  [PrivateShares$]
     comment = Dossiers Personnels Securises
     path = /srv/samba/privateshares/
     browseable = no
     read only = no
     guest ok = no
     valid users = @"Utilisateurs du domaine@entreprise.local"
     create mask = 0700
     directory mask = 0700
  ```
3. Enregistrez et quittez l'éditeur.
4. Redémarrez les démons Samba pour forcer la prise en compte des nouveaux paramètres :

   ```bash
   sudo systemctl restart smbd nmbd
   ```
---

## Étape 4 : Automatisation de la Création des Répertoires et Permissions POSIX

**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Cette phase industrialise la mise en place des espaces de stockage des utilisateurs en appliquant le principe du moindre privilège (isolation étanche `0700`). Pour garantir la compatibilité avec le sous-système d'authentification Winbind et éviter les anomalies de désynchronisation d'UID (notamment en cas de suppression et recréation de compte AD), les droits de propriété doivent être appliqués via la syntaxe de nom court.

1. Exécutez la boucle Bash suivante sur votre serveur de fichiers en adaptant la liste des comptes par les identifiants fonctionnels de votre Active Directory :

```bash
for user in prenom.nom1 prenom.nom2 prenom.nom3; do
    sudo mkdir -p /srv/samba/privateshares/$user
    sudo chown "$user:" /srv/samba/privateshares/$user
    sudo chmod 700 /srv/samba/privateshares/$user
done
```

Note: Le caractère `:` inséré juste après la variable `$user` indique à Linux d'assigner l'utilisateur AD comme propriétaire unique du dossier sans modifier ou interférer avec le groupe POSIX local.

---

## Étape 5 : Réparation Chirurgicale Ciblée (Gestion des Erreurs d'Orthographe)
**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

En cas d'échec ou d'anomalie d'authentification sur un compte précis suite à une mauvaise syntaxe ou une désynchronisation (ex: comptes spécifiques exclus de la boucle globale), n'exécutez plus le script complet. Appliquez une correction isolée en ligne droite :

1. Créez manuellement le répertoire manquant ou problématique :

   ```bash
   sudo mkdir -p /srv/samba/privateshares/nom.utilisateur
   ```
2. Réappliquez les droits de propriété exacts et les restrictions de sécurité de manière unitaire :
   ```bash
    sudo chown "nom.utilisateur:" /srv/samba/privateshares/nom.utilisateur
   ```
3. Rétablissez les restrictions d'accès strictes pour isoler l'espace privé:
   ```bash
   sudo chmod 700 /srv/samba/privateshares/nom.utilisateur
   ```

---

## Étape 6 : Forçage et Alignement des Droits Collaboratifs (Espace Commun X:)

**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Pour gérer un espace partagé commun destiné au travail collaboratif sans subir les blocages liés aux restrictions des ACLs POSIX classiques sur les groupes Active Directory (notamment ceux contenant des espaces), la gouvernance des droits d'écriture et de création est directement prise en charge et imposée par les directives du démon Samba.

1. Ouvrez à nouveau le fichier de configuration principal de Samba :
   ```bash
   sudo nano /etc/samba/smb.conf
   ```

2. Tout en bas du fichier, ajoutez ou modifiez le bloc de déclaration de l'espace commun en y intégrant les masques de force applicatifs:
   ```yaml
   [PublicShares]
       comment = Espace Collaboratif Commun
       path = /srv/samba/publicshares/
       browseable = yes
       read only = no
       guest ok = no
       valid users = @"Utilisateurs du domaine@votre-domaine.local"
       force create mode = 0660
       force directory mode = 0770
       directory mask = 0777
   ```
(Note : Les directives `force create mode` et `force directory mode` garantissent que n'importe quel fichier ou dossier créé par un collaborateur AD sera instantanément modifiable et navigable par l'ensemble des autres membres du groupe).

3. Enregistrez et quittez l'éditeur (`Ctrl+O` puis `Entrée`, puis `Ctrl+X`).
4. Redémarrer le service Samba pour appliquer la nouvelle politique de droits forcés:
   ```bash
   sudo systemctl restart smbd
   ```

---

## Étape 7 : Automatisation et synchronisation avec le script global

**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Pour éviter les interventions manuelles à chaque création d'utilisateur, le serveur de fichiers doit être capable de traiter les requêtes provenant du script PowerShell du contrôleur de domaine Windows Server via SSH. Cette étape assure que le dossier personnel est créé et correctement propriétaire dès qu'un nouvel utilisateur est provisionné dans l'Active Directory.

1. Assurez-vous que le service SSH est actif sur votre serveur Ubuntu :
   ```bash
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ```
2. Le script PowerShell (détaillé dans la documentation du serveur Windows) effectue désormais automatiquement les deux actions suivantes à chaque exécution sur le serveur Ubuntu :

La création du répertoire : sudo mkdir -p /srv/samba/privateshares/$samAccount

L'alignement des permissions de propriété (via nom court) et des droits : sudo chown "$samAccount:" ... && sudo chmod 700 ...

3. Si vous avez besoin de tester la réception d'une commande SSH depuis votre Windows Server vers Ubuntu manuellement pour vérifier la connectivité, exécutez depuis le terminal de votre serveur de fichiers :

```Bash
# Vérification que le système reconnaît bien l'utilisateur AD
getent passwd | grep nom.utilisateur
```
4. Une fois cette étape intégrée, votre serveur Ubuntu devient "passif" : il n'y a plus de boucle Bash à lancer manuellement. Le serveur de fichiers Ubuntu réagit instantanément aux ordres envoyés par le script d'automatisation centralisé situé sur votre Windows Server.
