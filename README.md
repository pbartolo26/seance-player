# Lecteur de séance de renforcement

Page web autonome (`index.html`, HTML/CSS/JS sans dépendance ni build) ouverte comme **Telegram Mini App**
depuis un bouton de clavier (`KeyboardButton` `web_app`) du bot coach. Elle déroule une séance de renfo
(vidéos Coros, minuteurs, repos, bips) puis renvoie le compte rendu au bot via `Telegram.WebApp.sendData`.

Publication prévue : <https://pbartolo26.github.io/seance-player/> (`.nojekyll` présent).

## Entrée : la séance dans le fragment d'URL

```
index.html#s=<base64url(JSON UTF-8), sans padding>
```

```json
{ "v":1, "n":"Gainage 15 min", "a":"Alex", "d":"2026-10-26", "rb":60,
  "ex":[ { "n":"Planche", "s":3, "t":30, "rest":30, "v":["c519b9fce76e46d285a1bc3c5740266d"], "c":"Bassin aligné" },
         { "n":"Squat", "s":3, "r":12, "rest":45, "v":[] } ] }
```

| champ | sens |
|---|---|
| `n`, `a`, `d` | nom de la séance, prénom, date `AAAA-MM-JJ` |
| `rb` | repos entre deux exercices (s), 60 par défaut |
| `ex[].s` | nombre de séries |
| `ex[].t` **ou** `ex[].r` | durée d'une série (s) **ou** répétitions |
| `ex[].rest` | repos entre séries (s), 30 par défaut |
| `ex[].v` | ids vidéo Coros → `https://s3eu.coros.com/source/exercise_gif/0/<id>.mp4` (plusieurs angles) |
| `ex[].c` | consigne (facultative) |

Sans vidéo (ou vidéo en erreur) : lien « 🔎 Voir une démo » vers une recherche YouTube.

Encodage côté bot (Node) : `Buffer.from(JSON.stringify(seance)).toString('base64url')`.

## Sortie : `Telegram.WebApp.sendData`

```json
{ "v":1, "n":"Gainage 15 min", "d":"2026-10-26", "start":"2026-10-26T18:05:12",
  "dur":812, "el":905,
  "ex":[ { "n":"Planche", "done":[30,30,25] }, { "n":"Squat", "done":[12,12,10] } ],
  "rpe":6, "note":"…", "up":true }
```

- `start` : heure locale du téléphone, sans fuseau ; `dur` : secondes actives (pauses exclues) ; `el` : secondes écoulées.
- `done` : secondes tenues ou répétitions faites, par série ; exercice sauté sans série faite = `[]`
  (les séries faites avant un saut sont conservées).
- `up` : enregistrer sur Intervals.icu (case cochée par défaut).
- Le JSON tient dans 4096 octets UTF-8 : le commentaire est tronqué (avec « … ») si besoin.
- `sendData` ne fonctionne que si la mini-app est ouverte depuis un bouton de **clavier** ; le bot reçoit
  un message de service `web_app_data`.

## Tester

- Démo intégrée : `index.html#demo` (Planche 3×30 s avec 2 angles, Squat 3×12 sans vidéo, repos de 10 s).
- Hors Telegram (navigateur classique), le bouton final affiche le JSON avec un bouton « Copier ».
- En local : `npx serve .` ou `python -m http.server`, puis ouvrir `http://localhost:8000/#demo`
  (en mode appareil mobile des outils de développement).
- Générer un lien de test :
  ```
  node -e "console.log('#s='+Buffer.from(JSON.stringify(require('./ma-seance.json'))).toString('base64url'))"
  ```

Les fonctions pures (décodage, durée estimée, construction/troncature du résultat) sont dans le bloc
`<script id="core">`, sans accès au DOM : on peut l'extraire et l'exécuter sous Node pour les tester.
