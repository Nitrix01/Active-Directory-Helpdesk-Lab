# Active Directory Helpdesk Lab

**Laboratoire Windows Server 2022 — Administration Active Directory et résolution d'incidents de support informatique N1**

## Présentation du projet

Ce projet présente la création et l'administration d'un environnement Active Directory virtualisé sous VMware Workstation.

L'objectif est de développer des compétences pratiques en support informatique de niveau 1 à travers la configuration d'une infrastructure Windows Server et la simulation d'incidents utilisateurs.

Le laboratoire permet notamment de travailler sur :

- L'administration des comptes utilisateurs Active Directory.
- La gestion des unités d'organisation (OU) et des groupes de sécurité.
- Les politiques de mots de passe et de verrouillage.
- La configuration de partages réseau SMB.
- La gestion des autorisations NTFS.
- Le diagnostic d'incidents avec PowerShell et les journaux Windows.
- La documentation des interventions de support informatique.

Les incidents sont reproduits volontairement, puis diagnostiqués, corrigés et validés.

---

## 1. Environnement technique

| Composant | Configuration |
|---|---|
| Hyperviseur | VMware Workstation |
| Système serveur | Windows Server 2022 Standard Evaluation |
| Contrôleur de domaine | DC01 |
| Domaine Active Directory | LAB.local |
| Adresse IP du serveur | 192.168.31.10 |
| Réseau virtuel | VMnet8 — NAT |
| Sous-réseau | 192.168.31.0/24 |
| Passerelle | 192.168.31.2 |
| DNS | DC01 — 192.168.31.10 |
| Mémoire virtuelle | 3 Go |
| Processeurs virtuels | 2 vCPU |
| Stockage virtuel | 60 Go |

### Rôles installés

- Active Directory Domain Services (AD DS)
- DNS Server

Le serveur `DC01` assure les fonctions de contrôleur de domaine et de serveur DNS pour l'environnement `LAB.local`.

---

## 2. Architecture Active Directory

L'annuaire est organisé autour d'une unité d'organisation principale nommée `Entreprise`, divisée en quatre services.

```text
LAB.local
│
└── Entreprise
    ├── Informatique
    │   └── Thomas Martin
    │
    ├── RH
    │   ├── Julie Durand
    │   └── Emma Petit
    │
    ├── Comptabilite
    │   └── Hugo Moreau
    │
    └── Direction
        └── Lucas Bernard
```

### Comptes utilisateurs

| Utilisateur | Identifiant AD | Service |
|---|---|---|
| Thomas Martin | t.martin | Informatique |
| Julie Durand | j.durand | Ressources humaines |
| Emma Petit | e.petit | Ressources humaines |
| Hugo Moreau | h.moreau | Comptabilité |
| Lucas Bernard | l.bernard | Direction |

Tous les utilisateurs sont fictifs et ont été créés pour les besoins du laboratoire.

---

## 3. Gestion des groupes et des permissions

Un groupe de sécurité Active Directory a été créé pour gérer les autorisations du service RH.

**Groupe :** `DL RH Modification`  
**Type :** Security  
**Portée :** Domain Local

### Membres

- Julie Durand
- Emma Petit

### Partage réseau

Le dossier RH est hébergé sur le contrôleur de domaine :

```text
C:\Partages\RH
```

Il est accessible par le réseau via :

```text
\\DC01\RH
```

### Autorisations

| Niveau | Groupe | Permission |
|---|---|---|
| SMB | DL RH Modification | Change |
| NTFS | DL RH Modification | Modify |

Les autorisations sont attribuées au groupe de sécurité plutôt qu'individuellement aux utilisateurs.

Cette organisation facilite l'administration des accès et permet de respecter le principe du moindre privilège.

---

## 4. Incidents de support informatique

Quatre incidents ont été simulés et résolus dans le laboratoire.

Chaque intervention comprend :

1. La description du problème utilisateur.
2. La reproduction de l'incident.
3. Le diagnostic technique.
4. L'identification de la cause.
5. L'application d'une correction.
6. La validation du rétablissement du service.
7. La documentation de l'intervention.

### INC-002 — Réinitialisation d'un mot de passe Active Directory

**Scénario :** une utilisatrice du service RH a oublié son mot de passe Windows.

**Interventions réalisées :**

- Vérification de l'état du compte AD.
- Réinitialisation administrative du mot de passe.
- Configuration du changement obligatoire à la prochaine connexion.
- Analyse de l'erreur Windows 1907.
- Diagnostic d'un refus lié à la politique de complexité.
- Changement du mot de passe.
- Validation de l'authentification SMB.

**Résultat :** authentification rétablie.

[Consulter INC-002](docs/incidents/INC-002-reset-password.md)

### INC-003 — Verrouillage d'un compte Active Directory

**Scénario :** un utilisateur du service Comptabilité ne peut plus se connecter après plusieurs tentatives avec un mot de passe incorrect.

**Interventions réalisées :**

- Configuration d'une politique de verrouillage à trois tentatives.
- Reproduction du verrouillage via SMB.
- Vérification de l'état `LockedOut`.
- Recherche de l'événement Windows 4740.
- Identification de l'ordinateur à l'origine du verrouillage.
- Déverrouillage du compte avec PowerShell.
- Validation de l'authentification.

**Résultat :** compte déverrouillé et authentification rétablie.

[Consulter INC-003](docs/incidents/INC-003-account-lockout.md)

### INC-004 — Permissions NTFS insuffisantes

**Scénario :** une utilisatrice du service RH peut consulter les fichiers partagés, mais ne peut plus en créer ou modifier.

**Interventions réalisées :**

- Vérification des permissions SMB et NTFS.
- Sauvegarde des ACL.
- Simulation d'une réduction des droits NTFS.
- Reproduction d'un refus d'écriture.
- Vérification des appartenances aux groupes AD.
- Identification de la permission `ReadAndExecute` comme cause.
- Restauration de l'autorisation `Modify`.
- Validation des opérations de lecture et d'écriture.

**Résultat :** droits de modification rétablis.

[Consulter INC-004](docs/incidents/INC-004-ntfs-permissions.md)

### INC-005 — Accès refusé à un dossier partagé

**Scénario :** une utilisatrice du service RH perd complètement l'accès à un dossier partagé.

**Interventions réalisées :**

- Reproduction de l'erreur `Access is denied`.
- Vérification de l'état du compte utilisateur.
- Diagnostic des appartenances aux groupes AD.
- Identification d'une appartenance manquante.
- Réintégration de l'utilisatrice dans le groupe de sécurité.
- Renouvellement de la connexion SMB.
- Validation des opérations de lecture et d'écriture.

**Résultat :** accès au partage rétabli.

[Consulter INC-005](docs/incidents/INC-005-shared-folder-access.md)

---

## 5. Outils et commandes utilisés

### Active Directory

- `Get-ADUser`
- `Get-ADGroupMember`
- `Get-ADPrincipalGroupMembership`
- `Add-ADGroupMember`
- `Set-ADAccountPassword`
- `Set-ADUser`
- `Unlock-ADAccount`
- `Get-ADDefaultDomainPasswordPolicy`

### Windows Server et permissions

- `Get-SmbShare`
- `Get-SmbShareAccess`
- `Get-Acl`
- `icacls`
- `net use`

### Diagnostic et journalisation

- Observateur d'événements Windows
- `Get-WinEvent`
- Événement de sécurité 4740
- `Get-ADUser -Properties LockedOut`

---

## 6. Compétences développées

### Administration Windows Server

- Installation et configuration d'un contrôleur de domaine.
- Configuration DNS.
- Gestion des utilisateurs, groupes et unités d'organisation.
- Administration des ressources partagées.
- Gestion des autorisations NTFS et SMB.

### Support informatique N1

- Analyse des demandes utilisateurs.
- Diagnostic des incidents d'authentification.
- Réinitialisation des mots de passe.
- Déverrouillage des comptes.
- Résolution des problèmes d'accès aux dossiers partagés.
- Vérification du fonctionnement après intervention.
- Documentation technique des incidents.

### PowerShell

- Administration des objets Active Directory.
- Consultation des propriétés des comptes.
- Analyse des journaux Windows.
- Vérification et modification des autorisations.
- Exécution de tests de diagnostic.

---

## 7. Captures d'écran

Les captures d'écran sont organisées dans le dossier `screenshots/`.

Elles permettent d'illustrer :

- L'architecture Active Directory.
- Les comptes et groupes de sécurité.
- Les permissions SMB et NTFS.
- Les messages d'erreur reproduits.
- Les résultats des commandes PowerShell.
- La validation des résolutions.

Les captures publiées ne doivent contenir aucun mot de passe ni aucune information d'identification sensible.

---

## 8. Limites et perspectives

Ce laboratoire constitue un environnement de formation et non une infrastructure de production.

Les tests ont principalement été réalisés depuis le contrôleur
