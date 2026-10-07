# Level09 — Explications

## Objectif
Décoder le fichier `token` (appartenant à `flag09`) pour obtenir le mot de
passe de `flag09`, lancer `getflag` et obtenir le token qui sert de mot de
passe à `level10`.

## La faille
```bash
ls -la
# -rwsr-sr-x 1 flag09 level09 7640 ... level09
# ----r--r-- 1 flag09 level09   26 ... token
```
`token` est lisible via la permission de **groupe** (`r--`, alors que le
propriétaire `flag09` n'a lui-même aucun droit), mais son contenu est
illisible : il est encodé.

Le binaire `level09` (SUID `flag09`) a des protections anti-debug
(`strings` révèle des vérifications `ptrace`, `LD_PRELOAD`,
`/proc/self/maps`) : pas question de l'ouvrir dans `gdb`. On teste donc en
boîte noire, avec différents arguments :
```bash
./level09 test   # tfuw   (t+0, e+1, s+2, t+3)
./level09 siuu   # sjwx   (s+0, i+1, u+2, u+3)
```
Le résultat dépend uniquement de l'argument fourni, et suit un pattern
clair : chaque caractère est décalé de sa **position** dans la chaîne
(`caractère + position`). `token` (même format/longueur attendue) a donc
vraisemblablement été produit avec ce même algorithme appliqué à un mot de
passe en clair : il suffit d'appliquer l'opération **inverse** (`caractère -
position`) à ses octets bruts pour le retrouver.

## Exploitation
`python` (2.x) est disponible sur la VM. Le home de `level09` étant en
lecture seule (`dr-x------`), le script est écrit dans `/tmp` puis exécuté
depuis `~` (le chemin `"token"` est relatif au dossier courant) :

```bash
cat > /tmp/reverse.py << 'EOF'
import sys

with open("token", "rb") as f:
    data = f.read()

for i in range(len(data)):
    byte = ord(data[i])
    original = (byte - i) % 256
    sys.stdout.write(chr(original))

print
EOF
cd ~
python /tmp/reverse.py
# f3iji1ju5yuevaus41q1afiuq
```

```bash
su flag09
Password: f3iji1ju5yuevaus41q1afiuq
getflag
# Check flag.Here is your token : s5cAJpM8ev6XHw998pRWG728z
```

Token obtenu : `s5cAJpM8ev6XHw998pRWG728z` — flag de level09 et mot de passe
de level10.

## Analyse de la faille

**Quelle était la faille ?**
Le secret (`token`) n'était protégé que par un **encodage maison trivial**
(décalage de chaque caractère par sa position), pas par un vrai chiffrement.
Une fois l'algorithme retrouvé — ce que les protections anti-debug ne
peuvent empêcher qu'en dynamique, pas en analyse statique du binaire — il
est immédiatement réversible, sans avoir besoin d'exécuter le programme
privilégié.

**Pourquoi c'est un problème ?**
- Les protections anti-debug (anti-ptrace, anti-LD_PRELOAD) ne sont qu'une
  gêne pour l'analyse **dynamique** ; elles n'empêchent en rien la lecture
  **statique** du code assembleur (`objdump`, désassembleurs, décompilateurs).
  S'appuyer dessus comme seule protection est une fausse sécurité.
- Un encodage réversible simple (type César/XOR fixe) n'est pas une
  protection cryptographique : dès que l'algorithme est connu, le secret est
  cassé instantanément, peu importe sa longueur.
- Les permissions du fichier (`r--` pour le groupe) permettaient de toute
  façon de lire le fichier encodé sans aucun privilège particulier.

**Comment l'empêcher ?**
- Utiliser un vrai algorithme de chiffrement (AES, etc.) avec une clé
  correctement protégée, jamais un encodage maison réversible par simple
  arithmétique.
- Ne jamais considérer les protections anti-debug comme une protection du
  secret lui-même : elles retardent l'analyse, elles ne la rendent pas
  impossible.
- Restreindre les permissions du fichier contenant un secret au strict
  nécessaire (aucun accès pour quiconque n'a pas besoin de le lire).
