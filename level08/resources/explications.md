# Level08 — Explications

## Objectif
Un binaire SUID `flag08` sait lire un fichier `token` que `level08` ne peut pas
lire, mais refuse qu'on lui demande ce fichier par son nom. On contourne ce
filtre pour lire le token (= mot de passe de `flag08`).

## Étapes

### 1. Repérer les fichiers
```bash
ls -la
# -rwsr-s---+ 1 flag08 level08  level08    (binaire SUID flag08)
# -rw-------  1 flag08 flag08   token      (lisible par flag08 SEUL)
```

- `level08` : binaire SUID → s'exécute avec les droits de **flag08**
- `token` (26 octets) : appartient à flag08 en `rw-------` → **level08 ne peut
  pas le lire** directement

### 2. Analyser le binaire
```bash
strings level08
```

Chaînes importantes :
```
%s [file to read]          → prend un fichier en argument et le lit
token                      → le mot "token" est connu du programme
You may not access '%s'    → il refuse certains fichiers
strstr                     → recherche d'une sous-chaîne (le filtre)
open / read / write        → ouvre, lit, affiche le fichier
```

**Comportement reconstitué :** le programme lit et affiche le fichier passé en
argument, **sauf** si le nom contient la chaîne `token` (détectée par `strstr`),
auquel cas il refuse.

Test :
```bash
./level08 token          # You may not access 'token'
./level08 /etc/passwd    # affiche le fichier → le lecteur fonctionne
```

### 3. Identifier la faille
Le filtre ne regarde que le **nom** fourni, pas le fichier réellement visé. Or
sous Linux un même fichier peut être désigné par plusieurs noms. Si on crée un
**lien symbolique** vers `token` avec un nom qui ne contient pas « token », le
filtre `strstr` ne détecte rien, mais le binaire (en tant que flag08) ouvre bien
le vrai fichier au bout du lien.

### 4. Exploiter avec un lien symbolique
```bash
ln -s ~/token /tmp/lien
./level08 /tmp/lien
# quif5eloekouj29ke0vouxean
```

- `ln -s` : crée un **lien symbolique** (un fichier qui contient juste le chemin
  vers la cible)
- `~/token` : la cible
- `/tmp/lien` : le nouveau nom, sans « token » → passe le filtre

**Pourquoi un lien symbolique et pas un lien physique (`ln` sans `-s`) ?**
Un lien physique vers `token` exigerait des droits sur le fichier cible, que
level08 n'a pas (`rw-------` à flag08) → sa création échouerait. Le lien
symbolique ne stocke qu'un chemin (une chaîne de texte), donc il ne demande
aucun droit sur la cible. C'est ensuite le binaire privilégié qui suit le lien
et ouvre le vrai fichier avec les droits de flag08.

Le résultat `quif5eloekouj29ke0vouxean` est le **contenu du fichier token**,
c'est-à-dire le **mot de passe de flag08**.

### 5. Récupérer le token
```bash
su flag08        # mot de passe : quif5eloekouj29ke0vouxean
getflag
```

Token obtenu : `25749xKZ8L7DkSCwJkT9dyv6f`

C'est le **flag de level08** et le **mot de passe de level09**.

## Résumé
binaire SUID flag08 lit un fichier mais filtre le nom "token" (`strstr`)
→ lien symbolique `/tmp/lien` → `token` → `./level08 /tmp/lien` → mot de passe
→ `su flag08` → `getflag`

## Rappel : les types de liens
- **Lien physique** (`ln`) : 2ᵉ nom pour le **même contenu** sur le disque (pas
  une copie). Nécessite des droits sur la cible.
- **Lien symbolique** (`ln -s`) : petit fichier contenant **le chemin** vers la
  cible. Aucun droit requis sur la cible ; cassé si la cible disparaît.

## Analyse de la faille

**Quelle était la faille ?**
Le programme filtrait l'accès en se basant sur le **nom** du fichier demandé
(`strstr(..., "token")`), pas sur le fichier réellement ouvert. Un lien
symbolique permet de viser le fichier interdit sous un autre nom.

**Pourquoi c'est un problème ?**
- Contrôler l'accès par le nom fourni est trompeur : un même fichier a plusieurs
  noms possibles (liens), et le nom ne dit rien du fichier réellement ouvert.
- Le binaire étant SUID flag08, il lit un fichier protégé pour le compte d'un
  utilisateur qui n'y a aucun droit.

**Comment l'empêcher ?**
- Ne pas filtrer sur le nom : vérifier le fichier réellement ciblé (résoudre le
  chemin réel, comparer l'inode, refuser de suivre les liens symboliques avec
  `O_NOFOLLOW`).
- Vérifier que l'utilisateur appelant a lui-même le droit d'accéder au fichier
  avant de le lire pour lui.
- Appliquer le moindre privilège (éviter le SUID si ce n'est pas indispensable).