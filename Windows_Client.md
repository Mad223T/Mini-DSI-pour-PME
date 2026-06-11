# Guide d'Installation et de Configuration du Poste Client (Windows 10/11)

Ce guide décrit pas à pas la création, l'installation et la configuration complète d'une station de travail cliente Windows sur l'hyperviseur VMware Workstation, ainsi que sa jonction au domaine et le montage automatisé de son répertoire personnel.

---

## Étape 1 : Préparation de l'environnement virtuel dans VMware Workstation
**📍 Position de l'utilisateur :** Logiciel VMware Workstation sur votre machine physique.

1. Ouvrez VMware Workstation et cliquez sur **Create a New Virtual Machine**.
2. Sélectionnez le mode **Typical (recommended)** et cliquez sur **Next**.
3. Sélectionnez **Installer disc image file (iso)**, cliquez sur **Browse**, chargez le fichier ISO de votre système client (Windows 10 ou Windows 11) puis cliquez sur **Next**.
4. Saisissez la clé de produit si demandée, sélectionnez l'édition de Windows souhaitée (Pro ou Enterprise recommandée pour la gestion de domaine) et définissez le nom d'utilisateur initial. Cliquez sur **Next**.
5. Nommez la machine virtuelle (ex: `Client-Windows-PC`) et définissez son emplacement de stockage.
6. Spécifiez la taille maximale du disque (minimum conseillé : `40 GB` à `60 GB`), sélectionnez **Store virtual disk as a single file** et cliquez sur **Next**.
7. Sur l'écran récapitulatif, cliquez sur **Customize Hardware...** pour associer la machine au réseau de l'entreprise :
   * Sélectionnez la carte réseau présente (**Network Adapter**).
   * Basculez sa configuration sur **LAN Segment**.
   * Dans le menu déroulant, sélectionnez précisément le même segment privé que celui utilisé par vos serveurs (ex: `LAN-Entreprise`). *Cela garantit l'étanchéité du réseau et permet au client de communiquer avec le contrôleur de domaine.*
   * Assurez-vous que la case **Connect at power on** est cochée.
8. Cliquez sur **Close** puis sur **Finish** pour valider et démarrer la machine virtuelle.

---

## Étape 2 : Installation du Système d'Exploitation Client
**📍 Position de l'utilisateur :** Console de la VM Client Windows nouvellement démarrée.

1. Si le système le demande, appuyez sur une touche pour démarrer à partir du CD/DVD virtuel.
2. Suivez l'assistant d'installation Windows : choisissez la langue, le format horaire et l'agencement du clavier, puis cliquez sur **Suivant** et **Installer maintenant**.
3. **⚠️ Choix de l'édition :** Si le choix vous est proposé, veillez à installer une version **Professionnelle (Pro)** ou **Entreprise**. Les éditions familiales (*Home / Famille*) ne possèdent pas les fonctionnalités nécessaires pour intégrer un domaine Active Directory.
4. Sélectionnez le type d'installation **Personnalisé : installer uniquement Windows (avancé)**, sélectionnez votre disque virtuel et cliquez sur **Suivant**.
5. Laissez l'installation se dérouler jusqu'au redémarrage automatique.
6. Lors de la phase de configuration initiale (OOBE) :
   * Choisissez votre région et votre disposition de clavier.
   * Lorsque Windows vous demande comment configurer l'appareil, sélectionnez **Configurer pour une organisation** (ou choisissez une configuration locale avec un compte hors connexion si vous préférez créer un compte local temporaire).
   * Créez l'utilisateur local initial et attribuez-lui un mot de passe.

---

## Étape 3 : Attribution des Paramètres Réseau et Validation DNS
**📍 Position de l'utilisateur :** Bureau de la VM Client Windows, connecté sur le compte local temporaire.

Pour joindre un domaine, la machine cliente doit impérativement obtenir une configuration réseau valide et utiliser le contrôleur de domaine comme serveur de résolution DNS.

1. **Vérification de l'adressage (DHCP) :**
   * Faites un clic droit sur le menu Démarrer et sélectionnez **Invite de commandes** ou **Terminal**.
   * Tapez la commande suivante pour vérifier si votre carte réseau reçoit bien une configuration de votre serveur DHCP d'infrastructure :
     ```cmd
     ipconfig
     ```
   * *Si l'adresse IPv4 obtenue est cohérente avec votre plan d'adressage (ex: `192.168.10.X`) et que le suffixe DNS correspond à votre organisation, passez à la sous-étape 3.*
2. **Alternative en cas d'IP Statique :**
   * Si vous n'utilisez pas de serveur DHCP actif, faites `Windows + R`, tapez `ncpa.cpl` et validez.
   * Faites un clic droit sur votre carte réseau $\rightarrow$ **Propriétés** $\rightarrow$ Double-cliquez sur **Protocole Internet version 4 (TCP/IPv4)**.
   * Cochez **Utiliser l'adresse IP suivante** et configurez manuellement une IP libre de votre sous-réseau (ex: `192.168.10.20`), le masque `255.255.255.0`.
   * **Crucial :** Dans le champ **Serveur DNS préféré**, saisissez scrupuleusement l'adresse IP fixe de votre contrôleur de domaine Windows Server (ex: `192.168.10.10`). Cliquez sur **OK** pour valider.
3. **Nettoyage et test de connectivité :**
   * Depuis votre invite de commandes, purgez le cache de résolution et effectuez un test de communication vers votre serveur de fichiers ou votre contrôleur de domaine pour valider la liaison :
     ```cmd
     ipconfig /flushdns
     ipconfig /registerdns
     ping votre-serveur-dns-ou-ip
     ```

---

## Étape 4 : Jonction de la Machine au Domaine Active Directory
**📍 Position de l'utilisateur :** Bureau de la VM Client Windows, connecté sur le compte local temporaire.

1. Sur votre clavier, faites la combinaison de touches `Windows + R`, tapez exactement `sysdm.cpl` et cliquez sur **OK**. La fenêtre des Propriétés système s'ouvre.
2. Dans l'onglet *Nom de l'ordinateur*, cliquez sur le bouton **Modifier...** situé en bas.
3. Dans la zone *Membre de*, cochez l'option **Domaine**.
4. Saisissez le nom complet de votre domaine d'entreprise (ex: `entreprise.local` ou `votre-domaine.local`) et cliquez sur **OK**.
5. Une invite de sécurité s'affiche à l'écran. Saisissez les identifiants d'un compte autorisé à joindre des machines au domaine (généralement le compte **Administrateur** du domaine Windows Server et son mot de passe associé), puis validez.
6. Un message de bienvenue apparaît instantanément : *« Bienvenue dans le domaine votre-domaine.local »*. Cliquez sur **OK**.
7. Le système vous informe qu'un redémarrage est nécessaire. Validez, fermez la fenêtre des propriétés et cliquez sur **Redémarrer maintenant**.

---

## Étape 5 : Ouverture de Session Utilisateur et Montage du Lecteur Réseau
**📍 Position de l'utilisateur :** Écran de verrouillage de la VM Client Windows après son redémarrage.

1. Sur l'écran de connexion, ne sélectionnez pas votre compte local habituel. Cliquez sur **Autre utilisateur** (en bas à gauche).
2. Dans le champ utilisateur, saisissez l'identifiant d'un collaborateur enregistré dans votre Active Directory sous la forme `identifiant@votre-domaine.local` (ex: `m.garnier@entreprise.local`). Saisissez son mot de passe de domaine et validez.
3. L'environnement utilisateur se charge pour la première fois. Une fois sur le Bureau, ouvrez une **Invite de commandes (CMD)**.
4. Pour monter de façon persistante et sécurisée le répertoire personnel hébergé sur le serveur de fichiers distant, exécutez la commande de montage en adaptant l'IP du serveur Samba, le nom du partage invisible et l'identifiant :
   ```cmd
   net use G: \\192.168.10.11\PrivateShares$\m.garnier
