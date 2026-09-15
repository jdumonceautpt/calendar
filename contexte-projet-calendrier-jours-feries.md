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
- Case du jour actuel : fond vert très clair (`--today-bg: #E9F6EC`)
- Case d'un jour férié : fond rouge très clair (`--holiday-bg: #FBEAEA`)
  (le vert "aujourd'hui" prend le pas visuellement si les deux coïncident)
- Sélection de pays en colonne de gauche : recherche, coche, suppression rapide
- **Favoris** : étoile sur chaque pays, persistée en `localStorage`
  (clé `jf-favorites`), fait remonter le pays en tête de liste
- **Sélecteur de langue** dans l'en-tête, à droite de la navigation
- Sélection par défaut au premier chargement : France uniquement (`['FR']`)

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

**Clés `localStorage` utilisées** : `jf-favorites`, `jf-lang`, `jf-wiki`.

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
- La sélection de pays elle-même (hors favoris) n'est pas persistée entre sessions.
- Le fichier dépend de plusieurs domaines externes (unpkg.com ×3, flagcdn.com,
  fonts.googleapis.com, date.nager.at, en.wikipedia.org) — à surveiller si déployé
  derrière un pare-feu d'entreprise filtrant.
- Idées évoquées mais pas construites : export `.ics`/Outlook des jours fériés,
  bouton "prochain jour ouvré commun à tous les pays sélectionnés", affichage des
  ponts (l'API Nager.Date expose un endpoint `LongWeekend`).

## Fichier de référence
La dernière version fonctionnelle complète est celle déployée sur GitHub Pages
à l'URL ci-dessus (`index.html`, ~1 360 lignes, HTML+CSS+JS en un seul fichier).

## Chantier en cours
Lot 2 des évolutions demandées, pas encore construit : **dark mode** et **refonte
visuelle** (épuré, moderne, lisible). Préalable identifié : une douzaine de
couleurs sont encore codées en dur hors des variables CSS de `.jf` (`#FAFBFC` des
cellules hors mois, couleurs du bandeau d'erreur `.jf-banner`) — à tokeniser avant
d'ajouter un thème sombre.
