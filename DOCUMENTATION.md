# UA Skill Evaluator — Documentation de fonctionnement

Application web mono-fichier (`index.html`) qui note un fichier **`SKILL.md`**
(définition de skill pour agent Claude) selon **9 critères de qualité**, sur
**45 points** (9 × 5). Tout est client-side : aucun back-end, aucune base de
données. Servie à la racine par Vercel (`https://skill-evaluator-nine.vercel.app`).

L'app propose **deux moteurs d'évaluation complémentaires** :

1. **Analyse automatique par heuristiques regex** — instantanée, hors-ligne (hors
   chargement des CDN), gratuite. Mesure la *présence de signaux* dans le texte.
2. **Évaluation par Claude (Opus 4.8)** — optionnelle, nécessite une clé API
   Anthropic. Mesure la *qualité réelle* via un vrai jugement du modèle.

Les deux scores s'affichent côte à côte ; l'utilisateur garde la main via des
sliders manuels.

---

## 1. Vue d'ensemble du flux

```
Import du SKILL.md (drag-drop / parcourir / copier-coller)
        │
        ├──► [Analyze skill]  → moteur regex : 9 scores auto + signaux
        │
        └──► [Évaluer avec Claude] → moteur LLM : 9 scores + justifications FR
        │
   Sliders (override manuel)  →  score final par critère
        │
   Résultats : score global /45, radar, verdict, points faibles, recommandations
        │
   [Export report] (Markdown copié) / [Reset]
```

---

## 2. Les 9 critères évalués

Chaque critère est noté **0 à 5**. Ci-dessous : ce que mesure le critère, et
**les signaux précis détectés par le moteur regex** (`analyze(text)`).

### 1. Grounded in real expertise (`expertise`)
Ancrage dans une expertise réelle plutôt que des généralités de LLM.
- **0** = générique, applicable à n'importe quoi · **5** = très spécifique, conscient du projet.
- Signaux **positifs** : chemins de fichiers (`` `…​.ext` ``), noms d'outils précis
  (pdfplumber, sfra, coremedia, adyen, threekit, yext, figma, contentful, shopify,
  stripe), numéros de version (`v1.2`, `1.2.3`), termes d'API/schéma (schema,
  endpoint, payload, query, mutation, env), identifiants internes (camelCase /
  ALL_CAPS).
- Signal **négatif** : tournures génériques (« best practice », « handle error »,
  « appropriately », « follow standard », « ensure proper »).

### 2. Refined with real execution (`tested`)
Skill itérée sur de vraies tâches, corrigée d'après ce qui s'est réellement passé.
- **0** = premier jet non testé · **5** = battle-tested, multiples itérations.
- Signaux : section **Gotchas** (`## gotcha / caveat / watch out / pitfall / known
  issue`), **corrections/interdictions** (do not, never, avoid, instead of, rather
  than, don't), **chemins de repli** (fallback, if…fail, on error, retry,
  otherwise), **cas limites** (edge case, special case, exception, unless, except
  when, only if). ≥ 3 corrections est valorisé.

### 3. Context efficiency (`context`)
Chaque token est justifié ; ni verbeux, ni redondant.
- **0** = verbeux, redondant · **5** = dense, juste taille.
- Signaux : **nombre de lignes** (≤ 300 excellent, ≤ 500 acceptable, > 500
  dépasse), **tokens estimés** (≈ mots × 1,3 ; ≤ 5000), **divulgation progressive**
  (`references/`, see/load/read `.md`), **phrases de remplissage** pénalisées (as
  mentioned, as noted above, as described, note that, please note, it is important).

### 4. Coherent scope (`scope`)
Couvre une unité de travail cohérente — ni trop étroite, ni trop large.
- **0** = trop étroit ou trop large · **5** = exactement une responsabilité.
- Signaux : **nombre de sections H2** (3 à 10 = bien structuré), présence d'une
  **description/déclencheur** (`>`, `description:`, `## overview / purpose / when to
  use`), **volume de mots** (200 à 5000), mots de **déclenchement** (trigger, use
  when, activate, invoke).

### 5. Calibrated control (`control`)
La prescriptivité s'adapte à la fragilité de chaque étape ; des défauts, pas des menus.
- **0** = rigidité uniforme ou vague · **5** = chaque étape calibrée à son risque.
- Signaux : **prescriptions strictes** (exactly, do not modify, run only, must use,
  required, mandatory), **souplesse** (depending on, you can, optionally, if needed,
  adapt, choose), **défauts définis** (by default, default to, recommended, prefer,
  use…instead), **commandes exactes** (blocs ```bash/sh/python/js```). Pénalité :
  menus d'options « you can use X, Y, Z » **sans** défaut.

### 6. Procedures over declarations (`procedures`)
Enseigne une méthode généralisable, pas une réponse spécifique à un cas.
- **0** = réponses spécifiques non réutilisables · **5** = méthode généralisable.
- Signaux : **étapes numérotées** (≥ 3), section **workflow** (`## workflow /
  process / approach / steps / procedure / how to`), **template de sortie** (`##
  template / output format / structure / format`), **formulations réutilisables**
  (for any, in general, when…use this, for each, across). Pénalité : réponses figées
  (the answer is, result should be, output is).

### 7. External system calls (`external`)
Appels MCP / API / web clairement nommés, avec gestion d'erreurs et repli.
- **0** = appels non nommés, sans gestion d'erreurs · **5** = outils nommés, erreurs
  gérées, replis définis.
- Signaux : **outils MCP** (`mcp__…`), **API/HTTP** (fetch, requests, axios, http,
  `https://`, endpoint, api key, bearer, authorization), **web search**,
  **ToolSearch / deferred tools**, **gestion d'erreurs** (non-200, status code, on
  error, if…fail, isError, try, catch, timeout), **repli** (fallback, if
  unavailable…), **conditions de NON-appel** (do not call, only call when…),
  **limites de débit / pagination**, **rationale de choix d'outil**, **auth /
  credentials**.
- ⚠️ Si **aucun appel externe** n'est détecté, le critère renvoie **3** (à noter
  manuellement si l'absence est intentionnelle).

### 8. Security & trust boundaries (`security`)
Pas de secrets en dur, gardes anti-injection, confirmation avant opérations destructives, PII protégées, moindre privilège.
- **0** = aucune conscience sécurité, secrets/opérations dangereuses non protégés ·
  **5** = secrets, injection, PII et opérations destructives tous gérés.
- 🔴 **Échec critique = 0** si un **secret en dur** est détecté (`sk-…`, JWT `eyJ…`,
  `ghp_…`, `xoxb-…`, `AIza…`, ou `api_key = "…"`).
- Autres signaux : **variables d'environnement / gestionnaire de secrets**
  (process.env, os.environ, `${X_KEY/TOKEN/SECRET}`, secrets., vault, keychain),
  **garde anti-injection de prompt** (treat as data, untrusted content, external
  content, prompt injection…), **opérations destructives** (delete, remove, drop,
  destroy, send, publish, deploy, overwrite, truncate, reset) **+ confirmation**
  (confirm, ask user, require approval, irreversible, cannot be undone), **protection
  PII** (redact, mask, do not log/expose PII…), **moindre privilège** (read-only,
  minimum scope, least privilege, no write), **validation d'entrée**, **frontières de
  confiance**.

### 9. Structure quality (`structure`)
Utilise des patterns efficaces : gotchas, templates, checklists, boucles de validation.
- **0** = prose plate, sans structure · **5** = gotchas, templates, checklists en place.
- Signaux : **section gotchas** (`## gotcha`), **checklist** (`- [ ]` / `- [x]`),
  **template** (```markdown, `## template`, `## output`), **boucle de validation**
  (`## validat / verif / check`), **pattern plan-valider-exécuter** (plan…then…
  execute, verify before, validate before), **blocs de code** (≥ 1).

---

## 3. Logique de notation

| Élément | Règle |
| --- | --- |
| **Score par critère** | 0–5, calculé par `analyze(text)` (regex) ou par Claude, ou fixé manuellement au slider. |
| **Score global** | Somme des 9 scores finaux, sur **/ 45**. |
| **Verdict** | ≥ **36** → 🟢 *Strong skill* · ≥ **23** → 🟡 *Needs iteration* · sinon 🔴 *Not production-ready*. |
| **Points faibles** | Critères dont le score final est **1 ou 2** (listés sous « Areas to improve »). |
| **Recommandations** | Pour chaque critère, un texte prescriptif **en français** indexé par `id` + score (0 à 5), trié des plus critiques aux meilleurs. |

**Statistiques affichées** (barre « Skill stats ») : nombre de lignes, tokens
estimés (mots × 1,3), sections H2, blocs de code, mots.

---

## 4. Évaluation par Claude (Opus 4.8)

Moteur optionnel qui complète (ne remplace pas) les regex.

- **Activation** : coller une **clé API Anthropic** (`sk-ant-…`) dans le champ dédié.
  Elle est stockée **uniquement** dans le `localStorage` du navigateur (jamais
  envoyée à un serveur tiers). Le bouton « Évaluer avec Claude » s'active dès qu'une
  clé et un contenu sont présents.
- **Appel** : requête directe navigateur → `https://api.anthropic.com/v1/messages`
  (en-tête `anthropic-dangerous-direct-browser-access`), modèle **`claude-opus-4-8`**,
  **pensée adaptative** activée, `max_tokens: 16000`.
- **Fiabilité** : la réponse est contrainte par une **sortie structurée** (JSON
  schema) — pour chacun des 9 critères : `id`, `score` (entier 0–5), `justification`
  (1–2 phrases en français, ancrée dans le contenu réel de la skill).
- **Affichage** : sous chaque critère, une carte bleue « Claude » montre le score +
  la justification, avec un bouton **« Utiliser ce score »** qui pousse la valeur
  dans le slider. Une **seconde série** sur le radar superpose Claude vs
  auto/manuel, et un total **« Score Claude / 45 »** s'affiche.
- **Erreurs gérées** : clé invalide (401), quota (429), refus du modèle, réponse
  vide/tronquée, erreurs réseau — message affiché sous le bouton, sans planter l'app.

> **Pourquoi les deux ?** Les regex mesurent le *vocabulaire* (présence de mots-clés),
> donc gonflables et sujettes aux faux positifs ; Claude *lit* réellement la skill.
> Utiliser les regex comme point de départ rapide, Claude comme jugement de fond,
> et le slider comme arbitrage humain final.

---

## 5. Mode d'emploi

1. **Importer** la skill : glisser-déposer un `.md`/`.txt`, cliquer pour parcourir,
   ou coller le contenu dans la zone de texte.
2. Cliquer **« Analyze skill »** → scores regex + signaux (✓ trouvé / ⚠ à vérifier /
   ✗ manquant) sur chaque critère (badge « Auto »).
3. *(Optionnel)* Saisir la clé API et cliquer **« Évaluer avec Claude »** → scores +
   justifications du modèle.
4. **Ajuster** au besoin via les sliders (le badge passe de « Auto » à « Manual »),
   ou cliquer « Utiliser ce score » pour adopter la note de Claude.
5. Lire le **score global**, le **radar**, le **verdict**, les **points faibles** et
   les **recommandations**.
6. **« Export report »** copie un rapport Markdown (scores, signaux, scores +
   justifications Claude, verdict) dans le presse-papier. **« Reset »** réinitialise.

**Dépendances runtime** (via CDN, connexion requise pour le rendu complet) :
Chart.js (radar) et la police Inter.

---

## 6. Architecture technique (pour développeurs)

- **Fichier unique** `index.html` (CSS + JS inline). Aucun build, lint ou test.
  Lancer en ouvrant le fichier dans un navigateur ou `python3 -m http.server`.
- **`CRITERIA`** (tableau de 9 objets) = source de vérité. Chaque entrée : `id`,
  `title`, `desc`, libellés `low`/`high`, et `analyze(text)` → `{ score, signals }`.
- **`RECOMMENDATIONS`** = objet indexé par `id` puis par score 0–5, textes en français.
- **Évaluation Claude** : `evaluateWithClaude()` (appel API + parsing),
  `applyClaudeScores()` (rendu + radar), état dans `llmScores` / `llmJustif`.
- Voir `CLAUDE.md` pour le détail d'architecture et les incohérences connues de
  dénominateurs (`/45` vs `/7` vs `/35`) si l'on ajoute/retire un critère.

---

## 7. Vérifications effectuées & déploiement

- **Test fonctionnel headless (Chromium / Playwright)** : page chargée, 9 cartes de
  critères et 9 blocs Claude rendus, activation correcte du bouton Claude (clé +
  contenu), `applyClaudeScores` peuplant le total et révélant les justifications,
  garde `if (chart)` validée. *(Seules erreurs en sandbox : CDN Chart.js/police
  bloqués hors-ligne — sans effet dans un vrai navigateur.)*
- **Schéma d'appel API** : aligné sur la documentation officielle (modèle
  `claude-opus-4-8`, sortie structurée `output_config.format`, en-tête d'accès
  navigateur direct).
- **Déploiement** : projet connecté à **Vercel**, déploiement auto depuis `main`.
  Correction du **404 à la racine** en faisant de l'app le fichier conventionnel
  **`index.html`** (servi nativement sur `/`), l'ancien chemin
  `/skill-evaluator.html` étant redirigé vers `/` via `vercel.json`.
