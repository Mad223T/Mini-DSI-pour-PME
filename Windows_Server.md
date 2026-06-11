\# Guide d'Installation et de Configuration du Serveur Active Directory (Windows Server)



Ce guide décrit pas à pas la création, l'installation et la configuration complète du contrôleur de domaine Windows Server sur l'hyperviseur VMware Workstation.



\---



\## Étape 1 : Préparation de l'environnement virtuel dans VMware Workstation

\*\*📍 Position de l'utilisateur :\*\* Logiciel VMware Workstation sur votre machine physique.



1\. Ouvrez VMware Workstation et cliquez sur \*\*Create a New Virtual Machine\*\* (Créer une nouvelle machine virtuelle).

2\. Choisissez le mode \*\*Typical (recommended)\*\* et cliquez sur \*\*Next\*\*.

3\. Sélectionnez \*\*Installer disc image file (iso)\*\*, cliquez sur \*\*Browse\*\* et chargez votre fichier ISO officiel de Windows Server. Cliquez sur \*\*Next\*\*.

4\. Saisissez vos informations de licence si vous en possédez une, ou laissez vide pour la version d'évaluation. Définissez le compte administrateur initial et cliquez sur \*\*Next\*\*.

5\. Nommez la machine virtuelle (ex: `Windows-Server-AD`) et choisissez son emplacement de stockage sur votre disque dur.

6\. Spécifiez la taille maximale du disque (minimum conseillé : `40 GB` à `60 GB`). Sélectionnez \*\*Store virtual disk as a single file\*\* pour de meilleures performances, puis cliquez sur \*\*Next\*\*.

7\. Sur le dernier écran, ne démarrez pas tout de suite la machine. Cliquez sur \*\*Customize Hardware...\*\* pour ajuster la couche réseau :

&#x20;  \* Par défaut, une première carte réseau est présente. Laissez-la configurée sur \*\*NAT\*\* (elle servira à télécharger les mises à jour initiales et les fonctionnalités).

&#x20;  \* Cliquez sur \*\*Add...\*\* en bas, sélectionnez \*\*Network Adapter\*\* et cliquez sur \*\*Finish\*\*.

&#x20;  \* Sélectionnez cette nouvelle carte réseau (\*\*Network Adapter 2\*\*) et basculez-la sur \*\*LAN Segment\*\*.

&#x20;  \* Cliquez sur le bouton \*\*LAN Segments...\*\*, créez un segment privé si aucun n'existe (ex: `LAN-Entreprise`), puis sélectionnez-le dans le menu déroulant.

&#x20;  \* Assurez-vous que la case \*\*Connect at power on\*\* est bien cochée pour les deux cartes.

8\. Cliquez sur \*\*Close\*\* puis sur \*\*Finish\*\* pour lancer la création.



\---



\## Étape 2 : Installation du Système d'Exploitation

\*\*📍 Position de l'utilisateur :\*\* Console de la VM Windows Server nouvellement démarrée.



1\. Démarrez la machine virtuelle. Si le message `Press any key to boot from CD or DVD` apparaît, appuyez immédiatement sur une touche de votre clavier.

2\. Dans l'assistant d'installation Windows Server :

&#x20;  \* Sélectionnez la langue, le format horaire et le clavier correspondant à vos préférences. Cliquez sur \*\*Suivant\*\*.

&#x20;  \* Cliquez sur \*\*Installer maintenant\*\*.

3\. \*\*⚠️ Choix crucial de la version :\*\* Lorsque la liste des systèmes s'affiche, veillez à sélectionner une version contenant la mention \*\*(Expérience de bureau)\*\* ou \*\*(Desktop Experience)\*\*. Si vous choisissez la version standard sans cette mention, vous n'aurez pas d'interface graphique (uniquement une invite de commandes). Cliquez sur \*\*Suivant\*\*.

4\. Acceptez les termes du contrat de licence.

5\. Choisissez le type d'installation \*\*Personnalisé : installer uniquement Windows (avancé)\*\*.

6\. Sélectionnez l'espace non alloué (votre disque virtuel) et cliquez sur \*\*Suivant\*\*. L'installation commence et la VM va redémarrer automatiquement à la fin du processus.

7\. Au redémarrage, définissez un mot de passe fort et complexe pour le compte \*\*Administrateur\*\* local.



\---



\## Étape 3 : Configuration Réseau Statique (LAN)

\*\*📍 Position de l'utilisateur :\*\* Bureau de Windows Server, connecté avec le compte Administrateur local.



Avant de promouvoir le serveur en contrôleur de domaine, son interface privée doit obligatoirement posséder des paramètres réseau fixes.



1\. Sur votre clavier, faites la combinaison de touches `Windows + R` pour ouvrir la fenêtre d'exécution.

2\. Tapez exactement `ncpa.cpl` et validez en cliquant sur \*\*OK\*\*. La fenêtre des connexions réseau s'ouvre.

3\. Identifiez vos deux cartes réseau :

&#x20;  \* L'une d'elles a reçu une IP automatique via VMware (il s'agit de l'interface \*\*NAT\*\*). Vous pouvez la renommer en `Internet-NAT`.

&#x20;  \* L'autre affiche un triangle jaune ou un réseau non identifié. C'est l'interface connectée au \*\*LAN Segment\*\*. Renommez-la `LAN-Prive`.

4\. Faites un clic droit sur la carte `LAN-Prive` et choisissez \*\*Propriétés\*\*.

5\. Dans la liste centrale, double-cliquez sur \*\*Protocole Internet version 4 (TCP/IPv4)\*\*.

6\. Cochez l'option \*\*Utiliser l'adresse IP suivante\*\* et renseignez les paramètres de votre plan d'adressage d'entreprise :

&#x20;  \* \*\*Adresse IP :\*\* Saisissez l'IP fixe de votre choix (ex: `192.168.10.10`)

&#x20;  \* \*\*Masque de sous-réseau :\*\* `255.255.255.0`

&#x20;  \* \*\*Passerelle par défaut :\*\* Laissez ce champ \*vide\* (la passerelle Internet est déjà gérée par votre autre carte en NAT).

7\. Dans la section des serveurs DNS, cochez \*\*Utiliser l'adresse de serveur DNS suivante\*\* :

&#x20;  \* \*\*Serveur DNS préféré :\*\* Saisissez la même adresse IP que celle que vous venez de donner au serveur (ex: `192.168.10.10`). Le futur contrôleur de domaine doit obligatoirement boucler sur lui-même pour résoudre les noms de l'annuaire.

8\. Cliquez sur \*\*OK\*\*, puis à nouveau sur \*\*OK\*\* pour appliquer instantanément.



\---



\## Étape 4 : Installation du Rôle Active Directory (AD DS)

\*\*📍 Position de l'utilisateur :\*\* Interface du Gestionnaire de serveur (Server Manager) qui s'ouvre automatiquement au démarrage.



1\. Dans le tableau de bord du \*\*Gestionnaire de serveur\*\*, cliquez sur \*\*Ajouter des rôles et des fonctionnalités\*\*.

2\. Dans l'assistant qui s'ouvre, cliquez sur \*\*Suivant\*\* sur la page de configuration préliminaire.

3\. Sélectionnez \*\*Installation basée sur un rôle ou une fonctionnalité\*\* et cliquez sur \*\*Suivant\*\*.

4\. Laissez l'option \*\*Sélectionner un serveur du pool de serveurs\*\* cochée. Vérifiez que votre serveur est bien sélectionné dans la liste en dessous, puis cliquez sur \*\*Suivant\*\*.

5\. Dans la liste des rôles de serveurs, cochez la case \*\*Services de domaine Active Directory (AD DS)\*\*.

6\. Une fenêtre contextuelle s'ouvre automatiquement pour vous proposer d'ajouter les outils d'administration requis. Cliquez sans modification sur \*\*Ajouter des fonctionnalités\*\*.

7\. Cliquez sur \*\*Suivant\*\* sur les fenêtres suivantes (Fonctionnalités, AD DS) sans cocher d'options supplémentaires.

8\. Sur la page de confirmation, vous pouvez cocher l'option \*Redémarrer automatiquement le serveur cible si nécessaire\*, puis cliquez sur \*\*Installer\*\*.

9\. Attendez que la barre de progression se termine complètement. Une fois l'installation réussie, cliquez sur \*\*Fermer\*\*.



\---



\## Étape 5 : Promotion du Serveur en Contrôleur de Domaine

\*\*📍 Position de l'utilisateur :\*\* Interface du Gestionnaire de serveur.



Le rôle est installé, mais le serveur n'est pas encore un contrôleur de domaine opérationnel. Il faut maintenant le promouvoir et créer votre forêt.



1\. En haut à droite du Gestionnaire de serveur, repérez l'icône de notification en forme de drapeau surmonté d'un \*\*triangle jaune d'avertissement\*\* et cliquez dessus.

2\. Cliquez sur le lien bleu intitulé \*\*Promouvoir ce serveur en contrôleur de domaine\*\*. L'assistant de configuration des services de domaine Active Directory s'ouvre.

3\. Sur la page de configuration du déploiement, sélectionnez l'option \*\*Ajouter une nouvelle forêt\*\*.

4\. Dans le champ \*\*Nom de domaine racine\*\*, saisissez le nom de domaine complet choisi pour votre organisation (ex: `entreprise.local` ou `votre-organisation.local`). Cliquez sur \*\*Suivant\*\*.

5\. Sur la page des options du contrôleur de domaine :

&#x20;  \* Laissez les niveaux fonctionnels de la forêt et du domaine sur la version par défaut proposée.

&#x20;  \* Assurez-vous que les cases \*\*Serveur DNS\*\* et \*\*Catalogue global (GC)\*\* sont bien cochées.

&#x20;  \* Définissez un mot de passe sécurisé pour le \*\*Mode de restauration des services d'annuaire (DSRM)\*\*. Ce mot de passe est indépendant du compte administrateur et servira en cas de maintenance critique de la base AD. Cliquez sur \*\*Suivant\*\*.

6\. Sur la page Options DNS, un avertissement concernant la délégation DNS peut apparaître. Ignorez-le (il est normal de ne pas pouvoir créer de délégation sur une nouvelle zone racine privée) et cliquez sur \*\*Suivant\*\*.

7\. Laissez l'assistant attribuer automatiquement le \*\*Nom NetBIOS du domaine\*\* (généralement la première partie de votre nom de domaine en majuscules, ex: `ENTREPRISE`), puis cliquez sur \*\*Suivant\*\*.

8\. Laissez les chemins d'accès par défaut pour les dossiers de la base de données NTDS, des journaux et du dossier SYSVOL. Cliquez sur \*\*Suivant\*\*.

9\. Vérifiez le récapitulatif de vos choix techniques sur la page d'examen des options et cliquez sur \*\*Suivant\*\*.

10\. L'assistant procède à la vérification des composants requis. Une fois le message de validation affiché en haut, cliquez sur \*\*Installer\*\*.

11\. Une fois la promotion achevée, le serveur va redémarrer automatiquement pour initialiser le nouvel annuaire d'entreprise.



\---



\## Étape 6 : Vérification et Maintenance Post-Installation

\*\*📍 Position de l'utilisateur :\*\* Écran de connexion et Gestionnaire de serveur.



1\. Lors de l'ouverture de session, le compte de connexion s'affiche désormais sous la forme `DOMAINE\\Administrateur` (ex: `ENTREPRISE\\Administrateur`), confirmant que la machine est devenue un contrôleur de domaine. Saisissez votre mot de passe pour ouvrir le bureau.

2\. Ouvrez le \*\*Gestionnaire de serveur\*\*. Les onglets rouges d'initialisation vont passer progressivement au vert.

3\. Si des alertes de services ou d'événements persistantes apparaissent à la suite du redémarrage, vous pouvez forcer le rafraîchissement ou aligner la pile des services en ouvrant une invite de commandes (CMD) en mode Administrateur pour exécuter :

&#x20;  ```cmd

&#x20;  net stop dns \&\& net start dns

&#x20;  net stop dhcpserver \&\& net start dhcpserver

