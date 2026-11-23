# INC-002 — Réinitialisation d'un mot de passe Active Directory

**Statut : Résolu — Clôturé**

**Catégorie : Support informatique / Active Directory / Gestion des identités**

**Priorité : P3 — Moyenne (simulation)**

**Environnement : Windows Server 2022 / Active Directory Domain Services**

---

## 1. Description de l'incident

**Utilisatrice concernée :** Emma Petit

**Service :** Ressources humaines

**Compte Active Directory :** `LAB\e.petit`

**Domaine :** `LAB.local`

**Contrôleur de domaine :** `DC01`

### Problème signalé

L'utilisatrice contacte le support informatique car elle a oublié son mot de passe Windows et ne peut plus accéder à sa session.

Une réinitialisation administrative du mot de passe est nécessaire pour lui permettre de retrouver l'accès à son compte.

**Impact :** impossibilité pour l'utilisatrice de s'authentifier avec son mot de passe oublié.

**Objectif de l'intervention :** réinitialiser le mot de passe, imposer son changement à la prochaine connexion et vérifier le rétablissement de l'authentification.

---

## 2. Environnement technique

| Composant | Configuration |
| --- | --- |
| Système d'exploitation | Windows Server 2022 |
| Contrôleur de domaine | DC01 |
| Domaine | LAB.local |
| Adresse IP du serveur | 192.168.31.10 |
| Compte utilisateur | e.petit |
| Service | Ressources humaines |
| Outil d'administration | PowerShell / Module ActiveDirectory |
| Protocole de test | SMB |
| Partage utilisé | `\\DC01\IPC$` |

L'intervention a été réalisée dans un environnement Active Directory virtualisé sous VMware Workstation.

---

## 3. Diagnostic initial

Avant toute modification, une vérification du compte utilisateur a été réalisée.

### Commande PowerShell

```
Get-ADUser -Identity "e.petit" -Properties Enabled,LockedOut,PasswordLastSet |
Select-Object Name,SamAccountName,Enabled,LockedOut,PasswordLastSet
```

### Objectifs

- Confirmer l'existence du compte dans Active Directory.
- Vérifier que le compte est activé.
- Contrôler son état de verrouillage.
- Consulter la date de dernière modification du mot de passe.

Cette étape permet de distinguer un problème de mot de passe d'un compte désactivé ou verrouillé.

Dans le scénario, l'identité de l'utilisatrice est considérée comme vérifiée conformément à la procédure interne de support.

---

## 4. Réinitialisation administrative du mot de passe

Un mot de passe temporaire a été défini pour permettre à Emma de retrouver l'accès à son compte.

### Saisie sécurisée du mot de passe temporaire

```
$NewPassword = Read-Host "Saisir le mot de passe temporaire" -AsSecureString
```

Le paramètre `-AsSecureString` évite l'affichage du mot de passe en clair lors de sa saisie.

### Réinitialisation du mot de passe

```
Set-ADAccountPassword -Identity "e.petit" -Reset -NewPassword $NewPassword
```

Le paramètre `-Reset` permet d'effectuer une réinitialisation administrative sans connaître l'ancien mot de passe.

### Obligation de changement à la prochaine connexion

```
Set-ADUser -Identity "e.petit" -ChangePasswordAtLogon $true
```

Cette configuration oblige l'utilisatrice à remplacer le mot de passe temporaire.

### Vérification

```
Get-ADUser -Identity "e.petit" -Properties pwdLastSet |
Select-Object Name,pwdLastSet
```

La valeur `pwdLastSet = 0` indique qu'un changement de mot de passe est exigé à la prochaine connexion.

---

## 5. Premier test d'authentification — Erreur 1907

Une tentative d'authentification SMB a été réalisée avec le mot de passe temporaire.

### Commande

```
net use \\DC01\IPC$ /user:LAB\e.petit *
```

### Résultat observé

```
System error 1907 has occurred.

The user's password must be changed before signing in.
```

### Analyse

L'erreur 1907 indique que Windows impose un changement de mot de passe avant d'autoriser cette connexion.

Ce comportement est cohérent avec l'option `ChangePasswordAtLogon` activée précédemment.

Une connexion SMB avec `net use` ne permet pas d'effectuer directement le changement interactif exigé.

**Conclusion :** ce message constitue un comportement attendu dans le cadre de la procédure de réinitialisation. Il ne démontre pas, à lui seul, que le mot de passe temporaire saisi est correct.

---

## 6. Changement du mot de passe temporaire

Afin de simuler le changement effectué par l'utilisatrice, une opération distincte de la réinitialisation administrative a été réalisée.

### Saisie de l'ancien et du nouveau mot de passe

```
$OldPassword = Read-Host "Mot de passe temporaire d'Emma" -AsSecureString

$NewPassword = Read-Host "Nouveau mot de passe d'Emma" -AsSecureString
```

### Commande de changement

```
Set-ADAccountPassword -Identity "e.petit" `
    -OldPassword $OldPassword `
    -NewPassword $NewPassword
```

Contrairement à la réinitialisation administrative, cette opération nécessite de fournir le mot de passe actuel.

Dans ce laboratoire, elle a été exécutée depuis une session administrateur pour simuler l'opération. En environnement professionnel, l'utilisatrice devrait normalement effectuer elle-même le changement de son mot de passe, sans communiquer son nouveau secret au support.

---

## 7. Incident rencontré — Refus du nouveau mot de passe

Lors de la première tentative de changement, Active Directory a refusé le nouveau mot de passe.

### Message d'erreur

```
The password does not meet the length, complexity,
or history requirement of the domain.
```

### Exception PowerShell

```
ADPasswordComplexityException
```

### Analyse

Cette erreur indique que le mot de passe proposé ne satisfait pas au moins une exigence de la politique de sécurité Active Directory.

Les causes possibles comprennent :

- Une longueur insuffisante.
- Une complexité insuffisante.
- La réutilisation d'un ancien mot de passe.
- La présence d'éléments du nom ou de l'identifiant utilisateur interdits par la politique.

Le message ne permet pas d'identifier à lui seul la règle exacte responsable du refus.

### Vérification de la politique de domaine

La commande suivante permet de consulter les principales exigences :

```
Get-ADDefaultDomainPasswordPolicy |
Select-Object MinPasswordLength, ComplexityEnabled, PasswordHistoryCount, MinPasswordAge
```

### Résolution

Un nouveau mot de passe compatible avec les exigences du domaine a été choisi.

La commande `Set-ADAccountPassword` a ensuite été exécutée avec succès.

**Aucun assouplissement de la politique de sécurité du domaine n'a été nécessaire.**

---

## 8. Validation du changement de mot de passe

### Vérification de la date de modification

```
Get-ADUser -Identity "e.petit" -Properties PasswordLastSet |
Select-Object Name,PasswordLastSet
```

La vérification de cette propriété permet de confirmer qu'un mot de passe a été défini récemment.

### Test d'authentification SMB

Une nouvelle connexion SMB a été effectuée avec le mot de passe définitif d'Emma.

```
net use \\DC01\IPC$ /user:LAB\e.petit *
```

### Résultat

```
The command completed successfully.
```

L'authentification SMB avec le nouveau mot de passe a été validée.

### Nettoyage de la connexion de test

```
net use \\DC01\IPC$ /delete
```

Cette opération permet de supprimer la connexion réseau utilisée pour la validation.

---

## 9. Synthèse des vérifications

| Vérification | Résultat |
| --- | --- |
| Compte Active Directory identifié | Réussi |
| Réinitialisation administrative | Réussie |
| Changement obligatoire configuré | Réussi |
| Erreur SMB 1907 analysée | Réussi |
| Refus initial du nouveau mot de passe | Identifié |
| Nouveau mot de passe accepté | Réussi |
| Authentification SMB finale | Réussie |
| Incident | Résolu |

**Résultat final :** le mot de passe du compte `LAB\e.petit` a été réinitialisé, remplacé par un nouveau mot de passe conforme et validé par une authentification SMB réussie.

---

## 10. Bonnes pratiques professionnelles

Cette intervention met en évidence plusieurs bonnes pratiques de support informatique :

1. Vérifier l'identité de l'utilisateur avant toute réinitialisation.
2. Contrôler l'état du compte Active Directory.
3. Utiliser un mot de passe temporaire conforme à la politique de sécurité.
4. Imposer son changement à la prochaine connexion.
5. Ne jamais demander à l'utilisateur de communiquer son nouveau mot de passe définitif.
6. Ne pas désactiver les règles de complexité pour contourner une erreur.
7. Tester l'authentification après la résolution.
8. Documenter les actions réalisées dans le ticket de support.

---

## 11. Compétences mobilisées

### Active Directory

- Administration des comptes utilisateurs.
- Réinitialisation de mots de passe.
- Gestion de l'obligation de changement à la prochaine connexion.
- Compréhension des politiques de mots de passe du domaine.

### PowerShell

- `Get-ADUser`
- `Set-ADAccountPassword`
- `Set-ADUser`
- `Get-ADDefaultDomainPasswordPolicy`
- `Read-Host -AsSecureString`

### Windows et réseau

- Authentification SMB.
- Utilisation de `net use`.
- Compréhension de l'erreur Windows 1907.
- Gestion des connexions réseau de test.

### Support informatique N1

- Qualification d'une demande utilisateur.
- Vérification d'un compte Active Directory.
- Application d'une procédure de réinitialisation.
- Diagnostic d'erreurs d'authentification.
- Respect des politiques de sécurité.
- Validation du rétablissement du service.
- Documentation d'une intervention.

---

## 12. Conclusion

Cet exercice a permis de simuler une demande courante de support informatique : la réinitialisation d'un mot de passe Active Directory oublié.

L'intervention a couvert l'ensemble du processus, depuis la vérification du compte jusqu'à la validation de la nouvelle authentification.

Deux comportements ont été particulièrement étudiés : l'erreur 1907 liée à l'obligation de changement du mot de passe et le refus d'un mot de passe non conforme à la politique du domaine.

Le changement définitif du mot de passe et le test SMB réussi ont confirmé la résolution de l'incident.

**Ticket INC-002 résolu et clôturé.**

---

*Incident fictif réalisé dans un environnement de laboratoire Windows Server 2022 à des fins de formation et de démonstration des compétences en support informatique.*
