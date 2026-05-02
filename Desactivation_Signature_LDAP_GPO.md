# Documentation Technique : Désactivation de l'exigence de signature LDAP via GPO

**Domaine technique :** Administration système et réseau, Cybersécurité
**Environnement :** Windows Server 2025
**Cible :** BTS SIO (SISR) - Épreuve E6

---

## 1. Contexte

Par défaut, les environnements Active Directory récents (notamment sur Windows Server 2025) imposent un haut niveau de sécurité par défaut, exigeant que les requêtes LDAP (Lightweight Directory Access Protocol) soient signées. C'est l'un des mécanismes visant à forcer l'utilisation de flux sécurisés (LDAPS, souvent associé au port 636).

Cependant, il arrive en entreprise de devoir autoriser temporairement ou durablement le **port 389 en clair** pour permettre à des services tiers spécifiques ou à des équipements "legacy" (tels que des copieurs vieillissants ou d'anciens applicatifs métier) d'interroger l'annuaire. Ces systèmes ne prenant souvent pas en charge le chiffrement TLS ni la gestion des certificats, la désactivation de l'exigence de signature s'impose pour maintenir la continuité de leur fonctionnement (authentification de base, requêtes de carnets d'adresses).

---

## 2. Procédure Technique (via gpmc.msc)

La configuration se déroule sur un contrôleur de domaine, en adaptant la politique de sécurité.

1. Appuyez sur `Touche Windows + R`, tapez `gpmc.msc` et validez pour ouvrir la **Console de gestion des stratégies de groupe**.
2. Modifiez la **Default Domain Controllers Policy** (ou de préférence, une GPO dédiée liée à l'unité d'organisation *Domain Controllers*).
3. Accédez au chemin exact suivant dans l'arborescence :
   `Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Options de sécurité`
   *(Dans certains raccourcis ou questions d'examen, on mentionne souvent `Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Options de sécurité`)*.
4. Repérez les deux stratégies de sécurité suivantes et positionnez leur valeur sur "**Aucun**" :
   - **Contrôleur de domaine : exigences de signature du serveur LDAP** : Définir sur `Aucun`.
   - **Sécurité réseau : exigences de signature de client LDAP** : Définir sur `Aucun`.
5. Validez vos paramètres et fermez l'éditeur de GPO.
6. Afin d'appliquer immédiatement les changements, ouvrez une invite de commandes en tant qu'administrateur et exécutez la commande suivante :

```cmd
gpupdate /force
```

---

## 3. Analyse Cybersécurité (Modèle DIC)

Désactiver la signature LDAP impacte directement votre posture de sécurité. Les impacts métiers sur la **Disponibilité**, l'**Intégrité** et la **Confidentialité** se résument ainsi :

| Critère de Sécurité | Impact de la modification |
| :--- | :--- |
| **Disponibilité** | **Amélioration fonctionnelle** : Cette modification permet de rétablir la disponibilité de l'annuaire LDAP pour les systèmes "legacy" qui seraient autrement bloqués par l'impossibilité de respecter les normes de chiffrement serveur. |
| **Intégrité** | **Risque élevé (Man-in-the-Middle)** : En l'absence de signature cryptographique garantissant l'intégrité, une personne malveillante sur le réseau local peut intercepter la requête (attaque *MitM*, de l'Homme du Milieu), puis falsifier ou altérer la réponse retournée par l'Active Directory vers l'application. |
| **Confidentialité**| **Vulnérabilité critique (Sniffing)** : En autorisant le transit en clair via le port 389, les identifiants d'accès d'un *LDAP simple bind* (nom d'utilisateur et mot de passe) peuvent très facilement être interceptés ("sniffés") au moyen d'analyseurs de paquets réseau comme Wireshark. |

---
