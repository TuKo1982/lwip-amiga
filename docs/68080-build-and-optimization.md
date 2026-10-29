# lwip-amiga sur 68080 — compilation et optimisation

Ce document rassemble ce que le dépôt `lwip-amiga` établit sur la compilation de
`bsdsocket.library` et sur les optimisations qui comptent sur un 68080 (Apollo/Vampire).
Les sources sont citées à chaque fois pour que chaque point puisse être vérifié.

> **En bref.** Rien dans le build ne cible le 68080 en particulier : tout est compilé en
> `-m68040` et tourne tel quel sur un 68080. La seule mesure faite sur 68080 concerne le
> **placement du code par LTO** : un chemin de contrôle rarement utilisé, s'il n'est pas
> marqué `cold`, fait perdre **6 à 12 %** de débit en envoi TCP.

---

## 1. Chaîne de compilation

| Élément | Valeur | Où |
|---|---|---|
| Compilateur | GCC m68k « Bebbo » (cross), fourni par le superprojet **emu68-driver-stack** | `CLAUDE.md`, `README.md` |
| Version GCC | Le code compile proprement en **GCC 16.1** | `RELEASE-NOTES.md` (1.2) |
| CPU cible | `-m68040`, réglé pour toute la stack par `cmake/toolchain.cmake` du superprojet | `src/CMakeLists.txt` l.30 |
| Pointeur de cadre | `-fomit-frame-pointer`, réglé pour toute la stack (même endroit) | `src/CMakeLists.txt` l.30 |
| Bibliothèque commune | `emu68-common` (`memcpy`, formateur, `Kprintf`, macros de layout/LTO) | `src/CMakeLists.txt` |
| Avertissements | `-Wall -Wextra` partout ; la library ajoute `-Wshadow -Wmissing-prototypes -Wstrict-prototypes` | `src/CMakeLists.txt` |

Il n'y a **aucun** drapeau `-m68080`, aucun code AMMX et aucune variante de build propre
au 68080. Le jeu d'instructions 68040 est un sous-ensemble de ce que le 68080 exécute, donc
le binaire est le même pour Emu68/PiStorm et pour un Vampire.

### Lancer le build

Depuis une copie du superprojet, toujours via le conteneur (jamais `cmake` sur l'hôte :
les arbres de build sont configurés sous `/work` dans le conteneur et un `cmake --build`
lancé sur l'hôte échoue sur le chemin du `CMakeCache`) :

```sh
./scripts/docker-build.sh --target lwip-amiga
```

- La library sort dans `install/LIBS/bsdsocket.library`, les outils dans `install/C/`.
- Backend de debug : `EMU68_CONFIGURE_ARGS="-DEMU68_DEBUG_BACKEND=serial"`
  (`pistorm` par défaut | `serial` | `off`).
- Pour compiler une copie locale de ce composant dans le superprojet :
  `-D LWIP_AMIGA_SOURCE_DIR=/chemin/vers/lwip-amiga`.

---

## 2. Niveaux d'optimisation par cible

| Cible | Optimisation | Pourquoi |
|---|---|---|
| `netstack` (cœur lwIP + couche port Amiga, lib statique) | **`-O3`** + LTO | chemin chaud RX/TX |
| `bsdsocket_library` | **`-O3`** + LTO, édition de liens `-nostdlib -nostartfiles -s` | chemin chaud des appels socket |
| `sockbench` (outil de banc, non livré) | **`-O3`** | ne doit pas être le goulet de la mesure |
| Outils CLI/GUI (`netinfo`, `mdns`, `ping`, `traceroute`, `arp`, `AddNetInterface`, `RemoveNetInterface`, `NetShutdown`, `GetNetStatus`, `ShowNetStatus`, `NetLogViewer`, `NetSpeed`, `netdev-stats`) | **`-Os`** | taille, pas de chemin chaud |

Historique : la version 1.2 avait fait passer les outils de `-O2` à `-Os` et `sockbench`
à `-O3` (commit `ed7422d`). Le cœur et la library sont en `-O3`.

### Effet secondaire connu de `-O3`

À `-O3`, GCC déroule les boucles d'options de `dhcp.c` et émet un faux
`-Wstringop-overflow` (il ne peut pas borner `options_out_len`, rendu par pointeur).
Le seul débordement réel a été corrigé en dimensionnant `DHCP_OPTIONS_LEN` dans
`lwipopts.h` ; l'avertissement est coupé **pour ce seul fichier**
(`set_source_files_properties(... dhcp.c ... -Wno-stringop-overflow)`).

---

## 3. LTO et placement du code — la mesure 68080

`netstack` et `bsdsocket_library` sont compilés avec LTO (`emu68_enable_lto(...)`).
LTO découpe et réordonne le code de toute la library ; ajouter une fonction qui n'a rien à
voir avec le chemin de données peut donc **déplacer le code chaud TX/RX**.

Mesure faite en ajoutant les ioctls `SIOCGIF*` (branche `sioc-gif-ioctls`, release 1.7) :

> Sans attribut, le build LTO a redécoupé le code chaud TX/RX et le débit d'envoi TCP a
> chuté de **6 à 12 % sur un 68080**.

La correction est de marquer le nouveau code `cold` :

```c
/* cold: a rare control path; keeping it (and sb_if_name) out of the hot
 * text stops LTO from re-partitioning the TX/RX code around it, which cost
 * 6-12 % of TCP send throughput on a 68080 in a plain build */
__attribute__((cold)) LONG sb_ifioctl(struct SocketBase *base, ULONG req, APTR argp)
```

**Règle pratique pour contribuer :**

- Tout nouveau chemin de **contrôle** (ioctl, configuration, requêtes rares, outils de
  diagnostic dans la library) doit être marqué `__attribute__((cold))`, ainsi que ses
  fonctions d'aide qui ne servent qu'à lui.
- Après tout ajout dans la library, refaire un `sockbench tx` (et `rx`) sur le 68080 et
  comparer avec la mesure d'avant. Une baisse sans rapport avec le code modifié indique
  un problème de placement, pas un bug de logique.
- Le layout du module est imposé par l'éditeur de liens, pas par l'ordre des sources
  (`emu68_module_layout(bsdsocket_library WRITABLE)` : point d'entrée à l'offset 0,
  `RT_ENDSKIP`). Ne pas compter sur l'ordre des fichiers pour placer le code.

---

## 4. Ce que le build retire hors debug

Sous le niveau debug (`EMU68_TIER`), `lwipopts.h` définit `LWIP_NOASSERT` :

- les ~490 `LWIP_ASSERT` de lwIP disparaissent (~19 Ko de moins dans la library) ;
- `LWIP_ASSERT_CORE_LOCKED()` devient vide (en debug il coûte un `FindTask()` à chaque
  entrée dans lwIP) ;
- `LWIP_ERROR` reste actif dans tous les builds (récupération propre des erreurs).

Conséquence : pour mesurer les performances, utiliser le build release ou le niveau
`profile` (`PROFILE=lwip-amiga`), jamais le niveau debug, dont les chiffres sont gonflés
par les assertions. Le contrôle `TCP_UNSENT_TAIL_DBGCHECK` n'est actif qu'au niveau TRACE.

---

## 5. Optimisations dans le code (indépendantes du CPU, utiles sur 68080)

### Copies mémoire
- `memcpy` de toute la library, y compris le `MEMCPY` de lwIP, est la routine `movem.l`
  d'emu68-common : appelable sous interruption, sans appel Exec (release 1.6).
- **Alignement SANA-II** : les trames reçues sont armées avec 2 octets de remplissage
  avant l'en-tête Ethernet, pour que l'en-tête IP, et donc le corps copié, soit sur un
  mot long. Avec un driver qui prépare ses trames de la même façon (genet.device 3.16/3.17),
  les copies se font en mots longs des deux côtés : une copie d'envoi pleine taille passe
  de **3,9 µs à 0,9 µs** (release 1.6).
- `MEM_ALIGNMENT` = 4 ; les arènes de slab sont pavées sur 64 octets (ligne de cache).

### Chemin TCP
- **Cache de fin de file `unsent_tail`** (patch lwIP local) : `tcp_write` ne parcourt plus
  toute la file `unsent` (~700 segments avec `TCP_SND_BUF` = 1 Mo, ~2200 appels/s en
  envoi saturé). C'était un point chaud mesuré.
- **GRO-lite** (`rx_gro.c`) : les paquets reçus ensemble sur une même connexion sont
  fusionnés et les ACK purs regroupés, environ 25 fois moins de passages dans la pile.
  Partagé entre netdev et SANA-II.
- **Politique d'ACK** (`LWIP_TCP_ACK_AGGREGATES`) : un lot reçu de plus d'un MSS est
  acquitté tout de suite, ce qui accélère les échanges requête/réponse (SMB2).
- Un seul « kick » d'émission par lot de paquets, et libération des tampons envoyés hors
  du verrou utilisé par l'application.

### Fenêtres et tampons (`lwipopts.h`)
- `TCP_MSS` 1460, `TCP_WND` = `TCP_SND_BUF` = 1 Mo, `TCP_SND_QUEUELEN` = 4·SND_BUF/MSS,
  `TCP_WND_UPDATE_THRESHOLD` = `TCP_WND/4`.
- Le pool de lectures SANA-II couvre toute la fenêtre TCP annoncée, sinon le driver jette
  les fins de rafale.

### Checksums
- **netdev** : déchargés sur le matériel quand le driver le permet.
- **SANA-II** : calculés en logiciel dans les deux sens (SANA-II n'a pas de déchargement).
  Le checksum TCP entrant est vérifié par la tâche pump **hors du verrou**, avant la fusion
  GRO. Sur un 68080, ce chemin est celui qui coûte le plus de CPU.

---

## 6. Mesurer

- `sockbench rx|tx|udprx|udptx <hôte> [flux] [secondes] [tailleKo]` contre
  `scripts/tcp-bench-peer.py` sur un PC du réseau.
- `sockbench rr|rrsel <hôte> ...` pour la latence requête/réponse (le chiffre qui compte
  pour SMB2/NFS).
- `udprx` est cadencé par défaut à 100 Mb/s (`0` = inondation sans cadence, ancien
  comportement, peu représentatif).
- `netdev-stats` (driver netdev) et `NetSpeed` pour observer le débit en direct.
- Comparer toujours deux builds **release** (ou `profile`), même driver, même pair, même
  réseau, plusieurs passes.

---

## 7. Ce qui n'existe pas (encore)

À ce jour le dépôt ne contient pas :

- de build ou de drapeaux propres au 68080 (`-m68080`, AMMX, instructions 68080) ;
- de chiffres de débit complets sur 68080 : les tableaux de performances des notes de
  version sont mesurés sur Raspberry Pi 4 / PiStorm / Emu68 avec `genet.device`.

Toute piste d'optimisation propre au 68080 (par exemple un checksum ou une copie en AMMX
pour le chemin SANA-II) devra être mesurée avec `sockbench` avant d'être retenue, comme
toutes les optimisations du projet.
