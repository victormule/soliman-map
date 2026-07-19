# Le Corps de Soliman al‑Halabi — cartographie interactive

Réseau navigable en 3D de la carte mentale Miro sur les restes humains en
contexte muséal. JavaScript vanilla, modules ES natifs, Three.js en `vendor/`.
**Aucune dépendance à installer.**

```bash
node build-graph.mjs   # export_miro.json -> graph.json  (à refaire après tout ré-export Miro)
npx serve .            # puis ouvrir l'URL — les modules ES interdisent file://
```

## Les trois fichiers qui comptent

| Fichier | Rôle |
|---|---|
| `export_miro.json` | **Source.** L'export brut du tableau. Ne se modifie pas à la main. |
| `build-graph.mjs` | **Les règles.** Tout ce qui transforme le tableau en réseau. |
| `graph.json` | **Produit.** Régénérable à volonté — ne jamais l'éditer, la prochaine exécution effacerait la correction. |

`index.html` charge `graph.json` par `fetch`. Il ne l'inline pas : le navigateur
peut le mettre en cache, et le fichier reste lisible et diffable.

## Ce que build-graph.mjs décide

Les règles sont en clair et commentées dans le script. Les cinq tables en tête
(`SECTIONS`, `REGARDS`, `SOUS_THEMES`, `PERIODES`, `FORMES`) sont **éditoriales**
— elles portent des noms et des couleurs qu'aucun algorithme ne devine :

- `SECTIONS` — les 14 sections = les **blocs titrés du tableau lui-même**
  (désocialisation, déontologie, monstration, nommer le corps, retours ESAA…),
  chacun avec ses `anchors` : une ou plusieurs ancres sur son contenu (et sous
  son titre quand il est excentré). Chaque post‑it rejoint le bloc de l'ancre la
  plus proche. `s_frame` fait exception : ce sont les enfants du cadre Miro.
  ⚠ L'ancienne grille de 6 ancres absorbait des blocs entiers dans le mauvais
  voisin (la désocialisation dans « Cas pratiques »…) — l'affectation a été
  revalidée bloc par bloc, chaque titre tombe dans le sien.
- `REGARDS` — les 11 regards et la couleur de remplissage qui les désigne —
  **hors du cadre uniquement** —, plus les `aliases` (les couleurs employées
  pour le même regard sans tomber pile sur celle de la légende). **Ce n'est pas
  un calcul de proximité**, c'est cette liste. Une couleur absente reste « non
  classée » : mieux vaut ne rien dire que mal attribuer un point de vue.
  `light_yellow` (la couleur PAR DÉFAUT d'un sticky Miro) n'est l'alias de
  rien : une couleur qu'on obtient sans la choisir ne désigne pas un point de vue.
- `SOUS_THEMES` — ⚠ **dans le cadre « Vue d'ensemble », la couleur ne dit pas
  un regard mais un sous-thème** : onze grappes, chacune titrée sur le mur par
  un post‑it « x … » de sa couleur (l'ancien build lisait « regard » partout :
  ~105 contresens). Le jaune sert deux grappes, départagées par l'ancre du
  titre ; le rouge du cadre n'a aucun titre → « sans sous-thème », annoncé.
- `FORMES` — le REGISTRE d'un post-it (« Comment ? ») : note · citation
  rapportée · lien/référence · image · appareil du tableau. ⚠ C'est un **crible,
  pas une coloration** : 4 post-its sur 5 sont des notes, et le geste que l'axe
  sert est de DÉCOCHER « Note » pour voir les cent autres. PRIORITÉ déclarée,
  une seule classe par nœud (image > appareil > lien/référence > citation >
  note) — un post-it qui cite ET porte un lien compte comme référence : le lien
  est un fait de l'item, la citation une lecture de son texte.
  ⚠ **La citation est la seule chose que cet axe devine.** Les guillemets ne
  suffisent pas : treize post-its les emploient en MENTION (« restes humains »)
  et non pour rapporter une voix. D'où `CITATION_MIN_LEN` / `CITATION_MIN_COVER`
  — un passage cité doit être long, ou occuper l'essentiel du post-it. Ces deux
  seuils partent dans `method.params` et la fiche (§ 5 ter) les AFFICHE.
  Les quatre autres classes se relèvent dans la structure (type, liens, nature),
  d'où le pointillé de l'onglet, comme « Quand ? ».

Un sixième vocabulaire éditorial ne vit PAS dans le build : `FLOW_DEFS`
(« Pourquoi ? ») est calculé dans le navigateur par `computeStructure`, comme
Degré et Tissu, parce qu'il se recompte sur l'étendue regardée.

La **légende du tableau** (13 items : pastilles des regards, « Légende »…) est
écartée du graphe : comptée comme corpus, elle gonflait chaque regard d'une
unité et tissait de faux fils « même texte ». Chaque nœud reçoit aussi une
**nature** (`contenu` / `titre` / `question` / `meta` / `reference` / `image`,
au corps typographique et aux marqueurs) : l'appareil éditorial reste visible
et cherchable, mais les statistiques (∑, lexique, croisements) l'écartent en
l'annonçant. Les deux images porteuses d'un `altText` le reçoivent pour texte.

Le reste est dérivé : le texte (HTML Miro → texte, insécables préservées), les
étiquettes (coupées au mot), le degré, la taille (`min(1.22, 0.62 + 0.075×degré)`),
les arêtes, les deux dispositions, et le **routage orthogonal** de la
cartographie (§ 9 — voir « Le tracé des liens »). La **direction** des connecteurs et leurs
**pointillés** sont conservés (`e.dir`, `e.dashed` — voir `addEdge`) et dessinés
tels quels : cônes en `InstancedMesh`, `LineDashedMaterial` pour les pointillés.

⚠️ **Cela ne fait PAS du graphe un graphe orienté**, et c'est le piège à ne pas
refermer : sur 451 connecteurs, 220 portent une pointe et **231 n'en portent
aucune**. Un trait sans pointe n'est pas un trait dont la direction manquerait —
c'est un rapprochement posé sans sens de lecture. Les quatre états (`dir` =
0 · 1 · −1 · 2) coexistent ; ne pas les ramener à un seul, ni « compléter » les
231 par des doubles pointes, ce qui inventerait 231 gestes.
⚠️ **Ce que la flèche SIGNIFIE reste inconnu** — dérivation ? réponse ?
chronologie ? Le tableau ne le dit nulle part. On restitue le geste sans
l'interpréter, et la fiche (§ 4, § 12) le déclare plutôt que de le taire.

**Le script dit ce qu'il jette.** Il annonce en fin d'exécution les items rejetés
et leur motif, les couleurs qu'il n'a pas su classer, et l'état du corpus. Si un
chiffre vous surprend, c'est le script qu'il faut lire — pas deviner.

## La fiche « à propos & méthode »

Trois onglets sous une en-tête collante (ⓘ dans le titre) : **À propos** (ce que
la carte est, ce qu'elle n'est pas, l'état du corpus), **Méthode** (§ 1 à 12 :
extraction, sections, regards, liens, attributs dérivés, les deux dispositions,
le recentrage, le rendu, la légende, l'analyse, les limites) et **Navigation**.

⚠️ **Aucun chiffre ni paramètre n'y est écrit en dur.** Les comptes se calculent
à l'ouverture depuis `graph.json` (même règle que le panneau ∑) et les
PARAMÈTRES de dérivation — graine, itérations, forces, seuils, échelles — sont
expédiés par `build-graph.mjs` dans `graph.json.method`, **depuis les constantes
mêmes qu'il applique**. Décrire un protocole avec des valeurs recopiées, c'est
le désynchroniser au premier réglage : une méthode fausse est pire qu'une
méthode absente. Si `method` manque (graph.json d'un build antérieur), la fiche
le DIT et `console.warn` — elle n'affiche pas des tirets.

⚠️ **Cette règle a été affirmée avant d'être tenue** (audit de juillet 2026). Le
§ 10 portait sept comptes recopiés à la main (195, 410, 594, 110, 355, 129,
125/129), le § 5 bis quatre écarts horaires, le § 8 le diamètre du graphe — et
ils avaient déjà dérivé : « sept sections » pour quatorze, « 22 sauts » pour
vingt-cinq, « 16 sans sous-thème » pour vingt-huit. Le build les MESURE
désormais (`census.arbre`, `census.silencesHours`, `census.giantDiameter`,
`census.textMedian`) et les annonce en fin d'exécution. Corollaire : **une
phrase de la fiche qui a besoin d'un nombre le demande à `census`** — et les
constantes qui appartiennent à la page, non au build (seuils du lexique
`COOC_MIN`/`COOC_MAX_LINKS`, rayons `EGO_RING`/`EGO_R_MAX`), se lisent depuis
la constante elle-même, jamais retapées.

⚠️ `census.arbre.crossConnector` **doit rester à 0** : c'est lui qui autorise la
phrase « aucun trait tracé à la main ne franchit la frontière tronc/branches »,
et le recalcul de Degré/Tissu sur l'étendue. S'il change, le build le crie et la
fiche cesse d'affirmer — c'est la thèse du § 10 qu'il faut alors rouvrir, pas le
compte qu'il faut ajuster.

Corollaire : une constante de simulation citée par la fiche vit au niveau module
dans `build-graph.mjs`, pas dans le bloc qui la consomme.

## Les deux dispositions

- **`miro`** — le tableau à plat, à l'échelle de la scène. Transformation affine
  des coordonnées Miro : ce que l'œil voit est la géographie composée à la main
  par les auteur·rices. La scène est un PLAN (z tient dans ±2).
- **`constellation`** — le tableau délié de sa géographie : ce qui rapproche
  n'est plus le voisinage sur le mur, c'est le lien. Les 14 sections tiennent une
  sphère de Fibonacci (fixes, ce sont les amers) ; les post‑its se placent par
  relaxation — les traits tirent, la gravité retient près de sa section, tout le
  monde se repousse.

L'aléatoire est à **graine fixe** (`SEED`) : trois exécutions donnent le même
fichier, au bit près. Vérifié.

⚠️ La relaxation avance d'un **pas borné qui refroidit**, jamais par une vitesse
accumulée d'un tour sur l'autre : garder la vitesse amplifie chaque à‑coup au
lieu de le dissiper, et la simulation part à l'infini (mesuré une fois : 1e+109).
Le plancher `MIN_SEP` joue le même rôle pour la répulsion en 1/d², qui explose
dès que deux post‑its se frôlent. Ne pas « simplifier » ces deux garde-fous.

## Le tracé des liens : direct ou circuit

Une disposition dit **où sont les nœuds** ; le tracé dit **comment les traits
vont de l'un à l'autre**. Ce sont deux questions, d'où deux réglages — et le
second ne déplace rien : `state.trace` vaut `'direct'` (une corde) ou
`'circuit'` (orthogonal, coudes biseautés à 45°).

**Le routeur vit dans `circuit-router.mjs`, et il n'y en a qu'un.** `build-graph.mjs`
l'appelle pour la cartographie (positions figées → routage figé, livré dans
`graph.json` : chaque arête porte `route: [x1,y1,x2,y2,x3,y3,x4,y4]`, ses points
INTÉRIEURS seulement, les extrémités se relisant sur les nœuds qui bougent). La
page l'appelle pour le **recentrage**, dont les positions sont calculées à la
volée. Deux copies auraient dérivé au premier réglage, et les deux vues
auraient dessiné deux circuits différents en prétendant suivre la même règle.

⚠️ **Le recentrage route dans un WORKER** (`circuit-worker.mjs`), et c'est une
mesure, pas une précaution : à pleine profondeur (395 nœuds, 465 arêtes) le
routage prend ~4,6 s. Sur le fil d'affichage ce serait un gel — plus de
rotation, plus de survol, plus de clic. Le résultat revient avec sa CLÉ
(`nœud@profondeur`) : entre l'envoi et la réponse le visiteur a pu changer de
vue, et l'on ne pose pas sur la carte le circuit d'une autre. Le vivant s'accorde
`ripup: 2` au lieu de 8 — c'est un budget de temps, pas une règle différente :
mêmes candidats, mêmes poids.

⚠️ **Le routeur est écrit en tableaux plats, sans allocation dans les boucles.**
La première version, en objets `{x,y}` avec des `Set` et des clés de case en
chaînes, mettait **39,6 s** pour 594 arêtes — correcte et inutilisable. Ce qui
coûtait n'était pas l'algorithme mais ce qu'on jetait. S'y ajoute un élagage
exact (un candidat déjà plus cher que le meilleur connu est abandonné en route :
tous les termes du coût étant positifs, la somme partielle ne redescend jamais).
39,6 s → 5,1 s, **au trait près le même résultat**. Ne pas « simplifier » en
revenant aux objets.

⚠️ **Un chemin tracé suit le trajet de l'arête qu'il surligne**, coudes compris
(`edgeBetween` retrouve l'arête et son sens). En corde par-dessus un circuit, le
surlignage désignait un lien en passant par où ce lien ne passe pas. Les arêtes,
les pointes de flèche et les chemins lisent tous `routePointAt` : **une seule
fonction calcule cette géométrie**, sinon le troisième code aurait divergé.

Le routeur est **glouton** : pose du plus court au plus long, 12 chemins
candidats par arête (2 L + 6 Z, dont des partages hors de [0,1] qui offrent un
détour), coût = croisements + recouvrements + nœuds rasés + longueur. Puis la
**reprise** (*rip-up and reroute*) : chaque arête est retirée et reposée face au
tracé complet, tant que ça bouge. Converge en 5 passes ; mesuré, 25 % de
croisements en moins et 43 % de nœuds rasés en moins qu'en ligne droite.

⚠️ **Un coude ne veut rien dire.** Ni détour, ni étape, ni hiérarchie : c'est un
évitement décidé par un compte de croisements. Seules les EXTRÉMITÉS d'un trait
portent du sens (et sa pointe, si elle en a une). La fiche le dit avec ces mots
au § 6 bis, et il faut que ça le reste : un tracé qui aurait l'air de signifier
quelque chose serait pire qu'une ligne droite.

⚠️ **Le circuit reste à plat**, et l'option se TAIT en constellation
(`traceMuteReason`) plutôt que d'agir à moitié : « se croiser » n'y a pas de sens
— deux traits qui se croisent à l'écran ne se touchent pas dans l'espace, et la
figure change à chaque tour de caméra. Le recentrage, lui, n'est PAS muet : il
route à la volée. Il est seulement en ATTENTE (`traceIsWaiting`, battement sur le
bouton) le temps du calcul — un bouton choisi mais pas encore honoré doit le
dire, sans quoi on croit à un clic perdu.

⚠️ **Le nombre de sommets par arête ne s'écrit pas dans la page** : `ROUTE_PTS`
se LIT sur les données (`e.route.length / 2 + 2`), le build en est seul maître.
Une arête vaut TOUJOURS ce nombre de sommets, y compris en tracé direct où ils
s'alignent sagement sur la corde — c'est ce qui fait de la bascule une simple
interpolation sommet par sommet, sans reconstruire un tampon ni traiter de cas
particulier. Les pointes de flèche s'orientent sur le **dernier segment** et non
sur la corde : une flèche qui vise là où le trait n'arrive pas se voit tout de
suite.

⚠️ `circuitW` (0 → 1) n'a **qu'un seul animateur à la fois** : `animateLayout`
le pilote pendant une métamorphose, `animateCircuit` le reste du temps. La
garde est à DOUBLE SENS (drapeau `layoutAnimating`, pas le simple test de
`layoutAnim` — qui ne redevient jamais `null` tout seul et ne peut donc pas
dire « en cours ») : `animateLayout` annule `circuitAnim` en entrant, ET
`animateCircuit` se tait tant que `layoutAnimating` est vrai, quitte à être
rappelée par `animateLayout` à l'atterrissage. Le sens qui manquait a été le
bug du recentrage en circuit (juillet 2026) : le routage du worker peut très
bien revenir PENDANT le vol des nœuds (le recentrage démarre à profondeur
« toute la composante ») — sans cette garde, `animateCircuit` démarrait quand
même, et les deux boucles écrivaient `circuitW` chacune de leur côté. Pire
qu'une oscillation : les sommets intérieurs visaient alors les coordonnées
ABSOLUES du circuit final (calculé pour la position d'arrivée) pendant que
les extrémités du trait étaient encore à mi-vol depuis l'ancienne disposition
— d'où des traits qui partaient dans tous les sens plutôt qu'une simple
saccade. `enterEgo` pose aussi ce drapeau AVANT `requestEgoRoutes` (pas
seulement `animateLayout`) : si le routage est déjà en cache, la réponse est
appliquée tout de suite, avant même le recul caméra qui précède la
métamorphose.

## Le brouillard : un piège documenté

La profondeur d'un brouillard se compte depuis la **caméra**, jamais depuis le
centre de la scène. La caméra se tient à ~230 du centre d'un nuage large de
~120 : avec une densité exponentielle calée sur le rayon, le nuage entier est
déjà « au loin ».

C'est l'erreur qu'a faite le réglage d'origine (`FogExp2(…, 0.0055)` en dur :
**91 % de la cartographie effacée** à la distance d'arrivée — la carte n'était
pas lointaine, elle était noyée). D'où le brouillard **linéaire** : il a ce que
l'exponentiel n'a pas, un DÉBUT. On l'accroche au bord proche du nuage (voile
nul) et on le porte jusqu'au bord lointain (voile `FOG_AT_FAR`).

Et il ne s'allume **qu'en constellation** : en cartographie la scène est un plan,
il n'y a aucune profondeur à dire — le voile n'y assombrissait que pour rien.

## Le geste dépend du mode

En cartographie la scène est plate : la faire tourner n'a aucun sens. Mais la
rotation vit sur le bouton gauche — la couper laissait le **bouton gauche mort**,
et le déplacement exilé sur le clic droit (inaccessible au trackpad, inexistant
au doigt). D'où `applyModeCameraLock()` : à plat, le bouton gauche DÉPLACE.

Un clic dans le vide fait un **bond** : la vue se recentre sur le point cliqué en
gardant sa distance et son angle (la caméra suit du même vecteur que la cible —
c'est ce qui préserve le recul et en fait un déplacement, pas un zoom).

Le bandeau du bas annonce le geste du mode **courant** : il promettait « glisser
pour tourner » dans un mode où la rotation était coupée.

Le bouton **⟳** (en-tête) tient la rotation automatique. Deux choses distinctes,
et c'est l'intérêt de les séparer : `state.rotateWanted` est la VOLONTÉ du
visiteur — elle survit à un changement de disposition — quand
`controls.autoRotate` est ce qui tourne à cet instant (suspendu par toute
interaction, repris après 10 s, impossible à plat). À plat le bouton est sourd
et le dit ; il ne se décoche pas pour autant, revenir en constellation retrouve
la volonté exprimée. Il ne tourne lui-même que quand la caméra tourne :
l'attente d'inactivité se voit, au lieu de passer pour une panne.

**⟲ réinitialise TOUT l'état de lecture**, pas la seule caméra : sélection,
recherche, chemins, recentrage, mise en lumière, **arborescence** (retour à
l'arbre entier), **cases des sept axes**,
**épingles** (y compris celles des onglets fermés — elles agissaient sans se
voir), étiquettes de proximité, axe de coloration, rotation. Une chose ne bouge
pas : la **disposition**, cadre de lecture délibérément choisi aux onglets — la
métamorphoser répondrait à une question que personne n'a posée.

## Cadrage

`distanceToFit()` interroge les deux axes séparément — la hauteur par le FOV
vertical, la largeur par le FOV **horizontal** (qui dépend de l'écran) — et garde
le plus exigeant. Un écran étroit ou un téléphone à l'horizontale obtient donc le
bon recul sans réglage séparé. L'ancien calcul prenait la demi-diagonale × 1,6 :
il ne connaissait ni la forme de ce qu'il cadrait, ni l'écran qui le regarde.

## Légende & analyse

- La légende se lit comme trois décisions emboîtées : la DISPOSITION (où sont
  les nœuds), l'ARBORESCENCE (quel corpus on regarde), puis l'AXE (ce qui colore).
- **L'arborescence — Arbre / Tronc / Branches.** Le tableau est deux corpus, et
  la mesure l'établit : sur 594 liens, 110 sont internes au tronc, 355 aux
  branches, et les 129 qui traversent sont TOUS des fils « même texte »
  (calculés) — **aucun trait tracé à la main ne franchit la frontière**. Le
  tronc PRÉCÈDE les branches de cinq jours. Restreindre est un filtre : cumulé
  aux cases, cité par l'URL (`sc=`), relâché par ⟲. Sur l'arbre rien n'est
  filtré, les 129 fils restent entiers. ⚠ « arbre » et « branche » sont
  RÉSERVÉS à cette lecture : le panneau des chemins tient le vocabulaire de la
  ROUTE (une route part d'un nœud et porte un ou plusieurs chemins).
- Les AXES (registre `AXES` : `keyOf`, `colorOf`, `items`) — cinq onglets en
  deux familles, la BORDURE disant la provenance (plein = posé par les
  auteur·rices, POINTILLÉ = relevé par l'archive, tireté = calculé) :
  - ⚠ **« Pourquoi ? » ne dit PAS l'intention** — rien ne l'enregistre. Il dit
    la place d'un post-it dans le raisonnement TRACÉ, lue dans le sens des
    flèches : amorce · relais · aboutissement, plus DEUX ignorances distinctes.
    Ne pas les fondre : « relié, mais sans flèche » (219) n'est pas « aucun
    trait tracé » (148) — la première est une lacune du tableau, la seconde un
    fait. Ensemble elles pèsent 60 %, et c'est la mesure de ce que l'axe ignore.
    Il siège avec les questions éditoriales mais porte le TIRETÉ du calculé :
    le regroupement sert la lecture, la bordure tient la rigueur.
    ⚠ « Où ? » n'a délibérément pas d'axe : la **cartographie** y répond déjà,
    la position d'un post-it EST son lieu (fiche § 5 quater).
  - **éditorial · les cinq questions** : « Quoi ? », « Qui ? », « Quand ? »,
    « Comment ? » (le REGISTRE — voir `FORMES` plus haut ; un crible, pas une
    coloration, et le seul axe dont une classe soit l'appareil du tableau : sa
    répartition dans ∑ porte donc sur TOUTES les feuilles, ce que le sous-titre
    dit plutôt que de laisser un total qui ne tombe pas juste).
    ⚠ **« Quoi ? » a DEUX vocabulaires** — les 13 blocs dans les branches,
    les 11 sujets (grappes de couleur) dans le tronc — présentés en deux groupes
    repliables. C'est ce qui a supprimé l'onglet « Sous-thème » (question déjà
    posée) et la ligne « Vue d'ensemble » (un lieu listé parmi des sujets).
    « Qui ? » se TAIT dans le tronc (aucun regard n'y est posé) : l'onglet reste
    visible, grisé, et refuse le clic en disant pourquoi.
    « Quand ? » : 4 PÉRIODES depuis `createdAt` (table `PERIODES` dans
    build-graph.mjs). Surtout pas « session » : à 48 h de coupure le tableau
    compte 8 séances, les 1re et 2e périodes n'en formant qu'UNE. Le nom se lit,
    la borne exacte paraît au SURVOL. ⚠ Le JOUR, jamais l'heure ; aucun nom
    d'auteur·rice nulle part.
  - **structurel · calculé sur l'étendue** : « Degré » (SEPT classes : une par
    nombre de liens jusqu'à 5, puis « 6 liens et plus ») et « Tissu », RECALCULÉS
    à chaque changement d'arborescence (`computeStructure`), titre à l'appui
    (« … dans le tronc »). Sinon un nœud s'annoncerait « 6 liens et plus »
    avec trois voisins visibles. Les NOMS portant des comptes, ils se réécrivent
    aussi (`refreshLegendNames`).
  Ces deux derniers axes n'ont AUCUNE donnée manquante, là où
  « regard » laisse ~54 % du hors-cadre non classé. Les seuils sont écrits
  dans les noms des catégories — une coupure arbitraire doit s'annoncer pour
  être discutable. Toute la machinerie (onglets, filtres cumulés, badges,
  pastilles, mise en lumière, épingles) lit le registre : ajouter un axe =
  une entrée + sa liste dans le HTML. Le repli des deux groupes de
  « Quoi ? » n'existe QUE sur l'arbre (ailleurs un seul vocabulaire
  s'applique : la liste est fixe, l'en-tête perd son chevron). Un groupe replié
  reste un confort de LECTURE, jamais un filtre : le rail replié montre toutes
  les couleurs de l'étendue (`scopedLegendItems`), et « tout cocher /
  tout décocher » agit sur le même ensemble. TROIS blocs légendés séparent la
  DISPOSITION (où sont les nœuds), l'ARBORESCENCE (quel corpus) et l'AXE (ce
  qui les colore) ; dans le dernier, deux familles — « éditorial · les quatre
  questions » et « structurel · calculé sur l'étendue » — et TROIS styles de
  bordure qui disent la provenance (plein = posé, pointillé = relevé, tireté =
  calculé ; même convention que la ligne d'option du recentrage).
  Couleurs : « Degré » en rampe ordonnée froid → chaud sur SEPT pas
  (bleu-gris → bleu → sarcelle → vert → or → orange → rouge : le degré est une
  grandeur, sa rampe est monotone), « Tissu » en trois teintes
  franches (bleu / orange / violet, classes qualitatives) — l'isolé est une
  donnée comme les autres, pas un gris qu'on efface.
- La légende vit REPLIÉE : un rail de symboles (mode ▦/✦/⊚, axe Qo/Qi/Qd/Cm/Pq/D°/C·,
  une pastille par catégorie, 🏷 étiquettes, compteur) qui redit l'état complet
  sans un mot. Le survol déplie, l'épingle 📌 force l'ouverture. Au doigt —
  pas de survol — le premier appui épingle (garde `canHover` : simuler le
  survol tactile piège, le « sticky hover » déplie sans jamais replier).
- Déplié, **Disposition ne défile JAMAIS** (`#lg-disposition`, hors de
  `#legend-scroll`) : c'est le cadre de lecture, il ne doit pas pouvoir
  glisser hors de vue pendant qu'on parcourt les catégories de « Coloré
  par » — seule cette zone-là défile, avec une barre TRÈS fine (4 px) et
  discrète plutôt que masquée, puisque c'est désormais la seule indication
  qu'il y a plus à voir. Le pied (étiquettes, curseur, épingles, compte,
  épingler-la-légende) reste entièrement fixe sous le défilement ; compte et
  bascule d'épingle partagent une rangée (`#lg-footrow`) pour tenir moins de
  hauteur, à contenu égal.
- Les sections n'ont PAS de sphère sur la carte : une section est un
  ensemble de données, pas un point dans l'espace. Leur position survit en
  interne (ancre de mise en page, cible du vol) mais rien ne se dessine.
- Cliquer le NOM d'une catégorie la met en lumière (pleine intensité, le
  reste recule, pastille pleine) et y mène ; re-cliquer éteint. La CASE,
  elle, filtre. Deux gestes, deux états : lecture vs sélection.
- ∑ (en haut à droite) : analyse calculée à l'ouverture depuis graph.json —
  corpus, répartitions cliquables, croisement section × regard (une cellule
  filtre la paire), croisement regard × regard, nœuds les plus connectés,
  lexique + concordancier. Rien d'écrit à la main.
- Lexique : fréquence DOCUMENTAIRE (nb de post-its contenant le terme),
  mots-outils exclus, formes non lemmatisées — et l'écran l'annonce. Cliquer
  un terme déplie son **concordancier** : chaque post-it qui le contient, en
  CONTEXTE (extrait de texte réel, accents et casse d'origine) plutôt qu'une
  liste de nœuds à rouvrir un par un. Réutilise l'ensemble de post-its déjà
  calculé pour la fréquence documentaire — aucun nouveau balayage du corpus.
  La position du terme se cherche dans le texte NORMALISÉ (même fonction que
  la recherche), mais l'extrait affiché vient du texte SOURCE à la même
  position : aplatir la casse et retirer un accent ne déplace aucun
  caractère, donc l'un vaut pour l'autre — ce qu'on lit garde ses vrais
  accents. Un seul terme s'examine à la fois (accordéon, même principe que
  `activeBranch` pour les chemins) ; au-delà de 60 post-its, le surplus est
  annoncé, jamais tronqué en silence.
- Croisement regard × regard (dans ∑) : compte des ARÊTES, pas des nœuds —
  pour chaque CONNECTEUR (à l'exclusion des fils « même texte » : un post-it
  répété n'est pas un dialogue, c'est la même voix redite) dont les deux
  extrémités ont un regard connu, la case du couple s'incrémente. La
  diagonale dit le dialogue d'un regard avec lui-même, le reste le pont entre
  deux points de vue — la question de recherche que pose ce croisement.
  Matrice SYMÉTRIQUE : un seul triangle s'affiche, diagonale comprise.
- Recherche : tous les mots = ET, `a|b` = OU, `-mot` = exclure, `"…"` =
  phrase exacte ; insensible aux accents ; sections hors du champ. Les
  textes sont normalisés UNE fois au chargement (`n.__norm`), pas à chaque
  frappe. Les résultats respectent les filtres cochés/décochés
  (`nodeMatchesFilter`) : un post-it éteint par une case n'apparaît ni dans
  le menu déroulant ni dans les résultats.
  - Tant qu'on tape sans valider, la recherche est FLOTTANTE : menu déroulant
    + lueur sur la carte, un état de passage. Un indice ↵ discret (invisible
    tant qu'il n'y a rien à valider) paraît dans la barre dès qu'il y a un
    résultat — le geste possible se montre, il ne se devine pas.
  - Entrée la fait basculer dans un état INSTALLÉ : le même volet que la fiche
    d'un post-it (« comme pour la sélection »), qui reste ouvert et VIVANT —
    retaper, changer d'axe (« Coloré par ») ou de filtre met à jour sa liste
    et sa répartition sans revalider (le recalcul vit dans `updateVisuals`,
    appelé par tout ce qui change déjà l'un ou l'autre). Il montre la
    répartition des résultats selon l'axe COURANT, leur liste complète
    (chacun coloré pareil, `colorFor`), et une case pour ÉPINGLER leurs
    étiquettes sur la carte (`state.searchPinned` — même mécanique que les
    épingles de catégorie, pour un ensemble de nœuds plutôt qu'une catégorie).
    Le zoom suit la même chorégraphie que le reste (`arriveAfterLayout` la
    reforme si on bascule cartographie ↔ constellation pendant que le volet
    est ouvert).
  - Le volet de recherche et la fiche d'un post-it PARTAGENT le même panneau :
    cliquer un résultat plonge dans sa fiche par-dessus (`mode-search`
    retiré) ; ✕/Échap n'y remonte QU'UN cran (retour à la liste), et
    seulement le second Échap referme tout — même principe que partout
    ailleurs dans l'outil, un niveau à la fois.
  - Une recherche installée est citable : `q=` dans l'URL (voir plus bas),
    restaurée après les filtres (dont elle dépend) et avant le nœud
    éventuellement sélectionné par-dessus.
- Chemins : depuis la fiche d'un nœud (« ⇢ Tracer un chemin »), BFS non
  pondéré sur TOUS les liens (connecteurs et « même texte »). L'absence de
  chemin est un résultat affiché — le graphe a plusieurs composantes — pas un
  échec silencieux. Le tracé est une COLLECTION : chaque « départ » est un
  ARBRE (une origine, une ou plusieurs arrivées) et l'on peut en tenir
  plusieurs à la fois. Deux gestes DISTINCTS : « ＋ arrivée » (sur un arbre)
  repart du MÊME point ; « ⇢ Tracer un chemin » (fiche d'un autre nœud) ouvre
  un NOUVEAU départ. Le panneau groupe chaque arbre (liseré gauche, en-tête =
  l'origine, ✕ pour le retirer) ; chaque branche a SA couleur, attribuée à la
  création et conservée (`segColors`), identique sur la carte (ligne à
  vertexColors) et dans le panneau — de quoi dire « ce chemin-ci » même quand
  plusieurs départs coexistent. La surbrillance travaille sur les sets
  COMBINÉS de toute la collection (`state.pathNodes`/`pathEnds`). Le panneau des
  chemins est ANCRÉ à la légende (positionné en JS, `positionPathPanel`) et la
  SUIT : replié, c'est une fine barre verticale SOUS elle (calée au bas de la
  fenêtre si la légende ouverte ne laisse pas la place — jamais hors-champ) ;
  déplié, il se colle à CÔTÉ, aligné en haut, avec la MÊME max-height que la
  légende (sa valeur résolue) → tous deux pleins, ils font la même hauteur,
  ascenseur au-delà. Quand la légende se déplie, le panneau se décale à droite
  pour lui laisser la place (le survoler la garde ouverte, sinon il glisserait
  sous le curseur) ; il s'adapte à la fenêtre. Déplié à la première
  construction, repliable ensuite (‹) — un armement le rouvre. Chaque branche
  se déplie en liste d'étapes cliquables. Quand un départ est ARMÉ,
  tout le dit : curseur en croix, liseré doré qui respire au bord de l'écran,
  invite dans le panneau, infobulle « ⇢ arrivée : … » — le clic suivant
  construit, il ne navigue pas.
- ⚠️ **Les arêtes se peignent par SOMMET**, depuis la couleur des deux nœuds
  qu'elles relient — et cette teinte est CUITE dans un tampon qui ne se réécrit
  que sur appel de `update()`. Tout ce qui change `colorFor` doit donc le
  rappeler : `recolorAll()` le fait. L'oubli donnait un bug intermittent —
  changer d'onglet laissait les liens aux couleurs de l'axe précédent, mais une
  bascule de disposition (qui rappelle `update()` à chaque image pour les
  positions) « réparait » l'affichage et masquait la cause.
- La fiche d'un nœud porte les **quatre** axes, pas deux : section et regard
  (traits pleins, éditoriaux), puis **degré** (« 6 liens et plus ») et
  **tissu** (« Composante géante · 395 post-its ») en TIRETÉ — la convention de
  la légende, tenue jusque dans la fiche. Les deux derniers ne manquent jamais.
- Deux repères distincts sur la carte, et il en faut deux : un anneau clair et
  fin cerne le nœud **choisi**, un **double anneau doré qui respire** le nœud
  **recentré**. Ils n'avaient pour se distinguer que leur échelle (×1,7 contre
  ×1,9) — deux sphères à peine plus grosses parmi six cents, on les perd ; et
  pendant un recentrage on peut parfaitement choisir un AUTRE nœud que le focal,
  si bien que les deux marques coexistent. Anneaux et non halos (le halo se
  confond avec le bloom), **billboardés** (sans quoi, vus par la tranche, ils
  disparaîtraient juste quand on tourne pour les chercher), et deux objets fixes
  réutilisés — rien ne se crée par clic.
- Un chemin tracé **survit au recentrage** et à sa sortie : ce n'est pas une
  géométrie mais une suite d'identifiants, et la polyligne se recalcule chaque
  frame depuis les positions courantes. Seule précaution : une branche est
  connexe, donc entièrement dans une composante — si le focal est dans une
  autre, on cesse de la tracer (ses nœuds sont au centre, invisibles : la tracer
  ferait une étoile qui ne veut rien dire) et le panneau l'estompe en le disant
  (« hors recentrage »). Voir `branchInEgo`.
- Déplier une branche la met **en avant** (`activeBranch`, accordéon) : sa
  couleur remonte au liseré, à la pastille et au nom dans le panneau, ses
  étapes gardent leur éclat sur la carte et les autres chemins reculent sans
  s'éteindre. C'est la COULEUR qui hiérarchise — `linewidth` n'a aucun effet en
  WebGL, toutes les lignes font un pixel quoi qu'on demande.
- Degré des voisins : dans la fiche, chaque connexion porte le nombre de liens
  du voisin (jeton à droite), et la liste est triée par là — les carrefours
  d'abord. On lit d'un coup vers où « creuser ».
- Les connexions d'une fiche RESPECTENT les filtres actifs — sur la carte
  (`neighborIds` dans `updateVisuals`, un voisin éteint par une case ne
  s'allume plus sous prétexte qu'il touche le nœud choisi) comme dans le
  panneau (`renderConnections`, même règle). Le titre dit « Connexions
  (8/12) » quand des voisins sont masqués, jamais « (8) » qui ferait
  disparaître les 4 autres sans le dire. Le bouton « Recentrer ici », lui,
  reste sur le compte BRUT : le recentrage ignore les filtres, un voisin
  masqué y reste un voisin. `renderConnections` est rappelée par
  `updateVisuals` (tout changement de filtre) ET par `recolorAll` (tout
  changement d'axe) — deux chemins, une seule fonction.
- Recentrage égocentrique (fiche → « ⊚ Recentrer ici ») : PAS une disposition
  de plus — une OPTION de la disposition courante. Sous cartographie : disque
  2D (couronnes par distance de graphe, angle = secteur proportionnel au poids
  du sous-arbre BFS). Sous constellation : coquilles 3D (chaque sous-arbre
  reçoit un « quartier de ciel », rectangle azimut × hauteur découpé en
  alternant les axes — v est le sinus de la latitude : à aire égale, part de
  ciel égale). Basculer d'onglet pendant un recentrage le RE-PROJETTE 2D↔3D.
  Les gestes et le cadrage suivent la disposition (plat = déplacer, vue de
  face ; 3D = tourner, de biais) — la rotation auto d'inactivité ne concerne
  donc jamais une vue plate. Tout est DÉTERMINISTE (aucun aléatoire, tri par
  libellé) : même nœud, même figure. Hors-composante = hors du monde (au
  centre, invisible, bornes intactes) ; la barre dit combien. Rayon total
  borné (la géante s'étire sur 25 sauts, mesurés par le build). Profondeur réglable (− / ＋) ; un
  chemin tracé ou le voisinage du nœud sélectionné restent entiers. TOUTE
  métamorphose de disposition suit la même chorégraphie en trois temps
  (`choreographTransition`) : recul dans l'orientation courante (lire la scène),
  métamorphose vue de loin, puis retour DE FACE et zoom en avant sur ce qui
  comptait (`arriveAfterLayout` : le nœud focal en recentrage, sinon le nœud
  sélectionné, sinon la zone en lumière, sinon la vue d'ensemble). Cela vaut
  aussi pour la bascule cartographie↔constellation, avec ou sans recentrage,
  et pour la re-projection 2D↔3D. Le retour de recentrage cadre le voisinage
  ≤ 2 sauts du focal — on ne le perd pas de vue.
  L'état se lit sous les onglets Disposition (« ⊚ recentré sur … ✕ », ligne
  tiretée) et dans le rail replié (▦⊚ / ✦⊚). La barre du recentrage tient sur
  DEUX lignes (nœud + sortie · profondeur + bilan). Elle VISE le centre de la
  fenêtre et ne se décale QUE si ce centre la ferait mordre sur un panneau
  ouvert (`positionEgoBar`) : bord gauche = ce qui est ouvert à gauche (légende
  + panneau des chemins), bord droit = le bord gauche mesuré de la fiche de
  détail (donc elle prévoit son ouverture). Tant qu'il y a la place, elle reste
  centrée ; resserrée, ses lignes s'enroulent et elle gagne en hauteur. Elle se
  tient LÉGÈREMENT au-dessus de l'aide de navigation (`#hint`) — pas de
  recouvrement. Sorties : ✕ de la ligne d'option, ✕ de la barre, Échap, ⟲.
- Aucune étoile décorative en fond : un point qui brille doit être une donnée.
- Épingles 🏷 : elles agissent depuis toutes les légendes à la fois — le pied
  de légende les compte (« 🏷 n épinglée·s · tout dépingler »), le rail replié
  aussi.
- Réseau de co-occurrences (dans ∑) : liens quand deux termes partagent
  ≥ 3 post-its, bornés aux 110 plus forts. Relaxation à pas borné qui
  refroidit, graine fixe — la même règle que la constellation, pour les
  mêmes raisons.
- L'URL mémorise le nœud ouvert (`#n=…`) : un état de lecture se cite.
- Échap remonte d'un niveau à la fois : panneau ∑, aide, chemin armé,
  chemin tracé, fiche, recherche, mise en lumière.

## Coûts par frame (ce qui a été mesuré et retiré)

- Les bornes de la scène (`currentBounds`) ne dépendent que des positions
  des nœuds ; `updateFog` les redemandait 60 fois/s. Elles sont en cache,
  invalidé uniquement pendant la transition de disposition.
- Le raycast de survol ne court que si le pointeur OU la caméra a bougé
  (`hoverDirty`) — curseur immobile sur scène immobile, zéro raycast.
- Les deux repères (choisi / recentré) coûtent trois positions et trois
  quaternions par frame : des objets FIXES qu'on déplace, jamais recréés.
- `makeEdgeRenderer.update()` réutilise deux `THREE.Color` de travail : il
  tourne à chaque image pendant une transition, et y allouer deux objets par
  arête, c'étaient ~1 200 objets jetés par image pour rien.

## L'ouverture (arrivée sur le site)

Trois temps : le titre seul sur le noir · la carte qui **s'allume en vague**
depuis son centre pendant que la caméra **recule** · l'UI qui monte en cascade
(la carte d'abord, les outils ensuite). Une seule fois — jamais lors d'une
bascule de disposition ni d'un recentrage (`intro.done`).

La vague vit **dans le canvas**, pas dans le DOM : elle touche 618 nœuds, et
618 transitions CSS parallèles ne tiennent pas 60 i/s (mobile compris). C'est
un facteur multiplicatif calculé en une passe dans la boucle de rendu — le
retard de chaque nœud est proportionnel à sa distance au centre, c'est ce qui
en fait une vague et non un fondu. Le facteur **multiplie** l'opacité que
`updateVisuals` vient de décider : filtres, mise en lumière et sélection
restent maîtres, l'intro ne fait que les révéler.

C'est le seul moment où `updateVisuals` tourne à chaque image (≈ 3 s).

L'UI part masquée par `body.booting` : sans cela elle serait à l'écran pendant
que le voile la cache, et surgirait d'un bloc à sa levée — une transition ne
peut pas partir d'un état qui n'a jamais existé. Les retards de la cascade sont
en CSS (`transition-delay`), pas dans une file de `setTimeout`.

Trois sorties : un geste quelconque **abrège** (clic, touche, molette — poser
l'état final, n'escamoter aucun état), `prefers-reduced-motion` et un lien
profond `#n=…` la **sautent** (on est venu pour un nœud, pas pour l'ouverture),
et un `setTimeout` de sécurité la termine si une image se perdait — une mise en
scène ne doit jamais retenir le site en otage.
- Le tracé du chemin est une polyligne recalculée chaque frame : quelques
  dizaines de points, et elle suit la transition sans cas particulier.

## État du corpus (à connaître avant d'en tirer des statistiques)

- 605 items retenus (+ 14 nœuds de section) · 451 connecteurs tracés à la
  main · 143 fils « même texte » · 13 items de légende écartés
- 564 post-its de CONTENU · 41 nœuds d'appareil (titres, questions,
  présentations) — visibles sur la carte, écartés des statistiques
- la « Vue d'ensemble » (195 nœuds, un tiers) est le CANEVAS DE DÉPART, non un
  résumé : ses 100 premiers post-its précèdent de cinq jours le moindre bloc, et
  sur 129 énoncés présents des deux côtés, 125 sont plus anciens dans le cadre.
  Ses couleurs disent des sous-thèmes (167/195 classés), jamais des regards
- **~54 % du hors-cadre n'a aucun regard** attribué (et le cadre n'en a pas,
  par construction)
- 89 composantes : un continent de 395 nœuds (65 %), quelques archipels
  (32, 25, 23…), et 61 nœuds isolés (degré 0)
- 28 références sur 26 nœuds seulement
- texte médian : 50 caractères
- **le temps** (605/605 datés, du 23 oct. 2025 au 16 févr. 2026) : 57 nœuds
  au canevas (9 %), **397 aux deux seuls jours de l'atelier** (66 %), 65 aux
  reprises de nov.–janv. (11 %), 86 à l'enquête de févr. (14 %)
