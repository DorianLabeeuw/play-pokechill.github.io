# Guide du contenu de Pokechill

Ce guide explique comment le jeu est organisé et comment y ajouter du contenu
**sans casser une sauvegarde existante**. Il décrit l'état du jeu en v5.0
(dernier commit upstream : « zcrystal team restriction »).

---

## 1. Vue d'ensemble

Pokechill est un jeu **100 % statique** (HTML + JavaScript, sans build ni
framework). Tout tourne dans le navigateur ; la sauvegarde vit dans le
`localStorage`.

- `index.html` contient toute l'interface (menus, panneaux) et charge les
  scripts dans un ordre précis.
- `styles.css` contient tout le style.
- `img/` contient les sprites et les illustrations.
- `scripts/` contient la logique et les données.

### Ordre de chargement des scripts (important)

Les fichiers partagent des variables globales, donc l'ordre compte : un
dictionnaire ne peut référencer que ce qui est chargé **avant** lui.

```
HackTimer.js, fuse.js           utilitaires (timers en arrière-plan, recherche floue)
moveDictionary.js               ability{} puis move{}
itemDictionary.js               item{}
pkmnDictionary.js               pkmn{} et ribbon{}
areasDictionary.js              field{}, skill{}, areas{}, season{}
PR/*.js                         contributions externes (auto-team, movesets, défis…)
script.js                       saved{}, migrations de version, thèmes, réglages
teams.js, explore.js            combat, exploration, menus, boucle de jeu
shop.js                         shop{}, ingredient{} (curry)
dictionarySearch.js, tooltip.js, decor.js
save.js                         sauvegarde / chargement / export / import
```

Le démarrage réel se fait dans `explore.js`, dans
`window.addEventListener('load', …)` : `loadGame()`, puis les rotations
(`getSeed()`), la saison, puis `updateGameVersion()` (migrations).

### Volume de contenu actuel (mesuré au chargement)

| Dictionnaire | Fichier | Nombre |
|---|---|---|
| `pkmn` | `pkmnDictionary.js` | 1 376 Pokémon (formes, mégas, Gmax et « clones » compris) |
| `move` | `moveDictionary.js` | 416 attaques |
| `ability` | `moveDictionary.js` | 195 talents |
| `item` | `itemDictionary.js` | 503 objets (held, key, decor, CT, mémoires) |
| `areas` | `areasDictionary.js` | 311 zones / dresseurs |
| `shop` | `shop.js` | 208 entrées de boutique |

---

## 2. Carte des systèmes de jeu

Toutes les « destinations » du jeu sont des entrées de `areas`, distinguées par
leur champ `type` :

| `type` | Nombre | Rôle | Rotation |
|---|---|---|---|
| `wild` | 48 | Zones sauvages, capture + drops | 12 rotations, change toutes les 12 h |
| `dungeon` | 21 | Donjons à objets (pierres, CT…) | 3 rotations, toutes les 12 h |
| `event` | 110 | Événements, raids, boss à tickets | 6 rotations, tous les 3 jours |
| `vs` | 40 | Échelle de dresseurs VS (progression principale) | aucune |
| `frontier` | 74 | Battle Frontier (équipes aléatoires par ligue et type) | 3 (ou 4 après Lusamine), tous les 3 jours |
| `dimension` | 8 | Failles et raids dimensionnels (méga) | 1 |
| `season` | 2 | Contenu saisonnier (Halloween) | selon les dates de `season{}` |

Zones spéciales sans `type` : `wildlifePark` (parc), `training` (entraînement),
`frontierBattleFactory`, et `customChallenge` (créée par `PR/challenges.js`).

Les rotations sont calculées dans `getSeed()` (`explore.js`) à partir de
l'heure UTC. Les constantes `rotationWildMax`, `rotationDungeonMax`,
`rotationEventMax`, `rotationFrontierMax` et `rotationDimensionMax` sont dans
`areasDictionary.js`.

### Progression principale

- **VS** : les dresseurs `type: "vs"` forment une échelle. Seul le premier non
  battu, **dans l'ordre de déclaration du fichier**, est jouable. Le dernier
  actuel est `vsPokemonProfessorOak` (niveau 150). Ajouter un dresseur après lui
  crée automatiquement un nouveau palier.
- **Déblocages** : les zones `wild` de niveau 30, 50 et 70 se débloquent avec
  Brock, Misty et Phoebe (boucle en fin de `areasDictionary.js`). Ailleurs, on
  utilise `unlockRequirement` et `unlockDescription`.
- **Menus verrouillés** : `switchMenu()` (`explore.js`) bloque la boutique
  avant Brock, les dimensions avant Brendan, etc.

### Autres systèmes

Battle Tower (« spiraling »), Battle Factory, génétique (héritage d'attaques et
d'IV), entraînement, curry (ingrédients), Wonder Trade, Mystery Gift,
Pokérus, starsigns, décorations, rubans, tags, défis personnalisés
(`PR/challenges*.js`), thèmes de couleur.

### Comment un Pokémon devient obtenable

`dictionarySearch.js` calcule `tagObtainedIn` pour chaque Pokémon : `wild`,
`event`, `park`, `frontier`, `mart` (boutique), `arceus`, sinon
`unobtainable` si aucun membre de sa famille n'est obtenable. Un Pokémon ajouté
au dictionnaire sans être placé dans une zone, une récompense ou la boutique
n'est donc **pas obtenable**. Le Wonder Trade ne pioche que parmi
`wild`/`frontier`/`park`.

Le taux de shiny de base est `1/400` (dans `givePkmn()` et pour les œufs).

---

## 3. Le système de sauvegarde (`scripts/save.js`)

### Format

Une seule clé `localStorage` : **`gameData`**, un objet JSON **plat** :

```js
{
  saved: { ... },   // état global (zone courante, version, réglages, compteurs…)
  team:  { ... },   // équipe active
  "<idObjet>":   { got, newItem },
  "<idBoutique>": { stock },
  "<idZone>":    { defeated, hpPercentage, ... },
  "<idPokemon>": { caught, level, shiny, ivs, moves, movepool, ribbons, ... }
}
```

Objets, entrées de boutique, zones et Pokémon partagent **le même espace de
clés**. Aujourd'hui aucun identifiant n'est en double entre ces quatre
dictionnaires, et il faut que ça reste vrai.

### Ce qui est sauvegardé

- `saved` et `team` en entier.
- Objets : `got`, `newItem`.
- Boutique : `stock`.
- Zones : `defeated`, `hpPercentage`, plus quelques champs pour les types
  `event`, `frontier`, `dimension`, `training`, `wildlifePark`.
- Pokémon : `caught`, `level`, `exp`, `ivs`, `moves`, `movepool`, `ability`,
  `shiny`, `ribbons`, `pokerus`, `nature`, `starsign`, `decor`, `tag`,
  `nickname`… (liste complète dans `saveGame()`).

Tout le reste (stats, types, spawns, récompenses, textes) vient du code, donc
**modifier une définition s'applique aussi aux sauvegardes existantes**.

### Quand ça sauvegarde

Automatiquement toutes les minutes, 5 s après le chargement, et quand on appuie
sur la touche `S`. Rien n'est sauvegardé tant que `saved.firstTimePlaying` est
vrai.

### Migrations de version

`updateGameVersion()` (`script.js`) contient des blocs
`if (saved.version < X) { … }` exécutés une seule fois, puis fixe
`saved.version = 5.0`. Sur une **nouvelle partie**, `saved.version` vaut
`undefined` et `undefined < X` est faux : aucun bloc ne s'exécute.

---

## 4. Règles d'or pour rester compatible avec une sauvegarde

1. **Ne jamais renommer ni supprimer un identifiant existant** (`pkmn.xxx`,
   `item.xxx`, `areas.xxx`, `shop.xxx`, `move.xxx`, `ability.xxx`).
   La sauvegarde est indexée par ces noms : un renommage fait perdre les
   captures, les shinies ou les objets associés, et une équipe ou un
   `saved.currentArea` qui pointe vers un id disparu peut casser le
   chargement. Pour changer un nom affiché, utiliser `rename:`.
2. **Ajouter, c'est sans risque.** Un nouveau Pokémon, objet, zone ou entrée de
   boutique absent de la sauvegarde garde ses valeurs par défaut (non capturé,
   0 exemplaire, non battu).
3. **Identifiants uniques entre dictionnaires.** Ne pas créer d'objet, de
   zone, d'entrée de boutique ou de Pokémon portant le même id qu'un élément d'un
   autre dictionnaire, ni nommé `saved` ou `team`.
4. **Nouveau champ dans `saved`** : `loadGame()` remplace tout l'objet
   `saved` par celui de la sauvegarde, donc une valeur par défaut posée en haut
   d'un script (`saved.monChamp = 0`) **disparaît** pour un joueur existant.
   Il faut l'initialiser **après** le chargement, comme le fait déjà le code :
   ```js
   saved.monChamp ??= 0   // dans le bloc 'load' de explore.js, après updateGameVersion()
   ```
5. **Nouveau champ sauvegardé sur les Pokémon, objets ou zones** : l'ajouter à
   la fois dans `saveGame()` et `loadGame()`, et au chargement ne l'écraser
   que s'il existe (`if (data[i].monChamp !== undefined) …`). Sinon les
   anciennes sauvegardes mettent `undefined` à la place de la valeur par
   défaut.
6. **Migrations** : pour une conversion ponctuelle (donner une compensation,
   convertir un ancien format), ajouter un bloc `if (saved.version < 5.1)`
   et monter la version finale. Il faut aussi penser au cas d'une nouvelle
   partie, où le bloc ne tourne pas.
7. **Toujours exporter sa sauvegarde** (Settings, « Export Data ») avant de tester une
   nouvelle version.

---

## 5. Recettes : ajouter du contenu

### Un Pokémon

Dans `pkmnDictionary.js`, avant la boucle `for (const i in pkmn)` de fin de
fichier :

```js
pkmn.monPokemon = {
    type: ["fire", "fairy"],
    bst: { hp: 70, atk: 60, def: 70, satk: 110, sdef: 90, spe: 80 },  // stats réelles, converties en note 1-6
    evolve: function() { return { 1: { pkmn: pkmn.saFormeEvoluee, level: evolutionLevel2 } } },
    hiddenAbility: ability.blaze,
    signature: move.mysticalFire,   // optionnel
    // rename: `Mon-Pokémon`,       // nom affiché si différent de l'id
    // hidden: true,                // hors Pokédex « normal »
}
```

- Les sprites sont résolus par l'id : `img/pkmn/sprite/<id>.png` et
  `img/pkmn/shiny/<id>.png`, en respectant la casse (GitHub Pages y est
  sensible).
- L'évolution peut se faire par niveau (`level`) ou par objet (`item`).
- Ensuite, **le rendre obtenable** : spawn dans une zone, `reward` d'un
  dresseur ou d'un raid, ou boutique.

### Une zone sauvage, un donjon ou un événement

Dans `areasDictionary.js`, en copiant une zone du même type :

```js
areas.maNouvelleZone = {
    rotation: 3,                 // à quelle rotation elle apparaît
    level: wildAreaLevel4,       // 10/30/50/70/90
    type: `wild`,                // ou `dungeon`, `event`
    background: `volcano`,       // img/bg/<nom>.png
    icon: pkmn.monPokemon,
    spawns: { common: [...], uncommon: [...], rare: [...] },
    drops:  { common: [item.mysteryEgg], rare: wildRareItemsFire },
    // unlockRequirement: function() { return areas.vsGymLeaderBrock.defeated },
    // unlockDescription: `Battre Brock pour débloquer`,
}
```

Pour ajouter une rotation complète, augmenter `rotationWildMax`,
`rotationDungeonMax` ou `rotationEventMax`. Attention : cela décale le
calendrier des rotations pour tout le monde, ce qui est sans danger pour la
sauvegarde.

### Un dresseur VS (nouveau palier de progression)

Ajouter un `areas.vsXxx` **après** `vsPokemonProfessorOak` :

```js
areas.vsMonDresseur = {
    name: `Champion Xxx`,
    background: `gym`,
    sprite: `oak`,               // img/trainers/<sprite>.png
    trainer: true,
    type: `vs`,
    level: 160,
    difficulty: 30,              // optionnel
    team: {
        slot1: pkmn.xxx, slot1Moves: [move.a.id, move.b.id, move.c.id, move.d.id],
        // ... jusqu'à slot6
    },
    fieldEffect: [field.noMercy.id],                       // optionnel
    reward: [pkmn.xxx],                                    // Pokémon donnés
    itemReward: { 1: { item: item.goldenBottleCap.id, amount: 10 } },
    // encounterEffect: function() { ... }                 // message ou déblocage
}
```

### Un objet

Dans `itemDictionary.js` : `item.monObjet = { type: "held" | "key" | "decor", info: function() { return `…` } }`,
plus l'image `img/items/monObjet.png`. Les CT (suffixe `Tm`) et les mémoires
(suffixe `Memory`) sont générées automatiquement d'après leur nom.

### Une entrée de boutique

Dans `shop.js` : `shop.shopMonObjet = { icon: item.monObjet.id, price: 10, currency: item.goldenBottleCap.id, category: `goods` }`.
`stock` limite la quantité ; `effect` permet un achat personnalisé ; une clé
`pkmn` vend un Pokémon.

### Une attaque ou un talent

Dans `moveDictionary.js` : `move.xxx = { moveset: [types…], split, type, power, rarity, info, hitEffect… }`
et `ability.xxx = { type: [...], rarity, info }`. Les bonus de talents
(Technicien, Sans Limite…) sont calculés automatiquement en fin de fichier.
Il faut ensuite brancher l'effet dans le combat (`explore.js`) si le
mécanisme est nouveau.

### Un défi

Dans `PR/challengesDictionary.js`, ajouter un objet à `mainChallenges` avec
les équipes au format texte de Showdown (`[Player]` / `[Enemy]`).

---

## 6. Tester en local

```bash
python3 -m http.server 8000
# puis ouvrir http://localhost:8000
```

Dans la console du navigateur, on peut inspecter `pkmn`, `areas`, `saved`,
etc. Des fonctions de debug existent : `debugGetPkmn(niveau, "shiny")` et
`debugGetItems()`. **Ne les utiliser que sur une sauvegarde de test**, car
elles modifient la sauvegarde.

Astuce : on peut importer sa vraie sauvegarde (exportée en JSON) dans un
profil de navigateur séparé pour tester sans risque.

---

## 7. Points d'attention propres au fork

- **La sauvegarde dépend du domaine.** Le `localStorage` est lié à l'adresse
  du site. Une partie jouée sur le site d'origine n'apparaît pas
  automatiquement sur le fork (par exemple sur GitHub Pages) : il faut
  l'exporter puis l'importer (Settings, « Export Data » ou « Data to Text »).
- **Vérification de mise à jour.** `PR/updateCheck.js` interroge le dépôt
  d'origine `play-pokechill/play-pokechill.github.io`, et non le fork. Comme
  l'upstream n'évolue plus, le bandeau « nouvelle version » ne s'affichera pas
  pour les nouveautés du fork. À rediriger vers le fork si besoin.
- **Sprites manquants.** 30 identifiants n'ont pas de sprite normal et 64
  n'ont pas de sprite shiny (formes d'Arceus et de Genesect, quelques mégas
  récentes, Palafin Héros…). Certains sont gérés par du code spécial
  (`arceusCheck()`). À vérifier avant de rendre ces formes obtenables.
- **Version.** Le jeu affiche `v5.0`. Pour le contenu du fork, il est proposé
  de continuer en 5.1, 5.2, etc., en ajoutant un bloc de migration seulement
  quand c'est nécessaire.
