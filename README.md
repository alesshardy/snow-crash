# snow-crash

Write-ups personnels du projet **Snow Crash** (42) : une introduction à la
cybersécurité à travers une série de challenges de type wargame, où chaque
niveau consiste à exploiter une faille (permissions, binaire SUID, injection
de commande, encodage faible, etc.) pour récupérer le mot de passe du compte
`flagXX` suivant.

> Sujet officiel du projet : [snowcrash.pdf](snowcrash.pdf)

## Principe du wargame

- On se connecte en SSH sur la VM fournie avec le compte `levelXX`.
- Le but est de trouver le mot de passe du compte `flagXX` correspondant
  (par exploitation d'une faille, pas par brute force).
- Une fois connecté à `flagXX`, on lance `getflag`, qui renvoie un **token**.
- Ce token est à la fois le flag du niveau et le mot de passe du niveau
  suivant (`levelXX+1`).

## Structure du repo

```
levelXX/
├── flag                    # token récupéré via `getflag` pour ce niveau
└── resources/
    └── explications.md     # démarche détaillée : analyse, faille, exploitation, remédiation
```

Chaque `explications.md` suit le même plan :

1. **Objectif** du niveau
2. **Étapes** suivies pour trouver la faille et récupérer le flag
3. **Résumé** de la méthode
4. **Analyse de la faille** : en quoi consiste la vulnérabilité, pourquoi
   c'est un problème, et comment s'en protéger

## Niveaux couverts

Partie obligatoire du projet, `level00` à `level09` :

| Niveau | Faille exploitée |
|---|---|
| level00 | Fichier secret lisible par tous, chiffré par un César trivial |
| level01 | Hash de mot de passe à casser |
| level02 | — voir [explications](level02/resources/explications.md) |
| level03 | Binaire SUID exploitable |
| level04 | Exécution de commande sous l'identité de `flag04` |
| level05 | Exécution de commande sous l'identité de `flag05` |
| level06 | Script PHP lancé par un binaire SUID |
| level07 | Binaire SUID construisant une commande à partir d'une variable d'environnement |
| level08 | Contournement d'un contrôle de nom de fichier par un binaire SUID |
| level09 | Décodage d'un fichier `token` |

Le détail de chaque exploitation se trouve dans le `resources/explications.md`
du niveau correspondant.

## Avertissement

Ces write-ups documentent la résolution de challenges pédagogiques réalisés
dans le cadre du cursus 42, sur une machine virtuelle dédiée fournie pour ce
projet. Ils n'ont pas vocation à être utilisés hors de ce contexte.
