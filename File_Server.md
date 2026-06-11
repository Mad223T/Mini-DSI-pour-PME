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
   Commandes : ip a

2. Éditez le fichier de configuration réseau Netplan (le nom du fichier .yaml peut varier) :
   Commandes : sudo nano /etc/netplan/00-installer-config.yaml

3. Modifiez ou ajoutez la configuration suivante en adaptant les adresses à votre plan réseau. Important : Le serveur DNS doit pointer explicitement sur l'adresse IP fixe de votre contrôleur de domaine Windows Server :

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

4. Enregistrez (Ctrl+O puis Entrée) et quittez l'éditeur (Ctrl+X).
5. Appliquez immédiatement la nouvelle politique réseau :
   Commandes : sudo netplan apply

---

## Étape 3 : Configuration du Fichier Samba (smb.conf)
**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Le service Samba est configuré pour monter un partage masqué, accessible uniquement par les identifiants validés auprès de l'annuaire de l'organisation.

1. Ouvrez le fichier de configuration principal de Samba :
   Commandes : sudo nano /etc/samba/smb.conf

2. Descendez tout en bas du fichier et insérez le bloc de déclaration du partage masqué. Le suffixe $ garantit l'invisibilité du dossier lors du scan réseau public. Les guillemets entourent le groupe Windows pour protéger les espaces systèmes :

[PrivateShares$]
   comment = Dossiers Personnels Securises
   path = /srv/samba/privateshares/
   browseable = no
   read only = no
   guest ok = no
   valid users = @"Utilisateurs du domaine@entreprise.local"
   create mask = 0700
   directory mask = 0700

3. Enregistrez et quittez l'éditeur.
4. Redémarrez les démons Samba pour forcer la prise en compte des nouveaux paramètres :
   Commandes : sudo systemctl restart smbd nmbd

---

## Étape 4 : Automatisation de la Création des Répertoires et Permissions POSIX
**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

Cette phase industrialise la mise en place des espaces de stockage des utilisateurs en appliquant le principe du moindre privilège (isolation étanche 0700).

1. Exécutez la boucle Bash suivante en modifiant la liste des comptes par les identifiants fonctionnels et vérifiés de votre Active Directory :

for user in a.martin b.leclerc m.garnier; do
    sudo mkdir -p /srv/samba/privateshares/$user
    sudo chown "$user@entreprise.local":"Utilisateurs du domaine@entreprise.local" /srv/samba/privateshares/$user
    sudo chmod 700 /srv/samba/privateshares/$user
done

---

## Étape 5 : Réparation Chirurgicale Ciblée (Gestion des Erreurs d'Orthographe)
**📍 Position de l'utilisateur :** Console en ligne de commande de la VM Ubuntu Server.

En cas d'échec ou d'anomalie d'authentification sur un compte précis suite à une mauvaise syntaxe ou une désynchronisation (ex: comptes spécifiques exclus de la boucle globale), n'exécutez plus le script complet. Appliquez une correction isolée en ligne droite :

1. Créez manuellement le répertoire manquant ou problématique :
   Commandes : sudo mkdir -p /srv/samba/privateshares/nom.utilisateur

2. Réappliquez les droits de propriété exacts et les restrictions de sécurité de manière unitaire :
   Commandes : sudo chown "nom.utilisateur@entreprise.local":"Utilisateurs du domaine@entreprise.local" /srv/samba/privateshares/nom.utilisateur
   Commandes : sudo chmod 700 /srv/samba/privateshares/nom.utilisateur
