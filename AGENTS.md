# AGENTS.md — contraintes de contribution

Ce fichier s'applique à **toute** contribution au dépôt, humaine ou automatique. Ce n'est pas un
guide de style : chaque règle traduit une capacité — ou une **absence** de capacité — de Telegram
Serverless ou de la Bot API. Les enfreindre ne produit pas du code moins élégant, ça produit du
code qui ne se déploie pas, ou qui détruit du contenu.

Conception complète : [`doc/classement-des-messages.md`](doc/classement-des-messages.md)
(obligatoire) et [`doc/verification.md`](doc/verification.md) (optionnel). Les `§n` ci-dessous
renvoient au premier, sauf mention contraire.

**Règle zéro.** Toute affirmation sur la plateforme doit être adossée à une ligne de
<https://core.telegram.org/bots/serverless> ou de <https://core.telegram.org/bots/api>. Une
capacité non documentée est traitée comme **absente**. Si une décision dépend d'un point non
documenté, elle va en §24 « Décisions ouvertes » — elle n'est pas devinée.

---

## 1. Cible d'exécution : Telegram Serverless, exclusivement

- Le seul environnement d'exécution est le bac à sable V8 de Telegram Serverless, déployé par
  `tgcloud`. Pas de VPS, pas de conteneur, pas de fonction cloud tierce, pas de processus local.
- N'introduire **aucune** dépendance à un runtime Node : pas de `process`, pas de `Buffer`
  (la doc l'exclut explicitement — utiliser `Uint8Array`), pas de `require`.
- Le CLI `tgcloud` s'exécute, lui, sur la machine de l'opérateur sous Node 18+. Ce qui vaut pour
  le CLI ne vaut pas pour le code déployé : ne pas confondre les deux mondes.

## 2. JavaScript seul — ni npm ni système de fichiers au runtime

- **Aucun paquet npm dans le code déployé.** Le runtime ne voit que le SDK et les modules du
  projet. Tout code utile — y compris le hachage SHA-256 (§4.4) et la normalisation de texte
  (§7.2) — est écrit en JavaScript pur dans `lib/`.
- **Aucun système de fichiers.** Pas de fichier de configuration lu à l'exécution : toute
  configuration est un **module JS versionné et déployé** (`lib/vocabulaire.js`,
  `lib/evaluation.js`, `lib/bornes.js`).
- **Aucune étape de compilation.** Pas de TypeScript, pas de bundler, pas de transpileur : ce
  qui est dans le dépôt est ce qui s'exécute.
- **Globales autorisées** : le cœur ECMAScript (`Object`, `Array`, `String`, `Number`, `Math`,
  `JSON`, `Map`, `Set`, `RegExp`, `Date`, `Promise`, `Uint8Array`, `Error`) et `console`.
- **Globales interdites**, faute d'être documentées par la plateforme : `crypto`,
  `crypto.subtle`, `TextEncoder` / `TextDecoder`, `setTimeout` / `setInterval`, `fetch` global
  (celui du SDK, importé, est le seul admis — voir §9 ci-dessous), `Buffer`, `process`.
- **Interdit aussi**, parce que le contenu Unicode/ICU embarqué n'est pas documenté :
  `String.prototype.normalize`, `Intl`, `toLocaleLowerCase` et consorts. Le repli d'accents est
  une table explicite et versionnée dans `lib/texte.js`.

## 3. Imports nus, et rien d'autre

Les seuls spécificateurs qui résolvent sont documentés :

```js
import { db, api, fetch, BotApiError } from 'sdk';
import { table, integer, text, eq, and, sql } from 'sdk/db';
import { api } from 'sdk/api';
import { fetch } from 'sdk/fetch';
import { evenements } from 'schema';
import { proposer } from 'lib/proposeur';
import { controles } from 'lib/audit/controles';
```

- **Jamais** de chemin relatif (`'./x'`, `'../x'`), **jamais** d'extension (`'lib/x.js'`) :
  la plateforme résout des noms de modules, pas des fichiers.
- `handlers/` est **plat** : un fichier par type d'update Bot API, aucun sous-dossier.
  `lib/` peut être imbriqué (`lib/audit/…`).
- Seuls `schema.js` et les `.js` sous `lib/` et `handlers/` sont déployés. `doc/`, `tests/`,
  `AGENTS.md`, `README.md` restent sur la machine.
- Ajouter un handler, c'est ajouter un type d'update auquel le bot se réveille et modifier
  `allowed_updates`. On n'en ajoute aucun sans justification écrite (§13.2).

## 4. Base : un seul `schema.js`, aucune clé étrangère

- Toutes les tables sont déclarées dans **un unique `schema.js` à la racine**, avec le DSL de
  `sdk/db`. Pas de second fichier de schéma, pas de schéma construit dynamiquement.
- **`.references()` et `foreignKey()` sont interdits** : la plateforme tourne avec
  `PRAGMA foreign_keys` **off** et le DSL lève une erreur à la déclaration. Les relations sont
  des colonnes ordinaires ; l'intégrité est maintenue en code, et les orphelins sont **signalés**
  par balayage `LEFT JOIN … WHERE parent IS NULL`, **jamais supprimés** (§14.6).
- **Toute requête est asynchrone** : `await` sur `.all()`, `.get()`, `.values()`, `.run()`.
- **Aucune transaction multi-instructions n'est documentée.** Toute transition d'état est une
  **instruction unique** en comparaison-et-échange, ou un `insert … onConflictDoUpdate …
  returning()` (§15.3). Aucune séquence lecture-puis-écriture n'est supposée atomique : la
  plateforme met le bot à l'échelle et deux updates peuvent être traitées en parallèle.
- Pas de clé primaire composite, pas de colonne dont on prévoit de changer le type (un changement
  de type est classé `manual` par `migrate`) — §14.13.

## 5. `push` et `migrate` sont deux gestes séparés

```bash
npx tgcloud status      # hors ligne
npx tgcloud diff        # hors ligne
npx tgcloud push        # déploie le code — ne touche JAMAIS la base
npx tgcloud migrate     # applique le schéma, après revue
```

- L'ordre est **toujours** `push` puis `migrate`. Ne jamais présenter un déploiement comme
  entraînant une migration : la plateforme les sépare exprès.
- **Suppression uniquement par `.deprecated('motif')`.** Retirer une déclaration de `schema.js`
  ne supprime rien.
- `--dry-run` en revue, `--safe` en routine. **`--yes` n'apparaît dans aucune procédure écrite**
  (il applique aussi les `warning`, donc les suppressions). `push --force` non plus : un conflit
  de révision se résout par `fetch` / `pull`.
- Aucun agent ne lance `push`, `migrate`, `webhook sync` ni `run` de sa propre initiative : ces
  commandes touchent un bot réel et sa base réelle. Elles sont proposées à l'opérateur.

## 6. Le bot ne produit jamais le contenu classé

C'est l'invariant central (§2, §21). Concrètement, dans le code du pipeline :

- **Le classement se fait exclusivement par `api.forwardMessage()` / `api.forwardMessages()`**,
  sur le `message_id` exact reçu, jamais recalculé ni deviné. Un album part en **un seul appel**
  `forwardMessages`, avec des identifiants **strictement croissants** (exigence documentée du
  paramètre `message_ids`).
- **Méthodes interdites**, dont l'absence est vérifiée par recherche textuelle en intégration
  continue (§18.4) :
  - `deleteMessage`, `deleteMessages` — le bot ne supprime rien ;
  - `editMessageText`, `editMessageCaption`, `editMessageMedia`, `editMessageReplyMarkup`
    appliqués à un message d'`INBOX` ou à une copie classée ;
  - `copyMessage`, `copyMessages` — une copie sans lien vers l'original est du contenu produit
    par le bot, pas un classement ;
  - toute méthode `…ForumTopic…` autre que `createForumTopic` et `editForumTopic` — en
    particulier `deleteForumTopic`, `closeForumTopic`, `unpinAllForumTopicMessages`,
    `editGeneralForumTopic` ;
  - toute manipulation d'octets de fichier : `getFile` + téléchargement et l'envoi de fichiers
    depuis un handler **ne sont pas pris en charge**. On ne manipule que des `file_id`.
- Le bot **rédige** des messages uniquement dans le sujet d'administration (propositions,
  confirmations, `/etat`, rapports d'audit), et ces messages ne recopient **jamais** le contenu
  d'un message d'`INBOX`.
- Un renommage de sujet est **proposé** par le bot et **appliqué par un humain** (§5.4). Aucune
  exception, aucun mode.

## 7. Échec fermé, toujours

- Si le `message_id` exact manque, si un album n'est pas démontrablement complet, si
  l'attribution exigible manque, si la capacité de transfert n'est pas acquise, ou si un secret
  manifeste est détecté : **le bot ne republie rien**. Il écrit un état et un dossier de preuve.
- **Trois causes d'arrêt, jamais confondues** : classification ⇒ `quarantaine` ; sécurité ⇒
  `protege` ; infrastructure (429, 5xx, réseau, droit manquant) ⇒ `bloque_infra`. Une issue
  d'appel inconnue ⇒ `incertain`, et **jamais** de reprise automatique.
- Aucun code ne « répare », ne devine, ne complète : une ligne de proposition non conforme est
  ignorée, **comptée** et journalisée avec son motif.
- Toute mutation laisse une ligne en base **avant** l'appel et une **après**. C'est la seule
  mémoire du bot : la plateforme ne documente aucun journal d'exécution consultable en
  production, et `console.*` n'est restitué que par `tgcloud run` et par BotFather.
- Les appels sont **bornés** : toutes les constantes de politique vivent dans `lib/bornes.js`,
  jamais dispersées, jamais en dur dans un handler.

## 8. Aucun secret, nulle part

- La plateforme **ne documente aucun mécanisme de secret d'exécution** : ni variable
  d'environnement pour les modules déployés, ni coffre. Il n'existe donc **aucun** code qui lise
  un secret, et aucune clé n'est écrite dans un module — un module déployé est récupérable par
  `tgcloud pull`.
- Le jeton d'API du bot n'est jamais manipulé : `sdk`/`api` est déjà authentifié.
- Le jeton CLI vit dans `.tgcloud/` (git-ignoré) ou dans `TGCLOUD_TOKEN`. Ne jamais le lire,
  l'afficher, le committer.
- La table `parametres` ne contient que des valeurs opérationnelles non secrètes, contrôlées par
  liste noire de noms de clés.
- **Un secret détecté dans un message n'est jamais recopié** — ni en base, ni dans un message, ni
  dans `console.*`. Le dossier de preuve porte le nom de la famille de motif et une position,
  jamais l'extrait.

## 9. Aucune donnée ne quitte Telegram en v1

- **Aucun appel `fetch` sortant dans le code v1**, vérifié par recherche textuelle en intégration
  continue. `fetch` est documenté et reste disponible, mais inutilisé.
- Par conséquent : **aucun LLM externe en v1**. L'extension est conditionnelle à l'apparition
  d'un mécanisme de secret d'exécution sûr, ou d'un service de classification authentifié sans
  secret embarqué — et la question de la divulgation des messages à un tiers est **séparée** de
  celle du secret ; débloquer l'une ne débloque pas l'autre (§12).
- Une contribution qui ajoute un appel sortant modifie une propriété de confidentialité du
  système : elle exige une décision explicite documentée, pas une revue de code ordinaire.

## 10. Tests : non négociables

Aucune contribution de code n'est complète sans eux.

- **Fonctions pures d'abord.** `lib/{empreinte,texte,proposeur,tags,verdict,secrets,unite}.js`
  n'importent **ni `sdk` ni `schema`** — contrôle statique. C'est la moitié du projet qui se teste
  sans plateforme : vecteurs dorés, table de décision complète, **corpus de propositions
  pathologiques** (prose, puces, ligne tronquée, slug inconnu, ligne de 10 ko, sortie vide).
- **Corpus de payloads** dans `tests/cas/*.json5`, un fichier par cas, joué par
  `npx tgcloud run handlers/<type> "$(cat …)"` avec la couture `ctx.essai` — qui n'effectue
  **aucun** appel Bot API et **aucune** écriture. `run` s'exécute sur la plateforme et il
  n'existe pas de base d'essai documentée : sans cette couture, un test écrit en production.
- **Contrôles statiques du dépôt** en intégration continue (§18.4) : imports, pureté, méthodes
  interdites, clés étrangères, `fetch`, bornes, secrets. Un contrôle qui ne casse pas quand on
  introduit volontairement la violation qu'il surveille n'est pas un contrôle : chaque contrôle a
  son test de régression.
- **Cas obligatoires** avant toute PR qui touche le transfert : rejeu ⇒ aucun second transfert ;
  album transféré en un appel et jamais découpé ; `429` ⇒ `bloque_infra` et non quarantaine ;
  interruption entre appel et confirmation ⇒ `incertain` sans reprise automatique.

## 11. Mise en service progressive : obligatoire

Aucun code de transfert ne passe directement en production.

| Mode | Transferts |
|---|---|
| `ombre` | aucun — on classe et on mesure |
| `revue` | seulement après approbation humaine par bouton |
| `canari` | automatiques pour une liste blanche de slugs |
| `actif` | automatiques |

- L'ordre `ombre` → `revue` → `canari` → `actif` n'est pas indicatif. Les seuils (`seuilScore`,
  les poids, `FENETRE_ALBUM_S`) **ne peuvent pas être fixés a priori** : ils se calibrent en
  `ombre` sur le corpus réel.
- Le retour en arrière est `/mode ombre` : immédiat, sans migration, et rien de ce qui a été
  transféré n'est défait — le bot n'annule pas, il n'a rien détruit.
- Les phases livrables (§23) sont déployables une à une ; les phases 0 à 2 ne transfèrent **aucun**
  message. Ne pas les fusionner.

## 12. Avant d'ouvrir une PR

- [ ] Aucun import relatif, aucune extension `.js`, aucun paquet npm, aucune globale hors liste.
- [ ] `handlers/` plat, uniquement des types d'update valides ; aucun handler ajouté sans motif.
- [ ] `schema.js` unique, aucun `.references(` ni `foreignKey(`, aucune suppression hors
      `.deprecated()`.
- [ ] Aucune méthode interdite du §6 ; aucun `fetch` ; aucune constante de politique hors
      `lib/bornes.js`.
- [ ] Aucun secret, aucune chaîne ressemblant à un jeton, aucun contenu de message recopié en base
      ou dans un log.
- [ ] Chaque nouveau chemin d'échec est **fermé** et laisse une trace exploitable
      (`{ regle, attendu, obtenu }`, jamais un booléen).
- [ ] Tests ajoutés — fonctions pures, cas `ctx.essai`, test de régression du contrôle statique.
- [ ] La documentation suit : un comportement décrit dans `doc/` et démenti par le code est un
      défaut au même titre qu'un test rouge.
- [ ] Toute hypothèse non adossée à la documentation officielle est inscrite en §24 « Décisions
      ouvertes », pas résolue en silence.
