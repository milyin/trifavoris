# trifavoris

Bot Telegram spécialisé dans le classement **non destructif** des messages d'un sujet `INBOX`
vers des sujets thématiques larges et réutilisables, à l'intérieur d'un même supergroupe forum.

Il tourne intégralement sur **[Telegram Serverless](https://core.telegram.org/bots/serverless)** :
des modules JavaScript déployés par `tgcloud`, une base SQLite intégrée, l'API Bot et `fetch`
comme seule bibliothèque. Pas de serveur, pas de paquet npm au runtime, pas de système de
fichiers.

## Le principe, en trois phrases

- **Classer, c'est transférer.** Le bot ne réécrit rien, ne résume rien, ne découpe aucun album,
  ne recrée aucun message. Il appelle `forwardMessage` / `forwardMessages` sur le message
  original, intégral et inchangé — texte, album, médias, auteur et source compris, quand
  Telegram les expose.
- **L'original ne bouge jamais.** Il reste dans `INBOX`. Un message qui relève de trois sujets
  est transféré trois fois, entièrement, dans chacun : il n'y a pas de catégorie « gagnante ».
- **En cas de doute, on ne publie rien.** Identifiant de message incertain, album non
  démontrablement complet, attribution manquante, capacité de transfert refusée, secret
  manifeste : le bot échoue fermé, conserve une preuve en base, et laisse un humain trancher.

Les sujets sont volontairement **larges** (`Applications utiles`, `Développement de logiciels`,
`Outils IA`). Un sujet existant est réutilisé ; s'il est trop spécifique, il est **renommé** pour
élargir son périmètre — jamais doublé. Tout renommage est proposé par le bot et **appliqué par un
humain**.

## Pas de LLM en v1, et pourquoi

La documentation de Telegram Serverless **ne décrit aucun mécanisme de secret d'exécution** :
pas de variable d'environnement pour les modules déployés, pas de coffre. Appeler une API LLM
externe exigerait d'écrire une clé dans un module déployé, donc dans le dépôt — ce que ce projet
s'interdit.

La v1 est donc **entièrement déterministe** : des règles versionnées produisent des lignes de
tags, un évaluateur déterministe décide. L'architecture réserve la place d'un modèle — même
format de sortie dégradable, même parseur, même cache adressé par empreinte, même borne dure sur
le nombre d'appels — mais l'extension est **conditionnelle** à l'apparition d'un mécanisme de
secret sûr documenté, ou d'un service de classification authentifié sans secret embarqué. Voir
[`doc/classement-des-messages.md`](doc/classement-des-messages.md) §12.

**Aucun secret n'est stocké en base ni dans le dépôt.**

## État

**Conception.** Ce dépôt ne contient pour l'instant aucun code d'exécution : ni `handlers/`, ni
`lib/`, ni `schema.js`. **Le code viendra dans une pull request ultérieure**, suivant les phases
livrables décrites dans la conception.

La conception est découpée en deux documents, correspondant aux deux composants du projet :

| Document | Composant |
|---|---|
| [`doc/classement-des-messages.md`](doc/classement-des-messages.md) | Le pipeline `INBOX → sujets` : ingestion, décision, transfert — **obligatoire** |
| [`doc/verification.md`](doc/verification.md) | L'audit indépendant de l'état du bot — **optionnel** |

Le bot classe sans le second composant : il conserve des contrôles en ligne non désactivables —
contraintes d'unicité en base, écriture avant et après chaque appel mutatif, échec fermé sur
chaque contrôle d'admissibilité. Le composant de vérification apporte autre chose : un audit
*a posteriori* de tout l'état, réimplémenté indépendamment du code qui l'a produit, et la
détection des dérives qui ne sont graves que par leur durée.

[`AGENTS.md`](AGENTS.md) rassemble les contraintes de plateforme que toute contribution — humaine
ou automatique — doit respecter.

## Prérequis prévus

- [Node.js](https://nodejs.org) 18 ou plus récent, pour le CLI `tgcloud` (`npm create @tgcloud/bot`).
- Un bot enregistré auprès de [@BotFather](https://t.me/BotFather), avec **Serverless activé**.
- Un **supergroupe forum** dans lequel le bot est administrateur avec le droit
  `can_manage_topics`, et qui contient au moins un sujet `INBOX` et un sujet d'administration.

Aucun service tiers, aucune clé d'API : la v1 ne fait aucun appel sortant.

## Licence

À définir.
