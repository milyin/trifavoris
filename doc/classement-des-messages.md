# trifavoris — classement des messages

> **Composant obligatoire.** Ce document décrit le bot lui-même : l'observation du sujet `INBOX`
> d'un supergroupe forum, la décision de classement, et le **transfert** du message original
> intact vers un ou plusieurs sujets thématiques. L'audit indépendant de l'état du bot fait
> l'objet d'un document et d'un composant séparés, **optionnels** : voir
> [`verification.md`](verification.md).

---

## 1. Contexte et objectif

`trifavoris` est un bot Telegram qui observe un unique sujet de forum, `INBOX`, dans un
supergroupe à sujets. Chaque message qui y est déposé — un lien, une note, une capture, un
album de photos, un fichier — est **classé** dans un ou plusieurs sujets thématiques larges et
réutilisables (`Applications utiles`, `Développement de logiciels`, `Outils IA`, …).

Classer, ici, veut dire une chose et une seule : **transférer le message original, intégral et
inchangé, dans le sujet cible**, par `forwardMessage` / `forwardMessages`. Le bot n'écrit jamais
un message de son cru à la place du vôtre.

| Axe | Bot « classique » hébergé | `trifavoris` |
|---|---|---|
| Hébergement | VPS, conteneur, fonction cloud | **Telegram Serverless** — modules JS déployés par `tgcloud` |
| Persistance | Postgres/Redis externes | base SQLite intégrée, une par bot |
| Réécriture du contenu | résumé, reformatage, extraction | **aucune** — transfert du message original |
| Sortie du classement | un message rédigé par le bot | une copie transférée par Telegram |
| Original | souvent consommé ou supprimé | **toujours conservé dans `INBOX`** |
| Multi-appartenance | une catégorie « gagnante » | **un transfert de l'original dans chaque sujet retenu** |
| Rôle d'un LLM | il rédige et décide | **aucun en v1** ; s'il revient, il *propose*, il ne décide pas (§12) |
| Échec | message ignoré, ou message d'erreur en clair | état terminal explicite + dossier de preuve en base (§16) |
| Secrets | variables d'environnement | **aucun secret** — ni en base, ni dans le dépôt (§11.4) |

**Hors périmètre.** Toute manipulation d'octets de fichier (télécharger une pièce jointe,
réenvoyer un média construit par le bot) : la plateforme documente explicitement que
`getFile` + téléchargement et l'envoi de fichiers depuis un handler **ne sont pas encore pris en
charge**. On ne manipule que des `file_id` et des transferts. Toute génération de contenu, tout
résumé, tout découpage d'album, toute « réparation » de message : c'est le sujet du §2.

---

## 2. Ce que « non destructif » signifie ici (décision d'architecture centrale)

Le bot a une seule action mutative sur le contenu : appeler `forwardMessage` /
`forwardMessages`. Tout le reste est de la lecture, de l'écriture en base, et — sous conditions
strictes — de la gestion de sujets. Cette contrainte est le cœur de la conception, pas une
politique de rédaction :

1. **Le bot ne produit jamais le contenu classé.** Il ne réécrit pas, ne résume pas, ne
   découpe pas un album, ne recompose pas un message perdu, ne « répare » pas un lien tronqué.
   La copie déposée dans le sujet cible est produite par Telegram à partir du message original,
   pas par le code.
2. **L'original reste dans `INBOX`, toujours.** Le bot n'appelle jamais `deleteMessage`,
   `editMessageText` ni aucune méthode qui altère un message d'`INBOX`. `INBOX` est un journal
   d'entrée ; le classement en est une projection, jamais un déplacement.
3. **Multi-appartenance par duplication de l'original, pas par arbitrage.** Un message qui
   relève de trois sujets est transféré trois fois, intégralement. Il n'y a pas de « catégorie
   principale » ; il n'y a pas de lien de renvoi vers une copie unique — Telegram n'offre pas de
   référence croisée entre sujets, et un message rédigé par le bot pour tenir lieu de renvoi
   serait précisément le contenu que le bot ne doit pas produire.
4. **Frontière nette entre proposition et décision.** Une *proposition* est un texte : des
   lignes de tags, produites en v1 par des règles déterministes (§7) et, si un jour c'est
   possible, par un modèle (§12). Une *décision* est le verdict d'un évaluateur déterministe et
   versionné (§8), qui lit ces lignes et rien d'autre. Le producteur de la proposition ne touche
   ni à la base, ni à l'API Telegram. Changer de producteur ne change pas une ligne du code de
   décision.
5. **Sortie de proposition dégradable.** Le format est « une ligne = un tag ». Une sortie
   tronquée reste partiellement exploitable : les N premières lignes sont lues, les autres sont
   perdues, et l'évaluation tranche sur ce qui reste. Le parseur **ignore silencieusement toute
   ligne non conforme**, en la comptant et en journalisant son motif ; il ne répare rien, ne
   devine rien. Effet de bord utile : prose d'introduction, puces, numérotation et fuites de
   balises internes tombent d'elles-mêmes.
6. **Échec fermé.** Si le `message_id` exact manque, si un album n'est pas démontrablement
   complet, si l'attribution exigible (`forward_origin`) manque, ou si la capacité de transfert
   n'est pas acquise, **le bot ne republie rien**. L'unité concernée reste en `pending` ou part
   en quarantaine, avec son dossier de preuve (§16). Republier « au mieux » serait produire du
   contenu — cf. point 1.
7. **Trois causes d'arrêt, jamais confondues.** Échec de **classification** (l'évaluation
   refuse) ⇒ quarantaine. Protection de **sécurité** (secret manifeste, §11) ⇒ état `protege`,
   jamais de transfert automatique. Panne d'**infrastructure** (429, 5xx, réseau) ⇒
   `bloque_infra`, reprise ultérieure, **jamais** de quarantaine. Confondre les trois, c'est
   soit noyer les vrais problèmes de classement dans le bruit réseau, soit traiter une fuite de
   secret comme un incident technique.
8. **Borne dure des appels.** Chaque invocation dispose d'un budget fixe d'appels Bot API
   (§13.4). Une unité de classement produit au plus `maxSujets` transferts. Il n'existe aucun
   chemin où le bot boucle sur l'API.
9. **Traçabilité intégrale en SQLite.** Toute mutation de sujet (création, renommage,
   enregistrement, refus) et tout transfert (réclamation, résultat, identifiants source et
   destination) laissent une ligne en base avant et après l'appel. C'est la seule mémoire dont
   dispose ce bot — la plateforme ne documente aucun journal d'exécution consultable en
   production (§19).

> **Pourquoi des lignes de tags plutôt qu'une structure contrainte.** En v1 le producteur est
> du code : il pourrait rendre un objet JavaScript. Le format ligne est choisi quand même,
> parce qu'il fixe dès maintenant la seule frontière qui compte — le décideur ne consomme que
> du texte, validé par lui. Le jour où un modèle produit ces lignes (§12), aucune ligne du
> décideur ne change, et le banc de tests du parseur, écrit contre des sorties pathologiques,
> est déjà là. La contrepartie — le format n'est garanti par personne — est exactement
> l'intention : tout le jugement est concentré au seul endroit déterministe et testable.

---

## 3. Le socle : ce que Telegram Serverless documente, et ce qu'il ne documente pas

Cette conception ne s'appuie que sur des capacités écrites dans la documentation officielle
(§25). Les absences comptent autant que les présences : chacune impose une décision. Les
sections référencées expliquent la contrainte en détail.

| Besoin | Documenté ? | Conséquence de conception |
|---|---|---|
| Modules JS déployés (`handlers/`, `lib/`, `schema.js`) | oui | §13 |
| Imports nus depuis `sdk`, `sdk/db`, `sdk/api`, `sdk/fetch`, `schema`, `lib/…` | oui | aucun chemin relatif, aucune extension `.js` |
| Paquets npm au runtime | **non** | tout le code utile vit dans `lib/` — y compris le hachage (§4.4) |
| Système de fichiers | **non** | la configuration est un **module JS versionné**, pas un fichier lu à l'exécution (§4) |
| Base SQLite par bot, `db` asynchrone | oui | §14 |
| Clés étrangères | **désactivées** (`.references()` lève une erreur) | intégrité en code + balayage d'orphelins (§14.6) |
| Transaction multi-instructions | **non documentée** | toute transition d'état est **une instruction unique** en comparaison-et-échange (§15.3) |
| Exécution périodique, minuterie, tâche de fond | **non documentée** | aucune reprise autonome : la reprise est portée par les updates suivantes et par `/balayage` (§15.6) |
| Primitive de hachage, `crypto`, `TextEncoder` | **non documentées** | `lib/empreinte.js` : UTF-8 et SHA-256 en JavaScript pur (§4.4) |
| Secret d'exécution (variable d'environnement, coffre) | **non documenté** | **pas de LLM externe en v1** (§12) |
| Journaux d'exécution consultables en production | **non** — `console.*` n'est restitué que par `tgcloud run` et BotFather | observabilité en base (§19) |
| Téléchargement / envoi d'octets de fichier | **non** — « pas encore pris en charge » | uniquement `file_id` et transferts (§10) |
| `fetch` sortant | oui — **textuel**, réponse plafonnée à **32 Mo** | seul canal externe ; inutilisé en v1 |
| Toute méthode Bot API via `api.<methode>()`, résultat déjà déballé | oui | §10.2 |
| Échec Bot API ⇒ `BotApiError` (`.code`, `.description`, `.method`, `.parameters`) | oui | §15.5 |
| Concurrence d'invocations | implicite — « scales with your bot automatically » | traitée comme réelle (§15.4) |
| `push` et `migrate` séparés ; `push` ne touche jamais la base | oui | §17 |
| Contrôle de concurrence optimiste au déploiement (révision) | oui | §17.3 |
| `tgcloud run <handler> <payload> [--ctx …]` sur les fichiers **locaux** | oui | banc d'essai (§18) |
| Webhook géré par la plateforme, `allowed_updates` dérivé des handlers déployés | oui | §17.4 |

Deux absences du côté **Bot API** pèsent autant que celles de la plateforme, et sont rappelées
là où elles mordent :

- **On ne peut pas énumérer les sujets d'un forum.** Il n'existe pas de méthode qui liste les
  sujets d'un supergroupe. Le bot ne connaît donc que les sujets **qu'il a créés lui-même** ou
  **qu'un administrateur lui a explicitement enregistrés** (§5.2).
- **On ne peut pas relire l'historique d'un salon** ni récupérer un message par son
  identifiant. Il n'existe aucune méthode non mutative qui réponde à « ce message existe-t-il
  encore ? » ou « cette copie a-t-elle bien été déposée ? ». C'est la racine de l'état
  `incertain` (§15.5) et la limite structurelle de l'audit
  ([`verification.md`](verification.md) §2.2).

---

## 4. Vocabulaire des sujets et contrat de proposition

### 4.1 Le vocabulaire est un module, et il est versionné

Il n'y a pas de système de fichiers : la configuration ne peut pas être un fichier YAML lu à
l'exécution. Elle est donc un **module JavaScript déployé**, `lib/vocabulaire.js`, dont
l'unique export décrit les sujets et porte une version explicite.

```js
// lib/vocabulaire.js — autorité unique du vocabulaire. Jamais écrit par le bot.
export const VERSION = 'vocab-2026-08-16.1';

export const SUJETS = [
  {
    slug: 'applications-utiles',
    nom: 'Applications utiles',
    // Cette description est le périmètre du sujet. Elle sert de critère de revue humaine,
    // et deviendra telle quelle le fragment de prompt si le §12 est un jour débloqué.
    description:
      "Logiciels et services prêts à l'emploi que l'on installe ou utilise directement : " +
      "utilitaires de bureau, applications mobiles, extensions, services en ligne.",
    couvre: ['utilitaires', 'extensions-navigateur', 'applications-mobiles'],
    motifs: [/* cf. §7.2 */],
  },
  {
    slug: 'developpement-de-logiciels',
    nom: 'Développement de logiciels',
    description:
      "Ce qui sert à écrire, tester, déployer ou comprendre du code : langages, bibliothèques, " +
      "dépôts, outillage, articles techniques, méthodes d'ingénierie.",
    couvre: ['bibliotheques', 'outillage-ci', 'articles-techniques'],
    motifs: [/* … */],
  },
  {
    slug: 'outils-ia',
    nom: 'Outils IA',
    description:
      "Modèles, agents, API et produits fondés sur l'apprentissage automatique, ainsi que la " +
      "documentation et les retours d'expérience qui les concernent.",
    couvre: ['modeles', 'agents', 'api-llm'],
    motifs: [/* … */],
  },
];
```

Trois propriétés en découlent, et elles sont testables :

- **Le vocabulaire est la seule autorité.** Aucun code ne crée un sujet dont le `slug` n'y
  figure pas (§5.3) ; aucune écriture du bot ne modifie ce module.
- **`VERSION` est inscrite dans chaque décision** (`classements.vocabulaireVersion`). Une
  décision prise sous un ancien vocabulaire reste identifiable, et l'audit peut la signaler
  sans la modifier ([`verification.md`](verification.md) §4.2).
- **`VERSION` fait partie de la clé du cache de propositions** (§7.4) : changer le vocabulaire
  invalide mécaniquement les propositions mémorisées.

Le champ `couvre` déclare la **relation de couverture** : les thèmes plus étroits qu'un sujet
large absorbe. C'est lui qui permet de refuser une création en doublon et de motiver un
élargissement (§5.4).

### 4.2 Grammaire des lignes

Une proposition est un texte. Chaque ligne est un tag, ou est ignorée.

```text
ligne     := tag
tag       := espace ":" segment (":" segment)*
espace    := "sujet" | "score" | "motif" | "signal"
segment   := [a-z0-9-]+ ou un entier selon l'espace, non vide
```

| Espace | Forme | Cardinalité | Rôle |
|---|---|---|---|
| `sujet:` | `sujet:<slug>` | 0..3 retenus | le sujet proposé |
| `score:` | `score:<slug>:<0-100>` | exactement 1 par `sujet:` retenu | la force de la proposition |
| `motif:` | `motif:<slug>:<regle>[:<detail>]` | 0..n | la preuve : quelle règle a déclenché |
| `signal:` | `signal:<nom>` | 0..n | un fait hors classement (`signal:secret-manifeste`, `signal:sans-contenu-classable`) |

Exemple de proposition bien formée :

```text
sujet:developpement-de-logiciels
score:developpement-de-logiciels:85
motif:developpement-de-logiciels:domaine:github.com
sujet:outils-ia
score:outils-ia:70
motif:outils-ia:terme:llm
```

Normalisation avant validation : `trim`, suppression d'un `:` final, passage en minuscules
**des seuls slugs et noms de règles** (les segments de détail sont laissés tels quels et ne sont
jamais réinjectés dans un message).

**Bornes dures, non configurables** : au plus 6 segments par tag, au plus 200 octets par ligne,
au plus 32 lignes retenues, au plus 3 `sujet:` retenus. Au-delà, la ligne est ignorée comme non
conforme. Ces bornes protègent le décideur d'une entrée hostile ou dégénérée quel que soit le
producteur.

### 4.3 Un score n'est pas une probabilité

`score:<slug>:80` est un **ordinal habillé en nombre**, même produit par des règles : c'est une
somme de poids déclarés, pas une fréquence observée. Trois conséquences, identiques qu'il vienne
de règles ou d'un modèle : le traiter comme un score monotone à seuil ; l'arrondir au multiple
de 5 (la fausse précision n'apporte rien et la granularité grossière stabilise les
comparaisons) ; **calibrer le seuil sur le corpus réel** — la phase de rodage en mode `ombre`
(§20) existe pour cela, et le seuil `seuilScore` est un paramètre versionné, pas une constante.

### 4.4 `lib/empreinte.js` — l'empreinte, en JavaScript pur

Le cache de propositions (§7.4), les clés d'idempotence (§15.2) et les dossiers de preuve
(§16) ont besoin d'une empreinte stable. La plateforme ne documente **ni `crypto`, ni
`crypto.subtle`, ni `TextEncoder`**. `lib/empreinte.js` fournit donc, en JavaScript pur et sans
dépendance :

- `octetsUtf8(chaine) -> Uint8Array` — encodage UTF-8 écrit à la main depuis `codePointAt`,
  paires de substitution comprises ;
- `sha256Hex(octets) -> string` — SHA-256, implémentation compacte et testable ;
- `empreinteUnite(unite) -> string` — le hachage d'une **chaîne canonique** dont l'ordre des
  champs est figé et documenté dans le module.

`Uint8Array` est acquis : la plateforme le documente comme type de lecture/écriture des colonnes
`blob()`. `Date` l'est aussi (mode `timestamp`). Le module est couvert par des vecteurs d'essai
dorés — l'empreinte est une **fonction pure**, donc le seul endroit du projet qui se teste
exhaustivement sans toucher à Telegram (§18.2).

> **Choisir SHA-256 plutôt qu'un hachage court.** L'empreinte sert de clé primaire et de clé
> d'idempotence de transfert : une collision n'y produit pas un doublon, elle produit un
> **transfert silencieusement absent** — le mode de défaillance le plus difficile à repérer,
> puisque rien n'échoue. Le coût d'un SHA-256 en JS sur quelques kilo-octets de texte est sans
> commune mesure avec celui d'un appel Bot API.

---

## 5. Politique des sujets

### 5.1 Sujets larges, réutilisés, élargis — jamais dupliqués

La règle produit, dans l'ordre de priorité :

1. **Réutiliser** un sujet connu dont le périmètre couvre le thème du message ;
2. **Élargir** un sujet connu dont le nom est trop spécifique, en le **renommant** ;
3. **Créer** un sujet, uniquement si son `slug` figure dans `lib/vocabulaire.js` et qu'aucun
   sujet connu ne le couvre ;
4. **Proposer** — enregistrer une `proposition_sujet` avec sa preuve, et ne rien faire d'autre.

Le point 4 est le comportement par défaut de tout ce qui n'entre pas dans les points 1 à 3.
Un slug hors vocabulaire n'est **jamais** promu automatiquement : c'est le pendant exact des
« valeurs hors catalogue » du modèle documentaire dont ce projet s'inspire.

### 5.2 Comment le bot connaît un sujet

L'API Bot ne permet pas de lister les sujets d'un forum. La table `sujets` (§14.3) est donc
peuplée par exactement deux chemins, tous deux journalisés :

- **création** — `api.createForumTopic({ chat_id, name, … })` renvoie le `message_thread_id` du
  sujet créé ; la ligne est écrite avec `origine: 'cree'` ;
- **enregistrement** — un administrateur poste `/enregistrer <slug>` **à l'intérieur du sujet
  concerné**. Le handler lit `message.message_thread_id` sur son propre payload : c'est la seule
  manière documentée d'apprendre l'identifiant d'un sujet préexistant. Ligne écrite avec
  `origine: 'enregistre'`.

`INBOX` lui-même est enregistré par ce chemin, sous le slug réservé `inbox`, et son
`message_thread_id` est recopié dans `parametres`. **Tant qu'`INBOX` n'est pas enregistré, le
bot n'ingère rien** : sans identifiant d'`INBOX`, il ne peut pas distinguer une entrée d'une
copie classée, et risquerait de reclasser ses propres transferts en boucle (§6.1).

### 5.3 Créer

Conditions cumulatives, vérifiées avant l'appel :

- le `slug` figure dans `lib/vocabulaire.js` ;
- aucun sujet connu ne le `couvre` (§4.1) ;
- le bot dispose du droit `can_manage_topics` — constaté par `api.getChatMember` sur lui-même,
  mis à jour par `handlers/my_chat_member.js`, mémorisé dans `parametres` avec un horodatage ;
- le mode de déploiement autorise la mutation (`canari` ou `actif`, §20).

À défaut : aucune création, `sujets_journal` enregistre un `refus` motivé, et l'unité part en
`bloque_infra` (droit manquant — c'est une panne, pas un défaut de classement) ou en
`quarantaine` (slug hors vocabulaire — c'est un défaut de classement).

Le nom créé est `nom` tel que déclaré dans le vocabulaire, jamais une variante engendrée.

### 5.4 Élargir par renommage — et pourquoi c'est toujours une décision humaine

Un sujet nommé `Extensions Firefox` que l'on veut voir accueillir toutes les applications
utiles doit être **renommé** `Applications utiles`, pas doublé d'un nouveau sujet. Telegram
fournit `api.editForumTopic({ chat_id, message_thread_id, name })` pour cela.

Le renommage est proposé automatiquement, **appliqué jamais automatiquement** :

- le bot écrit une ligne `sujets_journal` avec `action: 'renommage'`, `avant`, `apres`, `motif`,
  et `resultat: 'propose'` ;
- il émet dans le sujet d'administration un message portant deux boutons
  (`handlers/callback_query.js`) : appliquer, refuser ;
- l'application écrit une seconde ligne `sujets_journal` avec `acteur: 'admin:<id>'` et le
  résultat de l'appel.

> **Pourquoi ne pas automatiser.** Un renommage est visible par tous les membres du groupe,
> immédiat, et sans annulation propre : rebaptiser en sens inverse laisse une trace et perturbe
> les lecteurs. Surtout, c'est la seule opération de ce bot qui **modifie quelque chose que
> l'utilisateur a créé**. Le reste du système ne fait qu'ajouter des copies. Une décision
> automatique ici contredirait le §2 dans son esprit, même si elle n'altère aucun message.

Un renommage effectué **hors bande** — un humain renomme le sujet dans le client Telegram — est
détectable : le message de service correspondant arrive dans `handlers/message.js`, dans le
champ documenté `forum_topic_edited` du `Message`. Le bot met alors `sujets.nom` à jour et
journalise `action: 'renommage'`, `acteur: 'hors-bande'`. Si l'update n'est pas reçue, la
divergence entre `sujets.nom` et la réalité n'est pas détectable, et l'audit la signale comme
non vérifiable ([`verification.md`](verification.md) §2.2).

### 5.5 Ce que le bot ne fait jamais à un sujet

Il n'appelle ni `deleteForumTopic`, ni `closeForumTopic`, ni `unpinAllForumTopicMessages`, ni
`editGeneralForumTopic`. Ces méthodes n'apparaissent nulle part dans le code ; leur absence est
vérifiable par recherche textuelle et fait partie de la recette (§18.4). Un sujet fermé
manuellement est constaté à l'échec du transfert et traité comme une panne d'infrastructure
jusqu'à décision humaine.

---

## 6. La chaîne de traitement

```text
   update « message »  ──►  handlers/message.js
            │
            ├─ hors INBOX, service, commande, émis par le bot  ──────────────►  ignore
            │
   ┌────────v─────────────┐
   │ 1. ingestion         │  ecriture idempotente dans `evenements` ; la preuve est figée
   └────────┬─────────────┘
            │
   ┌────────v─────────────┐
   │ 2. unité de          │  message seul          ─────────────────────────►  pret
   │    classement        │  media_group_id présent ──►  attente_album  (§9)
   └────────┬─────────────┘
            │ pret
   ┌────────v─────────────┐
   │ 3. admissibilité     │  message_id exact, attribution exigible, capacité
   │    (déterministe)    │  de transfert, album démontrablement complet
   └───┬────────────┬─────┘
       │ ok         │ échec  ──────────────────────────────────►  quarantaine
       │
   ┌───v──────────────────┐
   │ 4. proposition       │  lib/proposeur : règles déterministes  ──►  lignes de tags
   └───┬──────────────────┘                                            (0 appel réseau)
       │
   ┌───v──────────────────┐
   │ 5. analyse           │  lignes conformes retenues, non conformes comptées et
   │                      │  journalisées — aucune réparation
   └───┬──────────────────┘
       │
   ┌───v──────────────────┐
   │ 6. évaluation        │  seuils versionnés  ──►  Verdict
   └───┬───────┬──────┬───┘
       │retenu │refusé│ signal:secret-manifeste
       │       v      └──────────────────────────►  protege  (jamais transféré, §11)
       │  quarantaine
   ┌───v──────────────────┐
   │ 7. résolution des    │  slug ──► message_thread_id  (réutiliser / élargir / créer /
   │    sujets            │           proposer, §5)
   └───┬──────────────────┘
       │
   ┌───v──────────────────┐
   │ 8. transfert         │  une réclamation par (unité, sujet), puis
   │                      │  forwardMessage / forwardMessages
   └───┬───────┬──────────┘
       │       └─ BotApiError 429/5xx ──────────►  bloque_infra  (reprise, §15.6)
       │       └─ résultat inconnu    ──────────►  incertain     (décision humaine, §15.5)
       v
     classe
```

### 6.1 Les quatre raccourcis déterministes, décidés sans rien évaluer

- **Message hors `INBOX`** (`message_thread_id` différent de celui enregistré, ou absent — cas
  du sujet *General*) ⇒ `ignore`. C'est aussi ce qui empêche le bot de reclasser ses propres
  copies : elles arrivent dans les sujets cibles, pas dans `INBOX`.
- **Message émis par le bot lui-même** (`from.id` égal à l'identifiant rendu par `getMe`,
  mémorisé dans `parametres`) ⇒ `ignore`, sans condition. Deuxième garde-fou anti-boucle,
  indépendant du premier.
- **Message de service** (`forum_topic_created`, `forum_topic_edited`, arrivées et départs de
  membres, épinglages…) ⇒ `ignore` pour le classement ; certains alimentent `sujets_journal`
  (§5.4).
- **Commande d'administration** (`/enregistrer`, `/mode`, `/etat`, `/balayage`, `/reprendre`,
  `/liberer`, `/audit`) ⇒ routée, jamais classée, jamais transférée. Une commande postée par un
  non-administrateur est ignorée sans réponse.

Un message ignoré n'écrit rien dans `evenements` **sauf** s'il s'agit d'une commande
d'administration, auquel cas c'est `journal_appels` qui porte la trace. Ne rien écrire pour le
bruit ordinaire est délibéré : la base est petite et doit le rester.

---

## 7. Le proposeur déterministe v1

### 7.1 Ce qu'il voit

Uniquement le payload `Message` que le handler reçoit, réduit à ce que Telegram expose sans
appel supplémentaire : `text` / `caption`, `entities` / `caption_entities` (dont les URL),
`document.mime_type` et `document.file_name`, la présence de `photo`, `video`, `audio`,
`voice`, `sticker`, `poll`, `story`, `link_preview_options`, et — pour un album — la
concaténation ordonnée des légendes de ses membres.

Aucun appel réseau, aucun appel Bot API : le proposeur est une fonction pure
`proposer(unite, vocabulaire) -> string`. C'est ce qui le rend rejouable hors ligne et
testable sans plateforme.

### 7.2 Les règles

Les motifs sont déclarés **dans le vocabulaire**, à côté du sujet qu'ils servent, jamais épars
dans le code :

```js
motifs: [
  { regle: 'domaine',  poids: 45, hotes: ['github.com', 'gitlab.com', 'crates.io', 'pypi.org'] },
  { regle: 'terme',    poids: 25, termes: ['bibliothèque', 'compilateur', 'framework', 'sdk'] },
  { regle: 'mime',     poids: 20, prefixes: ['text/x-'] },
  { regle: 'entite',   poids: 10, types: ['pre', 'code'] },
],
```

Le score d'un sujet est la somme des poids des règles déclenchées, plafonnée à 100 puis arrondie
au multiple de 5. Chaque déclenchement produit une ligne `motif:`, ce qui rend le score
**explicable ligne à ligne** — c'est le contenu du dossier de preuve en cas de quarantaine.

La normalisation du texte (minuscules, repli des accents, découpage en mots) est écrite dans
`lib/texte.js`, sans dépendance : `String.prototype.normalize` n'est pas documentée par la
plateforme et n'est donc pas utilisée. Le repli d'accents est une table explicite, versionnée,
couverte par des tests dorés.

### 7.3 Ce que le proposeur n'a pas le droit de faire

Il n'importe ni `sdk`, ni `schema`. Cette interdiction est **mécaniquement vérifiable** : une
recherche d'imports dans `lib/proposeur.js` et ses dépendances fait partie de la recette
(§18.4). C'est la traduction, dans un projet sans système de compilation, de la séparation de
composants que d'autres projets obtiennent par des unités de compilation distinctes.

### 7.4 Cache adressé par empreinte

Chaque proposition est mémorisée dans `cache_propositions` (§14.9) sous la clé

```text
empreinte( contenu canonique de l'unité ) + VERSION du vocabulaire + version du proposeur
```

En v1 le proposeur est déterministe et le cache n'économise rien de coûteux : il sert à
**rejouer** — reprendre une unité après incident et obtenir exactement la même proposition, et
comparer deux versions de vocabulaire sur le même corpus. Il est en place dès la v1 précisément
pour que le §12, s'il se débloque, n'ait rien à inventer : le jour où la proposition coûte un
appel payant, la clé, la table et la logique de rejeu existent déjà et ont été éprouvées.

---

## 8. Évaluation et verdict

Entièrement déterministe, entièrement versionnée, aucun appel réseau. C'est le point de décision.

```js
// lib/evaluation.js — autorité de décision. Jamais écrite par le bot.
export const VERSION = 'eval-2026-08-16.1';

export const REGLES = {
  seuilScore: 60,          // en dessous, la proposition ne compte pas
  minSujets: 1,            // moins : échec de classification
  maxSujets: 3,            // au-delà : on garde les 3 meilleurs scores, ordre stable
  maxRatioRejet: 0.5,      // au-delà, la proposition est jugée non exploitable
  slugInconnu: 'proposer', // proposer | refuser
  enDefaut: {
    aucunSujetRetenu:  'quarantaine',
    ratioRejet:        'quarantaine',
    slugInconnu:       'quarantaine',
    signalSecret:      'protege',      // prioritaire sur tout le reste
    droitManquant:     'bloque_infra',
  },
};
```

Verdict :

```js
// { type: 'retenu',      sujets: [{ slug, score, motifs }] }
// { type: 'quarantaine', defauts: [{ regle, attendu, obtenu }] }
// { type: 'protege',     signaux: ['secret-manifeste'] }
// { type: 'bloque',      cause: 'droit-manquant' | 'rate-limit' | 'api' }
```

Chaque défaut porte `{ regle, attendu, obtenu }` — jamais un booléen. C'est ce qui remplit le
dossier de preuve et le rend actionnable.

**L'ordre d'évaluation est figé** : signaux de sécurité d'abord, qualité d'analyse ensuite,
cardinalité et seuils enfin. Un message qui porte à la fois un secret manifeste et une
proposition parfaite part en `protege` : la sécurité l'emporte, et elle l'emporte avant même
qu'on sache si le classement aurait réussi.

**Il n'y a pas d'escalade en v1** — un seul producteur de proposition, donc au plus une
proposition par unité et zéro appel réseau. L'emplacement d'une escalade est réservé (§12) et
la borne est écrite dès maintenant : **au plus un producteur de secours, donc au plus deux
propositions par unité**, quoi qu'il arrive.

---

## 9. Groupes média (albums)

### 9.1 Le problème

Telegram livre les membres d'un album comme **des updates séparées**, liées par un
`media_group_id` commun. Rien dans l'API ne dit combien de membres compte l'album. Or le §2
interdit de découper un album : soit on transfère tous ses membres ensemble, soit on ne
transfère rien.

### 9.2 L'unité de classement

L'unité est donc soit un message seul, soit un album entier :

| Cas | Clé d'unité |
|---|---|
| message seul | `<chat_id>:<message_id>` |
| album | `<chat_id>:g:<media_group_id>` |

Toutes les décisions (`classements`), tous les transferts (`transferts`) et toutes les
quarantaines portent sur une clé d'unité, jamais sur un message isolé d'album.

### 9.3 Fenêtre de silence et clôture

Chaque membre reçu met à jour la ligne `groupes_media` : `dernierVu`, `membres`, bornes
`premierMessageId`/`dernierMessageId`, et la liste des clés. Le groupe est **clos** quand
`maintenant - dernierVu >= FENETRE_ALBUM` (proposition : 8 secondes), constaté lors d'une
invocation ultérieure — pas par une minuterie, qui n'existe pas (§3).

Trois déclencheurs de clôture, tous documentés :

1. l'update suivante, quelle qu'elle soit, exécute un **balayage borné** (§15.6) qui clôt les
   groupes échus ;
2. la commande `/balayage` postée par un administrateur ;
3. `npx tgcloud run handlers/message '<payload de balayage>'` par l'opérateur.

> **Le cas honnête :** un album déposé alors que plus rien n'arrive ensuite reste en
> `attente_album` indéfiniment. Ce n'est pas un défaut d'implémentation, c'est la conséquence
> directe de l'absence de minuterie documentée. Le comportement est **sûr** — rien n'est
> transféré, l'original est intact — mais il est **visible** : `/etat` affiche les unités en
> attente et leur âge, et l'audit les signale au-delà d'un seuil
> ([`verification.md`](verification.md) §4.3). Une reprise autonome exigerait une capacité que
> la plateforme ne documente pas ; l'inventer serait exactement ce que cette conception
> s'interdit.

### 9.4 Complétude : ce qu'on peut prouver, et ce qu'on ne peut pas

On ne peut pas prouver qu'un album est complet. On peut établir un faisceau de contrôles, et
**échouer fermé** dès que l'un manque :

| Contrôle | Règle | En défaut |
|---|---|---|
| Fenêtre écoulée | `maintenant - dernierVu >= FENETRE_ALBUM` | reste `attente_album` |
| Contiguïté | les `message_id` des membres forment une suite **sans trou** | `quarantaine`, catégorie `album` |
| Taille | 2 ≤ membres ≤ 10 (limite Telegram pour un groupe média) | `quarantaine`, catégorie `album` |
| Homogénéité de salon | tous les membres ont le même `chat_id` et le même `message_thread_id` | `quarantaine` |
| Ordre | la liste transmise à `forwardMessages` est strictement croissante | corrigé par tri ; un doublon ⇒ `quarantaine` |

La contiguïté est une **heuristique**, et elle est étiquetée comme telle dans le dossier de
preuve : Telegram attribue en pratique des identifiants consécutifs aux membres d'un album,
mais rien ne le garantit contractuellement. Elle est retenue parce que son mode de défaillance
va dans le bon sens — un trou fait échouer fermé, jamais transférer partiellement.

### 9.5 Atomicité du transfert d'album

Un album est transféré par **un seul appel** `api.forwardMessages({ chat_id, message_thread_id,
from_chat_id, message_ids })`, jamais par une boucle de `forwardMessage` : c'est la seule
manière d'obtenir dans le sujet cible un album regroupé plutôt qu'une suite de médias isolés —
c'est-à-dire de ne pas découper le message (§2.1).

`forwardMessages` renvoie un tableau de `MessageId` — **pas** des `Message` complets. Les
identifiants de destination sont donc enregistrés, mais aucune propriété de la copie n'est
relisible. C'est une limite à connaître pour l'audit ([`verification.md`](verification.md) §3.2).

Si l'appel échoue partiellement — cas non documenté, mais qu'on ne peut exclure — la
réclamation passe en `incertain` (§15.5) : aucune reprise automatique, décision humaine.

---

## 10. Attribution, transfert et confidentialité

### 10.1 L'attribution exigible

Le §2.6 impose d'échouer fermé quand `forward_origin` manque. La règle exacte, parce que la
nuance décide de tout :

- **Message d'origine posté directement dans `INBOX`** : `forward_origin` est légitimement
  absent. L'attribution exigible est alors `from` (utilisateur) ou `sender_chat` (message posté
  au nom d'un salon). Si les deux manquent, ⇒ `quarantaine`, catégorie `attribution`.
- **Message qui se présente comme transféré** : `forward_origin` doit être présent et porter un
  `type` connu (`user`, `hidden_user`, `chat`, `channel`). Absent, vide ou de type inconnu ⇒
  `quarantaine`, catégorie `attribution`. Le bot ne devine pas une origine.

Le contenu de `forward_origin` est recopié tel quel dans `evenements.origineJson`. Il n'est
jamais reformulé, jamais réémis dans un message.

### 10.2 Le transfert préserve ce que Telegram accepte de préserver

`forwardMessage` produit une copie qui porte, **quand Telegram l'expose** : le texte ou la
légende et sa mise en forme, le média ou l'album, l'auteur, et la chaîne d'origine si le
message était déjà un transfert. Le bot ne fournit aucun de ces éléments : il fournit
`from_chat_id` et `message_id`, et Telegram fait le reste. C'est précisément pourquoi le
transfert est la seule opération conforme au §2.

Paramètres retenus pour l'appel :

| Paramètre | Valeur | Motif |
|---|---|---|
| `chat_id` | le supergroupe forum | — |
| `message_thread_id` | le sujet cible | c'est le classement |
| `from_chat_id` | le même supergroupe | l'original est dans `INBOX` |
| `message_id` / `message_ids` | l'identifiant exact, jamais recalculé | §2.6 |
| `disable_notification` | `true` (paramétrable) | un classement n'est pas une nouvelle ; cela n'altère pas le message |
| `protect_content` | **non transmis** | le transmettre modifierait une propriété de la copie ; décision ouverte (§24.5) |

### 10.3 Ce que le transfert ne peut pas préserver, et qu'il faut assumer

- **Auteur masqué.** Si l'auteur d'origine a restreint la mention de son compte, la copie porte
  un nom non cliquable, ou `MessageOriginHiddenUser`. Le bot n'a aucun moyen de rétablir le
  lien, et n'essaie pas.
- **Contenu protégé.** Un message avec `has_protected_content`, ou provenant d'un salon qui
  interdit l'enregistrement, ne peut pas être transféré : l'appel échoue. Le bot **teste
  `has_protected_content` avant l'appel** et échoue fermé sans consommer d'appel, et traite
  l'échec Bot API comme une seconde ligne de défense (§15.5).
- **Confidentialité du classement.** Transférer un message d'`INBOX` vers un sujet du même
  supergroupe **ne change pas l'audience** : même salon, mêmes membres. C'est ce qui rend
  l'opération acceptable par défaut. Toute évolution qui transférerait vers un autre salon
  changerait cette propriété et sortirait du périmètre de cette conception.
- **Aucune donnée ne quitte Telegram en v1.** Aucun appel `fetch` sortant n'existe dans le code
  v1. C'est vérifiable par recherche textuelle et fait partie de la recette (§18.4).

---

## 11. Sécurité et secrets

### 11.1 Ne jamais amplifier un secret manifeste

Un message qui contient un mot de passe, une clé de licence ou un jeton ne doit **pas** être
recopié ailleurs, fût-ce dans le même salon : le classement le rend durablement plus visible,
plus indexable, plus difficile à retirer. Le détecteur `lib/secrets.js` est déterministe et
versionné, et produit une ligne `signal:secret-manifeste`.

Familles détectées (motifs versionnés, chacun nommé) :

| Famille | Exemples de motif |
|---|---|
| Étiquette explicite | `mot de passe :`, `password:`, `mdp`, `clé de licence`, `license key`, `serial` |
| Jetons de service | préfixes reconnaissables de jetons d'API, jetons porteurs |
| Jetons structurés | JWT (trois segments base64url séparés par des points) |
| Blocs de clés | en-têtes de blocs PEM, clés privées OpenSSH |
| Entropie | suite de ≥ 24 caractères sans espace, entropie de Shannon élevée, hors URL connue |

Le détecteur est **volontairement biaisé vers la protection** : un faux positif coûte une revue
humaine, un faux négatif coûte une divulgation durable. Le déséquilibre est assumé et
documenté.

### 11.2 L'état `protege`

Une unité marquée `protege` : n'est **jamais** transférée automatiquement ; reste intacte dans
`INBOX` ; est comptée dans `/etat` ; et n'apparaît nulle part ailleurs.

La libération est explicite, en deux temps, et journalisée : un administrateur poste
`/liberer <cle>`, le bot répond par un bouton de confirmation
(`handlers/callback_query.js`), et seule la confirmation fait repasser l'unité en `pret`.
`journal_appels` conserve l'acteur, l'horodatage et la famille de motif qui avait déclenché la
protection. Une libération n'efface jamais la trace.

### 11.3 Le bot ne recopie jamais un secret

Ni dans un message, ni dans `console.*`, ni dans une colonne de la base. Le dossier de preuve
d'une unité `protege` contient **le nom de la famille de motif et la position dans le texte**,
jamais l'extrait. C'est possible sans perte : l'original est dans `INBOX`, un humain peut le
lire. Recopier l'extrait dans la base reviendrait à créer une seconde copie du secret — dans le
seul endroit du système qu'un audit ou un export pourrait exposer.

### 11.4 Aucun secret dans le dépôt ni dans la base

- Le jeton d'accès CLI est géré par `tgcloud login`, stocké dans `.tgcloud/`, ignoré par git ;
  en intégration continue il passe par `TGCLOUD_TOKEN`, jamais écrit sur disque. Le dépôt n'en
  contient aucune trace.
- Le jeton d'API du bot n'est jamais manipulé : `sdk`/`api` est déjà authentifié par la
  plateforme — « nothing to install, no credentials to wire up ».
- La table `parametres` (§14.11) ne contient que des valeurs opérationnelles non secrètes :
  identifiant du salon, identifiant de sujet, mode de déploiement, seuils, droits constatés.
  L'audit vérifie qu'aucune clé de `parametres` n'appartient à une liste de noms interdits
  ([`verification.md`](verification.md) §4.5).
- Il n'existe aucun code qui lise un secret depuis quoi que ce soit — puisqu'il n'existe aucun
  mécanisme documenté pour en fournir un (§12).

### 11.5 Autorisation des commandes

Une commande n'est exécutée que si son auteur est administrateur du supergroupe, constaté par
`api.getChatMember({ chat_id, user_id })` et mis en cache dans `parametres` avec un horodatage
et une durée de validité courte. Statut inconnu ou expiré ⇒ la commande est refusée sans effet.
Aucune liste d'administrateurs n'est codée en dur : la source de vérité est Telegram.

---

## 12. Extension LLM — conditionnelle, et bloquée aujourd'hui

### 12.1 Le constat, sans détour

**La documentation de Telegram Serverless ne décrit aucun mécanisme de secret d'exécution.**
Il n'y a ni variable d'environnement pour les modules déployés, ni coffre, ni stockage de
configuration chiffré. Les seuls jetons documentés sont ceux du CLI (`TGCLOUD_TOKEN`,
`.tgcloud/credentials`), qui vivent sur la machine de l'opérateur et **ne sont pas accessibles
au code déployé**.

Conséquence directe : appeler une API LLM externe depuis un handler exigerait d'écrire sa clé
dans un module `lib/`, donc **dans le dépôt et dans l'espace de modules déployé**. C'est
interdit par le §11.4, et ce n'est pas une question de discipline : un module déployé est
récupérable par `tgcloud pull` par quiconque détient le jeton CLI du projet.

**Donc : la v1 est déterministe et n'appelle aucun LLM.** Ce n'est pas un choix de prudence
esthétique, c'est la seule conception compatible avec ce que la plateforme documente.

### 12.2 Les deux conditions de déblocage

L'extension n'est envisageable que si **l'une** de ces conditions est remplie et documentée :

1. **Un mécanisme de secret d'exécution sûr apparaît dans la plateforme** — un magasin de
   secrets injecté au runtime, absent de l'espace de modules et non restituable par `pull`.
2. **Un service de classification authentifié sans secret embarqué** devient disponible : un
   service qui authentifie l'appelant autrement que par un porteur à longue durée de vie stocké
   dans le code.

Aucune des deux n'est vraie aujourd'hui, et la seconde est plus difficile qu'elle n'en a l'air :
tout schéma « sans secret » finit par exiger une preuve que l'appelant peut produire, et le
runtime ne documente aucune identité vérifiable côté serveur.

### 12.3 La question de confidentialité, indépendante de celle du secret

Même avec un mécanisme de secret parfait, envoyer le texte des messages d'`INBOX` à un
prestataire externe est une **divulgation de données**, décidée par les membres du groupe et
non par ce document. Elle exigerait au minimum : un consentement explicite, une politique de
rétention connue, et l'exclusion inconditionnelle des unités `protege`. Les deux questions —
secret et divulgation — doivent être tranchées séparément ; débloquer la première ne débloque
pas la seconde.

### 12.4 Ce que la conception prépare quand même

Rien dans la v1 ne devra être défait :

- la frontière proposition/décision existe déjà (§2.4) ;
- le format de sortie dégradable et son parseur existent déjà (§4.2), éprouvés contre des
  sorties pathologiques ;
- le cache adressé par empreinte existe déjà (§7.4), avec la version du producteur dans la clé ;
- l'emplacement d'escalade est réservé et **borné dès maintenant** : au plus deux propositions
  par unité (§8) ;
- `fetch` est documenté (textuel, ≤ 32 Mo, lecture en flux possible pour du SSE) : le canal
  technique est connu et n'a rien à inventer.

L'extension serait alors un unique module `lib/proposeur-llm.js` exportant la même signature
que `lib/proposeur.js`, sélectionné par un paramètre. Aucun autre fichier ne changerait.

---

## 13. Architecture des handlers et des modules

### 13.1 Arborescence

```text
trifavoris/
├─ handlers/                  # plat, un fichier par type d'update — jamais de sous-dossier
│  ├─ message.js              # ingestion INBOX + routage des commandes d'administration
│  ├─ callback_query.js       # confirmations : renommage, libération, reprise
│  └─ my_chat_member.js       # droits du bot dans le forum (can_manage_topics)
├─ lib/
│  ├─ empreinte.js            # UTF-8 + SHA-256 en JS pur, fonctions pures
│  ├─ texte.js                # normalisation, repli d'accents, découpage — fonctions pures
│  ├─ vocabulaire.js          # SUJETS + VERSION — autorité, jamais écrite
│  ├─ evaluation.js           # REGLES + VERSION — autorité, jamais écrite
│  ├─ proposeur.js            # règles → lignes de tags ; n'importe ni sdk ni schema
│  ├─ tags.js                 # parseur de lignes, motifs de rejet typés — fonction pure
│  ├─ verdict.js              # moteur d'évaluation → Verdict — fonction pure
│  ├─ secrets.js              # détection de secrets manifestes — fonction pure
│  ├─ unite.js                # construction de l'unité de classement (message seul / album)
│  ├─ etat.js                 # transitions par comparaison-et-échange
│  ├─ sujets.js               # politique des sujets : résoudre, créer, proposer un renommage
│  ├─ transfert.js            # réclamation + forwardMessage / forwardMessages
│  ├─ balayage.js             # travaux en retard, bornés par invocation
│  ├─ journal.js              # écritures de traçabilité
│  ├─ commandes.js            # analyse et autorisation des commandes
│  ├─ bornes.js               # toutes les constantes dures, en un seul endroit
│  └─ audit/                  # OPTIONNEL — voir verification.md
├─ schema.js                  # toutes les tables, un seul fichier à la racine
├─ tests/                     # NON DÉPLOYÉ — payloads JSON5 et script d'exécution
├─ doc/
├─ AGENTS.md
└─ README.md
```

Seuls `schema.js` et les `.js` sous `lib/` et `handlers/` sont déployés. `tests/`, `doc/`,
`AGENTS.md` et `README.md` restent sur la machine — c'est ce qui permet d'avoir un banc d'essai
volumineux sans alourdir l'espace de modules.

### 13.2 Trois handlers, et pourquoi pas quatre

La documentation est explicite : chaque handler ajouté est un type d'update de plus pour lequel
la plateforme réveille du code, et `allowed_updates` du webhook est dérivé des handlers
déployés. On n'ajoute donc que le strict nécessaire.

| Handler | Rôle | Pourquoi il existe |
|---|---|---|
| `message.js` | ingestion, commandes, balayage | c'est le seul chemin d'entrée du contenu |
| `callback_query.js` | confirmations humaines | renommage (§5.4), libération (§11.2), reprise (§16.3) |
| `my_chat_member.js` | droits du bot | sans `can_manage_topics` constaté, la création de sujet doit échouer fermé (§5.3) |

`edited_message.js` est **volontairement absent** en v1 : un transfert est un instantané, une
édition ultérieure de l'original ne modifie pas les copies, et le bot ne doit rien y faire.
L'ajouter n'apporterait qu'une trace ; c'est une décision ouverte (§24.4).

### 13.3 Discipline des modules

- **Imports nus uniquement** : `from 'sdk'`, `from 'sdk/db'`, `from 'schema'`, `from 'lib/…'`.
  Jamais `'./x'`, jamais `'lib/x.js'`.
- **Aucun paquet npm**, aucune API globale non documentée. La liste des globales autorisées est
  explicite dans `AGENTS.md` ; tout ajout doit être justifié par une ligne de la documentation
  officielle.
- **Les fonctions pures ne touchent pas au monde** : `empreinte`, `texte`, `proposeur`, `tags`,
  `verdict`, `secrets`, `unite` n'importent ni `sdk` ni `schema`. C'est la moitié testable du
  projet, et c'est le siège des invariants.
- **Toutes les bornes dures dans `lib/bornes.js`**, jamais dispersées.

### 13.4 Budget d'une invocation

```js
// lib/bornes.js
export const MAX_APPELS_API = 12;          // par invocation, toutes méthodes confondues
export const MAX_SUJETS_PAR_UNITE = 3;
export const MAX_MEMBRES_ALBUM = 10;
export const MAX_TENTATIVES_TRANSFERT = 3;
export const MAX_UNITES_BALAYEES = 5;      // par invocation, en plus de l'unité courante
export const MAX_LIGNES_PROPOSITION = 32;
export const MAX_OCTETS_LIGNE = 200;
export const FENETRE_ALBUM_S = 8;
export const TTL_RECLAMATION_S = 60;
```

Le compteur d'appels est tenu en mémoire pour l'invocation et recopié dans `journal_appels`.
Budget épuisé ⇒ l'invocation s'arrête proprement, laisse le travail restant en base, et
n'écrit **aucun** état terminal qu'elle n'a pas vérifié. L'invariant est testé (§18.3).

---

## 14. Schéma SQLite conceptuel

Toutes les tables sont déclarées dans `schema.js` avec le DSL de `sdk/db`. Aucune clé étrangère
(la plateforme les interdit et le DSL lève une erreur) : les relations sont des colonnes
ordinaires, l'intégrité est maintenue en code (§14.6).

### 14.1 `evenements` — un message observé dans `INBOX`

| Colonne | Type | Rôle |
|---|---|---|
| `cle` | `text` **PK** | `<chat_id>:<message_id>` |
| `cleUnite` | `text` | clé de l'unité de classement (§9.2) — indexée |
| `chatId`, `messageId` | `integer` | identifiants exacts, jamais recalculés |
| `threadId` | `integer` | sujet d'origine (doit être `INBOX`) |
| `mediaGroupId` | `text` | présent si l'update appartient à un album |
| `dateMessage` | `integer` (`timestamp`) | date Telegram du message |
| `auteurKind`, `auteurId` | `text`, `integer` | `from` ou `sender_chat` (§10.1) |
| `origineKind` | `text` | `aucune` ou le `type` de `forward_origin` |
| `origineJson` | `json` | `forward_origin` recopié tel quel |
| `contenuKind` | `text` | `texte`, `photo`, `video`, `document`, … |
| `protege` | `boolean` | `has_protected_content` |
| `empreinte` | `text` | empreinte du contenu canonique (§4.4) |
| `etat` | `text` | §15.1 — indexée |
| `etatMaj` | `integer` (`timestamp`) | horodatage de la dernière transition |
| `tentatives` | `integer` | reprises consommées |
| `raison` | `text` | motif court de l'état courant |

Index : `idx_evenements_etat(etat)`, `idx_evenements_unite(cleUnite)`,
`idx_evenements_groupe(mediaGroupId)`.

Le texte du message **n'est pas stocké** : il est normalisé, haché, classé, et oublié. Seule
l'empreinte subsiste. C'est cohérent avec le §11.3 et avec le fait que l'original reste
disponible dans `INBOX`. La conséquence — on ne peut pas rejouer une proposition sans relire le
message dans Telegram, ce que l'API ne permet pas — est une décision ouverte (§24.2).

### 14.2 `groupes_media` — l'agrégation d'album

`cle` (**PK**, `<chat_id>:g:<media_group_id>`), `chatId`, `mediaGroupId`, `threadId`,
`premierMessageId`, `dernierMessageId`, `membres`, `clesMembres` (`json`), `premierVu`,
`dernierVu`, `etat` (`ouvert` | `clos` | `incomplet`).

### 14.3 `sujets` — la carte des sujets connus

`slug` (**PK**), `threadId` (`integer`, index **unique**), `nom`, `nomVocabulaire`, `etat`
(`actif` | `inconnu` | `indisponible`), `origine` (`cree` | `enregistre`),
`vocabulaireVersion`, `creeLe`, `majLe`.

L'unicité de `threadId` est la garantie qu'aucun sujet Telegram ne se retrouve associé à deux
slugs — le doublon que le §5.1 interdit, exprimé comme une contrainte de base.

### 14.4 `sujets_journal` — toute mutation de sujet

`id` (**PK**, auto-incrément), `slug`, `action` (`creation` | `renommage` | `enregistrement` |
`refus` | `constat`), `avant`, `apres`, `motif`, `methode` (nom de la méthode Bot API),
`resultat` (`propose` | `ok` | `erreur` | `refuse`), `erreurCode`, `erreurDescription`,
`acteur` (`bot` | `admin:<id>` | `hors-bande`), `horodatage`.

Une ligne est écrite **avant** l'appel (`resultat: 'propose'`) et une seconde **après**
(`ok`/`erreur`). Une mutation dont seule la première ligne existe est un appel dont l'issue
est inconnue : l'audit la signale ([`verification.md`](verification.md) §4.4).

### 14.5 `classements` — la décision

`id` (**PK**, auto-incrément), `cleUnite`, `sujetSlug`, `etat` (`propose` | `retenu` |
`refuse` | `approuve`), `score`, `motifs` (`json`), `vocabulaireVersion`, `evaluationVersion`,
`decideLe`.

Index **unique** `uidx_classements(cleUnite, sujetSlug)` : une unité ne peut pas être classée
deux fois dans le même sujet. C'est l'expression en base de l'idempotence du classement.

### 14.6 `transferts` — la colonne vertébrale de l'idempotence

| Colonne | Type | Rôle |
|---|---|---|
| `cle` | `text` **PK** | `<cleUnite>\|<sujetSlug>\|<empreinteLot>` |
| `cleUnite`, `sujetSlug`, `threadId` | | cible du transfert |
| `messageIdsSource` | `json` | les identifiants **exacts** transférés |
| `messageIdsDestination` | `json` | ce que l'API a renvoyé |
| `etat` | `text` | `reclame` \| `envoye` \| `incertain` \| `echoue` \| `abandonne` |
| `proprietaire` | `text` | identifiant d'invocation |
| `reclameLe`, `expireLe`, `termineLe` | `timestamp` | fenêtre de réclamation |
| `tentatives` | `integer` | bornées par `MAX_TENTATIVES_TRANSFERT` |
| `erreurCode`, `erreurDescription` | | recopiés de `BotApiError` |

`empreinteLot` est l'empreinte de la liste ordonnée des `message_id` sources : deux tentatives
sur le même lot produisent la même clé, une composition différente d'album en produit une autre.
C'est ce qui rend la clé à la fois idempotente et sensible au contenu réel du transfert.

**Intégrité sans clés étrangères.** `classements.cleUnite`, `transferts.cleUnite` et
`evenements.cleUnite` ne sont liés par aucune contrainte : le balayage (§15.6) inclut une passe
d'orphelins en `LEFT JOIN … WHERE parent IS NULL`, bornée, dont le résultat est **signalé et non
supprimé**.

### 14.7 `quarantaine`

`cleUnite` (**PK**), `categorie` (`classification` | `attribution` | `album` | `capacite`),
`defauts` (`json` — la liste des `{ regle, attendu, obtenu }`), `preuve` (`json` — §16.2),
`creeLe`, `statut` (`ouverte` | `reprise` | `classee`), `repriseLe`, `acteur`.

### 14.8 `propositions_sujet`

`valeur` (**PK**, le slug proposé), `occurrences`, `preuve` (`json`, bornée aux N dernières clés
d'unité), `statut` (`nouvelle` | `approuvee` | `refusee`), `premierVu`, `dernierVu`.

Jamais promue automatiquement en sujet (§5.1).

### 14.9 `cache_propositions`

`empreinte` (**PK**), `producteur` (`regles-v1` | `llm:<modele>`), `vocabulaireVersion`,
`sortieBrute` (`text`), `lignesRetenues` (`json`), `lignesRejetees` (`json`), `obtenuLe`.

### 14.10 `journal_appels`

`id` (**PK**, auto-incrément), `invocation`, `methode`, `cible`, `resultat`, `code`,
`description`, `dureeMs`, `horodatage`. Rétention bornée, purge par le balayage.

### 14.11 `parametres`

`cle` (**PK**), `valeur` (`text`), `majLe`, `acteur`. Clés attendues : `chatId`,
`inboxThreadId`, `adminThreadId`, `botId`, `mode`, `seuilScore`, `fenetreAlbum`,
`droitsConstates`, `droitsConstatesLe`. **Aucune clé secrète** — contrôlé par liste noire
(§11.4).

### 14.12 `compteurs`

`cle` (**PK**, p. ex. `2026-08-16:transferts`), `valeur` (`integer`). Alimente `/etat` (§19).

### 14.13 Discipline de schéma

- Pas de clé primaire composite : toutes les clés sont des `text` construites par le code, ce
  qui évite de dépendre d'une capacité que la documentation n'illustre pas.
- Pas de colonne dont on prévoit de changer le type : un changement de type est classé
  **manual** par `migrate` et doit être fait à la main (§17.2). Les identifiants Telegram sont
  `integer`, les structures sont `json`, tout le reste est `text`.
- `.deprecated('motif')` est le seul chemin de suppression (§17.2).

---

## 15. Machine à états, idempotence et concurrence

### 15.1 Les états

```text
                 ┌──────────┐
                 │  ignore  │  (§6.1 — rien n'est écrit)
                 └──────────┘

  recu ──► attente_album ──► pret ──► analyse ──┬──► retenu ──► transfert ──► classe
             │                  ▲               │                  │
             │                  │               ├──► quarantaine   ├──► bloque_infra ──┐
             │                  │               │        │         │                   │
             │                  └── reprise ────┘        │         └──► incertain      │
             │                                            │                 │           │
             └──► quarantaine (album incomplet)           │                 │           │
                                                          │        décision humaine     │
                          protege ◄────── signal secret ──┘                 │           │
                             │                                              v           │
                             └── libération explicite ──► pret        classe / retenu ◄─┘
```

| État | Signification | Sortie |
|---|---|---|
| `recu` | ingéré, preuve figée | `attente_album` ou `pret` |
| `attente_album` | album en cours d'agrégation | `pret` (clos) ou `quarantaine` (incomplet) |
| `pret` | unité complète et admissible | `analyse` |
| `analyse` | proposition produite et parsée | `retenu`, `quarantaine` ou `protege` |
| `retenu` | sujets résolus, transferts à faire | `transfert` |
| `transfert` | au moins une réclamation en cours | `classe`, `bloque_infra`, `incertain` |
| `classe` | **terminal** — tous les transferts confirmés | — |
| `quarantaine` | **terminal jusqu'à reprise** — échec de classification | `pret` sur `/reprendre` |
| `protege` | **terminal jusqu'à libération** — sécurité | `pret` sur libération confirmée |
| `bloque_infra` | panne — reprise automatique bornée | `retenu` |
| `incertain` | issue d'appel inconnue — **jamais** de reprise automatique | décision humaine |

### 15.2 Idempotence

Trois points d'idempotence, tous portés par une contrainte de base et non par du code :

1. **Ingestion** — `evenements.cle` est primaire. Une update rejouée par le webhook réécrit la
   même ligne sans jamais régresser l'état : la mise à jour ne touche que `vuLe`.
2. **Décision** — `uidx_classements(cleUnite, sujetSlug)` interdit la double décision.
3. **Transfert** — `transferts.cle` est primaire et déterministe. Deux invocations qui décident
   le même transfert produisent la même clé ; une seule obtient la réclamation.

### 15.3 Transitions : une instruction, toujours

**La plateforme ne documente aucune transaction multi-instructions.** Toute transition est donc
une **comparaison-et-échange en une seule instruction** :

```js
// transition sûre : n'aboutit que si l'état est encore celui qu'on a lu
await db.update(evenements)
  .set({ etat: 'analyse', etatMaj: new Date() })
  .where(and(eq(evenements.cle, cle), eq(evenements.etat, 'pret')))
  .run();
```

L'invocation doit ensuite **relire** la ligne pour savoir si elle a gagné la transition. Si
`.returning()` est disponible sur `update` (le constructeur de requêtes suit Drizzle, où il
l'est ; la documentation ne l'illustre que sur `insert`), la relecture disparaît — c'est une
décision ouverte (§24.1), et le code est écrit pour que seule `lib/etat.js` change.

La réclamation d'un transfert utilise le même principe sur `insert … onConflictDoUpdate …
returning()`, qui **est** documenté : la ligne n'est réattribuée que si la réclamation
précédente a expiré.

```js
const [ligne] = await db.insert(transferts)
  .values({ cle, cleUnite, sujetSlug, etat: 'reclame', proprietaire: moi,
            reclameLe: maintenant, expireLe: expiration })
  .onConflictDoUpdate({
    target: transferts.cle,
    set: {
      proprietaire: sql`case when ${transferts.etat} = 'reclame'
                             and ${transferts.expireLe} < ${maintenant}
                        then ${moi} else ${transferts.proprietaire} end`,
      // … reclameLe / expireLe suivent la même condition
    },
  })
  .returning()
  .run();

const jeLaTiens = ligne.proprietaire === moi && ligne.etat === 'reclame';
```

### 15.4 Concurrence

La plateforme met le bot à l'échelle automatiquement : **deux updates peuvent être traitées en
parallèle**, y compris deux membres d'un même album. La conception le suppose vrai partout :

- aucune séquence lecture-puis-écriture n'est supposée atomique ;
- aucun compteur n'est incrémenté par lecture puis écriture — toujours par `sql`
  (`set: { valeur: sql\`${compteurs.valeur} + 1\` }`) ;
- le balayage ne prend que des travaux qu'il a **réclamés** ;
- un test de concurrence fait partie de la recette (§18.3).

### 15.5 L'état `incertain` — la limite honnête

Une réclamation est écrite **avant** l'appel `forwardMessage(s)` et confirmée **après**. Si
l'invocation disparaît entre les deux, la ligne reste `reclame` et son issue réelle est
inconnue : le transfert a pu aboutir, ou non.

Aucune méthode Bot API ne permet de le savoir : on ne peut ni relire un message par son
identifiant, ni lister l'historique d'un sujet (§3). Les seules issues seraient de retenter —
au risque d'un doublon silencieux — ou de renoncer — au risque d'une perte silencieuse. Les
deux violent le §2.

**Décision : ni l'un ni l'autre.** À l'expiration de la réclamation, la ligne passe en
`incertain` ; le bot signale l'unité dans le sujet d'administration avec les identifiants source
et le sujet cible, et propose deux boutons : « déjà présent » (⇒ `envoye`, sans appel) et
« relancer » (⇒ nouvelle réclamation). Un humain regarde le sujet cible — c'est une opération de
trois secondes — et tranche. Le compte des `incertain` est un indicateur de santé de premier
plan (§19).

La fenêtre est réduite autant que possible : **un appel Bot API par réclamation**, jamais
plusieurs, et la confirmation est la toute première écriture après le retour de l'appel.

Traitement des erreurs `BotApiError` :

| Cas | Traitement |
|---|---|
| `code 429` avec `parameters.retry_after` | `bloque_infra`, aucune reprise dans l'invocation courante |
| `code 5xx` | `bloque_infra`, reprise bornée par `MAX_TENTATIVES_TRANSFERT` |
| `code 400` « message can't be forwarded » / contenu protégé | `quarantaine`, catégorie `capacite` |
| `code 400` « message to forward not found » | `quarantaine`, catégorie `capacite` (l'original a disparu) |
| `code 403` | `bloque_infra` — droit perdu, alerte dans `/etat` |
| `parameters.migrate_to_chat_id` | `bloque_infra` + alerte : le salon a migré, décision humaine |
| exception non `BotApiError` | `incertain` — on ne sait pas si l'appel est parti |

### 15.6 Reprise et balayage

Il n'existe pas de minuterie. La reprise est donc **portée par le trafic** :

- chaque invocation, après avoir traité son unité, exécute un balayage borné à
  `MAX_UNITES_BALAYEES` travaux : albums échus, `bloque_infra` dont le délai est passé,
  réclamations expirées, purge de `journal_appels` ;
- `/balayage` force un balayage immédiat ;
- `npx tgcloud run handlers/message '<payload>'` permet à l'opérateur de déclencher la même
  chose sans passer par un message réel.

Le balayage est **borné et journalisé** ; il ne crée jamais d'état terminal qu'il n'a pas
vérifié. Si le trafic s'arrête, le travail en retard reste en attente, visible, et sûr.

---

## 16. Quarantaine

### 16.1 Ce qui y va, ce qui n'y va pas

Y va : un **échec de classification** constaté — aucun sujet retenu, ratio de rejet trop élevé,
slug inconnu, attribution manquante, album non démontrablement complet, capacité de transfert
refusée par Telegram.

N'y va **jamais** : une panne d'infrastructure (⇒ `bloque_infra`), un secret manifeste
(⇒ `protege`), un résultat d'appel inconnu (⇒ `incertain`). Trois causes, trois états, trois
traitements — le §2.7.

Le message original **ne bouge pas** : la quarantaine est une ligne en base, pas un déplacement.
C'est la différence structurelle avec un dépôt de fichiers, et elle est à l'avantage du bot :
rien ne peut être perdu par une mise en quarantaine.

### 16.2 Le dossier de preuve

`quarantaine.preuve` (`json`) contient :

- la clé d'unité et la liste **exacte** des `message_id` concernés ;
- les `{ regle, attendu, obtenu }` en défaut ;
- la sortie brute du proposeur, telle quelle, et les lignes rejetées avec leur motif ;
- le ratio de rejet ;
- `vocabulaireVersion`, `evaluationVersion`, version du proposeur ;
- l'horodatage et l'identifiant d'invocation ;
- pour la catégorie `album` : bornes des identifiants, nombre de membres, trou constaté ;
- pour la catégorie `attribution` : `origineKind` et la raison exacte du refus.

**Jamais** : le texte du message, ni aucun extrait susceptible de contenir un secret (§11.3).

### 16.3 Reprise

`/reprendre <cleUnite>` — réservé aux administrateurs, confirmé par bouton — remet l'unité en
`pret`, incrémente `tentatives`, et conserve la ligne de quarantaine avec `statut: 'reprise'`.
C'est le chemin normal après correction du vocabulaire ou d'un seuil : corriger
`lib/vocabulaire.js`, `npx tgcloud push`, puis reprendre les unités concernées.

`/etat` groupe les quarantaines par `categorie` et par règle en défaut : c'est le premier écran
à regarder après un lot important, et le signal qui dit **quel réglage corriger**.

---

## 17. Déploiement, migrations et synchronisation

### 17.1 Le cycle

```bash
npx tgcloud status          # ce qui a changé localement, hors ligne
npx tgcloud diff            # ligne à ligne, hors ligne
npx tgcloud push            # déploiement atomique — ne touche jamais la base
npx tgcloud migrate         # applique les changements de schéma, après revue
npx tgcloud webhook         # état du webhook et de allowed_updates
```

`push` et `migrate` sont séparés par conception de la plateforme : un déploiement de code ne
peut jamais déclencher une migration par surprise. L'ordre est toujours **`push` puis
`migrate`** : `migrate` compare le schéma déployé à la base.

### 17.2 Politique de migration

- **Additif d'abord.** Nouvelle table, nouvelle colonne, nouvel index : statut `safe`, appliqué
  en une étape après confirmation.
- **Suppression uniquement par `.deprecated('motif')`.** Retirer une déclaration de `schema.js`
  ne supprime rien ; c'est délibéré et cela protège d'une suppression accidentelle. Le
  `.deprecated()` fait apparaître un `warning` confirmé individuellement, puis la déclaration
  est retirée.
- **Aucun changement de type.** Ils sont classés `manual` et doivent être faits à la main.
  §14.13 impose des choix de types qui évitent d'en arriver là ; si c'est inévitable, la
  procédure est : nouvelle colonne, recopie par `db.run` depuis une commande d'administration
  bornée, bascule du code, `.deprecated()` de l'ancienne.
- **`--dry-run` obligatoire en revue**, `--safe` en routine, `--yes` **jamais** dans une
  procédure écrite : il applique aussi les `warning`, c'est-à-dire les suppressions.
- Les objets `undocumented` — présents en base, absents du schéma — sont **signalés et jamais
  appliqués** ; ils sont aussi un contrôle d'audit
  ([`verification.md`](verification.md) §4.6).

### 17.3 Concurrence de déploiement

La plateforme tient une révision par projet et rejette un `push` en retard plutôt que d'écraser
le travail d'un autre. La règle du projet : `fetch` puis `pull`, jamais `--force`. `--force`
n'apparaît dans aucune procédure ; s'il devient nécessaire, c'est un incident à traiter comme
tel.

### 17.4 Webhook

`allowed_updates` est dérivé des handlers déployés. Ajouter ou retirer un handler peut laisser
le webhook désynchronisé ; `npx tgcloud webhook sync` le réaligne. `--drop-pending` **n'est pas**
utilisé en fonctionnement normal : jeter les updates en attente, c'est perdre des messages
d'`INBOX` que le bot n'aurait jamais vus — une perte silencieuse, exactement ce que le §2.6
interdit.

### 17.5 Ordre de mise en service

1. Créer le bot, activer Serverless dans BotFather, récupérer le jeton CLI (`tgcloud login`).
2. Ajouter le bot au supergroupe forum comme administrateur avec `can_manage_topics`.
3. `push` puis `migrate` (le schéma est entièrement additif au premier déploiement).
4. Poster `/enregistrer inbox` **dans** le sujet `INBOX`, puis `/enregistrer admin` dans le
   sujet d'administration, puis `/enregistrer <slug>` dans chaque sujet thématique préexistant.
5. `/mode ombre` — et seulement ensuite, le rodage du §20.

---

## 18. Tests

### 18.1 Le banc d'essai : `tgcloud run`

`npx tgcloud run handlers/message '<payload JSON5>'` exécute le handler **sur la plateforme avec
les fichiers locaux**, sans déployer, et restitue la valeur de retour, la sortie `console.*` et
la durée. C'est le seul mécanisme d'exécution documenté hors update réelle, donc le socle de la
recette.

```bash
npx tgcloud run handlers/message "$(cat tests/cas/album-incomplet.json5)"
npx tgcloud run handlers/message '{ chat:{id:-100}, message_thread_id: 42, text: "https://github.com/x" }' \
  --ctx '{ update: { update_id: 1 }, essai: true }'
```

### 18.2 La couture d'essai, et pourquoi elle est indispensable

**`run` s'exécute contre la base réelle du bot.** Il n'existe pas de base d'essai documentée.
Sans précaution, chaque essai écrirait dans la base de production et pourrait transférer de
vrais messages.

La couture retenue n'utilise que du documenté : le second argument du handler, `ctx`, est
fourni par `--ctx`. Le handler lit `ctx.essai === true` et, dans ce cas :

- n'effectue **aucun** appel Bot API — les appels sont enregistrés et retournés ;
- n'effectue **aucune** écriture — les mutations prévues sont retournées ;
- retourne l'intégralité du raisonnement : unité, proposition brute, lignes retenues et
  rejetées, verdict, sujets résolus, transferts qui auraient été faits.

Une update réelle ne peut pas porter `ctx.essai` : `ctx` est construit par la plateforme et ne
contient que ce qu'elle y met. La couture est donc inatteignable en production, et c'est
vérifiable par lecture du handler.

### 18.3 Recette du composant obligatoire

1. **Fonctions pures, hors plateforme.** `empreinte`, `texte`, `tags`, `verdict`, `secrets`,
   `unite`, `proposeur` sont testables par simple exécution : vecteurs dorés d'empreinte,
   table de décision complète de l'évaluation, et surtout **corpus de propositions
   pathologiques** — prose d'introduction, puces, numérotation, blocs de code, balises internes,
   ligne tronquée en fin de flux, espace de noms inconnu, slug inconnu, doublons, ligne de 10 ko,
   sortie vide, sortie intégralement non conforme. Chacune doit être ignorée sans exception et
   comptée au bon motif.
2. **Corpus de payloads.** `tests/cas/*.json5`, un fichier par cas : message texte simple, lien
   seul, album complet, album à trou, album de 11 membres, message transféré avec chaque type de
   `forward_origin`, message sans `from` ni `sender_chat`, message avec
   `has_protected_content`, message contenant chaque famille de secret, commande par un
   administrateur, commande par un non-administrateur, message hors `INBOX`, message émis par le
   bot, message de service.
3. **Chaque cas est joué en `ctx.essai`** et son verdict comparé à un attendu versionné dans
   `tests/attendus/`. Le script `tests/executer.sh` enchaîne les cas et sort non-zéro à la
   première divergence — les commandes du CLI sortent non-zéro en cas d'échec, elles se
   composent donc proprement.
4. **Idempotence.** Rejouer deux fois le même payload hors mode essai (sur un bot de recette)
   ne produit **aucun second transfert** : la clé de `transferts` est la garantie, le test la
   vérifie.
5. **Concurrence.** Deux `run` simultanés sur deux membres du même album : un seul groupe créé,
   un seul transfert, aucune ligne perdue.
6. **Panne ≠ échec de classification.** Un cas forcé en erreur `5xx` simulée doit laisser
   l'unité en `bloque_infra` et **jamais** en quarantaine.
7. **Borne d'appels.** Un cas qui produirait plus de `MAX_APPELS_API` appels doit s'arrêter
   proprement, laisser le reste en base, et ne créer aucun état terminal non vérifié.
8. **Album jamais découpé.** Aucun chemin ne produit un `forwardMessage` unitaire pour un membre
   d'album — vérifié par le journal d'appels du mode essai.

### 18.4 Contrôles statiques du dépôt

Exécutés en intégration continue, ce sont des recherches textuelles sur les sources — la seule
forme de contrôle structurel disponible dans un projet sans compilation :

| Contrôle | Règle |
|---|---|
| Imports | tout `import … from` cible `sdk`, `sdk/*`, `schema` ou `lib/…` ; aucun `./`, aucun `.js` |
| Pureté | `lib/{empreinte,texte,proposeur,tags,verdict,secrets,unite}.js` n'importent ni `sdk` ni `schema` |
| Méthodes interdites | aucune occurrence de `deleteMessage`, `editMessageText`, `deleteForumTopic`, `closeForumTopic`, `editGeneralForumTopic` |
| Réseau | aucune occurrence de `fetch` en v1 |
| Clés étrangères | aucune occurrence de `.references(` ni de `foreignKey(` |
| Handlers | `handlers/` est plat et ne contient que des types d'update valides |
| Bornes | toute constante numérique de politique vient de `lib/bornes.js` |
| Secrets | aucune chaîne ressemblant à un jeton dans le dépôt |

---

## 19. Observabilité

**La plateforme ne documente aucun journal d'exécution consultable en production** : `console.*`
n'est restitué que par `tgcloud run` et par l'exécution manuelle depuis BotFather. Toute
l'observabilité de production est donc en base, et rendue par une commande.

- `journal_appels` — chaque appel Bot API : méthode, cible, résultat, code, durée. Rétention
  bornée, purge par balayage.
- `compteurs` — agrégats journaliers : unités ingérées, classées, en quarantaine, protégées,
  transferts, échecs par code.
- `/etat` — rendu dans le sujet d'administration : mode courant, unités par état, âge de la plus
  ancienne unité non terminale, **nombre d'`incertain`** et **nombre d'`attente_album` échus**
  (les deux indicateurs qui exigent un humain), quarantaines groupées par catégorie et règle en
  défaut, propositions de sujets en attente, droits constatés et leur ancienneté.

`console.*` reste utilisé dans le code, pour la valeur qu'il a en `tgcloud run` : il est le
principal outil de mise au point. Il ne porte **jamais** de contenu de message (§11.3).

---

## 20. Mise en service progressive

Le mode vit dans `parametres.mode`, se change par `/mode <…>`, et chaque changement est
journalisé avec son acteur.

| Mode | Ingestion | Décision | Sujets | Transferts |
|---|---|---|---|---|
| `ombre` | oui | oui, écrite en base | aucune mutation | **aucun** |
| `revue` | oui | oui | proposées, jamais appliquées | **seulement après approbation** par bouton |
| `canari` | oui | oui | création autorisée pour les slugs du vocabulaire | automatiques pour une **liste blanche** de slugs |
| `actif` | oui | oui | création autorisée ; renommage toujours proposé | automatiques |

1. **`ombre`** — on classe sans rien transférer. Objectif : calibrer `seuilScore` et les poids
   sur le corpus réel, en lisant les décisions accumulées. C'est la seule manière honnête de
   fixer un seuil ; a priori, il serait deviné.
2. **`revue`** — chaque classement est proposé dans le sujet d'administration avec ses `motif:`,
   et un bouton l'applique. Objectif : mesurer le taux d'approbation par sujet. Un sujet dont
   les propositions sont refusées plus d'une fois sur cinq n'est pas prêt.
3. **`canari`** — un seul sujet, le mieux noté en `revue`, passe en automatique. Objectif :
   observer l'idempotence, les albums et les reprises en conditions réelles.
4. **`actif`** — tous les sujets du vocabulaire. Le renommage reste proposé (§5.4) : ce n'est
   pas un mode, c'est un invariant.

Le retour en arrière est immédiat et sans migration : `/mode ombre` suffit, et rien de ce qui a
été transféré n'est défait — conformément au §2, le bot n'annule pas, il n'a rien détruit.

---

## 21. Invariants

1. **Aucun message n'est réécrit, résumé, découpé, réparé ou recréé.** La seule production de
   contenu classé est un transfert.
2. **Le classement se fait exclusivement par `forwardMessage` / `forwardMessages`**, sur le
   `message_id` exact reçu, jamais recalculé ni deviné.
3. **L'original reste dans `INBOX`** : le bot n'appelle jamais de méthode qui altère ou supprime
   un message.
4. **Un message qui relève de plusieurs sujets est transféré intégralement dans chacun.**
5. **Un album est transféré en un seul appel, entier, ou pas du tout.**
6. **Aucun sujet n'est créé si un sujet connu le couvre** ; aucun sujet n'est créé pour un slug
   absent du vocabulaire.
7. **Aucun renommage de sujet n'est appliqué sans décision humaine explicite**, et toute
   mutation de sujet laisse deux lignes dans `sujets_journal` — avant et après l'appel.
8. **Échec fermé** : `message_id` incertain, album non démontrablement complet, attribution
   exigible manquante ou capacité de transfert refusée ⇒ **rien n'est republié**.
9. **Un secret manifeste n'est jamais amplifié** : état `protege`, aucun transfert automatique,
   aucune recopie de l'extrait où que ce soit.
10. **Idempotence stricte** : rejouer une update ne produit ni second classement ni second
    transfert ; les garanties sont des contraintes de base, pas du code.
11. **Toute transition d'état est une instruction unique** en comparaison-et-échange ; aucune
    séquence lecture-puis-écriture n'est supposée atomique.
12. **Une issue d'appel inconnue ne se résout jamais toute seule** : état `incertain`, décision
    humaine.
13. **Trois causes d'arrêt, jamais confondues** : classification ⇒ `quarantaine`, sécurité ⇒
    `protege`, infrastructure ⇒ `bloque_infra`.
14. **Les appels sont bornés** : au plus `MAX_APPELS_API` par invocation, au plus
    `MAX_SUJETS_PAR_UNITE` transferts par unité, au plus deux propositions par unité.
15. **Aucune ligne de proposition non conforme n'est réparée** : elle est ignorée, comptée et
    journalisée avec son motif.
16. **Le vocabulaire et les règles d'évaluation sont des autorités versionnées**, jamais écrites
    par le bot ; leur version est inscrite dans chaque décision.
17. **Toute mutation de sujet et tout transfert sont traçables en SQLite**, avec les
    identifiants source et destination.
18. **Aucun secret n'est stocké en base ni dans le dépôt**, et aucun code ne lit de secret
    d'exécution — il n'en existe pas de mécanisme documenté.
19. **Aucune donnée ne quitte Telegram** : aucun `fetch` sortant en v1.
20. **Le bot ne classe jamais ses propres copies** : double garde-fou — sujet d'origine et
    identité de l'émetteur.

Le bot **maintient** ces invariants. Le composant optionnel les **contrôle** de façon
indépendante ; la correspondance invariant → contrôle est en
[`verification.md`](verification.md) §6.

---

## 22. Matrice des invariants et des contrôles

> Dans les colonnes « Maintenu par » et « Contrôle en ligne », les renvois `§n` désignent des
> sections **de ce document**. Dans la colonne « Contrôle *a posteriori* », ils désignent des
> sections de [`verification.md`](verification.md).

| # | Invariant | Maintenu par | Contrôle en ligne | Contrôle *a posteriori* |
|---|---|---|---|---|
| 1 | aucune réécriture | §2, §10.2 | contrôle statique §18.4 | audit du code, non des données |
| 2 | transfert du `message_id` exact | §10.2 | `transferts.messageIdsSource` comparé à `evenements` | oui — §3.1 |
| 3 | original conservé | §2.2 | méthodes interdites §18.4 | non — hors portée de l'API |
| 4 | multi-appartenance complète | §8, §15.2 | un `transferts` par `classements` retenu | oui — §3.3 |
| 5 | album atomique | §9.5 | journal d'appels du mode essai §18.3.8 | oui — §3.2 |
| 6 | pas de sujet en doublon | §5.1, §5.3 | index unique `sujets.threadId` | oui — §4.4 |
| 7 | renommage humain, doublement journalisé | §5.4 | deux lignes `sujets_journal` | oui — §4.4 |
| 8 | échec fermé | §6, §9.4, §10.1 | quarantaine avec preuve | oui — §4.3 |
| 9 | secrets non amplifiés | §11 | ordre d'évaluation figé §8 | oui — §4.5 |
| 10 | idempotence | §15.2 | clés primaires et index uniques | oui — §3.3 |
| 11 | transitions atomiques | §15.3 | — | non — propriété d'exécution |
| 12 | `incertain` non auto-résolu | §15.5 | — | oui — §4.3 (comptage et âge) |
| 13 | trois causes distinctes | §2.7, §15.5 | table de correspondance des erreurs | oui — §4.3 |
| 14 | appels bornés | §13.4 | compteur d'invocation | oui — §4.6 (`journal_appels`) |
| 15 | lignes non conformes ignorées | §4.2 | `lignesRejetees` en base | oui — §3.4 |
| 16 | autorités versionnées | §4.1, §8 | version inscrite dans chaque décision | oui — §4.2 |
| 17 | traçabilité | §14.4, §14.6 | écriture avant/après appel | oui — §4.4 |
| 18 | aucun secret stocké | §11.4 | liste noire de clés `parametres` | oui — §4.5 |
| 19 | rien ne quitte Telegram | §12 | contrôle statique §18.4 | non — audit du code |
| 20 | pas d'auto-classement | §6.1 | double garde | oui — §4.1 |

Les invariants marqués « propriété d'exécution » ou « audit du code » ne sont pas vérifiables
sur les seules données : ils sont garantis par la conception et couverts par la recette (§18).

---

## 23. Phases livrables

Chaque phase est déployable et testable seule. Les phases 0 à 2 ne transfèrent **aucun**
message. Les phases du composant optionnel sont numérotées séparément (V1…V3) dans
[`verification.md`](verification.md) §8.

### Phase 0 — Socle déterministe, sans plateforme

- `lib/empreinte.js`, `lib/texte.js`, `lib/bornes.js`.
- `lib/vocabulaire.js`, `lib/evaluation.js` — autorités versionnées.
- `lib/tags.js` (parseur), `lib/verdict.js` (évaluation), `lib/secrets.js`, `lib/proposeur.js`.
- **Recette :** vecteurs dorés d'empreinte ; corpus de propositions pathologiques ; table de
  décision complète du verdict ; contrôles statiques §18.4. Aucune dépendance à `sdk`.

### Phase 1 — Schéma et ingestion

- `schema.js` complet (§14), `push` + `migrate`.
- `handlers/message.js` : raccourcis du §6.1, ingestion idempotente, construction de l'unité,
  agrégation d'album, états `recu`/`attente_album`/`pret`.
- `lib/etat.js` : transitions par comparaison-et-échange.
- Couture `ctx.essai` (§18.2), commandes `/enregistrer`, `/etat`.
- **Recette :** corpus de payloads §18.3.2 en mode essai ; rejeu d'une update ⇒ aucune ligne
  supplémentaire ; deux membres d'album en parallèle ⇒ un seul groupe.

### Phase 2 — Décision, mode `ombre`

- Chaînage proposition → analyse → évaluation → `classements`, sans aucun transfert.
- `cache_propositions`, `propositions_sujet`, `quarantaine` et son dossier de preuve.
- `/mode ombre`, `/reprendre`, `/etat` enrichi.
- **Recette :** chaque cas du corpus produit le verdict attendu ; une unité en quarantaine porte
  un dossier de preuve complet et **aucun texte de message** ; un secret manifeste produit
  `protege` avant toute autre règle.

### Phase 3 — Sujets

- `lib/sujets.js` : résolution, enregistrement, création sous conditions (§5.3), proposition de
  renommage (§5.4), `sujets_journal` avant/après.
- `handlers/my_chat_member.js` et le constat de `can_manage_topics`.
- `handlers/callback_query.js` : appliquer / refuser un renommage.
- **Recette :** création refusée pour un slug hors vocabulaire ; création refusée si un sujet
  connu couvre le slug ; droit manquant ⇒ `bloque_infra`, pas quarantaine ; deux invocations
  concurrentes ⇒ un seul sujet créé (index unique sur `threadId`).

### Phase 4 — Transfert

- `lib/transfert.js` : réclamation idempotente, `forwardMessage` / `forwardMessages`,
  confirmation, table de correspondance des erreurs (§15.5), état `incertain` et ses boutons.
- Balayage (§15.6), `/balayage`.
- Modes `revue` puis `canari`.
- **Recette :** rejeu ⇒ aucun second transfert ; album transféré en un appel ; `429` ⇒
  `bloque_infra` ; contenu protégé ⇒ quarantaine `capacite` ; interruption simulée entre appel
  et confirmation ⇒ `incertain`, et aucune reprise automatique.

### Phase 5 — Exploitation

- `/liberer` en deux temps, `compteurs`, purge de `journal_appels`, `/etat` complet.
- Passage en `actif` après la grille de sortie du §20.
- **Recette :** libération journalisée avec acteur ; purge bornée ; `/etat` reflète exactement
  le contenu des tables.

---

## 24. Décisions ouvertes

1. **`.returning()` sur `update`** (§15.3). La documentation ne l'illustre que sur `insert`,
   tout en indiquant que le constructeur suit Drizzle, où il existe aussi pour `update`. S'il
   fonctionne, chaque transition économise une lecture et devient strictement atomique ; sinon,
   `lib/etat.js` conserve la relecture. **À vérifier par un `tgcloud run` d'une ligne avant la
   phase 1**, car cela touche le cœur du modèle de concurrence.
2. **Conserver ou non le texte normalisé** (§14.1). Ne pas le stocker protège la
   confidentialité et allège la base, mais interdit de rejouer une proposition après un
   changement de vocabulaire — il faudrait relire le message, ce que l'API ne permet pas. Trois
   options : ne rien stocker (proposé) ; stocker uniquement le texte normalisé des unités **non**
   `protege`, avec purge après N jours ; stocker uniquement les traits extraits (domaines,
   termes déclenchés), qui suffisent à rejouer les règles mais pas un LLM. La troisième est
   probablement le bon compromis et mérite d'être tranchée avant la phase 2.
3. **Valeur de `FENETRE_ALBUM_S`** (§9.3). 8 secondes est un point de départ ; la valeur juste
   dépend de la latence réelle de livraison des membres d'album et se mesure en mode `ombre`.
4. **`handlers/edited_message.js`** (§13.2). Absent en v1. L'ajouter donnerait une trace des
   éditions d'originaux déjà classés — utile pour comprendre une divergence entre `INBOX` et une
   copie — au prix d'un type d'update de plus. À revoir après la phase 4.
5. **`protect_content` sur les copies** (§10.2). Non transmis, donc la copie hérite du
   comportement par défaut du salon. Le transmettre empêcherait la re-transmission des copies —
   ce qui peut être souhaitable — mais modifierait une propriété de la copie, contre l'esprit du
   §2. À trancher explicitement plutôt qu'à laisser par défaut.
6. **Contiguïté des `message_id` d'album** (§9.4). C'est une heuristique. Si le corpus réel
   montre des albums à identifiants non consécutifs, il faudra soit l'abandonner — et n'avoir
   plus que la fenêtre de silence, plus faible — soit la remplacer par un contrôle de taille
   observée. À mesurer en mode `ombre`.
7. **Seuils et poids** (§4.3, §8). `seuilScore: 60` et les poids du §7.2 sont des valeurs
   *a priori*. Elles ne peuvent être fixées qu'en mode `ombre`, sur le corpus réel. Un seuil
   trop haut laisse tout partir en quarantaine ; trop bas, il classe au hasard — et le second
   défaut est le plus coûteux, parce qu'il est silencieux et qu'il salit des sujets durablement.
8. **Portée du bot.** Un seul supergroupe forum, un seul `INBOX` (`parametres.chatId`). Étendre
   à plusieurs salons demanderait de revoir toutes les clés et les index uniques. Hors périmètre
   v1, à acter explicitement.

---

## 25. Références

Documentation officielle, faisant autorité pour tout ce document :

- **Telegram Serverless** — <https://core.telegram.org/bots/serverless>
  - *Projects & modules* (`#projects-and-modules`) — arborescence, modules déployés, imports nus,
    `handlers/` plat, absence de npm et de système de fichiers.
  - *The database* (`#the-database`) — DSL de `schema.js`, types et modes de colonne, index et
    contraintes, **absence de clés étrangères** (`#no-foreign-keys`), constructeur de requêtes,
    SQL brut, **migrations** (`#migrations`) et statuts `safe`/`warning`/`manual`/`undocumented`.
  - *The SDK* (`#the-sdk`) — `api` (résultat déballé, `BotApiError`), **limitations sur les
    fichiers** (`#file-limitations`), `fetch` (textuel, ≤ 32 Mo), `console`.
  - *Command-line interface* (`#command-line-interface`) — `push`, `migrate`, `run`, `status`,
    `diff`, `pull`, `fetch`, `reset`, `webhook`, authentification, *Staying in sync*.
- **Telegram Bot API** — <https://core.telegram.org/bots/api>
  - `forwardMessage` — <https://core.telegram.org/bots/api#forwardmessage>
  - `forwardMessages` — <https://core.telegram.org/bots/api#forwardmessages>
  - `Message` — <https://core.telegram.org/bots/api#message>
  - `MessageOrigin` — <https://core.telegram.org/bots/api#messageorigin>
  - `createForumTopic` — <https://core.telegram.org/bots/api#createforumtopic>
  - `editForumTopic` — <https://core.telegram.org/bots/api#editforumtopic>
  - `getChatMember` — <https://core.telegram.org/bots/api#getchatmember>
  - `getMe` — <https://core.telegram.org/bots/api#getme>
  - `Update` — <https://core.telegram.org/bots/api#update>

Les champs et méthodes Bot API cités dans ce document ont été vérifiés contre la référence
officielle liée ci-dessus. Toute évolution future de la plateforme doit être revérifiée contre
ces deux sources avant d'être utilisée dans le code.
