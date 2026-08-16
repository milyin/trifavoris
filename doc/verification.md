# trifavoris — vérification

> **Composant optionnel.** `trifavoris` observe, classe et transfère sans lui. Ce document
> décrit un audit **indépendant** de l'état du bot : il relit la base SQLite et recontrôle les
> invariants du pipeline sans partager son code de décision. Le bot lui-même est décrit dans
> [`classement-des-messages.md`](classement-des-messages.md).

---

## 1. Statut : optionnel, et ce que cela implique

Le composant se retire en supprimant `lib/audit/`, la route `/audit` de
`handlers/message.js`, et en marquant `.deprecated()` les deux tables du §7.2. Dans ce cas :

**Ce qui reste garanti.** Le pipeline conserve ses **contrôles en ligne**, qui ne dépendent pas
de ce composant et ne sont pas désactivables :

- les contraintes de base — `evenements.cle` primaire, `uidx_classements(cleUnite, sujetSlug)`,
  `transferts.cle` primaire, index unique sur `sujets.threadId` — qui portent l'idempotence et
  l'unicité des sujets sans une ligne de code applicatif ;
- l'écriture **avant** et **après** chaque appel mutatif (`sujets_journal`, `transferts`), qui
  rend une issue inconnue visible plutôt que silencieuse ;
- l'échec fermé de chaque contrôle d'admissibilité
  ([`classement-des-messages.md`](classement-des-messages.md) §6, §9.4, §10.1) ;
- les contrôles statiques du dépôt (§18.4 du même document), exécutés en intégration continue.

**Ce qui est perdu.** Quatre choses, toutes *a posteriori* :

1. **La détection des dérives accumulées** — une unité restée en `attente_album` depuis trois
   semaines, un `incertain` que personne n'a tranché, une quarantaine que personne n'a reprise,
   une ligne `sujets_journal` ouverte sans clôture. Aucune de ces situations n'est une erreur au
   moment où elle apparaît ; toutes sont des pannes quand elles durent.
2. **L'indépendance du contrôle.** Le pipeline vérifie ses écritures avec le code qui les
   produit : il détecte une écriture ratée, pas un défaut **dans** sa propre logique de
   décision. Le composant d'audit réimplémente les contrôles depuis le contrat de données, sans
   partager de module métier — c'est là qu'est sa valeur.
3. **La procédure de reprise après incident** (`/audit doctor`, §5), qui donne un feu vert motivé
   avant de repasser en `actif` après une panne.
4. **La détection d'un secret stocké par erreur** (§4.5), qui est le seul contrôle du système
   capable de constater qu'une future version aurait introduit une régression de confidentialité.

> **Compromis assumé.** Sans ce composant, les deux indicateurs qui exigent un humain —
> `incertain` et `attente_album` échus — ne sont visibles que si quelqu'un tape `/etat`. C'est
> acceptable tant que le bot est surveillé et le volume faible ; ça ne l'est plus dès qu'il
> tourne seul sur un groupe actif. Recommandation : livrer le composant, et le rendre
> optionnel plutôt qu'absent.

---

## 2. Principe : indépendance, et ses limites réelles

### 2.1 La règle

**Le composant ne partage aucun module de décision avec le pipeline, et ne réutilise aucune de
ses fonctions de calcul.**

- Entrées autorisées : les tables de `schema.js`, `lib/vocabulaire.js` et `lib/evaluation.js`
  **en lecture seule** (ce sont les autorités de configuration, pas du code métier), et
  `lib/bornes.js`.
- Entrées interdites : `lib/verdict.js`, `lib/proposeur.js`, `lib/tags.js`, `lib/etat.js`,
  `lib/transfert.js`, `lib/sujets.js`, `lib/unite.js`, `lib/secrets.js`. Les règles sont
  **réimplémentées** — sinon un défaut du moteur d'évaluation ou du parseur passerait inaperçu,
  puisque l'audit le reproduirait à l'identique.
- `lib/empreinte.js` est le seul cas discuté : recalculer une empreinte exige la même fonction
  de hachage. L'audit **réimplémente la chaîne canonique** (l'ordre des champs, la
  sérialisation) mais réutilise `sha256Hex`. Réimplémenter SHA-256 deux fois ne contrôlerait
  rien d'utile : une divergence signalerait un bug de transcription, pas un défaut de
  conception. La partie qui porte le sens — *ce qu'on hache* — est bien réimplémentée.

Dans un projet sans système de compilation, cette séparation ne peut pas être garantie par un
graphe de dépendances de paquets. Elle l'est par un **contrôle statique du dépôt** : aucun
fichier de `lib/audit/` ne peut importer un module de la liste interdite. Ce contrôle fait
partie de l'intégration continue, au même titre que ceux du §18.4 de
[`classement-des-messages.md`](classement-des-messages.md).

Réciproquement : **aucun module hors `lib/audit/` n'importe `lib/audit/`.** La seule exception
est la route `/audit` de `handlers/message.js`, qui se réduit à un import et un appel.

### 2.2 Ce que l'API ne permet pas de vérifier — et qu'il faut dire

C'est la différence majeure avec un audit de système de fichiers, où l'auditeur relit la
réalité. Ici, **la réalité est inaccessible** :

| Question | Vérifiable ? | Pourquoi |
|---|---|---|
| La copie transférée existe-t-elle encore dans le sujet cible ? | **non** | aucune méthode Bot API ne relit un message par identifiant |
| Le message original est-il toujours dans `INBOX` ? | **non** | même raison |
| La copie est-elle identique à l'original ? | **non** | et c'est Telegram qui l'a produite, pas le bot |
| Quels sujets existent réellement dans le forum ? | **non** | aucune méthode ne liste les sujets d'un forum |
| Le nom réel d'un sujet correspond-il à `sujets.nom` ? | **non** | pas de lecture de sujet |
| Le bot a-t-il encore ses droits ? | **oui** | `getChatMember` — non mutatif |
| Le salon existe-t-il encore, est-ce bien un forum ? | **oui** | `getChat` — non mutatif |

**Conséquence : l'audit est un audit de cohérence interne, pas de vérité terrain.** Il vérifie
que la base raconte une histoire cohérente et complète, que rien n'y est resté en suspens, et
que les invariants exprimables sur les données tiennent. Il ne peut pas certifier qu'un message
est bien arrivé quelque part.

**L'audit est strictement non mutatif.** Il n'appelle que `getChat` et `getChatMember`, n'écrit
que dans ses propres tables (§7.2), et ne corrige jamais rien : il constate et rapporte.
Prouver l'existence d'une copie exigerait de transférer un message de sonde — c'est-à-dire
d'amplifier du contenu pour vérifier qu'on l'a bien amplifié. C'est exclu.

---

## 3. Contrôles unitaires

Les quatre familles ci-dessous portent sur une unité de classement et sont réutilisées par
l'audit de corpus (§4).

### 3.1 Correspondance transfert ↔ événement

| Contrôle | Détail |
|---|---|
| Existence | tout `transferts.cleUnite` correspond à au moins une ligne `evenements` de même `cleUnite` |
| Identifiants exacts | `transferts.messageIdsSource` est **exactement** l'ensemble des `evenements.messageId` de l'unité — ni sur-ensemble, ni sous-ensemble, ni identifiant recalculé |
| Salon | tous les `evenements` de l'unité partagent `chatId` et `threadId`, et ce `threadId` est celui d'`INBOX` |
| Clé d'idempotence | `transferts.cle` recomposée depuis `cleUnite`, `sujetSlug` et l'empreinte de la liste ordonnée des identifiants sources est **identique** à la clé stockée |
| Sujet cible | `transferts.threadId` correspond à `sujets.threadId` pour `sujetSlug`, et ce n'est **pas** le `threadId` d'`INBOX` |
| Cohérence d'état | `envoye` ⇒ `messageIdsDestination` non vide et `termineLe` renseigné ; `reclame` ⇒ `expireLe` renseigné ; `echoue` ⇒ `erreurCode` renseigné |

Le contrôle de clé d'idempotence est le plus utile des six : il détecte une divergence entre la
composition réelle d'une unité et ce sur quoi le transfert a porté — c'est-à-dire exactement le
cas où un album aurait été transféré incomplet.

### 3.2 Albums

| Contrôle | Détail |
|---|---|
| Cardinalité | un `groupes_media` clos a entre 2 et `MAX_MEMBRES_ALBUM` membres |
| Concordance | `groupes_media.membres` égale le nombre de lignes `evenements` portant ce `mediaGroupId` |
| Contiguïté | les `messageId` des membres forment une suite sans trou, ou le groupe est `incomplet` et l'unité est en quarantaine catégorie `album` |
| Atomicité | pour un album `classe`, il existe **un** `transferts` par sujet retenu, et chacun porte **tous** les identifiants de l'album |
| Non-découpage | aucun `transferts` ne porte un sous-ensemble strict des identifiants d'un album |
| Destination | `messageIdsDestination` a la même cardinalité que `messageIdsSource` |

Le contrôle de non-découpage est le contrôle central du composant : il porte sur l'invariant 5,
et c'est le seul invariant du bot dont la violation serait à la fois grave et invisible depuis
l'interface Telegram, où une suite de médias isolés ressemble à un album.

### 3.3 Complétude des décisions

| Contrôle | Détail |
|---|---|
| Multi-appartenance | une unité `classe` a **autant** de `transferts` en état `envoye` que de `classements` en état `retenu` |
| Pas de transfert orphelin | tout `transferts` correspond à un `classements` `retenu` ou `approuve` de même `(cleUnite, sujetSlug)` |
| Pas de décision sans suite | un `classements` `retenu` sur une unité `classe` a son `transferts` en `envoye` |
| Pas de doublon | aucune paire `(cleUnite, sujetSlug)` en double — l'index unique le garantit, le contrôle vérifie qu'il est bien en place |
| Cardinalité | au plus `MAX_SUJETS_PAR_UNITE` `classements` `retenu` par unité |
| Terminaison | une unité `classe` n'a aucun `transferts` en `reclame` ou `incertain` |

### 3.4 Qualité d'analyse et versions

| Contrôle | Détail |
|---|---|
| Version de décision | tout `classements` porte un `vocabulaireVersion` et un `evaluationVersion` non vides |
| Dérive | un `classements` dont `vocabulaireVersion` diffère de la version déployée est **signalé**, jamais modifié : c'est le signal qui dit qu'une reprise s'impose |
| Slug connu | tout `classements.sujetSlug` figure dans `lib/vocabulaire.js`, ou est signalé comme hérité d'un vocabulaire retiré |
| Seuil | tout `classements` `retenu` a un `score` ≥ `seuilScore` de la version d'évaluation qu'il porte |
| Lignes rejetées | pour toute unité en quarantaine catégorie `classification`, `preuve` contient les lignes rejetées **et** leur motif |
| Réévaluation indépendante | l'audit **réimplémente** les règles de `lib/evaluation.js` et recalcule le verdict à partir des lignes retenues stockées ; toute divergence avec `classements` est signalée |

Le dernier contrôle est la raison d'être de la réimplémentation exigée au §2.1 : il compare deux
lectures indépendantes des mêmes règles sur les mêmes entrées. Une divergence est soit un défaut
du moteur d'évaluation, soit un défaut de l'audit — dans les deux cas, quelque chose qui ne se
serait jamais vu autrement.

---

## 4. Audit de l'état

### 4.1 Événements et périmètre

- Tout `evenements.threadId` est celui d'`INBOX` : une ligne portant un autre sujet signale une
  régression du filtre d'entrée (invariant 20).
- Aucun `evenements.auteurId` n'est égal à `parametres.botId` : le bot n'a jamais ingéré une de
  ses propres copies (invariant 20, seconde garde).
- Tout `evenements` porte un `cleUnite` ; toute unité d'album a une ligne `groupes_media`.
- Aucun `evenements.cle` ne dévie du gabarit `<chatId>:<messageId>` recomposé depuis ses propres
  colonnes.
- Orphelins : `classements`, `transferts` et `quarantaine` sans `evenements` correspondant —
  conséquence directe de l'absence de clés étrangères. Signalés, **jamais supprimés**.

### 4.2 Décisions et autorités

- Réexécution des contrôles du §3.4 sur l'ensemble du corpus.
- Distribution des `classements` par sujet et par version de vocabulaire : c'est la lecture qui
  permet de calibrer un seuil et de repérer un sujet devenu fourre-tout.
- Empreinte des autorités : l'audit recalcule `sha256Hex` de `lib/vocabulaire.js` et de
  `lib/evaluation.js` **tels que déployés** et la compare à celle enregistrée au dernier audit.
  Un changement non accompagné d'une reprise des unités concernées est signalé.

### 4.3 États non terminaux — le cœur du composant

C'est ici que se trouvent les pannes silencieuses. Aucun de ces états n'est une erreur ; tous le
deviennent avec le temps.

| État | Seuil d'alerte (paramétrable) | Pourquoi c'est grave |
|---|---|---|
| `attente_album` | âge > 10 × `FENETRE_ALBUM_S` | album jamais clos faute de trafic — rien ne le rappellera |
| `incertain` | toute occurrence, dès la première | exige une décision humaine, par conception |
| `bloque_infra` | âge > 1 h, ou `tentatives` = `MAX_TENTATIVES_TRANSFERT` | la panne n'est plus transitoire |
| `transfert` avec `transferts` `reclame` expiré | toute occurrence | réclamation abandonnée, doit devenir `incertain` |
| `quarantaine` `ouverte` | âge > 30 jours | une quarantaine qu'on ne vide jamais est une corbeille |
| `protege` | âge > 30 jours | un message protégé oublié reste non classé, et c'est peut-être voulu — le rappel n'est pas un reproche |
| `retenu` sans `transferts` | âge > 1 h | décision prise, transfert jamais réclamé |

Le rapport nomme l'unité, son état, son âge et sa dernière raison. Un compte agrégé sans les
clés serait inexploitable : un audit qui ne dit pas *quoi* regarder ne sert à rien.

### 4.4 Sujets et traçabilité

- **Unicité** : aucun `threadId` associé à deux slugs (l'index unique le garantit ; le contrôle
  vérifie qu'il existe) ; aucun slug associé à deux `threadId`.
- **Couverture** : aucun couple de sujets connus dont l'un `couvre` l'autre au sens de
  `lib/vocabulaire.js` — c'est le doublon que l'invariant 6 interdit, constaté sur les données.
- **Journal complet** : toute ligne `sujets_journal` en `resultat: 'propose'` a une ligne
  postérieure en `ok`, `erreur` ou `refuse` pour le même `slug` et la même `action`. Une
  proposition sans clôture est un appel dont l'issue est inconnue : **signalé nommément**.
- **Renommage humain** : aucune ligne `action: 'renommage'`, `resultat: 'ok'` ne porte
  `acteur: 'bot'`. C'est le contrôle de l'invariant 7, et il est absolu — une seule occurrence
  est un défaut.
- **Origine** : tout `sujets.origine` vaut `cree` ou `enregistre`, et une ligne `cree` a une
  ligne `sujets_journal` `action: 'creation'` correspondante.
- **Sonde non mutative** : `getChat` confirme que le salon existe et est un forum ;
  `getChatMember` confirme les droits du bot et met à jour l'âge de `droitsConstates`. Ce sont
  les deux seuls appels que l'audit s'autorise.

### 4.5 Sécurité et confidentialité

- **Aucun secret en base.** `parametres.cle` est comparée à une liste noire (`token`, `key`,
  `secret`, `password`, `mdp`, `api_key`, `bearer`, …) et `parametres.valeur` est passée au
  détecteur d'entropie **réimplémenté** par l'audit. Toute correspondance est signalée par le
  nom de la clé — jamais par la valeur.
- **Aucun texte de message stocké.** Les colonnes qui ne doivent pas contenir de texte libre
  (`quarantaine.preuve`, `cache_propositions.lignesRetenues`, `journal_appels.description`) sont
  contrôlées : longueur bornée, absence de motifs de secret, absence de contenu ressemblant à un
  corps de message. C'est le contrôle qui attraperait une régression où un dossier de preuve se
  mettrait à recopier ce qu'il ne doit pas.
- **Aucune unité `protege` transférée.** Aucun `transferts` ne porte une `cleUnite` dont
  l'`evenements` est en `protege` — sauf si une libération explicite figure dans
  `journal_appels`, avec son acteur. Le contrôle vérifie la **présence de la trace de
  libération**, pas seulement l'état courant : c'est ce qui distingue une libération légitime
  d'un contournement.
- **Aucune donnée sortie.** Aucune ligne `journal_appels` ne porte une méthode qui ne soit pas
  une méthode Bot API attendue ; la liste des méthodes réellement appelées est rapportée telle
  quelle, ce qui rend visible tout appel inattendu.

### 4.6 Appels, bornes et schéma

- Aucune invocation de `journal_appels` ne dépasse `MAX_APPELS_API` appels (invariant 14).
- Aucune unité n'a plus de `MAX_SUJETS_PAR_UNITE` transferts, ni plus de deux propositions.
- Distribution des codes d'erreur Bot API : une part inhabituelle de `429` ou de `403` est
  rapportée comme signal d'exploitation, pas comme défaut.
- **Objets non documentés** : la sortie de `npx tgcloud migrate --dry-run` liste les objets
  présents en base et absents de `schema.js`. L'audit en base ne peut pas les voir seul ; le
  rapport rappelle donc de lancer cette commande, et la procédure du §5 l'exige.

---

## 5. Reprise après incident (`/audit doctor`)

Préflight à exécuter avant de repasser en `canari` ou `actif` après une panne. Contrôles, dans
l'ordre :

1. `getChat` répond et le salon est bien un forum ; `parametres.chatId` et
   `parametres.inboxThreadId` sont renseignés ;
2. `getChatMember` confirme les droits du bot, `can_manage_topics` compris ;
3. aucun `transferts` en `reclame` dont `expireLe` est dépassé — sinon ils doivent d'abord
   devenir `incertain` ;
4. **aucun `incertain` non tranché** — c'est la condition bloquante : repasser en `actif` avec
   des transferts d'issue inconnue, c'est accepter de ne jamais savoir ;
5. aucune ligne `sujets_journal` `propose` sans clôture ;
6. contrôles §3.1 à §3.4 sur les unités des sept derniers jours ;
7. cohérence des états non terminaux (§4.3) sous les seuils d'alerte ;
8. le webhook est en synchronisation (`npx tgcloud webhook`) et `allowed_updates` correspond
   aux handlers déployés — hors base, donc rappelé dans le rapport plutôt que contrôlé ;
9. `npx tgcloud migrate --dry-run` ne signale aucun changement en attente.

Si l'une de ces conditions échoue, `/audit doctor` rend un avis **négatif et motivé** et
recommande de rester en `ombre` ou en `revue`. Il ne modifie rien lui-même — y compris les
états qu'il juge incohérents.

---

## 6. Correspondance invariant → contrôle

Reprise des invariants de [`classement-des-messages.md`](classement-des-messages.md) §21.

| # | Invariant | Contrôlé par | Contrôlable *a posteriori* ? |
|---|---|---|---|
| 1 | aucune réécriture | contrôle statique du dépôt | non — propriété du code |
| 2 | `message_id` exact | §3.1 | oui |
| 3 | original conservé dans `INBOX` | absence des méthodes mutantes (statique) | **non** — l'API ne relit pas un message (§2.2) |
| 4 | multi-appartenance complète | §3.3 | oui |
| 5 | album atomique, jamais découpé | §3.2 | oui — sur les identifiants, pas sur le rendu |
| 6 | pas de sujet en doublon | §4.4 | oui |
| 7 | renommage humain, journalisé | §4.4 | oui — contrôle absolu |
| 8 | échec fermé | §3.2, §4.3 | oui |
| 9 | secrets non amplifiés | §4.5 | oui |
| 10 | idempotence | §3.3 + contraintes de base | oui |
| 11 | transitions atomiques | — | **non** — propriété d'exécution |
| 12 | `incertain` non auto-résolu | §4.3, §5.4 | oui |
| 13 | trois causes distinctes | §4.3 | oui |
| 14 | appels bornés | §4.6 | oui |
| 15 | lignes non conformes ignorées | §3.4 | oui |
| 16 | autorités versionnées | §3.4, §4.2 | oui |
| 17 | traçabilité | §4.4, §3.1 | oui |
| 18 | aucun secret stocké | §4.5 | oui |
| 19 | rien ne quitte Telegram | §4.5 + contrôle statique | partiellement |
| 20 | pas d'auto-classement | §4.1 | oui |

Les invariants marqués « propriété d'exécution » ou « propriété du code » ne sont pas auditables
sur les données : ils sont garantis par la conception et couverts par la recette
([`classement-des-messages.md`](classement-des-messages.md) §18).

---

## 7. Mise en œuvre et distribution

### 7.1 Contrainte de plateforme

Il n'y a pas d'unité de compilation séparable, pas de binaire distinct, et `schema.js` est un
**fichier unique**. Le composant ne peut donc pas être un artefact indépendant : c'est un
sous-arbre `lib/audit/` plus deux tables. « Optionnel » signifie ici : **retirable sans toucher
au reste**, et vérifié comme tel.

### 7.2 Ce que le composant ajoute

```text
lib/audit/
├─ regles.js       # réimplémentation des règles d'évaluation (§2.1)
├─ controles.js    # §3 et §4
├─ doctor.js       # §5
└─ rapport.js      # rendu texte pour le sujet d'administration
```

Deux tables dans `schema.js` :

- `audit_executions` — `id` (**PK**, auto-incrément), `lanceLe`, `acteur`, `portee`,
  `constats`, `bloquants`, `dureeMs`, `empreinteAutorites` ;
- `audit_constats` — `id` (**PK**, auto-incrément), `executionId`, `controle`, `gravite`
  (`info` | `signal` | `bloquant`), `cible`, `attendu`, `obtenu`.

`{ controle, cible, attendu, obtenu }` — jamais un booléen, jamais un simple compte : un rapport
qui ne dit pas quoi regarder ne sert à rien.

### 7.3 Invocation

- `/audit [tout|transferts|albums|sujets|securite|etats]` — réservé aux administrateurs, rendu
  dans le sujet d'administration, tronqué à une taille de message et complété par un renvoi vers
  `audit_constats` ;
- `/audit doctor` — la procédure du §5 ;
- `npx tgcloud run handlers/message '{ … "/audit tout" … }' --ctx '{ … }'` — même code, sortie
  intégrale par `console.*`, sans poster dans le groupe. C'est la forme à privilégier pour un
  audit complet, puisque `console.*` n'est restitué que là
  ([`classement-des-messages.md`](classement-des-messages.md) §19).

### 7.4 Retrait

`.deprecated('composant d'audit retiré')` sur les deux tables, suppression de `lib/audit/` et de
la route `/audit`, `push` puis `migrate` (deux `warning` confirmés un par un). Le contrôle de
retirabilité fait partie de la recette (§9.7).

---

## 8. Phases de développement

Numérotation séparée de celle du pipeline. V1 peut démarrer dès que la phase 2 du pipeline
(décision, mode `ombre`) est figée ; V2 dépend de la phase 4 (transfert) ; V3 de la phase 5.

### Phase V1 — Réévaluation indépendante et cohérence des décisions

- `lib/audit/regles.js` : réimplémentation des règles de `lib/evaluation.js` depuis leur
  description, sans importer le moteur.
- Contrôles §3.3, §3.4, §4.1, §4.2 ; tables `audit_executions` et `audit_constats` ;
  `/audit` en lecture.
- **Recette :** pour chaque contrôle, un jeu de données sain et au moins un jeu corrompu —
  `classements` sans version, score sous le seuil, slug hors vocabulaire, décision en double
  contournant l'index, transfert orphelin. Chaque injection doit être détectée et attribuée à la
  bonne cible. **Test croisé :** toute décision produite par le pipeline en phase 2 doit passer
  la réévaluation indépendante.

### Phase V2 — Transferts, albums et sécurité

- Contrôles §3.1, §3.2, §4.4, §4.5.
- **Recette :** injections — `messageIdsSource` amputé d'un membre d'album, clé d'idempotence
  incohérente avec la liste des identifiants, deux slugs sur un même `threadId`, ligne
  `sujets_journal` `propose` sans clôture, renommage `ok` avec `acteur: 'bot'`, valeur
  ressemblant à un jeton dans `parametres`, transfert d'une unité `protege` sans trace de
  libération. Chaque injection doit sortir un constat `bloquant` nommant la cible.
- **Le test le plus important :** un album de quatre membres dont un transfert ne porte que
  trois identifiants. C'est la violation de l'invariant 5, elle est invisible dans Telegram, et
  c'est la justification d'existence du composant.

### Phase V3 — États non terminaux et `doctor`

- Contrôles §4.3, §4.6 ; `lib/audit/doctor.js` (§5) avec avis motivé et condition bloquante sur
  les `incertain`.
- **Recette :** jeux de données avec `attente_album` vieilli artificiellement, `incertain` non
  tranché, `bloque_infra` à `tentatives` maximales, réclamation expirée, quarantaine de 40
  jours. `doctor` doit rendre un avis négatif et nommer la condition en défaut, sans rien
  modifier.

---

## 9. Recette du composant

1. **Indépendance réelle** — le contrôle statique refuse tout import interdit depuis
   `lib/audit/`, et refuse tout import de `lib/audit/` depuis le reste du projet hormis la route
   `/audit`. Test de régression : ajouter volontairement `import { evaluer } from 'lib/verdict'`
   dans `lib/audit/regles.js` doit faire échouer l'intégration continue.
2. **Divergence détectée** — muter volontairement `lib/verdict.js` (par exemple inverser une
   comparaison de seuil) : les tests du pipeline peuvent rester verts, mais la réévaluation
   indépendante du §3.4 doit signaler la divergence. Ce test est la justification d'existence du
   composant et il est documenté comme tel.
3. **Matrice de corruption** — pour chaque injection des phases V1 à V3, `/audit tout` doit
   produire un constat nommant le contrôle et la cible. Un audit qui ne détecte pas est pire
   qu'un audit absent, parce qu'il rassure.
4. **Non-mutation** — `/audit tout` sur un jeu de données figé : aucune écriture hors
   `audit_executions` et `audit_constats`, aucun appel Bot API autre que `getChat` et
   `getChatMember`. Contrôlé par comparaison du contenu des tables avant et après, et par
   `journal_appels`.
5. **Bornes** — l'audit respecte lui aussi `MAX_APPELS_API` et se termine toujours, y compris
   sur un corpus volumineux : la portée est bornée par requête (`/audit transferts` plutôt que
   `/audit tout`) et le rapport indique explicitement s'il a été tronqué.
6. **Dérive du vocabulaire** — durcir `lib/evaluation.js` (par exemple `seuilScore: 90`),
   `push`, puis `/audit tout` : les décisions prises sous l'ancienne version doivent être
   signalées, **sans être modifiées**. C'est le signal qui dit qu'une reprise s'impose.
7. **Retirabilité** — après suppression de `lib/audit/` et de la route `/audit`, `npx tgcloud
   push` réussit, le bot fonctionne, et `/audit` répond par une erreur explicite « composant non
   installé » — jamais par un silence ni par une exception.
