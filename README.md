# Audit de Sécurité sur une Machine Debian 12 avec Serveurs OpenSSH et Web

## 1. Introduction

Cet audit de sécurité se concentre sur une machine Debian 12 fraîchement installée, comportant un serveur OpenSSH et un serveur Web. Les configurations utilisées sont celles par défaut, sans durcissement préalable. L'audit ne couvre pas l'application **Vulnerable Light App** (VLA), qui sera auditée ultérieurement. L'objectif est de découvrir les potentielles failles liées aux services actifs (SSH, Web) et aux configurations système par défaut, ainsi que de proposer des recommandations d'amélioration.

> **Note** : Pour faciliter l'administration de la machine, l'utilisateur principal a été ajouté au groupe `sudoers`, lui conférant des privilèges administratifs.

## 2. Audit de la Configuration de la Machine

### 2.1. Informations Système
- **Version de Debian** : 12 (Bookworm)
- **Services en cours d'exécution** :
  - Serveur OpenSSH
  - Serveur Web (type à déterminer)
- **Configuration par défaut** : Pas de durcissement supplémentaire appliqué.

## 3. Audit du Serveur OpenSSH

### 3.1. Configuration par Défaut
Le serveur OpenSSH est configuré avec des paramètres standards sur une installation Debian fraîche :
- **Port** : 22 (port par défaut)
- **Authentification par mot de passe** : Activée
- **Accès root** : Possiblement activé (```PermitRootLogin yes``` par défaut)
- **Version d'OpenSSH** : Dernière version stable fournie avec Debian 12
- **Autres paramètres** : La configuration par défaut utilise des clés de chiffrement modernes mais permet l'accès par mot de passe.

### 3.2. Vulnérabilités Potentielles
- **Accès Root via SSH** : Par défaut, l'accès root est activé, ce qui expose la machine à des tentatives de force brute ciblant le compte administrateur.
- **Authentification par mot de passe** : L'utilisation d'une authentification par mot de passe sans restrictions supplémentaires augmente les risques d'attaques par dictionnaire ou force brute.
- **Absence de limitation de tentatives de connexion** : Aucune limitation par défaut pour les tentatives de connexion SSH, ce qui permet des attaques répétées sans restriction.
- **Absence de journalisation renforcée** : Le journal des connexions SSH est enregistré, mais sans mesures proactives comme l'alerte ou la détection d'attaques répétées.

### 3.3. Recommandations d'Amélioration
- **Désactiver l'accès root** : Modifier ```/etc/ssh/sshd_config``` pour désactiver l'accès root en modifiant ```PermitRootLogin``` à ```no```.
  ```PermitRootLogin no```
- **Désactiver l'authentification par mot de passe** : Favoriser l'utilisation des clés SSH en désactivant l'authentification par mot de passe.
  ```PasswordAuthentication no```
- **Activer la journalisation et l'alerte des tentatives échouées** : Configurer ```fail2ban``` pour surveiller les tentatives SSH échouées et bannir automatiquement les adresses IP après plusieurs tentatives.
  ```sudo apt install fail2ban```
- **Changer le port SSH par défaut** : Utiliser un port non standard pour réduire l'exposition aux scanners de port.
  ```Port 2222```

## 4. Audit du Serveur Web

### 4.1. Configuration par Défaut
Le serveur Web installé (Apache ou Nginx selon les paquets par défaut) est configuré avec des paramètres de base :
- **Port** : 80 (HTTP), 443 (HTTPS si activé)
- **Indexation des répertoires** : Possiblement activée
- **Support SSL** : Non activé par défaut
- **Fichiers de configuration** : Utilisation des fichiers de configuration par défaut sans restrictions spécifiques.

### 4.2. Vulnérabilités Potentielles
- **Manque de HTTPS** : Sans configuration HTTPS par défaut, le serveur expose les communications en clair, ce qui peut permettre des attaques de type Man-in-the-Middle (MITM).
- **Indexation des répertoires** : Si l’indexation des répertoires est activée, des informations sensibles peuvent être exposées publiquement.
- **Absence de configuration de sécurité** : Les headers de sécurité comme ```Content-Security-Policy```, ```X-Frame-Options```, ```X-XSS-Protection```, etc., ne sont pas configurés par défaut, rendant le serveur vulnérable aux attaques de type cross-site scripting (XSS) ou clickjacking.
- **Fichiers de configuration accessibles** : Les fichiers de configuration peuvent être consultables si des erreurs de permission existent.

### 4.3. Recommandations d'Amélioration
- **Forcer l'utilisation de HTTPS** : Installer et configurer un certificat SSL (Let’s Encrypt recommandé) pour forcer les connexions en HTTPS.
  ```sudo apt install certbot python3-certbot-nginx```
  ```sudo certbot --nginx```
- **Désactiver l’indexation des répertoires** : Dans la configuration du serveur Web (Apache ou Nginx), désactiver l’indexation des répertoires.
  - **Pour Apache** :
    ```Options -Indexes```
  - **Pour Nginx** :
    ```autoindex off;```
- **Ajouter des en-têtes de sécurité** : Configurer les en-têtes de sécurité dans le fichier de configuration du serveur.
  - **Pour Apache** :
    ```Header always set X-Frame-Options 'DENY'```
    ```Header always set X-XSS-Protection '1; mode=block'```
    ```Header always set Content-Security-Policy 'default-src 'self''```
  - **Pour Nginx** :
    ```add_header X-Frame-Options 'DENY';```
    ```add_header X-XSS-Protection '1; mode=block';```
    ```add_header Content-Security-Policy 'default-src 'self';```
- **Restreindre l'accès aux fichiers de configuration** : Vérifiez les permissions sur ```/etc/nginx/``` ou ```/etc/apache2/``` pour vous assurer que seuls les utilisateurs administratifs peuvent y accéder.

## 5. Audit de la Configuration Système de Base

### 5.1. Comptes Utilisateurs et Permissions
- **Utilisateurs par défaut** : La configuration par défaut inclut plusieurs comptes système qui, même s'ils ne permettent pas de connexions directes, pourraient être exploités si leurs permissions sont mal configurées.
- **Permissions SUID/SGID** : Les fichiers avec des permissions SUID/SGID peuvent représenter une surface d'attaque s’ils sont mal sécurisés.

### 5.2. Vulnérabilités Potentielles
- **Utilisateurs inutilisés** : Certains comptes système peuvent représenter une surface d'attaque s'ils ne sont pas correctement désactivés ou configurés.
- **Fichiers SUID/SGID** : Les fichiers avec des permissions SUID/SGID par défaut peuvent être exploités pour des élévations de privilèges.

### 5.3. Recommandations d'Amélioration
- **Vérifier et limiter les comptes utilisateurs** : Désactiver ou supprimer les comptes inutilisés ou non nécessaires.
  ```sudo userdel <username>```
- **Vérifier les fichiers SUID/SGID** : Inspecter les fichiers avec des permissions SUID/SGID et désactiver ces permissions si elles ne sont pas nécessaires.
  ```find / -perm /4000```
  ```chmod u-s <fichier>```

## 6. Journalisation et Supervision

### 6.1. Journalisation par Défaut
- Le système Debian enregistre les événements via ```rsyslog``` et stocke les logs dans ```/var/log/```. Cependant, la supervision active n’est pas configurée par défaut.

### 6.2. Vulnérabilités Potentielles
- **Absence de surveillance proactive** : Sans surveillance active, les incidents de sécurité peuvent passer inaperçus.

### 6.3. Recommandations d'Amélioration
- **Configurer une solution de supervision** : Installer des outils comme ```auditd``` ou ```fail2ban``` pour surveiller les activités suspectes.
  ```sudo apt install auditd```
  ```sudo systemctl enable auditd```
- **Configurer la rotation des logs** : Assurez-vous que ```logrotate``` est correctement configuré pour éviter l'épuisement de l'espace disque.

## 7. Conclusion

Cet audit révèle que la configuration par défaut de Debian 12, bien que sécurisée dans une certaine mesure, présente des vulnérabilités potentielles sur les services OpenSSH et Web. Des améliorations simples, telles que la désactivation de l'accès root via SSH, l'installation de certificats SSL, et l'ajout d'en-têtes de sécurité sur le serveur Web, peuvent significativement renforcer la sécurité du système.

Des étapes supplémentaires peuvent être mises en œuvre pour sécuriser les permissions des fichiers sensibles et activer des mécanismes de surveillance proactive.
>>>>>>> c15abb2 (rendu V0.2)
