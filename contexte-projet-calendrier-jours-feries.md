# Contexte projet — Calendrier universel des jours fériés

## Objectif métier
Outil pour un chef de projet dans l'audiovisuel (sous-titrage, doublage, distribution
multilingue) : visualiser en un coup d'œil quels pays partenaires sont fériés à une
date donnée, pour planifier des livraisons sans mauvaise surprise.

## Dépôt / déploiement
- Repo GitHub : `jdumonceautpt/calendar`
- Fichier unique : `index.html` à la racine (branche `main`)
- Hébergement : GitHub Pages ("Deploy from a branch", `main` / `/root`)
- URL publique : https://jdumonceautpt.github.io/calendar/
- Mode de travail : les évolutions sont poussées **directement sur `main`**, pas
  par pull request. Contrepartie : tester dans un navigateur (Console **et**
  Network) *avant* de pousser, `main` étant la branche déployée.
- Attention : deux comptes GitHub existent sur la machine. `jdumonceautpt` est le
  propriétaire ; `Jouw82` n'a qu'un accès en lecture. Vérifier `gh auth status`
  avant toute écriture.

## Stack technique (fichier HTML autonome, aucun build)
- React 18 + ReactDOM, chargés en UMD depuis unpkg.com (pas de bundler)
- Babel Standalone (unpkg) pour transformer le JSX **dans le navigateur**
- Icônes : SVG faits main (pas de dépendance lucide-react, pour rester 100% CDN-free
  côté build)
- Données jours fériés : API publique **Nager.Date** — `https://date.nager.at/api/v3/`
  (`AvailableCountries`, `PublicHolidays/{year}/{countryCode}`), gratuite, sans clé
- Wikipédia : `en.wikipedia.org/w/api.php` (`prop=langlinks`, `origin=*`), sans clé,
  pour retrouver l'article traduit d'un jour férié et son titre dans la langue choisie
- Drapeaux : images depuis `flagcdn.com/{code}.svg` (fallback texte si l'image échoue)
- Noms de pays, formats de date et libellés de jours dérivés de la langue choisie
  via `Intl.DisplayNames` / `Intl.DateTimeFormat` (plus aucun tableau de jours codé
  en dur)
- Police : IBM Plex Sans / IBM Plex Mono (Google Fonts)

## Design
- Vues : Jour / Semaine / Mois / Année (Mois par défaut)
- Semaine commençant le lundi
- Un pays = une couleur (pastille), palette de ~14 couleurs distinctes
- Vue Mois/Année : pastilles colorées uniquement (pas de texte), détail au clic
  dans un panneau latéral (drawer)
- Vue Jour : "verdict" textuel du type *"2 pays fériés sur 4 affichés"*
- Case du jour actuel : fond vert très clair (`--today-bg`), numéro sur pastille d'accent
- Case d'un jour férié : fond rouge très clair (`--holiday-bg`)
  (le vert "aujourd'hui" prend le pas visuellement si les deux coïncident)
- Sélection de pays en colonne de gauche : recherche, coche, suppression rapide
- **Favoris** : étoile sur chaque pays, persistée en `localStorage`
  (clé `jf-favorites`), fait remonter le pays en tête de liste
- **Sélecteur de langue** dans l'en-tête : menu sur mesure, pas un `<select>`.
  Un select natif ne peut pas porter d'image et Windows ne dispose d'aucune
  police d'emojis drapeaux (`🇫🇷` s'y affiche « FR ») — d'où un bouton + menu en
  `position: fixed`, avec les drapeaux `flagcdn` et fermeture au clic extérieur
  et à Échap. Les six drapeaux sont préchargés au montage, sinon le menu
  s'ouvre sur des carrés vides.
- **Sélection par défaut : le pays de référence de la langue active**
  (`LANG_COUNTRY` : fr→FR, en→GB, es→ES, de→DE, it→IT, pt→PT), mis en favori
  *et* sélectionné. Remplace l'ancienne sélection `['FR']` codée en dur. Un
  garde par référence n'applique la règle qu'au changement de langue, pour que
  le retrait manuel du pays ne soit pas annulé au rendu suivant.

## Thèmes (déployés le 2026-09-16, commit `773ee19`)
Trois états : **système**, **clair**, **sombre**, mémorisés en `localStorage`
(clé `jf-theme`). L'état « système » *retire* l'attribut `data-theme` pour
laisser `prefers-color-scheme` décider ; les deux autres le posent sur `<html>`.
Le thème sombre a donc deux déclencheurs — une media query et l'attribut — dont
les blocs de jetons sont volontairement dupliqués : les factoriser ferait
prendre le pas à l'un des deux au mauvais moment.

**Toutes les couleurs passent par des jetons CSS sur `:root`.** C'était le
préalable au mode sombre : une douzaine de valeurs étaient codées en dur hors
de `.jf` (cellules hors mois, bandeau d'erreur, ombre du volet latéral, liseré
des drapeaux, anneau des pastilles). Ne pas réintroduire de couleur littérale
ailleurs que dans les blocs de jetons — seule exception légitime : `PALETTE`,
qui est de la donnée (l'identité colorée d'un pays), pas du thème.

**Rôles texte et bordure séparés.** L'ancienne variable `--faint` servait à la
fois de couleur de bordure et de couleur de texte (placeholder de recherche,
libellés de section, codes pays) à **2,6:1 sur blanc**, sous le seuil de 4,5:1.
Il y a maintenant `--text-3` pour le texte et `--glyph` pour les traits et
icônes. Contrastes mesurés après correction : texte 14,3:1, secondaire 7,5:1,
tertiaire 5,7:1, placeholder sur champ 5,1:1, accent 8,6:1, blanc sur accent
5,65:1. **Ne jamais utiliser `--glyph` pour du texte.**

**Week-ends : le thème sombre va dans l'autre sens.** Jetons dédiés
`--weekend` / `--weekend-out` / `--weekend-hover`. En thème clair le week-end
est plus foncé que le jour ouvré (ratio 1,28, écart de clarté 9,7 points L*).
En thème sombre il est plus **clair** : assombrir n'offre pas de marge quand le
fond est déjà à L* 11 — un premier essai en plus foncé ne gagnait que
1,098 → 1,118, contre 1,098 → **1,366** en éclaircissant. Ne pas « harmoniser »
les deux directions, ce serait revenir à un écart invisible. Conséquence de ces
fonds plus marqués : le numéro des jours hors mois et l'indicateur « +N »
utilisent `--text-2` et non `--text-3`, sinon ils tombent sous 4,5:1.

**États des cellules et spécificité.** Les règles `.jf-cell` et `.jf-mday`
passent par `:where()` pour aplatir la spécificité : l'ordre d'écriture seul
décide, et « aujourd'hui » prime sur « férié », qui prime sur « week-end ».
Sans cela, `.jf-cell.wknd.out` (trois classes) battait `.jf-cell.holiday`
(deux classes) et un jour férié tombant un week-end hors mois s'affichait en
gris. Ne pas réintroduire de sélecteurs composés ici.

## Adaptation aux écrans
L'en-tête est une **grille en trois colonnes** (`1fr auto 1fr`) : le bloc central
— sélection de vue, libellé de période, bouton « aujourd'hui » — est ainsi centré
sur la fenêtre et non repoussé au milieu par la largeur de ses voisins (mesuré à
640 px sur 1280, écart nul). Bascule de thème puis sélecteur de langue à droite,
dans cet ordre.

Points de rupture : 1180 (année sur 3 colonnes), 1100 (le bloc central descend
sur sa propre rangée), 980 (sous-titre masqué), 880 (colonne latérale en tiroir),
720 (téléphone), 520 (navigation et sélection de vue empilées, cette dernière en
pleine largeur), plus un bloc `pointer: coarse` qui agrandit les cibles tactiles.

**Les rangées de grille sont épinglées explicitement** (`grid-row`) sous 1100 px.
Le placement automatique traite les éléments dans l'ordre du DOM : le bloc central,
qui occupe toute la largeur, poussait le bloc droit à une quatrième rangée et
faisait passer l'en-tête de 132 à 171 px sur téléphone.

À 375 px, l'en-tête fait **132 px** de haut (contre ~350 avant réorganisation)
et la grille occupe le reste (778 px sur 812). Deux arbitrages qui l'expliquent,
à ne pas défaire sans mesurer : le libellé de langue cède la place au seul
drapeau, et le groupe de trois boutons de thème devient **un bouton cyclique**
(`.jf-themecycle`) — sinon la sélection de vue partait à la ligne. Le volet
latéral passe en plein écran et la vue Semaine en colonne unique.

## Internationalisation (déployée le 2026-09-15, commit `d333008`)
Six langues : `fr`, `en`, `es`, `de`, `it`, `pt`. Langue mémorisée en
`localStorage` (clé `jf-lang`), devinée depuis `navigator.languages` au premier
chargement, repli sur `fr`.

**Noms des jours fériés — cascade.** L'API Nager.Date ne fournit que `localName`
(langue du pays) et `name` (anglais) ; aucune traduction n'existe côté source. Donc :
1. **langue choisie** — `localName` si le pays parle cette langue (table
   `LANG_HOME`), sinon le dictionnaire embarqué `HOL_RAW` (85 entrées, ~75 % des
   2 666 jours fériés de 2026), sinon le titre de l'article Wikipédia traduit
2. **anglais** — le champ `name`, renseigné sur 2 666 entrées sur 2 666
3. **langue d'origine** — `localName`, de toute façon affiché en information
   secondaire dès qu'il diffère du nom retenu

**Lien Wikipédia** par jour férié (volet latéral et vue Jour), même cascade :
article dans la langue choisie via les interlangues, sinon article anglais
*explicitement étiqueté comme tel*, sinon lien de recherche. Réponses mises en
cache dans `localStorage` (clé `jf-wiki`), absences comprises, plafond 600 entrées.

**`LANG_HOME` est volontairement restrictive** — vérifié contre l'API : la Suisse
renvoie `localName` en allemand, la Belgique en néerlandais, le Luxembourg en
luxembourgeois, et l'Afrique francophone (SN, CI, ML, CM, GA) en anglais. Les
inscrire dans la liste `fr` afficherait la mauvaise langue. Ne pas « compléter »
cette table sans vérifier pays par pays ce que renvoie réellement l'API.

**La langue coche son pays, elle ne l'étoile jamais.** Les favoris
n'appartiennent qu'à l'utilisateur : aucun mécanisme automatique n'y touche.

`jf-auto-select` retient les pays cochés *par la langue*, et eux seuls sont
décochés lors d'une bascule. **Toute action de l'utilisateur sur un pays le sort
de cet ensemble** — l'étoile comme la case à cocher, dans les deux sens : à
partir de là il reste en place quelle que soit la langue, coché ou non.

Garde-fou à ne pas retirer : un pays **déjà étoilé ou coché à la main ne devient
pas automatique** quand on passe dans sa langue (variable `dejaManuel`). Sans
lui, étoiler l'Espagne, passer en espagnol puis en allemand la décochait, alors
qu'elle avait été choisie explicitement.

Comportement attendu, vérifié : avec France, Pologne et Espagne étoilées et
cochées, passer en espagnol, en anglais puis en italien laisse les trois cochées
et favorites à chaque étape ; le pays de la nouvelle langue s'ajoute, celui de
l'ancienne ne se décoche que s'il n'avait été posé que par la langue.

**Migration au démarrage** (`readFavorites`) : une version antérieure étoilait
d'office le pays de la langue et notait ces codes dans `jf-favorites-auto`. Au
premier chargement, ces favoris sont retirés et l'ancienne clé supprimée, sinon
des étoiles non désirées survivaient. Ne pas retirer ce code sans être certain
que plus aucun navigateur ne porte l'ancienne clé.

**Clés `localStorage` utilisées** : `jf-favorites`, `jf-auto-select`,
`jf-lang`, `jf-wiki`, `jf-theme`. Ancienne clé migrée puis supprimée :
`jf-favorites-auto`.

## Bugs déjà rencontrés et corrigés (ne pas les réintroduire)

1. **Page blanche sans erreur console** → en réalité une erreur *était* présente
   mais masquée par un filtre Console oublié. Toujours vérifier Console **et**
   Network avant de conclure à un problème réseau/CDN.

2. **`Uncaught SyntaxError: Cannot use import statement outside a module`**
   Cause : Babel Standalone utilise par défaut le *runtime automatique* de
   preset-react, qui injecte un `import` ES module — incompatible avec un
   `<script>` classique (non-module) chargeant React en UMD/global.
   Correctif appliqué : le bloc JSX n'est plus dans un `<script type="text/babel">`
   auto-scanné par Babel, mais dans `<script type="text/babel-src" id="app-src">`
   (inerte). Un petit script de bootstrap le récupère et le transforme
   explicitement avec `Babel.transform(src, { presets: [['react', { runtime: 'classic' }]] })`
   avant de l'injecter et l'exécuter. **Ne pas revenir à `data-presets="react"` seul.**

3. **Nom du pays sélectionné vide dans toutes les vues.**
   `i18n.country()` cherchait un champ `label` dans l'état `countries`, alors
   que ce champ n'est calculé que dans le mémo `localized`. La fonction
   renvoyait `undefined` : le nom disparaissait des lignes sélectionnées du
   volet, des cartes de jours fériés, de la vue Semaine et des pastilles
   « Ouvrés » — sans aucune erreur console. Leçon : deux structures dérivées
   du même état n'ont pas les mêmes champs, vérifier laquelle on interroge.

4. **Liseré coloré de 3 px** sur les cartes de la vue Semaine : remplacé par
   une pastille, cohérente avec les vues Mois et Année. Le détecteur de la
   skill de design le signale comme marqueur d'interface générée, et la règle
   plafonne ce type de bordure à 1 px.

## Détection des jours fériés reportés
Badge « Reporté » et note donnant la date habituelle, dans la carte de détail
(vue Jour et volet latéral). **L'API ne signale rien** : mesuré sur les 2 666
entrées de 2026, le champ `fixed` vaut `false` partout, `types` n'en dit rien,
et seules trois entrées portent une annotation dans leur nom — le 3 juillet
américain s'appelle simplement « Independence Day ». Le report est donc calculé.

Règle retenue : la date habituelle est la **(mois, jour) majoritaire sur cinq
années** (`OBS_SPAN = 2`), exigée dans au moins trois d'entre elles. Un report
est retenu quand le jour chômé tombe un vendredi ou un lundi, que la date
habituelle tombe un week-end, et que l'écart ne dépasse pas trois jours.

**Ne pas revenir à une comparaison avec une seule année voisine.** Cette
première version, mesurée sur 30 pays, signalait **90 reports sur 447** jours
fériés, contre **12** pour la version retenue. Les faux positifs venaient des
fêtes du type « troisième lundi de janvier », qui se déplacent de quelques jours
chaque année et dont l'année voisine tombe souvent un week-end : Martin Luther
King Day, Labor Day, les anniversaires régionaux néo-zélandais, le Coming of Age
Day japonais. Le mode sur cinq ans les écarte, ces fêtes n'ayant aucune date
stable. Contre-exemple vérifié : Noël 2026 aux États-Unis tombe un vendredi sans
être reporté et ne déclenche rien.

Limite connue : pour une fête quasi fixe calée sur un terme solaire — Ching Ming
à Hong Kong — le mode peut désigner une date habituelle décalée d'un jour. Le
report lui-même est correctement identifié, c'est la date citée qui peut être
approximative.

Les années de référence ne sont demandées qu'à **l'ouverture d'un détail de
journée** (quatre appels par pays, mis en cache pour la session) : la grille
reste à un appel par pays et par année visible.

## Comportement qui ressemble à un bug mais n'en est pas un
Le 4 juillet 2026 (Independence Day, US) tombe un samedi. La règle fédérale
américaine (5 U.S.C. § 6103) reporte le jour chômé au vendredi précédent quand le
jour férié tombe un samedi. L'API Nager.Date renvoie donc le **3 juillet** comme
jour férié, ce qui est correct pour la planification (bureaux fermés ce jour-là).
Aucune correction nécessaire ; envisagé mais non fait : un badge "Reporté" pour
distinguer visuellement ces cas.

## Limites connues / pistes non implémentées
- Le week-end est fixé samedi-dimanche pour tous les pays affichés. Ne gère pas
  les pays à week-end vendredi-samedi (Golfe, Israël) — impacterait le "verdict"
  ouvré/fériés si on travaille avec ces zones.
- La traduction des noms de jours fériés n'est pas complète et ne peut pas l'être :
  739 noms anglais distincts, dont une longue traîne strictement locale (« Day of
  Andalucía », « Coming of Age Day ») sans équivalent ni article dans la langue
  cible. Environ 10 % des noms restent affichés en anglais — explicitement, jamais
  vides ni inventés. Pour améliorer la couverture : ajouter des entrées à
  `HOL_RAW`, c'est le seul levier fiable.
- Les favoris sont locaux au navigateur/appareil (pas de synchronisation compte).
- La **sélection** de pays n'étant pas persistée, seuls les pays de la langue
  sont cochés au démarrage d'une session : le scénario « mes trois favoris
  restent cochés » ne vaut qu'à l'intérieur d'une session. Persister la
  sélection lèverait cette limite.
- La sélection de pays elle-même (hors favoris) n'est pas persistée entre sessions.
- Le fichier dépend de plusieurs domaines externes (unpkg.com ×3, flagcdn.com,
  fonts.googleapis.com, date.nager.at, en.wikipedia.org) — à surveiller si déployé
  derrière un pare-feu d'entreprise filtrant.
- Idées évoquées mais pas construites : export `.ics`/Outlook des jours fériés,
  bouton "prochain jour ouvré commun à tous les pays sélectionnés", affichage des
  ponts (l'API Nager.Date expose un endpoint `LongWeekend`).

## Fichier de référence
La dernière version fonctionnelle complète est celle déployée sur GitHub Pages
à l'URL ci-dessus (`index.html`, ~2 150 lignes, HTML+CSS+JS en un seul fichier).

## Reste à faire
Aucun chantier en cours. Idées évoquées, jamais construites : export `.ics`,
bouton « prochain jour ouvré commun », ponts via l'endpoint `LongWeekend`,
persistance de la sélection de pays entre sessions.


Deux relectures qui demandent un œil humain : les libellés d'interface en
es/de/it/pt, écrits sans relecture native, et les noms de jours fériés qui
restent en anglais sur les pays réellement utilisés — chaque ajout au
dictionnaire `HOL_RAW` est ciblé et rapide.
