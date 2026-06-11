# Mini-DSI-pour-PME

# Projet Réseau - Infrastructure Active Directory

## Présentation

Ce projet consiste à mettre en place une infrastructure réseau complète sous VMware Workstation comprenant :

- Un contrôleur de domaine Active Directory
- Un serveur de fichiers Linux
- Un serveur de sauvegarde Linux
- Un poste client Windows
- Un service DNS
- Un service DHCP
- Une gestion centralisée des utilisateurs et des unités d'organisation (OU)

L'objectif est de simuler l'infrastructure informatique d'une entreprise afin d'administrer les utilisateurs, les ressources et les services réseau.

---

# Architecture

## Machines virtuelles

| Machine | Rôle | Système |
|----------|--------|----------|
| DC01 | Contrôleur de domaine | Windows Server 2022 |
| FILE01 | Serveur de fichiers | Ubuntu Server |
| BACKUP01 | Serveur de sauvegarde | Ubuntu Server |
| PC01 | Poste client | Windows 10 / 11 |

---

# Plan d'adressage IP

| Équipement | Adresse IP | Masque | Passerelle | DNS |
|-------------|------------|---------|-------------|------|
| Routeur | 192.168.10.254 | 255.255.255.0 | - | - |
| DC01 | 192.168.10.10 | 255.255.255.0 | 192.168.10.254 | 192.168.10.10 |
| FILE01 | 192.168.10.11 | 255.255.255.0 | 192.168.10.254 | 192.168.10.10 |
| BACKUP01 | 192.168.10.12 | 255.255.255.0 | 192.168.10.254 | 192.168.10.10 |
| PC01 | 192.168.10.50 | 255.255.255.0 | 192.168.10.254 | 192.168.10.10 |

---

# Installation de l'infrastructure

## 1. Création des machines virtuelles

Les machines ont été créées sous VMware Workstation :

- DC01
- FILE01
- BACKUP01
- PC01

---

## 2. Installation de Windows Server 2022

Installation du système sur DC01.

Configuration :

- Adresse IP statique
- Renommage du serveur en DC01
- Installation des rôles :
  - Active Directory Domain Services (AD DS)
  - DNS
  - DHCP

---

## 3. Création du domaine

Nom du domaine :

```text
ecoleit.com
```

Promotion du serveur en contrôleur de domaine.

---

## 4. Configuration DNS

Le serveur DNS est hébergé sur :

```text
192.168.10.10
```

Toutes les machines utilisent ce serveur DNS.

---

## 5. Configuration DHCP

Création d'une plage DHCP permettant l'attribution automatique des adresses IP aux postes clients.

Exemple :

```text
192.168.10.100 - 192.168.10.200
```

---

# Organisation Active Directory

## Unités d'organisation (OU)

```text
ecoleit.com
│
├── Direction
├── Tech
└── Commercial
```

---

# Utilisateurs

## Direction

| Nom | Login |
|------|--------|
| Jean Dupont | j.dupont |
| Alice Martin | a.martin |
| Pierre Durand | p.durand |

---

## Tech

| Nom | Login |
|------|--------|
| Marc Garnier | m.garnier |
| Lucas Rousseau | l.rousseau |
| Bob Leclerc | b.leclerc |
| Charlie Masson | c.masson |
| David Moreau | d.moreau |
| Thomas Petit | t.petit |

---

## Commercial

| Nom | Login |
|------|--------|
| Eva Mercier | e.mercier |
| Sophie Lefevre | s.lefevre |
| Thomas Bertrand | t.bertrand |
| Julie Legrand | j.legrand |
| Nicolas Girard | n.girard |
| Emma Andre | e.andre |

---

# Import des utilisateurs

Les utilisateurs sont importés à partir d'un fichier CSV.

Exemple :

```csv
Prenom,Nom,SamAccountName,OU,Role
Marc,Garnier,m.garnier,Tech,Expert-SuperUser
Lucas,Rousseau,l.rousseau,Tech,Expert-SuperUser
Jean,Dupont,j.dupont,Direction,Utilisateur
```

Import réalisé via PowerShell.

---

# Serveur de fichiers

## FILE01

Ubuntu Server 24.04 LTS

Fonctions :

- Partage réseau
- Hébergement des fichiers utilisateurs
- Gestion des droits d'accès

---

# Serveur de sauvegarde

## BACKUP01

Ubuntu Server 24.04 LTS

Fonctions :

- Sauvegarde des données
- Stockage des copies de sécurité
- Protection contre la perte de données

---

# Poste client

## PC01

Machine cliente intégrée au domaine :

```text
ecoleit.com
```

Tests réalisés :

- Authentification AD
- Résolution DNS
- Attribution DHCP
- Accès aux ressources réseau

---

# Tests de validation

✅ Ping entre toutes les machines

✅ Résolution DNS

✅ Attribution DHCP

✅ Création des utilisateurs

✅ Création des OU

✅ Intégration du poste client au domaine

✅ Connexion avec les comptes Active Directory

✅ Communication entre les serveurs

---

# Compétences mises en œuvre

- Virtualisation VMware
- Administration Windows Server
- Active Directory
- DNS
- DHCP
- PowerShell
- Ubuntu Server
- Réseaux TCP/IP
- Gestion des utilisateurs
- Gestion des droits
- Administration système

---

# Auteur

Projet réalisé dans le cadre d'un TP d'administration systèmes et réseaux.

**Samir & Mamadou**
