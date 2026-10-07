# Level02 — Explications

## Objectif
Trouver le mot de passe de `flag02`, lancer `getflag` pour obtenir le token
qui sert de mot de passe à `level03`.

## Étapes

### 1. Repérer le fichier du niveau
```bash
ls -la
# level02.pcap
```

L'extension `.pcap` = *packet capture* : un enregistrement de paquets réseau.
`cat` affiche du binaire illisible — il faut un outil qui comprend le format.

### 2. Comprendre ce que contient la capture
Un `.pcap` est une capture de trafic réseau. On la lit avec un outil d'analyse
réseau :
- **tcpdump** : outil en ligne de commande qui capture et affiche les paquets.
- **Wireshark** : équivalent graphique, plus confortable (vue des conversations,
  "Follow TCP Stream"). Sa version terminal s'appelle `tshark`.

Ici on ne capture rien (la capture existe déjà), on s'en sert seulement pour
**relire** le fichier. En l'inspectant, on voit des fragments lisibles :
`wwwbugs login:`, `Password:`, `Login incorrect`. C'est une session de
connexion **telnet** enregistrée. Telnet transmet tout **en clair**, donc la
frappe du mot de passe est entièrement présente dans la capture.

### 3. Lire la capture avec tcpdump
Wireshark n'étant pas disponible, on utilise `tcpdump` (présent sur la VM).
On affiche le contenu en hexadécimal, en ne gardant que les paquets envoyés
**vers** le serveur (ce que l'utilisateur tape) et non vides :

```bash
tcpdump -r level02.pcap -nn -x 'tcp dst port 12121 and greater 54' 2>/dev/null
```

- `-r level02.pcap` : lire le fichier (au lieu de capturer en direct)
- `-nn` : ne pas résoudre les noms / ports
- `-x` : afficher chaque paquet en hexadécimal
- `tcp dst port 12121` : seulement les paquets vers le serveur (la frappe)
- `greater 54` : ignorer les paquets vides (accusés de réception)

**Pourquoi l'hexadécimal (`-x`) ?** C'est la lecture sans ambiguïté. En mode
texte (`-A`), les octets non imprimables comme le backspace (`0x7f`) ne
s'affichent pas ou deviennent des symboles illisibles, et le caractère se
mélange aux octets d'en-tête. En hexa, chaque octet est un code net : on lit le
dernier octet, on le convertit via ASCII, et on voit même les backspaces que le
mode texte masquerait — indispensable pour reconstituer le mot de passe exact.

### 4. Lire les caractères tapés
En telnet, **chaque touche = un paquet de `length 1`**. Le caractère tapé est
le **dernier octet hexadécimal** du paquet. On le convertit en lettre via la
table **ASCII** (ex : `66`→f, `74`→t, `65`→e…).

Ce qu'il faut regarder :
- **Ignorer** les paquets `length 0` (accusés de réception, vides).
- **Ignorer** les gros paquets `length > 1` du début : c'est la négociation
  telnet (`xterm`, `DISPLAY`, taille d'écran…), pas la frappe.
- Repérer le paquet `Password:` (envoyé par le serveur) : tout ce qui est tapé
  **avant** = login (inutile), tout ce qui est tapé **après** = le mot de passe.

**Pourquoi le mot de passe ne commence qu'à ~07:23:34 ?** L'heure n'a rien
d'arbitraire : c'est le **serveur** qui envoie `Password:` à un instant précis.
Avant ce paquet, l'utilisateur tapait son **login** (vers 07:23:24-26) ; une
fois le login validé (Entrée), le serveur affiche `Password:`, et seulement à
partir de là (dès 07:23:34) les frappes suivantes sont celles du mot de passe.
C'est donc la position du paquet `Password:` qui sépare login et mot de passe,
pas une heure fixe.

Conversion d'un octet si besoin :
```bash
echo -n "66" | xxd -r -p    # affiche : f
```

### 5. Reconstituer le mot de passe (attention aux backspaces)
Les octets tapés après `Password:` :

```
66 74 5f 77 61 6e 64 72   → f t _ w a n d r
7f 7f 7f                  → 3x BACKSPACE (efface r, d, n)
4e 44 52 65 6c            → N D R e l
7f                        → 1x BACKSPACE (efface l)
4c 30 4c                  → L 0 L
0d                        → Entrée (fin)
```

**`0x7f` = touche Backspace** : elle efface le caractère précédent. Il faut
donc simuler la frappe réelle, pas juste coller les lettres :

1. `ft_wandr`
2. −3 → `ft_wa`
3. +`NDRel` → `ft_waNDRel`
4. −1 → `ft_waNDRe`
5. +`L0L` → `ft_waNDReL0L`

Mot de passe trouvé : **`ft_waNDReL0L`**

### 6. Récupérer le token
```bash
su flag02        # mot de passe : ft_waNDReL0L
getflag
```

Token obtenu : `kooda2puivaav1idi4f57q8iq`

C'est le **flag de level02** et le **mot de passe de level03**.

## Résumé
`.pcap` → session telnet en clair → `tcpdump -x` → lire les octets `length 1`
→ convertir en ASCII + appliquer les backspaces → `ft_waNDReL0L` → `su flag02`

## Ce qu'il faut retenir
- Un `.pcap` est un enregistrement du trafic réseau, relisible a posteriori.
- `length 1` = une frappe clavier ; `length 0` = paquet vide ; `length > 1` au
  début = négociation du protocole.
- Le caractère tapé est le dernier octet du paquet, à convertir via ASCII.
- `0x7f` = Backspace : toujours appliquer les corrections.

## Analyse de la faille

**Quelle était la faille ?**
La connexion utilisait **telnet**, un protocole qui transmet **tout en clair**
(identifiants et mot de passe compris). Quiconque capture le trafic peut relire
la frappe du mot de passe. De plus, cette capture (`.pcap`) avait été laissée
accessible dans le dossier du niveau.

**Pourquoi c'est un problème ?**
- Aucune confidentialité : le mot de passe circule en clair sur le réseau et se
  lit directement dans une capture (rien à "déchiffrer").
- N'importe qui sur le chemin réseau (même une machine qui écoute) peut voler
  les identifiants.

**Comment l'empêcher ?**
- Ne jamais utiliser telnet : le remplacer par **SSH**, qui chiffre toute la
  connexion. Même capturé, le trafic SSH est illisible sans la clé.
- Plus généralement, chiffrer toute communication sensible (TLS/HTTPS pour le
  web, etc.).
- Ne pas laisser traîner des captures réseau accessibles aux autres comptes.