# AMPTemplates-KAG — King Arthur's Gold pour AMP

Template **AMP Generic Module** pour héberger un serveur dédié
**King Arthur's Gold** (Transhuman Design) sur Linux.

CubeCoders n'en propose aucun : celui-ci est écrit de zéro, sur le modèle du
template Teeworlds officiel (même cas de figure — une archive à télécharger et
un fichier `.cfg`, pas de SteamCMD).

## Pourquoi ce jeu vaut le coup

Tout a été vérifié, pas supposé :

| Point | Valeur | Comment c'est vérifié |
|---|---|---|
| Prix | **gratuit** | API Steam de l'app 219830 : `is_free: true` |
| Poids du serveur | **37 Mo** | `Content-Length` de l'archive officielle |
| Ports à ouvrir | **un seul**, 50301/UDP | jeu, téléchargement des cartes et annonce au serveur maître passent tous par là |
| Modes livrés | CTF, SmallCTF, TDM, WAR, Sandbox, Challenge | dossiers réellement présents sous `Base/Rules/` dans l'archive |

Gratuit et léger, c'est l'inverse exact d'un ARK : n'importe qui peut rejoindre
sans rien acheter, et la machine ne le sent pas.

## Le seul vrai piège : le binaire est en 32 bits

Il n'existe **pas** de build 64 bits. Vérifié directement :

```
kag-linux32-dedicated-release.tar.gz → 200, 37 Mo
kag-linux64-dedicated-release.tar.gz → 404
```

Mais les images Docker que l'on trouve installent une trentaine de paquets
i386 (mesa, X11, GL…) — ce sont les dépendances du **client**, pas du serveur.
En lisant les `DT_NEEDED` de l'ELF de `KAGdedi`, la liste réelle est de **cinq
bibliothèques**, dont aucune graphique :

```
libc.so.6   libdl.so.2   libgcc_s.so.1   libm.so.6   libstdc++.so.6
```

Soit trois paquets Debian : `libc6:i386`, `libstdc++6:i386`, `libgcc-s1:i386`
(`libdl` et `libm` font partie de `libc6`). Ils sont déclarés dans
`Meta.ExtraContainerPackages`, donc installés automatiquement en conteneur.

**Sur l'hôte, sans conteneur**, il faut les avoir une fois pour toutes :

```bash
sudo dpkg --add-architecture i386 && sudo apt update && sudo apt install -y libc6:i386 libstdc++6:i386 libgcc-s1:i386
```

À vérifier avant de chercher plus loin si l'instance refuse de démarrer :

```bash
ls -la /lib/ld-linux.so.2
```

Pas de loader 32 bits, pas de serveur — et le message d'erreur d'AMP ne le dira
pas clairement.

## Installation

### 1. Ajouter le dépôt dans ADS

Configuration → Instance Deployment → Configuration Repositories → ajouter :

```
hydrocut/AMPTemplates-KAG:main
```

puis **Fetch Latest**.

### 2. Créer l'instance

Create Instance → **King Arthur's Gold (TeamKit)**. Puis **Update** : AMP
télécharge l'archive, pose le droit d'exécution sur `KAGdedi`, et écrit un
`autoconfig.cfg` de départ (sans l'écraser s'il existe déjà).

### 3. Régler et démarrer

Onglet Configuration, quatre choses avant le premier démarrage :

1. **Mot de passe RCON** — sans lui, personne ne peut administrer depuis le jeu
2. **Nom du serveur** et **description** (bon endroit pour le lien du Discord)
3. **Mode de jeu** — `CTF` pour une communauté, `SmallCTF` si vous êtes peu
4. **Joueurs maximum** — 32 est confortable ; au-delà, les cartes CTF serrent

Puis **Start**. La console d'AMP lit et écrit directement dans le serveur
(`App.AdminMethod=STDIO`) : pas besoin d'activer la console TCP.

## Deux choses à savoir, qui ne sont écrites nulle part ailleurs

### Les KAG Guards peuvent modérer ton serveur

Extrait des conditions de THD livrées dans l'archive (`terms.txt`) :
`sv_allow_globals_mods` est **actif par défaut**, ce qui autorise les
modérateurs bénévoles de THD à kicker et bannir sur ton serveur **sans te
demander**. En échange, ils le défendent quand ton staff n'est pas là.

Le template **garde le réglage du jeu** (actif) plutôt que de le changer dans
ton dos, mais le réglage est exposé dans AMP avec cette explication en clair,
sous *Modération*. C'est ta décision, pas la mienne.

### KAG réécrit son `autoconfig.cfg` en s'arrêtant

Comme Barotrauma avec son `serversettings.xml` : AMP écrit le fichier au
démarrage, le jeu le réécrit à l'arrêt — en y remettant ses commentaires en fin
de ligne. La regex du `metaconfig.json` est donc écrite pour les tolérer :

```
^(?<key>[A-Za-z0-9_]+)\s*=\s*(?<value>[^#]*?)\s*(?:#.*)?$
```

Conséquence à connaître : une valeur qui contiendrait un `#` serait tronquée
au `#`. C'est le prix à payer pour relire le fichier que le jeu réécrit, et le
compromis est le bon — sauf si tu veux un `#` dans le nom du serveur.

## Vérifier le template avant de le pousser

```bash
python verifier.py
```

Il teste ce qui casse vraiment en silence : JSON valide, `@IncludeJson` qui
pointent sur des fichiers existants, `PrimaryApplicationPortRef` qui
correspond à un port déclaré, **chaque réglage exposé dans AMP présent dans le
fichier de config livré**, et l'aller-retour de la regex y compris avec un
commentaire en fin de ligne. Il doit afficher `Tout est bon.`

## Licence et redistribution

Les conditions de Transhuman Design interdisent de redistribuer le jeu sous
quelque forme que ce soit, **binaires du serveur compris**. Ce dépôt ne
contient donc que le template : les fichiers du jeu sont téléchargés depuis
`dl.kag2d.com` à l'Update, et le `.gitignore` est écrit pour qu'aucun fichier
du jeu ne puisse partir par accident dans un commit.

King Arthur's Gold est une marque de Transhuman Design. Ce template n'est ni
affilié ni approuvé par eux.
