# Level00 — Explications

## Objectif
Trouver le mot de passe de `flag00`, lancer `getflag` pour obtenir le token
qui sert de mot de passe à `level01`.

## Étapes

### 1. Chercher un fichier lié au niveau suivant
Les flags appartiennent à des comptes `flagXX`. On cherche les fichiers
possédés par `flag00` sur toute la machine :

```bash
find / -user flag00 2>/dev/null
```

Résultat utile : `/usr/sbin/john`
(`2>/dev/null` masque les erreurs "Permission denied", inévitables et sans intérêt.)

### 2. Analyser le fichier
```bash
ls -l /usr/sbin/john     # droits : ----r--r--  → "others" peut lire, donc level00 aussi
file /usr/sbin/john      # ASCII text → simple fichier texte
```

La faille : le fichier appartient à `flag00` mais reste **lisible par tout le monde**
(mauvaise configuration des permissions).

### 3. Lire le contenu
```bash
cat /usr/sbin/john
# cdiiddwpgswtgt
```

Cette chaîne n'est pas le mot de passe direct : que des lettres minuscules,
aucun chiffre ni symbole → typique d'un texte chiffré par décalage.

> Note : le fichier est nommé `john` en référence à **John the Ripper**, l'outil
> de cassage de mots de passe. C'est un indice pour la suite — il ne sert pas à
> résoudre level00 (résolu par un simple César), mais annonce level01, où un hash
> devra être cassé.

### 4. Déchiffrer (chiffre de César)
Le **chiffre de César** décale chaque lettre d'un nombre fixe de positions dans
l'alphabet. Ici le décalage est de **11**.

En testant les décalages possibles :

```bash
echo cdiiddwpgswtgt | tr 'a-z' 'l-za-k'
# nottoohardhere
```

Chaîne lisible → c'est le mot de passe de `flag00`.

### 5. Récupérer le token
```bash
su flag00          # mot de passe : nottoohardhere
getflag            # affiche le token
```

Token obtenu : `x24ti5gi3x0ol2eh4esiuxias`

C'est le **flag de level00** et le **mot de passe de level01**.

## Résumé
`find` → fichier mal protégé → César (décalage 11) → `su flag00` → `getflag`

## Analyse de la faille

**Quelle était la faille ?**
Un fichier contenant un secret (`/usr/sbin/john`) était lisible par tout le
monde (`----r--r--`, droit `r` pour "others"). Son contenu n'était "protégé"
que par un chiffrement César, trivial à casser.

**Pourquoi c'est un problème ?**
- Permissions trop permissives : un secret de `flag00` accessible à `level00`.
- Sécurité par l'obscurité : César n'est pas un chiffrement sérieux, il se
  casse en testant 25 décalages.

**Comment l'empêcher ?**
- Principe du **moindre privilège** : ne donner que les droits strictement
  nécessaires. Ici le fichier devrait être `0400` (lecture pour le propriétaire
  seul), voire inaccessible aux autres comptes.
- Ne jamais compter sur un encodage faible (César, base64…) pour protéger un
  secret : ce n'est pas du chiffrement.