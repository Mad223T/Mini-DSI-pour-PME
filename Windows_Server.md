# Guide d'Installation et de Configuration du Serveur Active Directory (Windows Server)

Ce guide décrit pas à pas la création, l'installation et la configuration complète du contrôleur de domaine Windows Server sur l'hyperviseur VMware Workstation.

---

## Étape 1 : Préparation de l'environnement virtuel dans VMware Workstation
**📍 Position de l'utilisateur :** Logiciel VMware Workstation sur votre machine physique.

1. Ouvrez VMware Workstation et cliquez sur **Create a New Virtual Machine** (Créer une nouvelle machine virtuelle).
2. Choisissez le mode **Typical (recommended)** et cliquez sur **Next**.
3. Sélectionnez **Installer disc image file (iso)**, cliquez sur **Browse** et chargez votre fichier ISO officiel de Windows Server. Cliquez sur **Next**.
4. Saisissez vos informations de licence si vous en possédez une, ou laissez vide pour la version d'évaluation. Définissez le compte administrateur initial et cliquez sur **Next**.
5. Nommez la machine virtuelle (ex: `Windows-Server-AD`) et choisissez son emplacement de stockage sur votre disque dur.
6. Spécifiez la taille maximale du disque (minimum conseillé : `40 GB` à `60 GB`). Sélectionnez **Store virtual disk as a single file** pour de meilleures performances, puis cliquez sur **Next**.
7. Sur le dernier écran, ne démarrez pas tout de suite la machine. Cliquez sur **Customize Hardware...** pour ajuster la couche réseau :
   * Par défaut, une première carte réseau est présente. Laissez-la configurée sur **NAT** (elle servira à télécharger les mises à jour initiales et les fonctionnalités).
   * Cliquez sur **Add...** en bas, sélectionnez **Network Adapter** et cliquez sur **Finish**.
   * Sélectionnez cette nouvelle carte réseau (**Network Adapter 2**) et basculez-la sur **LAN Segment**.
   * Cliquez sur le bouton **LAN Segments...**, créez un segment privé si aucun n'existe (ex: `LAN-Entreprise`), puis sélectionnez-le dans le menu déroulant.
   * Assurez-vous que la case **Connect at power on** est bien cochée pour les deux cartes.
8. Cliquez sur **Close** puis sur **Finish** pour lancer la création.

---

## Étape 2 : Installation du Système d'Exploitation
**📍 Position de l'utilisateur :** Console de la VM Windows Server nouvellement démarrée.

1. Démarrez la machine virtuelle. Si le message `Press any key to boot from CD or DVD` apparaît, appuyez immédiatement sur une touche de votre clavier.
2. Dans l'assistant d'installation Windows Server :
   * Sélectionnez la langue, le format horaire et le clavier correspondant à vos préférences. Cliquez sur **Suivant**.
   * Cliquez sur **Installer maintenant**.
3. **⚠️ Choix crucial de la version :** Lorsque la liste des systèmes s'affiche, veillez à sélectionner une version contenant la mention **(Expérience de bureau)** ou **(Desktop Experience)**. Si vous choisissez la version standard sans cette mention, vous n'aurez pas d'interface graphique (uniquement une invite de commandes). Cliquez sur **Suivant**.
4. Acceptez les termes du contrat de licence.
5. Choisissez le type d'installation **Personnalisé : installer uniquement Windows (avancé)**.
6. Sélectionnez l'espace non alloué (votre disque virtuel) et cliquez sur **Suivant**. L'installation commence et la VM va redémarrer automatiquement à la fin du processus.
7. Au redémarrage, définissez un mot de passe fort et complexe pour le compte **Administrateur** local.

---

## Étape 3 : Configuration Réseau Statique (LAN)
**📍 Position de l'utilisateur :** Bureau de Windows Server, connecté avec le compte Administrateur local.

Avant de promouvoir le serveur en contrôleur de domaine, son interface privée doit obligatoirement posséder des paramètres réseau fixes.

1. Sur votre clavier, faites la combinaison de touches `Windows + R` pour ouvrir la fenêtre d'exécution.
2. Tapez exactement `ncpa.cpl` et validez en cliquant sur **OK**. La fenêtre des connexions réseau s'ouvre.
3. Identifiez vos deux cartes réseau :
   * L'une d'elles a reçu une IP automatique via VMware (il s'agit de l'interface **NAT**). Vous pouvez la renommer en `Internet-NAT`.
   * L'autre affiche un triangle jaune ou un réseau non identifié. C'est l'interface connectée au **LAN Segment**. Renommez-la `LAN-Prive`.
4. Faites un clic droit sur la carte `LAN-Prive` et choisissez **Propriétés**.
5. Dans la liste centrale, double-cliquez sur **Protocole Internet version 4 (TCP/IPv4)**.
6. Cochez l'option **Utiliser l'adresse IP suivante** et renseignez les paramètres de votre plan d'adressage d'entreprise :
   * **Adresse IP :** Saisissez l'IP fixe de votre choix (ex: `192.168.10.10`)
   * **Masque de sous-réseau :** `255.255.255.0`
   * **Passerelle par défaut :** Laissez ce champ *vide* (la passerelle Internet est déjà gérée par votre autre carte en NAT).
7. Dans la section des serveurs DNS, cochez **Utiliser l'adresse de serveur DNS suivante** :
   * **Serveur DNS préféré :** Saisissez la même adresse IP que celle que vous venez de donner au serveur (ex: `192.168.10.10`). Le futur contrôleur de domaine doit obligatoirement boucler sur lui-même pour résoudre les noms de l'annuaire.
8. Cliquez sur **OK**, puis à nouveau sur **OK** pour appliquer instantanément.

---

## Étape 4 : Installation du Rôle Active Directory (AD DS)
**📍 Position de l'utilisateur :** Interface du Gestionnaire de serveur (Server Manager) qui s'ouvre automatiquement au démarrage.

1. Dans le tableau de bord du **Gestionnaire de serveur**, cliquez sur **Ajouter des rôles et des fonctionnalités**.
2. Dans l'assistant qui s'ouvre, cliquez sur **Suivant** sur la page de configuration préliminaire.
3. Sélectionnez **Installation basée sur un rôle ou une fonctionnalité** et cliquez sur **Suivant**.
4. Laissez l'option **Sélectionner un serveur du pool de serveurs** cochée. Vérifiez que votre serveur est bien sélectionné dans la liste en dessous, puis cliquez sur **Suivant**.
5. Dans la liste des rôles de serveurs, cochez la case **Services de domaine Active Directory (AD DS)**.
6. Une fenêtre contextuelle s'ouvre automatiquement pour vous proposer d'ajouter les outils d'administration requis. Cliquez sans modification sur **Ajouter des fonctionnalités**.
7. Cliquez sur **Suivant** sur les fenêtres suivantes (Fonctionnalités, AD DS) sans cocher d'options supplémentaires.
8. Sur la page de confirmation, vous pouvez cocher l'option *Redémarrer automatiquement le serveur cible si nécessaire*, puis cliquez sur **Installer**.
9. Attendez que la barre de progression se termine complètement. Une fois l'installation réussie, cliquez sur **Fermer**.

---

## Étape 5 : Promotion du Serveur en Contrôleur de Domaine
**📍 Position de l'utilisateur :** Interface du Gestionnaire de serveur.

Le rôle est installé, mais le serveur n'est pas encore un contrôleur de domaine opérationnel. Il faut maintenant le promouvoir et créer votre forêt.

1. En haut à droite du Gestionnaire de serveur, repérez l'icône de notification en forme de drapeau surmonté d'un **triangle jaune d'avertissement** et cliquez dessus.
2. Cliquez sur le lien bleu intitulé **Promouvoir ce serveur en contrôleur de domaine**. L'assistant de configuration des services de domaine Active Directory s'ouvre.
3. Sur la page de configuration du déploiement, sélectionnez l'option **Ajouter une nouvelle forêt**.
4. Dans le champ **Nom de domaine racine**, saisissez le nom de domaine complet choisi pour votre organisation (ex: `entreprise.local` ou `votre-organisation.local`). Cliquez sur **Suivant**.
5. Sur la page des options du contrôleur de domaine :
   * Laissez les niveaux fonctionnels de la forêt et du domaine sur la version par défaut proposée.
   * Assurez-vous que les cases **Serveur DNS** et **Catalogue global (GC)** sont bien cochées.
   * Définissez un mot de passe sécurisé pour le **Mode de restauration des services d'annuaire (DSRM)**. Ce mot de passe est indépendant du compte administrateur et servira en cas de maintenance critique de la base AD. Cliquez sur **Suivant**.
6. Sur la page Options DNS, un avertissement concernant la délégation DNS peut apparaître. Ignorez-le (il est normal de ne pas pouvoir créer de délégation sur une nouvelle zone racine privée) et cliquez sur **Suivant**.
7. Laissez l'assistant attribuer automatiquement le **Nom NetBIOS du domaine** (généralement la première partie de votre nom de domaine en majuscules, ex: `ENTREPRISE`), puis cliquez sur **Suivant**.
8. Laissez les chemins d'accès par défaut pour les dossiers de la base de données NTDS, des journaux et du dossier SYSVOL. Cliquez sur **Suivant**.
9. Vérifiez le récapitulatif de vos choix techniques sur la page d'examen des options et cliquez sur **Suivant**.
10. L'assistant procède à la vérification des composants requis. Une fois le message de validation affiché en haut, cliquez sur **Installer**.
11. Une fois la promotion achevée, le serveur va redémarrer automatiquement pour initialiser le nouvel annuaire d'entreprise.


# Complément de Configuration : Création de la Plage d'Adresses DHCP (Étendue)

Ce module s'insère directement dans le guide du serveur Windows après l'Étape 5 (Promotion en Contrôleur de Domaine). Il détaille l'activation et la configuration de l'étendue DHCP pour distribuer automatiquement les configurations IP aux clients du LAN privé.

---

## Étape 6 : Configuration de l'Étendue du Serveur DHCP
**📍 Position de l'utilisateur :** Bureau de Windows Server — Connecté en tant qu'Administrateur du domaine.

Le rôle DHCP étant installé, il est nécessaire de définir la plage réseau (l'étendue) que le serveur va distribuer de manière dynamique sur votre segment privé.

1. **Ouverture de la console DHCP :**
   - Dans le **Gestionnaire de serveur**, cliquez sur le menu **Outils** en haut à droite.
   - Sélectionnez **DHCP** dans la liste déroulante. La console de gestion DHCP s'ouvre.
2. **Création d'une Nouvelle Étendue :**
   - Dans le volet de gauche, développez le nom de votre serveur (ex: `WINDOWS-AD.entreprise.local`).
   - Faites un clic droit sur le nœud **IPv4** et sélectionnez **Nouvelle étendue...**.
   - L'assistant de création s'ouvre. Cliquez sur **Suivant**.
3. **Nommage de l'étendue :**
   - Saisissez un nom explicite (ex: `LAN-Clients-Entreprise`) et une description facultative, puis cliquez sur **Suivant**.
4. **Définition de la Plage d'Adresses :**
   - Renseignez les bornes de la plage IP que le serveur distribuera automatiquement à vos machines clientes :
     * **Adresse IP de début :** Saisissez la première IP de votre plage (ex: `192.168.10.50`).
     * **Adresse IP de fin :** Saisissez la dernière IP de votre plage (ex: `192.168.10.200`).
   - En bas, vérifiez ou ajustez la configuration du masque de sous-réseau :
     * **Longueur :** `24`
     * **Masque de sous-réseau :** `255.255.255.0`
   - Cliquez sur **Suivant**.
5. **Ajout d'exclusions (Optionnel) :**
   - Si vous devez réserver des adresses spécifiques au sein de cette plage pour d'autres serveurs ou équipements fixes, saisissez-les ici. Sinon, laissez vide et cliquez sur **Suivant**.
6. **Durée du Bail :**
   - Laissez la valeur par défaut (généralement 8 jours) ou adaptez-la selon vos besoins, puis cliquez sur **Suivant**.
7. **Configuration des Options DHCP (Crucial pour le Domaine) :**
   - Sur la page vous demandant si vous souhaitez configurer les options DHCP maintenant, cochez **Oui, je veux configurer ces options maintenant** et cliquez sur **Suivant**.
   - **Option 003 (Passerelle par défaut / Routeur) :** Saisissez l'adresse IP de votre routeur ou passerelle si vous en possédez une sur ce LAN (ex: `192.168.10.1`), cliquez sur **Ajouter**, puis sur **Suivant**. Si le LAN est purement isolé sans passerelle, cliquez directement sur **Suivant**.
   - **Option 006 (Serveur DNS) :** L'assistant doit normalement avoir pré-rempli cette zone avec le nom de votre domaine racine et l'IP fixe de votre contrôleur de domaine (ex: `192.168.10.10`). Si ce n'est pas le cas, ajoutez manuellement l'IP fixe de votre serveur Windows AD. *C'est cette option qui permet aux clients de localiser l'Active Directory.* Cliquez sur **Suivant**.
   - **Option 044 (Serveurs WINS) :** Laissez vide et cliquez sur **Suivant**.
8. **Activation de l'Étendue :**
   - Cochez **Oui, je veux activer cette étendue maintenant** et cliquez sur **Suivant**, puis sur **Terminer**.
9. **Autorisation du serveur dans l'Active Directory :**
   - Dans la console DHCP, si une icône rouge apparaît sur le nœud IPv4, faites un clic droit sur le nom de votre serveur dans le volet de gauche et sélectionnez **Autoriser** (Authorize). 
   - Appuyez sur `F5` pour rafraîchir : les icônes doivent passer au vert, confirmant que le serveur distribue désormais les adresses IP sur votre LAN segment.

---

## Étape 7 : Vérification et Maintenance Post-Installation
**📍 Position de l'utilisateur :** Écran de connexion et Gestionnaire de serveur.

1. Lors de l'ouverture de session, le compte de connexion s'affiche désormais sous la forme `DOMAINE\Administrateur` (ex: `ENTREPRISE\Administrateur`), confirmant que la machine est devenue un contrôleur de domaine. Saisissez votre mot de passe pour ouvrir le bureau.
2. Ouvrez le **Gestionnaire de serveur**. Les onglets rouges d'initialisation vont passer progressivement au vert.
3. Si des alertes de services ou d'événements persistantes apparaissent à la suite du redémarrage, vous pouvez forcer le rafraîchissement ou aligner la pile des services en ouvrant une invite de commandes (CMD) en mode Administrateur pour exécuter :
   ```cmd
   net stop dns && net start dns
   net stop dhcpserver && net start dhcpserver
