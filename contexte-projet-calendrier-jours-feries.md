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

## Stack technique (fichier HTML autonome, aucun build)
- React 18 + ReactDOM, chargés en UMD depuis unpkg.com (pas de bundler)
- Babel Standalone (unpkg) pour transformer le JSX **dans le navigateur**
- Icônes : SVG faits main (pas de dépendance lucide-react, pour rester 100% CDN-free
  côté build)
- Données jours fériés : API publique **Nager.Date** — `https://date.nager.at/api/v3/`
  (`AvailableCountries`, `PublicHolidays/{year}/{countryCode}`), gratuite, sans clé
- Drapeaux : images depuis `flagcdn.com/{code}.svg` (fallback texte si l'image échoue)
- Noms de pays traduits en français via `Intl.DisplayNames(['fr'], {type:'region'})`
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
- Sélection par défaut au premier chargement : France uniquement (`['FR']`)

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
- Les favoris sont locaux au navigateur/appareil (pas de synchronisation compte).
- La sélection de pays elle-même (hors favoris) n'est pas persistée entre sessions.
- Le fichier dépend de trois CDN externes (unpkg.com ×3, flagcdn.com, fonts.googleapis.com,
  date.nager.at) — à surveiller si déployé derrière un pare-feu d'entreprise filtrant.
- Idées évoquées mais pas construites : export `.ics`/Outlook des jours fériés,
  bouton "prochain jour ouvré commun à tous les pays sélectionnés", affichage des
  ponts (l'API Nager.Date expose un endpoint `LongWeekend`).

## Fichier de référence
La dernière version fonctionnelle complète est celle déployée sur GitHub Pages
à l'URL ci-dessus (`index.html`, ~780 lignes, HTML+CSS+JS en un seul fichier).
