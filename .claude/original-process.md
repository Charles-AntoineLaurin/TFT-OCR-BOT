# Fonctionnement du bot original (code dans `legacy/`)

Ce document résume le fonctionnement du code legacy, analysé avant le début de la
réécriture/modification du projet. Il sert de référence pour ne pas avoir à
re-découvrir ces mécanismes plus tard.

## 1. Reconnaissance des éléments à l'écran

Le bot n'utilise **aucune vision par ordinateur avancée** (pas de template matching,
pas de détection d'objets, pas d'images de référence stockées dans le repo — aucun
fichier `.png`/`.jpg` n'est présent). Tout repose sur deux mécanismes simples :

### a) OCR sur des zones d'écran fixes (`legacy/ocr.py`)

- `screen_coords.py` définit des coordonnées **codées en dur pour 1920x1080** via les
  classes `Vec2`/`Vec4`, qui appliquent un offset basé sur la position de la fenêtre
  League détectée au démarrage (`legacy/game.py:40-60`, via `win32gui`).
- `ocr.get_text()` (`legacy/ocr.py:41`) fait : capture d'écran (`ImageGrab.grab`) →
  resize ×3 → niveaux de gris → seuillage Otsu (`cv2.threshold`) → passage à Tesseract
  (`tesserocr`) avec une whitelist de caractères et un mode de segmentation de page
  (`psm`).
- Utilisé pour : le nombre de rounds, l'or, les noms de champions du shop, le texte des
  tooltips d'objets (au survol), le message des enclumes ("ChooseOne"/"Feelinglucky"),
  le texte des augments, "VICTORY"/"EXIT NOW".
- Le texte OCR brut est ensuite **fuzzy-matché** (`difflib.SequenceMatcher`) contre les
  listes connues de champions/objets (`arena_functions.py:57-69, 121-131`) pour
  corriger les erreurs de reconnaissance.

### b) Détection de couleur de pixel (pas de l'OCR)

- Pour savoir si un slot du banc est occupé (`arena_functions.bench_occupied_check`,
  `legacy/arena_functions.py:109`) ou trouver un slot vide (`empty_slot`), le bot
  capture une petite zone sous chaque portrait et cherche une **couleur exacte**
  `[0, 255, 18]` (la barre de vie verte), via un produit de convolution `numpy` pour
  confirmer une série de pixels verts.

### Cas particuliers notables

- Les "orbes"/enclumes ne sont pas détectées visuellement — le bot presse "E" sur
  chaque slot vide du banc puis lit le **texte du popup** qui apparaît (`ChooseOne`)
  pour savoir s'il doit choisir (`legacy/arena.py:215-230`).
- Le type de round (PvE/PvP/carousel/encounter) au tour X-1 est déterminé en survolant
  les icônes de round en haut de l'écran et en OCR-isant le tooltip, puis en cherchant
  des mots-clés ("Carousel", "Krugs", etc.) — `legacy/game_functions.py:33-53`.
- **Vie et niveau ne viennent pas de l'OCR** : ils sont lus via l'API locale officielle
  de Riot (`https://127.0.0.1:2999/liveclientdata/allgamedata`), donc ce ne sont pas
  des captures d'écran (`legacy/arena_functions.py:17-40`).

## 2. Comment le bot connaît les champions disponibles

C'est **entièrement codé en dur pour le set en cours au moment du développement**,
aucune récupération dynamique (pas d'appel à une API Riot Data Dragon, pas de fichier
de données externe) :

- `legacy/game_assets.py:43-103` — dict `CHAMPIONS` : nom → coût en or, taille du
  board, et jusqu'à 3 traits, saisi manuellement.
- `legacy/game_assets.py:140-185` — dict `FULL_ITEMS` : recettes d'objets (combinaison
  de 2 composants de base), aussi en dur.
- `legacy/comps.py` — le fichier destiné à l'utilisateur : `COMP` (composition cible
  avec position sur le board), `AUGMENTS`/`AVOID_AUGMENTS` (listes de texte à
  matcher), tout ça change à **chaque set** et parfois à chaque patch d'augments.

Conséquence pratique : un changement de set TFT nécessite de réécrire `CHAMPIONS`,
`FULL_ITEMS`, `COMP`, `AUGMENTS` (et potentiellement des coordonnées d'écran si l'UI
change). Les commits récents ("Update for set11...") montrent que c'est déjà la source
principale de maintenance du projet. Il n'y a aucune validation de cohérence — si un
nom de champion dans `comps.py` ne correspond pas exactement à une clé dans
`CHAMPIONS`, ça plante avec un `KeyError` à l'exécution plutôt qu'à l'initialisation.

## 3. Tester sur des images statiques

Une partie du code est **déjà découplée** de la capture d'écran live :

- `ocr.get_text_from_image(image, whitelist)` (`legacy/ocr.py:56`) prend directement un
  objet `PIL.Image` en paramètre. On peut donc dès maintenant faire :

  ```python
  from PIL import Image
  import ocr
  ocr.get_text_from_image(Image.open("fixtures/shop_champ1.png"), whitelist=ocr.ALPHABET_WHITELIST)
  ```

  sans avoir besoin du jeu ouvert.

- En revanche, `ocr.get_text(screenxy, scale, psm, whitelist)` (`legacy/ocr.py:41`) et
  les fonctions de détection couleur (`bench_occupied_check`, `empty_slot` dans
  `arena_functions.py`) appellent **directement** `ImageGrab.grab(bbox=...)` —
  impossible de leur injecter une image sans soit :
  1. **monkeypatcher** `PIL.ImageGrab.grab` dans les tests (solution rapide, pas de
     changement de code prod), ou
  2. **refactorer** pour séparer "obtenir l'image" de "traiter l'image" (comme c'est
     déjà fait pour `get_text_from_image`), ce qui rendrait tout testable proprement et
     permettrait aussi, plus tard, de remplacer Tesseract par autre chose si besoin.

### Recommandation

Constituer un petit dossier de fixtures (captures réelles : barre de shop, compteur
d'or, popup d'enclume, indicateur de round, tooltip d'objet) et écrire des tests
`pytest` qui passent ces images à travers `get_text_from_image`/une version refactorée
de `get_text`. Ça donnerait un filet de sécurité pour valider l'OCR à chaque
changement de set/patch, sans dépendre d'une partie en cours — et ce serait un bon
prérequis avant de retravailler l'OCR ou les coordonnées.
