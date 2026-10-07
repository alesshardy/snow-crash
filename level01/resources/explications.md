# Level01 — Explications

## Objectif
Trouver le mot de passe de `flag01`, lancer `getflag` pour obtenir le token
qui sert de mot de passe à `level02`.

## Étapes

### 1. Chercher la faille
Les `find` habituels (user/group/perm pour `flag01` et `level02`) ne donnent
rien → la faille n'est pas un fichier qui traîne. On va voir là où le système
range les comptes et mots de passe : `/etc/passwd`.

```bash
cat /etc/passwd
```

### 2. Repérer l'anomalie
Toutes les lignes ont un `x` dans le champ mot de passe (le vrai hash est dans
`/etc/shadow`, lisible par root seul)… sauf `flag01` :

```
flag01:42hDRfypTqqnw:3001:3001::/home/flag/flag01:/bin/bash
       └─────┬──────┘
         le hash, laissé en clair dans passwd
```

`42hDRfypTqqnw` est le **hash** du mot de passe de flag01 (pas le mot de passe
lui-même). Il aurait dû être dans `/etc/shadow`.

### 3. Casser le hash avec John the Ripper
Un hash est à sens unique : on ne le "déchiffre" pas, on le **casse** (tester
des candidats jusqu'à retrouver le bon). On met le hash dans un fichier au
format `nom:hash`, puis on lance John.

```bash
echo 'flag01:42hDRfypTqqnw' > hash.txt
john --show hash.txt
# flag01:abcdefg
```

> John n'étant pas exécutable sur la VM, le hash a été cassé depuis la machine
> perso (méthode "external software" prévue par le sujet). Pour transférer le
> fichier de la VM vers son PC, on utilise `scp` (à lancer depuis sa propre
> machine, pas depuis la VM) :
>
> ```bash
> scp -P 4242 level01@localhost:/etc/passwd ~/Desktop/
> ```
>
> - `-P 4242` : port SSH de la VM (majuscule pour scp)
> - `level01@localhost:/etc/passwd` : fichier source sur la VM
> - `~/Desktop/` : destination sur le PC
>
> Ici le hash a simplement été recopié à la main depuis `/etc/passwd`, mais
> `scp` est la méthode propre et indispensable dès qu'il s'agit d'un binaire.

Mot de passe trouvé : **`abcdefg`**

### 4. Récupérer le token
```bash
su flag01        # mot de passe : abcdefg
getflag
```

Token obtenu : `f2av5il02puano7naaf6adaaf`

C'est le **flag de level01** et le **mot de passe de level02**.

## Résumé
`/etc/passwd` → hash en clair → John the Ripper → `abcdefg` → `su flag01` → `getflag`

## Analyse de la faille

**Quelle était la faille ?**
Le hash du mot de passe de `flag01` était stocké en clair dans `/etc/passwd`
(lisible par tous) au lieu de `/etc/shadow` (root seul). De plus, le mot de
passe choisi (`abcdefg`) était trivial.

**Pourquoi c'est un problème ?**
- `/etc/passwd` est lisible par tout le monde : exposer un hash dedans permet à
  n'importe quel utilisateur de le récupérer et de le casser hors-ligne.
- Un mot de passe faible se casse en une fraction de seconde.

**Comment l'empêcher ?**
- Stocker les hash uniquement dans `/etc/shadow` (c'est le rôle du système
  shadow, et la raison du `x` dans passwd).
- Imposer des mots de passe forts (longs, aléatoires) pour résister au cassage.
- Utiliser des fonctions de hachage lentes et salées (bcrypt, etc.) plutôt que
  le vieux DES/crypt.