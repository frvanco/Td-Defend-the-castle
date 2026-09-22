# Tower Defense

### Jouer : **[td-defendthecastle.com](https://td-defendthecastle.com/)**

> **Gratuit, et il le restera.** Le jeu se joue en entier sans payer : toutes les
> tours, tous les creeps et tout le contenu de jeu sont accessibles à tout le
> monde, sans limite de temps et sans monnaie à acheter.
> Les seuls achats possibles sont des **skins cosmétiques optionnels**
> (l'apparence du constructeur et celle du château). Ils ne donnent **aucun
> avantage** : ni or, ni statistique, ni déblocage, ni raccourci. La garantie est
> structurelle et non déclarative — `packages/sim`, qui décide de tout ce qui se
> passe en jeu, **ne connaît pas les skins** : ils n'existent que dans le rendu.

> **Projet en développement actif.** Le jeu est jouable de bout en bout, mais il
> est en cours d'optimisation et de remaniement continus : l'équilibrage, le
> rendu et l'interface bougent d'une semaine à l'autre. Rien ici n'est figé, et
> les chiffres cités dans ce fichier datent de la dernière mesure, pas d'une
> version stable. Ce qui manque encore est listé en fin de fichier,
> [Ce qu'il reste à développer](#ce-quil-reste-à-développer).

Un tower defense multijoueur en 3D, jouable dans le navigateur : six joueurs
défendent chacun leur arène et s'envoient des vagues d'unités. Un moteur de
simulation pur et déterministe (`packages/sim`) fait tourner toutes les arènes en
permanence ; un client web en 3D (Three.js) permet de jouer contre des bots et
d'observer les arènes adverses ; un serveur NestJS sert de porte d'entrée
(comptes, journal des parties, boutique de skins, signalement de bug) ; un runner
headless mesure l'équilibrage en accéléré, sans rendu.

Ce fichier est la **référence d'architecture** : il explique les contrats et les
raisons. `CLAUDE.md` est la consigne de travail courte.

## État du projet

Ce qui marche aujourd'hui :

- une partie complète en solo contre cinq bots, du lancement à l'écran de fin ;
- un didacticiel en vingt-et-un temps, joué dans une vraie partie ;
- comptes invités ou email, journal des quinze dernières parties ;
- une boutique de skins branchée sur Stripe Checkout, et l'équipement stocké
  côté serveur ;
- un runner headless pour mesurer l'équilibrage.

Ce qui est en chantier permanent, et ce qu'il faut garder en tête en lisant la
suite :

- **L'équilibrage n'est pas arrêté.** `balance.json` est retouché à chaque
  session de jeu ; les valeurs citées dans « Équilibrage actuel » sont un
  instantané.
- **Le rendu évolue.** La direction artistique, le décor et les modèles de
  creeps sont repris par passes successives ; les mesures de performance
  bougent avec.
- **Il n'y a pas de vrai multijoueur.** La partie tourne entièrement dans
  l'onglet, contre des bots. Le salon privé n'existe qu'en mémoire côté client.
- **Aucune promesse de compatibilité.** Pas de versionnage sémantique, pas de
  migration de données de jeu : une partie enregistrée aujourd'hui peut ne plus
  se relire après un changement de format.

## Démarrer

```bash
pnpm install

pnpm test          # tests de déterminisme et de règles (363 tests, 30 fichiers)
pnpm typecheck     # tsc --noEmit sur tout le monorepo
pnpm headless 20   # 20 parties bot contre bot (2e arg : easy|medium|hard)

pnpm dev           # client web seul   -> http://localhost:5173
pnpm dev:server    # serveur seul      -> http://localhost:3000 (routes sous /api)
pnpm dev:all       # les deux ensemble
```

Il n'y a **ni ESLint, ni Prettier, ni CI** : `pnpm test` et `pnpm typecheck` sont
le seul filet. Les deux doivent passer avant de rendre la main.

Le runner accepte un 3e argument d'effectif, mais **le format est verrouillé à
6 joueurs** : il n'existe que pour explorer, jamais pour mesurer. Les parties
sont coupées à 40 minutes simulées ; une partie sans vainqueur a atteint ce
plafond, ce qui ne veut pas dire qu'elle est bloquée.

Le client proxie `/api` vers `http://localhost:3000` (voir
`apps/web/vite.config.ts`) : lancer les deux, ou `pnpm dev:all`.

Le serveur a besoin d'une base PostgreSQL. En local, `docker-compose.yml` à la
racine en démarre une (`docker compose up -d`) sur un port non-standard pour ne
pas entrer en conflit avec une instance déjà installée. Copier
`apps/server/.env.example` en `apps/server/.env`, ajuster `DATABASE_URL` si
besoin, puis :

```bash
pnpm --filter @tower-defense/server exec prisma migrate dev
```

### Outils de développement

Tous opt-in par paramètre d'URL, tous sans effet en production :

| Paramètre | Effet |
|---|---|
| `?dev=1` | console de dev (`window.__dev`, `window.__lobbyDev`) et contrôles de vitesse Pause/1x/2x/4x |
| `?perf=1` | instrumentation ; rapport via `window.__perf.getReport()` |
| `?seed=N` | rejoue exactement la même partie d'un lancement à l'autre |
| `?decor=0` | démarre avec le décor masqué (comparaison de performance) |

`?dev=1` n'a d'effet que sur un serveur de développement : les modules concernés
sont derrière `import.meta.env.DEV`, donc élagués du bundle de production — ils
n'y sont pas seulement inactifs, ils ne sont pas livrés.

En partie, avec `?dev=1`, la console expose (`apps/web/src/dev.ts`) :

| Appel | Effet |
|---|---|
| `__dev.setGold(50000)` | fixe l'or du joueur (commande `debugSetGold`, arène 0) |
| `__dev.setLives(3)` | fixe ses vies — **ne déclenche pas la défaite à 0**, l'événement `defeat` n'est pas émis |
| `__dev.unlockTier(3)` | débloque les boutiques d'envoi jusqu'à ce palier — API 1-indexée (1 = Enrôlés, déjà acquise ; 2 = Augmentés ; 3 = Machines), traduite en `unlockShop` répétés |
| `__dev.disableSendTimers()` | remplit tous les stocks d'envoi et supprime l'attente (`debugMaxStock`) |
| `__dev.setSceneryEnabled(false)` | masque le décor sans toucher aux surfaces jouables |
| `__dev.camera()` | position et cible courantes de la caméra, pour figer un cadrage |

Les améliorations de tours ne sont bloquées **que par l'or** : `setGold` suffit à
les ouvrir toutes, il n'y a pas de palier à débloquer de ce côté.

## Structure

```
packages/data       données de la carte, des unités et des tours + surcharges d'équilibrage
packages/sim        moteur pur : tick(state, commands) -> events
packages/renderer   géométrie procédurale 3D des tours (Three.js), indépendante du DOM
packages/lobby      contrat du salon privé + implémentation en mémoire
apps/headless       runner de parties en accéléré, sans rendu
apps/web            client jouable (Three.js), launcher, didacticiel, partie solo contre des bots
apps/server         comptes, parties, boutique, signalements — NestJS + Prisma + PostgreSQL
```

`packages/data/src/map_data.json` est la **référence figée** des données de jeu
et n'est jamais modifié. Tout changement d'équilibrage passe par `balance.json`,
qui l'écrase à la volée — c'est le fichier à ajuster en jouant.

Les données source décrivent 8 arènes ; le format du jeu est verrouillé à
**6 joueurs** (`rules.maxPlayers`, surchargé dans `balance.json`). Les lanes 6
et 7 restent présentes, simplement inutilisées.

## Le contrat du moteur

```ts
const state = createGame(seed, playerCount);
const events = tick(state, commands);
```

- Timestep fixe à 20 Hz. Le temps est toujours un entier de ticks, jamais un
  delta flottant — c'est ce qui garantit le déterminisme.
- Aucun `Math.random` ni `Date.now` dans `packages/sim`. Le PRNG est seedé et
  son état vit dans le state, donc il se sérialise avec (le bot a son propre
  PRNG interne, jamais `Math.random` non plus).
- L'état est plat et sérialisable en JSON tel quel. Conséquence pratique :
  `hashState()` couvre tout nouvel état ajouté à `GameState` sans code dédié, et
  le client qui observe une arène adverse lit cet état tel quel au lieu de le
  reconstruire.
- `tick` mute l'état en place. Pour un snapshot : `structuredClone(state)`.
- **Toutes les arènes sont simulées à chaque tick, tout le temps** — que le
  client les affiche ou non. Le coût mesuré est négligeable (~0,03 ms/tick sur
  une partie à 6 joueurs avec 180 creeps et 109 tours). C'est ce qui permet au
  client de faire naviguer le joueur entre les arènes sans jamais mettre la
  simulation en pause.

Le déterminisme n'est **pas** nécessaire au netcode (il n'y a pas de serveur de
jeu autoritaire pour l'instant) — il sert à rendre les analyses d'équilibrage
reproductibles d'une exécution à l'autre. `packages/sim/test/determinism.test.ts`
garde ce contrat.

Toute égalité doit être départagée par une règle stable et explicite (index de
joueur, index d'emplacement), jamais par l'ordre d'itération d'une `Map`.

La cadence de l'income vit **dans l'état** (`state.roundIntervalSec`), pas
seulement dans les règles : le didacticiel l'accélère le temps d'un temps de
scénario, et une cadence en cours n'est jamais rallongée par une nouvelle
commande — sinon le décompte affiché ferait un bond en arrière sous les yeux du
joueur.

## Le constructeur

Une tour n'apparaît pas au clic. Chaque arène a un **constructeur**
(`builder` dans le code, `packages/sim/src/builder.ts`), un par joueur et donc
six, bots compris, qui se rend sur place avant de construire. Le temps de trajet
est une mécanique : construire loin coûte du temps, et l'ordre dans lequel on
construit devient un moyen indirect de le placer pour la suite.

- **Déplacement en ligne droite, aucun pathfinding.** Les tours ne bloquent
  pas : à l'échelle de rendu le recouvrement ne se voit pas, et les rendre
  bloquantes créerait une mécanique absurde où mieux on construit, plus son
  constructeur est gêné.
- **Le seul obstacle est le couloir des creeps**, franchi en vol. Le test porte
  sur la géométrie *fixe* du chemin (`packages/data/src/builder.ts`, cinq
  rectangles dérivés des waypoints), jamais sur une notion de « bloqué » qui
  n'existe pas. Un trajet qui recoupe le couloir redécolle à chaque fois, sans
  état conservé entre les ticks.
- **L'or est débité au clic et l'emplacement réservé** dès la planification.
  Cette réservation est structurante : les bots choisissent leur case via
  `!arena.occupied`, ils héritent donc de la règle sans code dédié.
- **File de 5 constructions** au maximum (`rules.builderQueueMax`), celle en
  cours comprise. Un ordre au-delà est refusé sans rien débiter.
- **Annulation** (`cancelBuildQueue`) : rembourse à 100 % les ordres planifiés et
  libère leurs emplacements. Celle en cours va au bout et n'est pas remboursée —
  « en cours » signifie `mode === 'building'`, un ordre vers lequel il marche
  encore est annulable.
- **Il ne retourne nulle part** une fois la file vide : il reste exactement là
  où il a fini.
- Il ne combat pas, n'a pas de points de vie, ne peut pas être ciblé ni
  perturbé. Il n'apparaît dans aucune boucle de combat.

Un détail assumé : pour la rangée d'emplacements qui borde le chemin, la
distance d'arrêt (49,6) dépasse l'écart entre l'emplacement et le bord du ruban
(39). Il construit donc ces cases en se tenant d'une dizaine d'unités sur le
couloir. `packages/sim/test/builder.test.ts` le documente pour que ce ne soit
pas pris pour un bug.

Réglages, tous dans `balance.json` :

| Clé | Valeur | Rôle |
|---|---|---|
| `builderGroundSpeed` | 192 | vitesse au sol, unités monde/s (≈ celle d'un creep) |
| `builderFlySpeed` | 320 | vitesse en survol du couloir |
| `builderBuildSec` | 3 | durée de construction, identique pour toutes les tours |
| `builderQueueMax` | 5 | taille de la file |
| `builderRadius` | 9,6 | recul par rapport au centre de l'emplacement |
| `builderFlyMargin` | 16 | piste de décollage avant le bord du couloir |

Les vitesses sont en **unités monde**, la seule unité que connaît la simulation.
Le rendu applique `WORLD_TO_SCENE = 2/64` par-dessus : 192 et 320 valent 6 et 10
unités de scène par seconde. Une durée de construction proportionnelle au prix a
été écartée — elle punirait deux fois les tours chères.

Le mot **« Constructeur »** est celui affiché au joueur ; le code, lui, dit
partout `builder` (fichiers, types, clés de règles).

## Client web (`apps/web`)

Écran d'accueil (pseudo ou connexion) puis menu (`launcher.ts`) ; `startGame()`
(`main.ts`) lance une partie locale contre bots dans le markup existant et peut
être rappelée sans recharger la page (retour à l'accueil, Rejouer) — elle libère
alors proprement la scène 3D précédente (`disposeScene3D`) avant d'en recréer
une.

**Équipement** : le joueur choisit son skin de constructeur et de château depuis
l'accueil (`renderEquipmentScreen`), parmi ce que son compte possède. Cet
équipement vit **côté serveur** (table `EquippedSkin`), pas dans le
`localStorage` du navigateur : une possession ne se garde pas chez l'acheteur.
`apps/web/src/equipment.ts` en tient un cache synchrone, rempli à l'ouverture de
session — donc aussi après une connexion, une inscription ou une entrée en
invité, pas seulement au démarrage avec une session déjà valide. Seule la
difficulté choisie reste dans le `localStorage` : ce n'est pas une possession.

Les catalogues clients (`builders.ts`, `castles.ts`) sont la liste des modèles
*jouables*, indépendamment de qui les possède ; le serveur décide de la
possession, le client du rendu. `resolveBuilderId` accepte aussi la forme courte
des identifiants (`n1` → `contremaitre`), pour qu'une valeur enregistrée sous
cette forme reste lisible.

**Navigation entre arènes** : une barre de pastilles (une par joueur, couleur +
libellé + vies restantes) permet d'observer n'importe quelle arène adverse
pendant que la sienne continue de tourner. Le rendu bascule sans recharger le
décor ni recréer de géométrie : chaque joueur a sa propre instance
`TowerEntities`/`CreepEntities` (son propre sous-groupe Three.js), synchronisée à
chaque frame que son arène soit affichée ou non — changer de vue n'est qu'un
changement de visibilité. Aucune action n'est possible sur une arène qu'on ne
contrôle pas ; le HUD continue d'afficher les valeurs de sa propre arène en
toutes circonstances.

**Chaque joueur a son propre constructeur**, tiré au sort parmi le catalogue
(`drawBuilders`) — seul le joueur garde le sien, celui qu'il a équipé. Le rendu
suit donc le même schéma que les tours et les creeps : une entité par arène,
dont une seule est visible.

**Une entité par joueur, pas par lane.** Les données décrivent 8 arènes, le
format en joue 6 : en créer 8 fabriquerait deux constructeurs fantômes pour des
lanes jamais jouées.

**Chaque instance possède ses matériaux d'équipe.** `clone(true)` partage les
matériaux avec le modèle source, donc avec tous les autres clones — or
`tintTeam` les *mute*. Sans copie, reteindre un constructeur repeint celui de
tout le monde et c'est la dernière teinte qui gagne. Seuls les matériaux
d'équipe sont copiés (le reste peut rester partagé), et la copie se fait
**avant** toute teinte : `Material.copy` sérialise `userData` en JSON, une
`THREE.Color` copiée après coup n'en serait plus une. Pour la même raison,
`dispose()` ne libère que ces copies : parcourir l'instance pour tout disposer
libérerait les ressources GPU du modèle en cache, que three devrait
re-téléverser à la partie suivante.

Le tirage utilise `Math.random` — c'est du rendu, pas de la simulation : il ne
change à aucun moment `GameState` et reste donc hors du contrat de déterminisme.
Il diffère à chaque lancement, y compris à `?seed=N` identique. Les doublons sont
assumés : le catalogue compte moins d'entrées qu'une partie n'a de joueurs.

**Barre de commandes** (bas de l'écran, hauteur fixe de 182 px) : trois modes
exclusifs — construction, envoi de creeps, abilités — un panneau d'information à
gauche, les ressources à droite, et la colonne de modes au centre droit. Le
compteur de la file de construction y est un badge sur le bouton Construction,
avec son bouton d'annulation juste en dessous. Convention du projet : **jamais de
rouge** pour signaler une indisponibilité, jaune et vert uniquement.

**Lecture de la construction** : dès qu'un ordre est planifié, la tour visée
apparaît en translucide sur son emplacement (`buildPreview.ts`). Quand le
constructeur commence à bâtir, cet aperçu *devient* le chantier — couleurs
pleines, échafaudage, croissance, poussière — pendant exactement
`builderBuildSec`, et la tour réelle apparaît déjà finie. L'avancement est lu sur
la simulation (`buildTicksLeft`), pas accumulé image par image : arriver en cours
de chantier le reprend à son avancement réel, et rien ne dérive en x2 ou x4. Les
ordres *planifiés* ne sont montrés que sur sa propre arène (ils révéleraient les
intentions d'un adversaire) ; le chantier *en cours*, lui, est visible partout.

**Un creep qui atteint la sortie s'efface, il ne meurt pas.** L'événement `leak`
porte l'identifiant de l'entité, et le rendu la retire sans jouer le clip de
mort : l'animation de mort raconterait une défense réussie alors que le joueur
vient de perdre une vie.

**Écran de fin** : il s'affiche aussi quand on quitte en cours de partie
(élimination, retour à l'accueil, quitter). Le classement ne liste alors que les
joueurs qui ont réellement fini — afficher un rang pour des arènes encore en vie
donnerait un résultat qui n'existe pas.

### Décor et rendu 3D

Le décor est procédural et entièrement visuel : il n'entre dans aucune
simulation, aucune collision, aucun raycast. Deux couches :

- `terrain3d.ts`, `plateau.ts`, `terrainGrass.ts` : le plateau jouable, ses
  falaises, ses crêtes et son dallage. Les falaises sont bruitées par sommet, et
  l'esplanade du château est dérivée de la géométrie du couloir plutôt que
  posée à la main.
- `apps/web/src/sanctuary/` : les ruines, le château, les ateliers, les arches,
  la statue et le pont. Ce module a **son propre README**
  (`apps/web/src/sanctuary/README.md`) : options, densité, palette et protocole
  de comparaison de performance.

Le rendu passe par un **tone mapping ACES Filmic** (`renderer.toneMapping`,
exposition dans `apps/web/src/tonemap.ts`). C'est le piège central de toute
retouche de couleur ici : une valeur hexadécimale écrite dans le code n'est
**pas** celle qui s'affiche, la courbe l'écrase. `pourAfficher(hex)` inverse la
courbe pour les cas où l'on veut vraiment obtenir la couleur demandée à l'écran
(le dégradé du ciel, par exemple). La brume, elle, est laissée littérale :
Three.js mélange le fog **après** le tone mapping et la conversion d'espace
colorimétrique (`fog_fragment` suit `colorspace_fragment`), la compenser la
décalerait deux fois.

La caméra ne monte jamais au-dessus de l'horizon (`maxPolarAngle`) : tout ce qui
est peint au zénith est invisible en jeu, c'est une erreur classique à éviter en
retouchant le ciel.

Ordre de grandeur mesuré avec `?perf=1` sur une arène affichée : environ
190 appels de dessin et 210 000 triangles. Ces chiffres bougent à chaque passe
de décor — les remesurer avant de conclure quoi que ce soit.

L'interpolation entre deux ticks de simulation (20 Hz) n'existe pas côté rendu
(60 Hz) : le mouvement des creeps est donc discret, pas lissé. Chantier séparé,
pas encore traité.

### Écran de chargement

Entre le clic qui lance une partie et son premier tick, rien ne tourne : tout ce
que la partie consommera est chargé et préparé d'abord, barre de progression à
l'appui. Sans lui, une partie démarrerait avec des creeps en sphère et des tours
procédurales remplacées au fil des arrivées, et la première apparition d'un creep
jamais vu compilerait ses shaders en pleine frame.

Trois modules, trois responsabilités :

| Module | Rôle |
|---|---|
| `modelManifest.ts` | **données pures** : quel `.glb` pour quelle entité, rien d'autre |
| `preload.ts` | `preloadAll(onProgress, builders)` — télécharge tout, ne charge rien lui-même |
| `loadingScreen.ts` | l'overlay : la table des joueurs, la barre, le libellé d'étape |

`modelManifest.ts` n'importe **ni three.js ni aucun chargeur**. C'est ce qui
permet au launcher — donc à la page d'accueil — de connaître la liste complète
des fichiers sans traîner le moteur de rendu dans son bundle. Les modules qui
chargent réellement (`entities3d.ts`, `towerModel.ts`) lisent le même manifeste :
une seule liste, jamais deux à tenir en phase.

- **Ce qui est préchargé** : les 36 `.glb` de creeps, les 27 de tours, les
  modèles de constructeur **réellement tirés** pour cette partie (dédoublonnés),
  et les chunks `main.js` et du didacticiel. Charger le code fait partie du
  chargement : c'est la première étape de la barre, pas un préalable invisible.
- **Aucun échec ne rejette.** Un fichier illisible compte comme terminé et
  laisse jouer son repli (sphère générique, géométrie procédurale). Un `.glb`
  manquant coûte son modèle, jamais la partie — c'est pourquoi
  `loadAnimatedCreepModel` résout `null` au lieu de rejeter.
- **L'écran dure au moins 7 secondes**, le temps de lire la table. C'est un
  plancher, jamais un plafond : un chargement plus long n'est pas raccourci.
- **La barre est le minimum entre le travail fait et le temps écoulé** sur ces
  7 secondes. Tout en cache, elle met quand même 7 s à se remplir, au rythme du
  temps ; chargement lent, c'est le travail réel qui commande de bout en bout.
  Elle n'annonce donc jamais plus que ce qui est fait.

#### La table

Une carte par joueur, au centre de l'écran, barre en dessous. De haut en bas :
un onglet et un liseré à la couleur de sa lane — la même qu'en jeu dans la barre
d'arènes, c'est ce qui permet de reconnaître un adversaire une fois la partie
lancée —, le rôle (`P1`, `P2`…), la vignette de son constructeur, son nom, puis
une seconde ligne. Le pseudo et « Vous » pour le joueur ; « Bot » et le niveau
pour les adversaires, qui partagent tous celui choisi à l'accueil.

**Les vignettes portent la couleur du joueur**, et seulement sur leurs pièces
d'équipement — comme `tintTeam` en jeu, qui ne repeint que les matériaux
d'équipe et laisse la peau, la barbe, le cuir et l'acier intacts.

Une vignette est une image plate : elle ne porte aucune information de matériau.
Une fusion de calques CSS repeindrait donc le personnage entier, peau comprise.
`builderTint.ts` travaille pixel par pixel sur un canvas et croise **deux
critères**, aucun ne suffisant seul (mesures faites sur les fichiers livrés) :

| | équipement | zone à préserver | ce qui les sépare |
|---|---|---|---|
| n1 | chemise, 0-29°, sat 0,77 | peau, 20-30°, sat 0,27 | la **saturation** seule |
| n2 | équipement, 15-44° | peau verte, 150-165° | la **teinte** seule |

Sur n1 la teinte ne distingue rien ; sur n2 les deux zones ont des saturations et
des surfaces comparables. D'où une fenêtre de teinte autour de `tintHue`
multipliée par une rampe de saturation. Ces valeurs sont **mesurées sur le dessin
livré**, jamais reprises du `.glb` : les vignettes sont des visuels dessinés qui
ne suivent pas la couleur d'équipe du modèle (n2 est bleu dans son `.glb` mais
orange sur son dessin, n3 rouge mais bleu).

Le plancher de saturation est bas quand `tintHue` est connue — la fenêtre fait
déjà le tri, la saturation n'a plus qu'à écarter les gris — et haut sinon, où
elle est le seul critère. Un dessin dont la peau partage la teinte de
l'équipement relève du second cas malgré sa teinte connue : il le déclare via
`tintMinSat`.

Le `src` de l'image n'est posé qu'une fois la teinte calculée : attacher le
dessin non teinté puis le remplacer donnerait à voir une couleur qui n'est pas
celle du joueur.

Le cadre reprend le vocabulaire du launcher : bleu acier, surtitre encadré de
filets, titre sérif en capitales espacées, losange de séparation, équerres
d'angle. Le fond réutilise l'illustration d'accueil, fortement assombrie —
plutôt qu'une image de plus à produire, et c'est ce qui fait lire l'écran comme
la suite du launcher et non comme un interlude. C'est ce contenu qui justifie la
durée plancher : sans lui, l'écran n'aurait aucune raison de durer.

#### Montage en deux temps

`prepareGame()` monte la scène, les entités et les listeners puis fait le
warmup ; `start()` lance la boucle. **Rien ne tique avant `start()`.** Le canvas
doit être visible et dimensionné à l'appel de `prepareGame()` : la compilation
des shaders porte sur la taille réelle de la cible de rendu, d'où un `#app`
révélé *derrière* l'overlay de chargement, qui le couvre encore.

Le warmup occupe les 10 derniers pour cent de la barre (« Préparation… ») :

1. créer les `AnimatedCreepController` des 36 modèles **pour les 6 arènes** —
   sinon le premier envoi de T2, puis de T3, puis le premier changement d'arène
   construisent leurs `InstancedMesh` en pleine frame ;
2. rendre tous les groupes visibles, appeler `renderer.compileAsync()`, puis
   **restituer la visibilité d'avant**. `compileAsync` ne compile que ce qu'il
   voit : sans ce passage, les arènes masquées compileraient au premier
   basculement.

Chaque étape **rend la main au navigateur** avant la suivante. Sans cette pause,
les six arènes s'enchaînent en une seule tâche bloquante : le libellé et le
segment 90-100 % sont bien écrits dans le DOM, mais jamais peints.

`start()` remet `last` et l'accumulateur à zéro. Sans ça, le délai du warmup
serait livré d'un coup à la simulation et la partie démarrerait par un bond de
plusieurs dizaines de ticks.

Toutes les entrées en partie passent par `enterGame()` (`launcher.ts`) : c'est ce
qui garantit qu'aucun bouton ne peut lancer une partie sans ce chemin.
**Rejouer** ne repasse pas par là : il réutilise la scène montée et ne fait que
réinitialiser l'état, donc il est immédiat par construction.

**Vérifier** : `?perf=1` expose `game.viewedSphereAlive` — le nombre de creeps
tombés sur la sphère de repli faute de modèle. Avec le préchargement en place il
doit rester à **0**.

### Didacticiel (`apps/web/src/tutorial/`)

Une **vraie partie**, pas une maquette : mêmes commandes, mêmes règles, aucun
raccourci qui ferait apparaître une tour ou un creep hors du moteur. Le scénario
tient en vingt-et-un temps dans `steps.ts`, en données pures — le contrôleur ne
fait que les dérouler. Rien du didacticiel ne vit ailleurs que dans ce dossier ;
la partie n'en connaît que le pont `TutorialBridge`.

| Module | Rôle |
|---|---|
| `types.ts` | le vocabulaire, et le pont par lequel `startGame()` le branche |
| `steps.ts` | le scénario, en données : répliques, attentes, montants crédités |
| `scriptedDriver.ts` | les cinq adversaires, pilotés par une file d'ordres au lieu d'une IA |
| `watcher.ts` | traduit l'état de la partie en événements de scénario |
| `controller.ts` | déroule, gèle, crédite, guide |
| `overlay.ts` | la bulle du constructeur, les surlignages, les flèches, les panneaux |

- Le pilote scripté satisfait la **même forme que `Bot`**, donc la boucle de jeu
  ne distingue pas les deux. Il attend son or *et* son stock au lieu de forcer.
- Le moteur n'émet **aucun événement** pour une tour terminée, une amélioration
  ou un creep tué. `watcher.ts` déduit ces trois-là en comparant l'état d'un tick
  au précédent — la simulation n'est pas touchée.
- Une réplique gèle la partie en cessant d'**accumuler** du temps, sans toucher à
  `speed`, qui appartient aux contrôles de dev. Une commande émise pendant un gel
  s'applique au dégel : c'est pourquoi la cadence d'income est écrite sur l'état
  neuf plutôt que reposée à chaque temps.
- **L'income est accéléré à 5 s** pendant les premiers temps, puis rendu à
  `rules.roundIntervalSec` : attendre 30 secondes son premier revenu, sans rien
  à faire, serait le point mort du didacticiel.
- La bulle est ancrée au **modèle du constructeur**, reprojeté à chaque frame :
  c'est lui qui parle, pas un bandeau anonyme. Elle ne capte pas le pointeur —
  seuls ses boutons le font. Posée en plein terrain, la rendre cliquable
  empêcherait de faire tourner la caméra ou de viser un emplacement à travers
  elle.
- **Des flèches** désignent l'élément surligné (`tuto-arrow`), replacées à chaque
  frame sur la boîte réelle de la cible ; celles qui pointent un élément du bord
  droit se posent à sa gauche pour ne pas sortir de l'écran.
- **Les panneaux DÉPART et ARRIVÉE** ne vivent que le temps du surlignage du
  chemin, reprojetés sur les deux bouts du couloir. Les laisser toute la partie
  encombrerait le terrain longtemps après avoir servi.
- Le deuxième temps demande de **bouger la caméra**, et dit qu'il faut *maintenir*
  le clic gauche pour pivoter et le clic droit pour se déplacer. Il est placé là
  parce que c'est le seul moment où l'on parle du terrain, et parce que les
  premiers temps seraient sinon entièrement passifs. Le mouvement est détecté par
  un `change` des `OrbitControls` survenu **entre** `start` et `end` — `start`
  seul partirait sur un simple clic, `change` seul sur l'amortissement et sur les
  `update()` de la boucle de rendu.

**Toutes les racines sauf la Tourelle sont grisées** (`LOCKED_TOWERS`) :
visibles dans la grille, non sélectionnables. Le didacticiel enseigne une
défense, pas un choix de branche. La liste est **dérivée** des racines
constructibles, pas écrite à la main : ajouter une branche au jeu la grise ici
sans intervention, et un test verrouille l'invariant.

Les montrer inactives plutôt que les retirer dit au joueur que le jeu en compte
plus que ce qu'il peut essayer ici. Le bouton porte `disabled`, ce qui empêche
réellement le clic sans ajouter de garde dans le gestionnaire — au prix d'un
effet de bord : un bouton désactivé n'émet aucun événement de souris, donc la
zone info reste muette sur ces tuiles, et c'est le `title` du conteneur qui
explique pourquoi.

## Assets 3D

| Dossier | Contenu |
|---|---|
| `public/models/towers/` | 27 `.glb`, un par palier, rangés par slug de branche |
| `public/models/creeps/` | 36 `.glb` |
| `public/models/builders/` | les `.glb` de constructeurs ; **seuls ceux inscrits dans `builders.ts` sont jouables** |
| `public/models/castles/` | les `.glb` de châteaux (`castles.ts`) |
| `public/icons/` | vignettes 128×128 correspondantes |

Ces fichiers sont **produits hors du dépôt** : ne pas les régénérer ni les
modifier. Un `.glb` déposé dans `models/builders/` sans entrée dans
`builders.ts` est simplement ignoré : le catalogue fait foi, pas le contenu du
dossier.

Tous sont préchargés avant la première frame (voir « Écran de chargement ») : les
replis ci-dessous ne sont pas le chemin normal du démarrage : ils sont la
réponse à un fichier illisible.

Aucun de ces modèles n'a de squelette : ce sont des assemblages de pièces rigides
animés par transformations de nœuds. Il n'y a donc **jamais de `SkinnedMesh`**
dans ce projet, et `SkeletonUtils` n'est pas nécessaire — `clone(true)` suffit.

Trois pipelines distincts, pour trois contraintes différentes :

- **Tours** (`towerModel.ts` + `packages/renderer/src/towers/fromModel.ts`) : un
  modèle chargé une fois, cloné par tour, matériaux et géométries partagés.
  Repli sur la géométrie procédurale de `packages/renderer` si le `.glb` manque
  ou est illisible — **garder ce repli fonctionnel**.
- **Creeps** (`animatedCreepModel.ts`, `animatedCreepInstances.ts`) : fusionnés
  et rendus en `InstancedMesh`, parce qu'il peut y en avoir des centaines. Deux
  clips suffisent et sont **pré-échantillonnés** en tables de poses — `Walk`
  (32 pas, en boucle) et `Death` (24 pas, jouée une fois) : le rendu indexe
  dedans, sans `AnimationMixer` ni évaluation de courbe par instance. C'est ce
  choix qui rend des centaines de creeps tenables.
- **Constructeurs** (`builderModel.ts`, `builderEntity.ts`) : ni l'un ni l'autre.
  Un seul est visible, donc aucune instanciation, aucun cache de géométrie,
  aucune fusion — un `AnimationMixer` standard sur une hiérarchie clonée.

La **couleur du joueur** suit la convention `TeamColor` : le matériau portant ce
nom est repeint par instance. Les modèles peuvent aussi déclarer leurs
métadonnées dans les `extras` du `.glb` (`teamColorMaterial`, `actionProps`,
`heightMeters`) — le chargeur les lit et ne connaît alors aucun modèle en
particulier. Attention : Three.js ne recopie **pas** les extras de la racine du
fichier, il faut les lire sur `gltf.parser.json.extras`.

Ajouter un constructeur : déposer le `.glb` dans `models/builders/`, sa vignette
dans `icons/builders/`, et une entrée dans `apps/web/src/builders.ts` qui déclare
les deux chemins (`url`, `iconUrl` — les noms de fichiers sont libres, le
catalogue fait foi). Y renseigner `tintHue` — la teinte des pièces d'équipement
**telle qu'elle est dessinée sur la vignette** — sinon la recoloration de l'écran
de chargement retombe sur la seule saturation.

**La vignette doit rendre l'équipement séparable de la peau**, sans quoi la
recoloration déborde. Une des deux conditions suffit : soit la peau est
nettement moins saturée que l'équipement (au plus 0,45 contre au moins 0,65,
avec `tintMinSat: 0.55`), soit leurs teintes s'écartent d'au moins 50°.

Le Contremaître tient la première par construction, et il le faut : sa peau
partage la teinte de son équipement, la fenêtre de teinte ne les sépare donc pas.
Bras, visage, front, barbe et cheveux sont dessinés à 0,44 de saturation contre
0,72-0,74 pour l'équipement, et les pixels semi-transparents du contour suivent
la même valeur, sans quoi un liseré coloré cerne les bras. Casquette, chemise,
panneau du marteau et jetpack restent saturés : ils portent tous
`Builder_vermilion` dans le `.glb`, ils doivent donc bien prendre la couleur du
joueur.

Les vignettes sont des **visuels dessinés**, comme toutes les icônes du projet.
À défaut, un dépannage rend le modèle depuis le `.glb` :

```bash
pnpm --filter @tower-defense/web gen-builder-icons
```

Le script est autonome (il sert lui-même les fichiers, aucun `pnpm dev` requis),
reprend l'éclairage du jeu et **ne remplace jamais une vignette existante** sans
`--force`. Il joue une image du clip `Idle` avant de rendre : sans cela la
hiérarchie reste en pose de bind, où *tous* les accessoires sont déployés.

Une vignette livrée doit porter une **vraie couche alpha**. Un export qui
aplatit le damier de transparence dans les pixels affiche un carré blanc
quadrillé sur le fond sombre du panneau.

## Serveur (`apps/server`)

NestJS + Prisma + PostgreSQL, monté à la main (pas de scaffold `nest new`) ;
`apps/server/tsconfig.json` est **volontairement autonome**, il n'étend pas
`tsconfig.base.json` (NestJS a besoin des décorateurs legacy et de CommonJS,
incompatibles avec le reste du monorepo — ne pas « corriger »). Session par
cookie `httpOnly` (90 jours), mot de passe haché en argon2id, rate limiting par
IP (`@nestjs/throttler`), `ValidationPipe({ whitelist: true })` global.

| Préfixe | Rôle |
|---|---|
| `/api/auth` | compte invité par pseudo, création de compte, connexion, rattachement d'un email à un invité, déconnexion, profil courant |
| `/api/matches` | journal personnel des parties, borné aux 15 dernières par joueur |
| `/api/shop` | catalogue, inventaire, réclamation, achat, équipement |
| `/api/bug-reports` | signalement de bug depuis l'accueil, relayé dans un salon Discord |
| `/api/health` | sonde de vivacité ; **ne touche pas la base** |
| `/webhooks/stripe` | **hors du préfixe `/api`** : l'adresse est posée dans le tableau de bord Stripe |

Un **invité est un compte à part entière** (`User.isGuest`) : il a une ligne, un
discriminant `#n` et une session. « Invité » ne veut donc jamais dire « sans
compte », ce qui explique que `BugReport.accountId` soit rempli pour lui aussi.

### Boutique de skins (Stripe Checkout)

Un skin est une **apparence** de constructeur ou de château, identifiée par le
même slug que le modèle côté client (`barista`, `forteresse-arcanique`). Trois
tables : `Skin` (catalogue), `OwnedSkin` (possession) et `EquippedSkin` (ce que
le compte porte).

**Rien de ce qui se vend n'entre dans le jeu.** Un skin ne traverse jamais la
frontière du rendu : `packages/sim` n'a aucun champ pour lui, aucune commande ne
le mentionne, et `hashState()` ne le voit pas. Le seul contenu payant du projet
est donc, par construction, incapable d'influer sur une partie — c'est ce qui
rend la promesse de la première ligne de ce fichier vérifiable plutôt que
déclarative.

Le catalogue distingue trois familles par `inShop` et `isDefault` :

| | `inShop` | accordé à la création | visible en boutique |
|---|---|---|---|
| Bastion, Citadelle royale (`isDefault`) | non | oui | non |
| Contremaître, Mécanicien xéno, Mage noir, Bastion de guerre | non | oui | non |
| Canard de débug (offert), Barista, Mécano, Forteresse arcanique | oui | non | oui |

Les règles de sécurité de l'intégration, chacune couverte par un test
(`apps/server/test/shop-checkout.test.ts`, `shop-webhook.test.ts`) :

- **Le montant vient toujours de la base.** Le client n'envoie qu'un slug :
  `/api/shop/checkout` relit le skin, vérifie qu'il est actif, payant, non
  possédé, puis crée la session Checkout avec la référence de prix Stripe — que
  le client ne voit jamais, et sans qu'aucun montant ne transite.
- **`success_url` n'accorde rien** : elle ne sert qu'à afficher un message et à
  rafraîchir l'inventaire, et s'appelle à la main sans rien obtenir.
- **Le skin n'est accordé que par `/webhooks/stripe`**, dont la signature est
  vérifiée sur le **corps brut** (`rawBody: true` dans `main.ts` ; un JSON
  re-sérialisé ne reproduit pas les octets signés). Sans secret de webhook
  configuré, **tout** webhook est refusé.
- **On ne vend jamais ce qu'on ne peut pas accorder.** Une clé Stripe présente
  sans `STRIPE_WEBHOOK_SECRET` est le seul réglage qui coûte de l'argent au
  joueur : il paie sur Checkout, le webhook est refusé, aucun skin n'arrive. En
  production `readShopConfig` coupe donc la vente dans ce cas (catalogue
  consultable, `/api/shop/checkout` en 503) et journalise une erreur ; hors
  production il se contente d'un avertissement, puisqu'on y ouvre Checkout pour
  regarder sans payer.
- **Idempotence par contrainte unique** : la clé primaire `(accountId, skinId)`.
  Stripe rejoue ses événements ; un rejeu n'écrit rien et ne notifie rien.
- **L'achat est réservé aux comptes inscrits.** Un invité n'a pas d'email, donc
  aucun moyen de récupérer un achat perdu avec son cookie ; il peut en revanche
  réclamer les skins offerts.
- La notification Discord d'achat ne porte **ni email, ni coordonnées, ni donnée
  de paiement** — pseudo, skin, montant, identifiant de session.

Mise en route :

```bash
# 1. structure et catalogue (les price_id ne sont PAS dans la migration)
pnpm --filter @tower-defense/server exec prisma migrate deploy
# 2. rattache les price_id du tableau de bord, depuis l'environnement
pnpm --filter @tower-defense/server exec tsx prisma/seed.ts
# 3. pour tester les webhooks en local ; la commande affiche le whsec_ a mettre
#    dans STRIPE_WEBHOOK_SECRET
stripe listen --events checkout.session.completed --forward-to localhost:3000/webhooks/stripe
```

Variables : `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `APP_URL`,
`STRIPE_PRICE_BARISTA`, `STRIPE_PRICE_MECANO`,
`STRIPE_PRICE_FORTERESSE_ARCANIQUE`, `DISCORD_PURCHASE_TRACKING_URL`. Sans clé
secrète la boutique reste consultable et les achats répondent 503.

Le secret de webhook de **production** ne vient pas de `stripe listen` : il est
affiché à la création du point de terminaison dans le tableau de bord Stripe
(Developers → Webhooks → le point sur `https://<domaine>/webhooks/stripe` →
« Signing secret »). Il commence par `whsec_`, diffère de celui du local, et ne
s'obtient plus qu'en le révélant depuis cette page. Vérifier après déploiement
que les journaux du serveur ne contiennent **ni** `STRIPE_WEBHOOK_SECRET absent`
**ni** `vente desactivee` : ces deux lignes signifient qu'aucun achat ne peut
aboutir.

**Aucune clé, aucun secret dans le dépôt** : il est public. `.env` est dans le
`.gitignore`, et une clé Stripe ou une URL de webhook poussée sur GitHub est
révoquée par balayage automatique.

### Signalement de bug

Une modale sur l'écran d'accueil (`apps/web/src/bugReportModal.ts`, chargée au
premier clic) poste en `multipart/form-data` trois champs — catégorie, résumé,
description — et une capture facultative. **Tout le reste de la ligne est posé
par le serveur** : compte, session, pseudo dénormalisé, commit du build, agent et
empreinte d'IP salée. Un client qui les enverrait quand même se les fait retirer
en silence par le `ValidationPipe` global (`whitelist`).

Le bouton **n'existe qu'à l'accueil** : il est masqué à l'entrée en partie, au
salon et à l'écran de fin. Attention au piège CSS qui a déjà coûté deux
corrections : une règle `#bug-report-btn { display: flex }` (sélecteur d'ID) bat
la règle `[hidden] { display: none }` du navigateur. Poser l'attribut `hidden`
ne suffit donc pas, il faut la règle `#bug-report-btn[hidden] { display: none }`
— et vérifier sur `getComputedStyle().display`, jamais sur la propriété
`hidden`.

La capture n'est jamais écrite telle quelle : type reconnu **aux premiers
octets** (JPEG et PNG seulement, jamais l'extension ni le `Content-Type`), puis
re-encodage systématique en JPEG 1280 px par `sharp`, sous un nom tiré au hasard.
La base ne garde que le chemin relatif ; au-delà de 10 jours une tâche
quotidienne efface le fichier et remet la colonne à `NULL`, la ligne texte
restant l'historique.

Discord part **après** la réponse au joueur et sans être attendu : un webhook
absent, lent ou en erreur laisse le signalement enregistré avec
`discordSent = false`. La base est la source de vérité, le salon n'est qu'un
affichage.

Variables (voir `apps/server/.env.example`, toutes facultatives en
développement) : `DISCORD_BUG_WEBHOOK_URL`, `BUG_REPORT_STORAGE_DIR`,
`IP_HASH_SALT`, `BUILD_COMMIT`.

## Déploiement

Le serveur Nest sert **l'API et le client compilé sur la même origine** : le
client appelle `/api` en relatif (`apps/web/src/api.ts`), donc rien à ouvrir en
CORS et le cookie de session reste en `sameSite: lax`. Servir les deux
séparément demanderait les trois à la fois — base d'URL injectée au build du
client, CORS avec `credentials`, et cookie en `sameSite: none`.

`Dockerfile` à la racine construit l'image : client puis serveur, dans une seule
image qui démarre sur `node apps/server/dist/main.js`.

**Les migrations ne sont pas jouées au démarrage** — deux instances qui démarrent
ensemble les joueraient en concurrence. C'est une étape de release, à part :

```bash
pnpm --filter @tower-defense/server migrate:deploy   # structure + catalogue
pnpm --filter @tower-defense/server seed             # price_id Stripe, depuis l'environnement
```

Variables à fournir, en plus de celles déjà décrites plus haut :

| Variable | Rôle |
|---|---|
| `PORT` | imposé par la plupart des hébergeurs, qui arrêtent le conteneur s'il n'écoute pas dessus |
| `NODE_ENV=production` | **sans lui le cookie de session n'est pas `secure`** |
| `CLIENT_DIST_DIR` | client compilé à servir ; vide = `apps/web/dist`, absent = seule l'API répond |
| `TRUST_PROXY` | nombre d'intermédiaires de confiance : 1 derrière un seul reverse proxy, 2 avec un CDN devant |
| `IP_HASH_SALT` | sans lui, un sel est tiré à chaque démarrage et les empreintes d'IP ne se rapprochent plus |
| `APP_URL` | racine publique, pour les URL de retour de Stripe Checkout |

La sonde `GET /api/health` ne touche pas la base : une plateforme qui redémarre
le conteneur parce que Postgres a eu une seconde difficile aggraverait la panne
au lieu de la corriger.

Le webhook Stripe se déclare dans le tableau de bord sur
`https://<domaine>/webhooks/stripe`, **hors du préfixe `/api`**.

Trois limites connues, à traiter le jour où il y aura plus d'une instance :

- la limitation d'envoi (throttler Nest et limiteur du signalement de bug) vit
  **en mémoire** : à deux instances, les quotas doublent ;
- la purge quotidienne des captures tourne dans chaque instance ;
- les captures s'écrivent sur le **disque local** (`BUG_REPORT_STORAGE_DIR`).
  Sur un disque éphémère elles disparaissent à chaque redéploiement — il faut un
  volume persistant, ou passer à un stockage objet.

## Salon privé (`packages/lobby`)

Salons de 6 places avec code à 4 caractères. `contract.ts` définit l'interface
sans rien présumer de l'implémentation, et n'importe **ni `sim` ni `data`** : ces
types décrivent ce qui circulera un jour sur le fil, pas l'état interne du
moteur. `mock.ts` en fournit une implémentation **en mémoire, côté client** — il
n'existe aucune route serveur de salon, deux navigateurs ne peuvent pas se
rejoindre aujourd'hui. Le jour où un client réseau arrivera, il devra satisfaire
exactement le même contrat.

L'alphabet des codes exclut I, O, 0 et 1 : un code se lit à voix haute ou se
recopie depuis une capture d'écran, et ces confusions sont la première source
d'erreur.

## Équilibrage actuel

Tout dans `packages/data/src/balance.json`, jamais dans `map_data.json`. Cette
section est un **instantané** : elle change à chaque passe.

**Règles** : 30 vies (`startLives`), revenu de départ 60 par round, un round
toutes les 30 s (`roundIntervalSec`), prime de mise à mort 15 % du coût du creep
tué, multiplicateur de vitesse des creeps 0,7 (`creepSpeedMultiplier`).

**Six branches de tours** totalisant **27 paliers**, dont 6 constructibles
directement (les racines) :

| Branche | Paliers | Chaîne |
|---|---|---|
| Balistique | 5 | Tourelle → Canon → Canon lourd → Obusier → Canon à rail |
| Acide | 5 | Acide → Corrosive → Dissolvante → Nécrose → Solvant |
| Givre | 5 | Givre → Gel → Blizzard → Cryogène → Zéro absolu |
| Anti-aérien | 5 | Anti-aérien → Arc → Foudre → Orage → Tempête |
| Cadence | 5 | Répétiteur → Mitrailleuse → Gatling → Fauchoir → Moissonneuse |
| Réacteur | 2 | Réacteur → Soleil artificiel |

Le Réacteur est la seule branche de fin de partie (30 000 puis 180 000 or) et n'a
que deux paliers, alignés sur les paliers 4-5 des autres ; c'est pour cela qu'il
est placé en dernier dans la barre d'achat.

**Aucune règle de prix uniforme n'est visée.** Le rapport entre le prix d'un
palier et son efficacité en DPS par or n'est pas régulier, ni d'un palier au
suivant ni d'une branche à l'autre, et ce n'est pas un défaut : les prix se
règlent branche par branche, au ressenti de jeu. Ne pas chercher à leur imposer
une progression arithmétique, et ne pas traiter une irrégularité comme une dérive
à corriger.

`towers` contient une 28e entrée, `h003` (« Canon »), qui n'est ni une racine ni
la cible d'une amélioration : elle est donc inatteignable en jeu. Elle n'a pas de `.glb`, d'où les 27 modèles de tours.

**Trois branches ont une ability** (`packages/sim/src/status.ts`) :
ralentissement de zone (Givre, deux sources s'additionnent jusqu'à `SLOW_CAP`),
poison mono-cible qui ignore l'armure (Acide), chaîne d'éclair à rebonds
décroissants sur cibles aériennes uniquement (Anti-aérien).

**39 creeps, trois boutiques d'envoi** de 12 unités chacune. Enrôlés est acquise
dès le départ (`unlockedShopTier` vaut 0) ; les deux suivantes se débloquent
*séquentiellement*, jamais en visant un palier précis — Augmentés à 2 000 or,
Machines à 20 000. Un creep d'une boutique non débloquée n'est pas « en
rupture » : il est inaccessible, et la barrière est dans la simulation, pas
seulement dans l'interface.

**Bots** (`packages/sim/src/bot.ts`) : trois niveaux (`easy`/`medium`/`hard`,
défaut `medium`) qui ne changent que la *compétence* — vitesse de décision,
placement selon la longueur de chemin couverte à portée, réaction aux vagues
aériennes. La *personnalité* (agressivité, composition préférée) est tirée du RNG
propre du bot, indépendamment du niveau : un bot agressif n'est pas plus fort,
juste différent. Chaque bot est plafonné à 200 tours simultanées, puis continue
les améliorations et les envois.

Les bots sont soumis **exactement aux mêmes règles que le joueur humain** : même
constructeur, même file de 5, même délai de construction. Ils respectent ces
plafonds *à l'émission* — la convention du dépôt est qu'un bot n'émet jamais une
commande qu'il sait vouée au rejet, sans quoi sa comptabilité interne se
débiterait d'achats qui n'ont jamais eu lieu.

### Économie

Le revenu d'un joueur a deux sources : l'income versé à chaque round (le gros du
revenu), et la prime de mise à mort — quand un creep meurt dans l'arène d'un
joueur, ce joueur reçoit `rules.bountyPct` de son coût en or (arrondi au
supérieur, minimum 1). C'est le propriétaire de l'arène qui est payé, jamais
l'envoyeur ; un creep qui leak, ou qui est engendré par la mort d'un autre
(Porte-essaim), ne rapporte rien. `arena.goldFromBounty` et
`arena.goldFromIncome` cumulent chaque source séparément — c'est ce que
`pnpm headless` utilise pour afficher la part du revenu qui vient de la défense
plutôt que de l'income pur.

Acheter un creep achète surtout du **revenu permanent** : `income` augmente du
`pointValue` de l'unité envoyée. Le creep part chez tous les autres joueurs
vivants, jamais chez l'acheteur.

### Mesurer une modification

```bash
pnpm headless 20          # 20 parties, difficulté et effectif par défaut
pnpm headless 20 hard
```

Ce que le headless sert à détecter : une **régression grossière**, pas la qualité
de l'équilibrage. Le ressenti de jeu se juge en jouant. Un changement
d'équilibrage ne s'annonce jamais comme validé sur la seule sortie du runner :
donner les chiffres avant/après et laisser l'arbitrage.

## Emplacements de construction

Une tour ne se pose pas n'importe où dans la zone constructible : elle se pose
sur un **emplacement** précis, défini dans `packages/data/src/build_slots.json`
(**236 par arène**, regroupés en rangées/colonnes nommées — grille pleine à
l'intérieur du U, bandes collées au chemin sur les bras et le connecteur).

- `packages/data/scripts/gen_slots.ts` régénère le layout à partir de
  `packages/data/src/zoneFootprints.ts` (géométrie du couloir de la lane 0, les
  autres arènes sont des copies translatées) ; le JSON produit peut aussi être
  modifié à la main sans toucher au code. **Ne pas le régénérer sans demande
  explicite : cela change la carte.** Après un changement de `SLOT_SIZE`, mettre
  à jour le nombre attendu dans `packages/sim/test/slots.test.ts`.
- `SLOT_SIZE` vaut 80 : moins d'emplacements par arène qu'une grille plus fine,
  mais assez de place par tour pour qu'elles soient visiblement plus imposantes
  que les creeps.
- `packages/data/src/slots.ts` expose `buildSlots(player)` (emplacements d'une
  arène, coordonnées déjà translatées) et `nearestSlot(player, x, y)`
  (l'emplacement le plus proche d'un clic, ou `null` au-delà d'une case).
- Dans `packages/sim/src/sim.ts`, `buildTower` snap sur `nearestSlot()` : la tour
  prend la position exacte de l'emplacement, jamais celle du clic — c'est ce qui
  garde la sim déterministe. L'occupation est suivie par `arena.occupied`, indexée
  par l'id stable de l'emplacement, **marquée dès la planification** et libérée à
  la vente ou à l'annulation.

## Ce qu'il reste à développer

Par ordre d'importance. Rien ici n'est commencé sauf mention explicite.

### 1. La partie multijoueur

C'est le manque structurant : aujourd'hui la partie tourne **entièrement dans
l'onglet**, contre des bots, et deux navigateurs ne peuvent pas se rejoindre.

- Pas de serveur de jeu autoritaire, pas de WebSocket. Le moteur est déjà pur et
  déterministe, ce qui rend l'option « commandes répliquées + états rejoués »
  crédible, mais aucun choix de netcode n'est arrêté.
- Le salon privé existe en contrat (`packages/lobby/contract.ts`) et en
  implémentation mémoire (`mock.ts`) ; il n'a **aucune route serveur**. Le
  contrat a été écrit pour que le client réseau le remplace sans toucher à
  l'interface.
- `GameState` ne modélise **aucune déconnexion** : seul `alive` existe. Un joueur
  qui part en cours de partie n'a pas de représentation.
- Le lancement d'une partie passe tout entier par `enterGame()`, ce qui laisse un
  seul point d'entrée à adapter.

### 2. L'équilibrage global

Les valeurs bougent à chaque session ; plusieurs points sont identifiés et non
traités :

- **La Tourelle (racine Balistique) domine.** Sur la dernière campagne de 20
  parties, elle représente 49 % de l'or investi dans les tours encore debout,
  loin devant l'Acide (21 %) et l'Anti-aérien (18 %) — malgré les rééquilibrages
  successifs.
- **Les durées du runner ne sont pas celles d'une partie jouée.** En jeu, une
  partie dure 20 à 25 minutes, 30 au plus. Le headless, lui, fait s'affronter six
  bots et mesure 21,9 à 37,9 minutes (médiane 27,6) sur la dernière campagne. Cet
  écart n'est pas expliqué aujourd'hui, et rien ne dit qu'il vienne de
  l'équilibrage : les durées du runner servent à comparer deux états du jeu entre
  eux, jamais à décrire ce que vit un joueur. Ne pas conclure d'une durée de
  runner que les parties sont trop longues.
- **Le délai de construction n'a jamais été mesuré isolément** contre l'état
  antérieur au constructeur.
- **`h00T` (Soleil artificiel) a une portée de 2000**, la plus grande du jeu :
  au-dessus des 1500 de Balistique, Acide et Givre, et du double des 1000
  d'Anti-aérien et Cadence.
- **`h011` (Tempête) tire toutes les 0,10 s pour 18 000 DPS**, la borne basse de
  cadence du jeu avec `h010`. Signalé en marge du nerf des six branches, qui ne
  touchait pas l'Anti-aérien ; `packages/data/test/towers.test.ts` a dû descendre
  sa garde de cadence à 0,10 pour l'englober.
- La baseline de comparaison vit dans `CLAUDE.md` (campagne datée), mais elle
  n'est qu'un point de départ : aucune cible n'est arrêtée pour le taux de
  victoire, la durée ou le palier médian atteint.

### 3. Le rendu des creeps

C'est le point faible visuel du jeu. Les leviers disponibles, sans qu'aucun soit
arrêté :

- **L'interpolation entre deux ticks.** La simulation tourne à 20 Hz, le rendu à
  60 : le mouvement est discret, pas lissé. C'est ce qui se voit le plus quand
  plusieurs dizaines de creeps avancent ensemble, et c'est du rendu pur — la
  simulation n'a pas à changer.
- **La variation entre exemplaires d'une même unité.** L'échelle appartient au
  modèle, donc douze exemplaires ont exactement la même stature ; seule la phase
  de marche varie déjà, décalée par entité via le nombre d'or
  (`phaseOffsetFor`).
- **Les modèles eux-mêmes.** Les 36 `.glb` passent tous par le même pipeline :
  ce qui les distingue tient donc entièrement aux assets livrés, produits hors du
  dépôt. Améliorer le rendu des creeps, c'est autant refaire des assets que
  changer du code.

### 4. Le reste

- **Les abilités n'existent pas dans la simulation.** Elles ont un panneau dans
  la barre de commandes et trois branches ont des effets passifs, mais aucune
  *commande* d'ability n'est implémentée côté moteur.
- **Multi-instance.** Trois limites à lever avant de passer à plus d'une
  instance : throttler en mémoire, purge des captures dupliquée, captures sur
  disque local (détail en « Déploiement »).
- **Équilibrage des bots contre un humain.** Les niveaux sont réglés contre
  d'autres bots en headless ; leur difficulté ressentie par un joueur n'a pas été
  mesurée.
- **Aucune persistance de partie.** Une partie interrompue est perdue ; seul le
  résumé va au journal (`/api/matches`).
- **Accessibilité et écrans étroits** : l'interface est dessinée pour un écran
  large, la barre de commandes a une hauteur fixe de 182 px et rien n'est prévu
  pour le tactile.
