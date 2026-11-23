# INC-003 — Verrouillage d'un compte Active Directory

**Statut : Résolu — Clôturé**

**Catégorie : Support informatique / Active Directory / Gestion des comptes**

**Priorité : P3 — Moyenne (simulation)**

**Environnement : Windows Server 2022 / Active Directory Domain Services**

---

## 1. Description de l'incident

**Utilisateur concerné :** Hugo Moreau

**Service :** Comptabilité

**Compte Active Directory :** `LAB\h.moreau`

**Domaine :** `LAB.local`

**Contrôleur de domaine :** `DC01`

### Problème signalé

L'utilisateur indique ne plus pouvoir accéder à son compte après plusieurs tentatives de connexion avec un mot de passe incorrect.

Le compte est verrouillé par la politique de sécurité du domaine.

**Impact :** impossibilité de réaliser de nouvelles authentifications avec le compte Active Directory tant que celui-ci reste verrouillé.

## 2. Environnement technique

| Composant | Configuration |
| --- | --- |
| Système d'exploitation | Windows Server 2022 |
| Contrôleur de domaine | DC01 |
| Domaine Active Directory | LAB.local |
| Adresse IP | 192.168.31.10 |
| Utilisateur | h.moreau |
| Politique de verrouillage | 3 tentatives incorrectes |
| Durée de verrouillage | 30 minutes |
| Journal d'événements | Windows Logs / Security |
| Événement étudié | 4740 |

## 3. Reproduction de l'incident

Afin de simuler un incident utilisateur, la politique de verrouillage du domaine a été configurée avec un seuil de trois tentatives incorrectes.

### Vérification de la politique

```
Get-ADDefaultDomainPasswordPolicy |
Select-Object LockoutThreshold, LockoutDuration, LockoutObservationWindow
```

### Simulation des échecs d'authentification

Depuis `DC01`, plusieurs tentatives d'authentification SMB ont été effectuées avec un mot de passe incorrect :

```
net use \\DC01\IPC$ /user:LAB\h.moreau *
```

Le partage administratif `IPC$` a été utilisé pour tester l'authentification réseau sans modifier de fichiers.

Après trois tentatives incorrectes, le compte a été verrouillé.

### Confirmation du verrouillage

```
Get-ADUser -Identity "h.moreau" -Properties LockedOut |
Select-Object Name, LockedOut
```

**Résultat observé :**

```
Name          LockedOut
----          ---------
Hugo Moreau   True
```

## 4. Diagnostic de l'incident

Deux méthodes ont été utilisées pour identifier le verrouillage dans les journaux Windows.

### Méthode 1 — Observateur d'événements

Ouverture de l'Observateur d'événements :

```
eventvwr.msc
```

Navigation :

**Journaux Windows → Sécurité → Filtrer le journal actuel**

Filtre appliqué :

```
4740
```

L'événement correspondant au verrouillage de Hugo Moreau a été identifié.

### Méthode 2 — PowerShell

```
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4740
} -MaxEvents 5 |
Select-Object TimeCreated, Id, Message |
Format-List
```

Cette commande permet d'afficher les derniers événements de verrouillage enregistrés dans le journal Sécurité.

### Informations identifiées

| Champ | Résultat |
| --- | --- |
| Event ID | 4740 |
| Target Account Name | h.moreau |
| Target Domain Name | LAB |
| Caller Computer Name | DC01 |

Le champ `Caller Computer Name` a permis d'identifier `DC01` comme ordinateur à l'origine de la demande de verrouillage.

## 5. Cause racine

**Dépassement du seuil de tentatives d'authentification incorrectes défini dans la politique de verrouillage Active Directory.**

Dans ce scénario, trois mauvais mots de passe ont été volontairement soumis depuis `DC01` avec le compte `LAB\h.moreau`.

Le verrouillage correspond donc au fonctionnement attendu de la politique de sécurité du domaine.

Aucune panne du contrôleur de domaine ou du service Active Directory n'a été identifiée.

## 6. Résolution appliquée

Après vérification du compte concerné et analyse de l'événement 4740, un déverrouillage manuel a été effectué avec PowerShell.

### Commande de résolution

```
Unlock-ADAccount -Identity "h.moreau"
```

Cette commande déverrouille le compte sans réinitialiser son mot de passe.

### Vérification de l'état du compte

```
Get-ADUser -Identity "h.moreau" -Properties LockedOut |
Select-Object Name, LockedOut
```

**Résultat confirmé :**

```
Name          LockedOut
----          ---------
Hugo Moreau   False
```

## 7. Validation de la résolution

Une nouvelle authentification SMB a été effectuée avec le mot de passe correct de Hugo Moreau.

### Test d'authentification

```
net use \\DC01\IPC$ /user:LAB\h.moreau *
```

**Résultat observé :**

```
The command completed successfully.
```

La réussite de cette connexion confirme que le compte peut de nouveau s'authentifier auprès du serveur.

### Nettoyage de la connexion de test

```
net use \\DC01\IPC$ /delete
```

### Résultats des vérifications

| Vérification | Résultat |
| --- | --- |
| Compte initialement verrouillé | Confirmé |
| Événement 4740 identifié | Réussi |
| Ordinateur à l'origine du verrouillage | DC01 |
| Déverrouillage manuel | Réussi |
| État `LockedOut = False` | Confirmé |
| Authentification SMB avec mot de passe correct | Réussie |

**Résultat final :** le compte de Hugo Moreau est déverrouillé et son authentification fonctionne à nouveau.

## 8. Bonnes pratiques et recommandations

Dans un environnement professionnel, la résolution d'un verrouillage de compte nécessite plusieurs précautions :

- Vérifier l'identité de l'utilisateur avant toute intervention.
- Consulter les événements de sécurité afin de déterminer l'origine du verrouillage.
- Vérifier si les tentatives incorrectes peuvent provenir d'une application, d'un service ou d'identifiants enregistrés.
- Signaler toute activité suspecte à l'équipe de sécurité.
- Déverrouiller le compte uniquement lorsque l'intervention est autorisée.
- Vérifier le rétablissement de l'authentification.
- Documenter l'intervention dans l'outil de ticketing.

## 9. Compétences mobilisées

**Active Directory**

- Consultation des comptes utilisateurs.
- Vérification de l'état de verrouillage.
- Déverrouillage manuel d'un compte.
- Compréhension des politiques de sécurité du domaine.

**Windows Server**

- Consultation du journal Sécurité.
- Analyse de l'événement 4740.
- Identification de l'ordinateur à l'origine du verrouillage.

**PowerShell**

- `Get-ADUser`
- `Get-ADDefaultDomainPasswordPolicy`
- `Get-WinEvent`
- `Unlock-ADAccount`
- `net use`

**Support informatique N1**

- Reproduction d'un incident utilisateur.
- Diagnostic des problèmes d'authentification.
- Identification de la cause racine.
- Application d'une correction.
- Validation du retour à la normale.
- Documentation et clôture d'un ticket.

## 10. Conclusion

Cet exercice a permis de reproduire et résoudre un incident de verrouillage de compte Active Directory sur Windows Server 2022.

L'analyse de l'événement de sécurité 4740 a permis d'identifier le compte concerné ainsi que l'ordinateur à l'origine du verrouillage.

Le compte a ensuite été déverrouillé avec PowerShell, et une authentification SMB réussie a confirmé le rétablissement du service.

**Ticket INC-003 résolu et clôturé.**

---

*Incident simulé dans un environnement de laboratoire à des fins de formation et de démonstration des compétences en support informatique.*
