# Mise en œuvre de Black Duck Seeker (IAST) sur Kubernetes avec PostgreSQL externe

**Guide de compréhension, configuration et administration — Infrastructure on-premise**

| Métadonnée | Valeur |
|---|---|
| Responsable du guide | À désigner — Lead technique / DevSecOps |
| Public | DevSecOps, plateforme Kubernetes, DBA, développeurs, QA et AppSec |
| Édition documentaire | 1.0 — 31 octobre 2026 |
| Cible | Seeker sur Kubernetes on-premise ; PostgreSQL self-hosted administré par l’équipe |
| Statut | Guide de référence et de qualification ; validation locale obligatoire avant production |
| Format de maintenance | Markdown ; publication possible dans Confluence ou Notion |

> **Décision d’architecture proposée.** Déployer les composants officiels Seeker avec le chart fourni par l’installeur ; exploiter une base PostgreSQL indépendante du cycle de vie Seeker avec CloudNativePG ; exposer uniquement le frontal Seeker en HTTPS sur le réseau interne. PostgreSQL reste « externe » au sens de Seeker, même s’il tourne dans le même cluster Kubernetes.

> **Point de vigilance sur les versions.** Les exemples historiques mentionnant PostgreSQL 10.10 ou une prise en charge limitée à 10–14 ne doivent pas servir de référence pour une nouvelle installation. La page officielle PostgreSQL consultée, dans la branche `latest`, annonce PostgreSQL 14 ou supérieur, y compris 18. Cette information n’est pas une certification rétroactive pour une ancienne release Seeker : archiver les exigences de la version réellement achetée et vérifier également le support amont PostgreSQL et CloudNativePG. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/postgresql-database-requirements.html)

## Mode d’emploi du guide

Les mentions **OBLIGATOIRE**, **RECOMMANDÉ** et **OPTIONNEL** représentent les exigences proposées pour cette plateforme interne, sauf attribution explicite à l’éditeur. Un exemple n’est pas une preuve d’exécution en environnement réel : aucune installation sur l’infrastructure cible n’a été réalisée dans le cadre de ce document.

Les marqueurs `<VERSION_SEEKER>`, `<CHART_VERSION_CNPG>`, `<IMAGE_POSTGRES_VALIDEE>`, `<STORAGE_CLASS>` et `<FQDN_SEEKER>` désignent des valeurs à fournir, pas des versions ou noms de ressources présumés. Les fichiers et scripts `ci/*` proposés plus loin sont des développements internes à réaliser ; ils ne sont pas des exécutables Black Duck.

La branche documentaire `latest` est évolutive. Avant une installation, conserver dans Git les fichiers non sensibles du package livré, leurs empreintes, les versions retenues, les valeurs Helm et les décisions d’architecture ; conserver les secrets dans le coffre-fort de l’entreprise.

## Sommaire

1. Résumé exécutif et positionnement
2. Fonctionnement et limites de Seeker
3. Fonctionnalités et bénéfices
4. Architecture cible on-premise
5. Prérequis et dossier de préparation
6. PostgreSQL self-hosted : installation et configuration
7. Seeker : installation officielle sur Kubernetes
8. Agents : intégration et validation
9. CI/CD et politiques de sécurité
10. Administration et exploitation
11. Sécurité et durcissement
12. Montée en compétence et recette
13. Annexes et runbooks
14. Checklist Go-Live de 15 points

## 1. Résumé exécutif

### Comprendre l’outil

Black Duck Seeker est une solution d’analyse interactive de sécurité applicative, ou IAST. Elle observe une application exécutée et instrumentée pendant les tests fonctionnels, API ou manuels afin de détecter des vulnérabilités avec leur contexte runtime ; elle ne se réduit pas à un scan du dépôt Git. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/seeker-overview.html)

Pour le lead technique, le résultat attendu est une chaîne opérationnelle : une plateforme disponible, des agents correctement raccordés, des tests représentatifs, des résultats attribuables à une version applicative, puis un processus de correction et de retest. Le périmètre recommandé est d’abord un environnement de test ou de recette isolé ; l’utilisation sur des applications servant de vrais utilisateurs exige une analyse séparée des risques et du support éditeur.

### Place dans le SDLC

| Approche | Objet observé | Positionnement proposé | Limite à retenir |
|---|---|---|---|
| SAST | Code source ou représentation compilée | Pull requests et build | Un résultat statique ne prouve pas, seul, l’exécution du parcours. |
| SCA | Composants tiers et dépendances | Build et gouvernance open source | Ne remplace pas l’analyse des failles du code métier. |
| DAST | Application en fonctionnement, depuis l’extérieur | Recette et vérification de surface exposée | Visibilité interne du code généralement limitée. |
| IAST / Seeker | Exécution instrumentée et flux de données | Tests d’intégration, API et E2E | Dépend des technologies compatibles et des parcours exercés. |

Ces méthodes sont complémentaires : la documentation Seeker distingue l’analyse statique, l’analyse des composants et l’instrumentation runtime. L’organisation ci-dessus est une proposition de chaîne DevSecOps, non une obligation de licence. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/static-application-security-testing-sast-.html) · [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/software-composition-analysis-sca-.html) · [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/instrumentation.html)

### Responsabilités du lead

| Acteur | Responsabilité proposée | Livrable attendu |
|---|---|---|
| Lead technique Seeker | Architecture, configuration, recette, support et coordination | Dossier technique, runbooks et indicateurs |
| Équipe plateforme | Kubernetes, ingress, CSI, réseau et observabilité | Services d’infrastructure exploitables |
| DBA ou référent PostgreSQL | Base, performances, sauvegarde et restauration | Procédure PRA et contrôles réguliers |
| AppSec | Politiques, arbitrage des risques et exceptions | Règles de triage et de gate |
| Développeurs | Intégration agent et remédiation | Application instrumentée, correctifs testés |
| QA | Couverture des parcours et comptes de test | Suites de tests représentatives |

Le rôle d’administration Seeker décrit par l’éditeur inclut l’installation, les mises à jour et la maintenance du serveur et des agents. La répartition entre ces personnes est à formaliser localement pour éviter qu’une seule personne détienne les accès et la connaissance d’exploitation. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/roles-and-personas.html)

## 2. Fonctionnement de Seeker

### Composants réels

Le déploiement Kubernetes documenté ne correspond pas à un unique conteneur générique. Il comporte notamment un frontal **NGINX**, un composant **Server** et un composant **Sensor**, plus PostgreSQL lorsque la base est gérée par Seeker ; en mode base externe, celle-ci est exploitée séparément. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html)

| Composant | Rôle et attention opérationnelle |
|---|---|
| Seeker Agent | Instrumente le runtime de l’application et remonte les détections et informations associées. |
| NGINX | Point d’entrée des agents, de la console et des intégrations ; transporte aussi les communications WebSocket. |
| Sensor | Composant de traitement lié aux communications d’instrumentation ; conserver la configuration fournie par le package. |
| Server | Centralise les résultats et les fonctions de gestion de la plateforme. |
| PostgreSQL | Persistance ; le diagramme éditeur indique des connexions depuis Server et Sensor. |
| Application testée | Traite les requêtes des tests ; conserve sa propre base et ses propres dépendances métier. |

Cette décomposition et les flux doivent être conservés lors du durcissement réseau ; l’agent n’a pas à connaître le mot de passe PostgreSQL de Seeker. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/seeker-architecture-and-communications.html) · [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/agent.html)

### Instrumentation et flux

L’instrumentation observe notamment les entrées HTTP et la propagation des données jusqu’aux opérations sensibles, par exemple les accès à la base ou au système de fichiers. Un checker applique une règle de détection à ce contexte runtime ; la présence et la couverture des checkers dépendent de l’agent et de sa version. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/instrumentation.html) · [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/features-and-capabilities.html)

```text
Tests QA / API / E2E
        |
        v
Requête HTTP -> Contrôleur -> Service -> Appel SQL / fichier / HTTP sortant
        |          |            |                 |
        +----------+------------+-----------------+
                   Agent instrumentant le runtime
                                  |
                                  v
                   Frontal Seeker -> Sensor / Server -> Résultats
```

**Exemple pédagogique :** un paramètre de recherche atteint une requête SQL construite sans protection adaptée. L’IAST peut associer l’entrée et l’opération SQL observée ; l’équipe analyse ensuite la preuve et corrige le code. Ne pas introduire volontairement une application vulnérable sur un réseau partagé : le laboratoire doit être isolé.

### Vérification active

Seeker peut produire une requête dérivée avec des paramètres adaptés, la renvoyer vers l’application instrumentée et observer si la donnée atteint le même point sensible. Ce mécanisme n’est pas disponible pour tous les checkers et peut ne pas aboutir, par exemple avec un jeton à usage unique expiré. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/about-active-verification.html)

| Tag technique documenté | Interprétation opérationnelle |
|---|---|
| `Seeker-Verified` | La vérification active a confirmé le comportement vulnérable selon le checker. |
| `Seeker-Invalidated` | Le mécanisme a invalidé la détection dans les conditions observées. |
| `Seeker-Fix-Confirmed` | Le mécanisme a confirmé une correction dans une nouvelle version applicative. |
| Aucun tag de vérification | Ne signifie ni « sûr » ni « faux positif » ; examiner le contexte. |

Ces tags sont distincts de la sévérité, du statut de traitement et d’une acceptation métier du risque. La documentation permet aussi d’exclure des zones de l’application et de contrôler les requêtes pouvant modifier les données. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/about-active-verification.html)

**OBLIGATOIRE — maîtrise des effets de bord :** utiliser des données synthétiques, neutraliser paiements/envois réels, définir les exclusions sensibles, puis vérifier les paramètres de vérification active avant la première campagne. L’IAST n’est pas totalement passif dès lors que cette vérification rejoue des requêtes. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/about-active-verification.html)

### Langages et compatibilité

Les ressources officielles décrivent notamment Java/JVM, .NET, Node.js ainsi que des évolutions d’agents Python, Go et PHP. Il ne faut pas en déduire que tous les runtimes, frameworks ou checkers sont équivalents : utiliser l’**Agent Technical Specification** et le **Connect Agent Wizard** de la release installée. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/features-and-capabilities.html) · [Source](https://docs.blackduck.com/r/seeker/latest/release-notes/release-notes-2025.9.0.html) · [Source](https://docs.blackduck.com/r/seeker/latest/release-notes/release-notes-2025.4.0.html)

| Technologie | Principe à comprendre | Qualification locale requise |
|---|---|---|
| Java / JVM | Chargement d’un agent dans la JVM ; distribution fournie par Seeker | JDK, framework, serveur, autres agents APM et image |
| .NET | Instrumentation du runtime CLR | .NET Framework / .NET, OS, architecture, mécanisme de démarrage |
| Node.js | Chargement de l’instrumentation au démarrage | Node.js, CommonJS/ESM, ordre de chargement et framework |
| Python, Go, PHP | Intégration propre à chaque agent | Instructions exactes, mode de packaging et checkers pris en charge |

Pour Java et .NET, l’attachement concerne le runtime de l’application, pas un simple conteneur sidecar qui analyserait magiquement les processus voisins. Pour Node.js, les instructions ESM ont évolué vers `--import` dans des releases documentées : ne pas réutiliser une commande ancienne sans validation. [Source](https://community.blackduck.com/s/article/Seeker-Attaching-an-Agent) · [Source](https://docs.blackduck.com/r/seeker/latest/release-notes/release-notes-2025.4.0.html)

### Limites à expliquer aux équipes

- **Couverture :** l’absence de détection ne prouve pas l’absence de vulnérabilité ; vérifier que le parcours a été exercé et observé.
- **Vérification :** une alerte non vérifiée reste à analyser ; ne pas la classer automatiquement comme faux positif.
- **Autorisation métier :** prévoir des tests multi-rôles et multi-utilisateurs ; aucun outil ne connaît à lui seul toutes les règles du produit.
- **Performances :** mesurer les coûts CPU, mémoire et latence avant/après instrumentation ; ne pas promettre un pourcentage d’overhead générique.
- **HA :** trois instances PostgreSQL ne rendent pas automatiquement les composants Seeker hautement disponibles.

## 3. Fonctionnalités principales

| Fonctionnalité | Bénéfice | Cas d’usage et limite |
|---|---|---|
| Détection runtime | Contexte d’exécution au moment des tests | Campagnes QA instrumentées ; pas un audit exhaustif automatique. |
| Vérification active | Meilleure confiance dans certaines alertes | Priorisation des résultats vérifiés ; disponibilité selon checker. |
| Flux de données et code | Comprendre l’origine et la propagation du risque | Investigation et correction avec le développeur. |
| Conseils de remédiation | Raccourcir l’investigation | Revue du correctif puis retest ; conseil à contextualiser. |
| Découverte API | Inventaire des routes et visibilité sur les échanges | REST, GraphQL et autres interfaces selon support effectif. |
| Sécurité API / JWT | Détection de classes de risques d’authentification et d’autorisation | Coupler aux tests avec comptes et rôles représentatifs. |
| Données sensibles | Repérer des paramètres selon règles intégrées ou définies | Minimisation et contrôle des exports. |
| SCA via BDBA | Détecter des vulnérabilités connues de composants tiers | Intégration et licence à qualifier ; appliance séparée si retenue. |
| Référentiels | Rattacher les résultats à des catégories reconnues | OWASP et standards disponibles dans la release ; pas une certification. |
| Gestion et reporting | Suivre les projets et exporter les résultats | Backlog, tendances et rapports restreints aux personnes habilitées. |
| API et CI/CD | Automatiser les opérations | Contrat API propre à Seeker ; ne pas substituer Detect/Bridge SCA sans preuve de support. |

Les fonctions runtime, remédiation, données sensibles, SCA intégrée via Black Duck Binary Analysis et exports sont décrites dans le guide produit. La SCA n’est donc pas à présenter comme inexistante, ni comme autonome sans composant supplémentaire : le déploiement et l’habilitation BDBA sont à vérifier. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/features-and-capabilities.html)

La page produit et les ressources Black Duck décrivent les API REST/GraphQL, la découverte des interfaces et des contrôles liés à JWT et à l’autorisation. Ces descriptions ne remplacent pas la matrice des checkers propre au runtime ciblé. [Source](https://www.blackduck.com/interactive-application-security-testing.html) · [Source](https://www.blackduck.com/blog/appsec-decoded-continuous-appsec-testing-in-devsecops-with-seeker-iast.html)

**RECOMMANDÉ — mesure de couverture :** rapprocher l’inventaire d’API Seeker de l’OpenAPI de référence, des routes attendues et des scénarios QA. Distinguer une route découverte d’une route réellement testée avec un compte ayant les droits appropriés.

## 4. Architecture cible on-premise

### Schéma de référence

```text
Developpeur / QA -> CI/CD -> Cluster Kubernetes on-premise
                              |
                              +-- namespace app-test
                              |     Application + Seeker Agent
                              |          | HTTPS / WebSocket
                              |          v
                              +-- Ingress interne / PKI entreprise
                              |          |
                              +-- namespace seeker
                              |     NGINX Seeker
                              |       |        |
                              |     Server   Sensor
                              |       |        |
                              |       +---+----+
                              |           | TCP 5432 / TLS valide
                              +-- namespace seeker-db
                              |     Service seeker-postgres-rw
                              |       |        |        |
                              |     primaire replique replique
                              |       PVC      PVC      PVC
                              |        \        |       /
                              +---------\-------|------/----------+
                                         Sauvegardes + WAL
                                                |
                                      S3 interne / MinIO qualifie
                                      + copie hors domaine de panne
```

**Choix proposé :** CloudNativePG dans `cnpg-system`, données dans `seeker-db`, composants Seeker dans `seeker`. Pour une cible HA PostgreSQL, prévoir trois instances sur des workers distincts et des domaines de panne autant que possible distincts. Pour un laboratoire, une instance est acceptable avec une absence de HA explicitement documentée.

CloudNativePG gère le primaire, les répliques, les PVC et le service de connexion lecture/écriture. Il ne repose pas sur un StatefulSet utilisateur pour orchestrer ce cluster ; ne pas modifier ses pods manuellement comme s’il s’agissait d’une installation PostgreSQL artisanale. [Source](https://cloudnative-pg.io/documentation/1.24/)

### Ressources Kubernetes

| Ressource | Usage proposé | Règle |
|---|---|---|
| Namespace | Séparer plateforme, données et applications | OBLIGATOIRE |
| Chart Seeker | Générer les Deployments/Services/ConfigMaps requis | Utiliser celui du package qualifié |
| CR `Cluster` CNPG | Décrire PostgreSQL | Ne pas cumuler avec un StatefulSet PostgreSQL manuel |
| Secrets | Mots de passe, certificats, accès registre | Restreindre lecture, stockage et rotation |
| ConfigMaps | Configuration non sensible | Ne pas créer de variables Seeker non documentées |
| Service ClusterIP | Communication interne | Pas d’exposition publique PostgreSQL |
| Ingress | UI/API/agents, HTTPS et WebSocket | Contrôleur et timeouts à qualifier |
| PVC | Données PostgreSQL et volumes Seeker exigés par le chart | Inventaire, expansion et restauration |
| Probes | Startup, readiness et liveness | Garder les sondes du package avant personnalisation |
| NetworkPolicy | Contrôle des échanges | Ajouter tous les flux opérateur, DNS et sauvegarde |

### Flux à autoriser

| Source | Destination | Port / protocole | Remarque |
|---|---|---|---|
| Utilisateurs, agents, CI/CD | Ingress interne | 443 / HTTPS, WebSocket | URL et CA d’entreprise |
| Ingress | Frontal NGINX Seeker | Port HTTPS réel du Service | Lire les manifests rendus |
| NGINX Seeker | Sensor / Server | Ports internes du package | La documentation indique 8082 / 9911 par défaut |
| Server et Sensor | PostgreSQL primaire | 5432 / TCP avec TLS qualifié | Les deux consommateurs doivent fonctionner |
| Server | LDAP / SSO / SMTP | Selon service validé | LDAPS, SAML 2.0 et SMTP documentés |
| Server | BDBA, Jira ou autres intégrations | HTTPS selon besoin | OPTIONNEL et explicitement autorisé |
| Opérateur CNPG | Instances PostgreSQL | 8000 / TCP | Contrôle d’état par l’opérateur |
| Instances PostgreSQL | Instances PostgreSQL | 5432 / TCP | Réplication et administration selon opérateur |
| PostgreSQL / sidecar backup | Stockage objet | HTTPS selon endpoint | Certificat et droits bucket dédiés |
| Pods concernés | DNS, API Kubernetes | Ports locaux à identifier | API requise par opérateur et instance manager |
| Prometheus | PostgreSQL exporter | 9187 / TCP selon configuration | Ne pas exposer sur Internet |

Les ports internes Seeker sont des valeurs par défaut documentées, pas une instruction d’exposition publique. Les ports CloudNativePG et ses besoins d’API doivent être pris en compte avant de poser un « deny all ». [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/seeker-architecture-and-communications.html) · [Source](https://cloudnative-pg.github.io/docs/1.26/security/)

### Stockage et disponibilité

**OBLIGATOIRE :** qualifier le CSI et la StorageClass avec l’équipe plateforme : latence, IOPS, `fsync`, modes d’accès, expansion, reprise après panne, politique de reclaim et chiffrement. Ne pas valider ou exclure un stockage uniquement parce qu’il porte le nom Ceph, Longhorn, vSphere, SAN ou NFS.

**RECOMMANDÉ :** un PVC par instance PostgreSQL ; des répliques réparties sur des workers et hyperviseurs distincts ; sauvegardes dans un domaine de panne indépendant. La réplication n’est pas une sauvegarde, et une sauvegarde stockée uniquement sur les mêmes disques ne couvre pas la perte de ces disques.

Ne pas promettre un RPO nul sur la seule présence de répliques. Qualifier le mode de réplication retenu et les résultats des exercices de panne en tenant compte de la latence et de la disponibilité attendue.

## 5. Prérequis

### Fiche de qualification

| Paramètre à renseigner | Valeur attendue avant changement |
|---|---|
| Version / build Seeker et licence | Version du package réellement fourni |
| Kubernetes et distribution | Release maintenue, validée pour Seeker et CNPG |
| Versions opérateur / chart / plugin | Versions figées et matrice de support archivée |
| PostgreSQL | Version croisant support Seeker, CNPG et PostgreSQL amont |
| Registre privé | FQDN, certificat, proxy, authentification et droits push/pull |
| Stockage | StorageClass, capacité par instance, marge et expansion |
| URL Seeker | FQDN interne, DNS, SAN, certificat, propriétaire du renouvellement |
| Ingress | Contrôleur, mode TLS backend, WebSocket et timeouts |
| Identité | Annuaire ou SAML 2.0 si retenu ; compte de secours |
| CI/CD | Outil et runners autorisés à joindre les environnements |
| PRA | RPO/RTO approuvés, copie hors domaine de panne et responsable |
| Applications pilotes | Runtime, framework, APM, routes, jeux de données et propriétaires |

**OBLIGATOIRE :** disposer de `kubectl`, Helm qualifié, d’un poste/bastion Linux pour l’installeur et de droits limités mais suffisants. La page Kubernetes Seeker consultée cite Kubernetes 1.19+ et Helm 2.x/3.x ; ce minimum historique n’est pas une recommandation d’utiliser ces branches anciennes en production en 2026. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html)

### Dimensionnement

Le nombre d’agents simultanés, le débit de tests, la rétention et le volume de résultats déterminent la capacité. Le guide éditeur distingue plusieurs tailles et fournit des fichiers de dimensionnement à appliquer au chart ; les chiffres suivants proviennent de sa branche `latest`, pas d’une mesure sur l’infrastructure cible. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/sizing-guidelines.html)

| Élément Kubernetes | Small, jusqu’à 100 agents | Medium, jusqu’à 500 agents |
|---|---|---|
| Server | 1,5 CPU ; 4 GB mémoire | 2 CPU ; 8 GB mémoire |
| Sensor | 1,5 CPU ; 4 GB mémoire | 2 CPU ; 8 GB mémoire |
| NGINX | 0,5 CPU ; 0,5 GB mémoire | 1 CPU ; 0,5 GB mémoire |
| Base, référence mono-instance du guide | 1 CPU ; 2 GB mémoire | 1,5 CPU ; 4 GB mémoire |
| Volume persistant de référence | 100 GB | 150 GB |

Ces valeurs constituent un ordre de dimensionnement éditeur, pas des `requests/limits` prêts à recopier, et ne couvrent pas automatiquement trois instances CNPG, ses sidecars, le stockage objet et les autres charges du cluster. Les unités GB du guide ne sont pas identiques aux unités GiB des exemples Kubernetes. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/sizing-guidelines.html)

### Préparation hors ligne

**RECOMMANDÉ si Internet est filtré :** préparer une liste d’artefacts incluant installeur Seeker, package Docker/Helm téléchargé par l’installeur, images Seeker, opérateur CNPG, images PostgreSQL, plugin Barman, cert-manager et images de diagnostic. Répliquer les artefacts autorisés dans le registre interne, distribuer les CA aux nœuds/runtimes et tester le pull depuis un worker sans accès Internet.

## 6. PostgreSQL self-hosted

### Choix d’hébergement

La cible de ce guide est **CloudNativePG on-premise**. Une base sur VM dédiée gérée par l’équipe reste une alternative si celle-ci dispose déjà d’une plateforme PostgreSQL maîtrisée ; ne pas déplacer une responsabilité DBA dans Kubernetes sans compétence et procédures associées.

| Option | Usage proposé | Point d’attention |
|---|---|---|
| CNPG dans Kubernetes | Cible privilégiée si l’opérateur est accepté | Responsabilité du stockage, de la sauvegarde et du PRA |
| PostgreSQL sur VM | Alternative self-hosted valide | Exploitation OS, HA et réseau à assurer |
| Chart PostgreSQL tiers, dont Bitnami | Alternative à qualifier | Support, images disponibles et procédures exactes à vérifier |
| RDS, Cloud SQL, Azure Database | Hors cible self-hosted retenue | À réévaluer uniquement si la stratégie change |

### Exigences Seeker vérifiées

La documentation PostgreSQL Seeker consultée précise UTF-8, des locales compatibles UTF-8/anglais, l’utilisation du schéma déterminé par le `search_path` et des minima de configuration : `shared_buffers=512MB`, `work_mem=32MB`, `max_connections=120`. Elle n’impose pas un superutilisateur pour le fonctionnement normal. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/postgresql-database-requirements.html)

Elle distingue des droits élevés pour des fonctions particulières : utilisation de `pgstattuple` avec superutilisateur et inclusion de petites bases dans un diagnostic avec `pg_read_all_settings` ou superutilisateur. Ne pas accorder ces droits par défaut et ne pas présenter `pgcrypto` comme une dépendance Seeker établie par cette page. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/postgresql-database-requirements.html)

**OBLIGATOIRE :** contrôler la mémoire réelle sous concurrence. `work_mem` ne constitue pas une enveloppe globale unique ; ne pas multiplier des valeurs de tuning sans mesure. Les minima Seeker doivent être conciliés avec les limites mémoire, les requêtes simultanées et les processus PostgreSQL.

### Installation de l’opérateur

Exemple à exécuter après substitution de la version du chart validée. Le dépôt et le nom du chart correspondent au projet officiel CloudNativePG ; en environnement filtré, utiliser leur copie approuvée dans le dépôt interne. [Source](https://cloudnative-pg.github.io/charts)

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace \
  --version <CHART_VERSION_CNPG> \
  --wait --timeout 10m

kubectl -n cnpg-system get deployments,pods
kubectl get crd clusters.postgresql.cnpg.io
```

### Namespaces et secret applicatif

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: seeker
---
apiVersion: v1
kind: Namespace
metadata:
  name: seeker-db
```

**Choix proposé :** laisser CNPG générer initialement le secret applicatif, puis organiser sa rotation et son éventuelle synchronisation avec le coffre-fort. Cela évite de saisir un mot de passe dans une ligne de commande. CNPG peut créer automatiquement les mots de passe et certificats ; un secret fourni est également possible si son `username` correspond au propriétaire demandé. [Source](https://cloudnative-pg.github.io/docs/1.26/security/)

Schéma de secret fourni, seulement si une injection sécurisée est retenue ; ne pas appliquer cette valeur littérale :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: seeker-postgres-app-user
  namespace: seeker-db
type: kubernetes.io/basic-auth
stringData:
  username: seeker
  password: "<INJECTION_PAR_GESTIONNAIRE_DE_SECRETS>"
```

### Cluster PostgreSQL

Exemple d’ingénierie pour une base dédiée. Les ressources et `100Gi` sont des valeurs initiales proposées pour un pilote avec marge, pas une certification de capacité. Le tag/digest de l’image et la locale doivent être vérifiés dans l’image qualifiée. Ne pas appliquer avant d’avoir substitué les marqueurs.

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: seeker-postgres
  namespace: seeker-db
spec:
  instances: 3
  imageName: "<REGISTRE_INTERNE>/<IMAGE_POSTGRES_VALIDEE>@sha256:<DIGEST>"
  enableSuperuserAccess: false
  bootstrap:
    initdb:
      database: seekerdb
      owner: seeker
      encoding: UTF8
      localeCType: en_US.UTF-8
      localeCollate: en_US.UTF-8
      # Si un secret fourni est retenu, ajouter :
      # secret:
      #   name: seeker-postgres-app-user
  storage:
    storageClass: "<STORAGE_CLASS>"
    size: 100Gi
  resources:
    requests:
      cpu: "1"
      memory: 4Gi
    limits:
      cpu: "2"
      memory: 8Gi
  affinity:
    podAntiAffinityType: required
  postgresql:
    parameters:
      shared_buffers: 512MB
      work_mem: 32MB
      max_connections: "120"
    pg_hba:
      - hostnossl seekerdb seeker all reject
      - hostssl seekerdb seeker <CIDR_CLIENTS_EFFECTIF> scram-sha-256
      - host seekerdb seeker all reject
```

La séquence `pg_hba` proposée refuse le trafic non chiffré pour le compte applicatif, autorise les sources attendues en TLS puis interdit les autres sources pour ce compte. Inspecter le fichier effectivement généré et l’ordre des règles de la version CNPG retenue ; les autres règles opérateur/réplication ne doivent pas être supprimées. PostgreSQL applique la première règle correspondante, sans repli vers une règle suivante en cas d’échec. [Source](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

**OBLIGATOIRE — avant démarrage Seeker :** faire valider la configuration TLS cliente Seeker, y compris Server et Sensor. Les noms de paramètres TLS du client Seeker n’ont pas été établis dans les pages publiques utilisées ici ; il ne faut pas inventer une variable `SEEKER_DATABASE_SSLMODE`. Si cette partie ne peut pas être qualifiée avec le package ou le support, le passage en production est bloqué.

```bash
kubectl apply -f 00-namespaces.yaml
kubectl apply --dry-run=server -f 10-postgresql-cluster.yaml
kubectl apply -f 10-postgresql-cluster.yaml

kubectl -n seeker-db get clusters.postgresql.cnpg.io
kubectl -n seeker-db get pods -o wide
kubectl -n seeker-db get svc,pvc
kubectl -n seeker-db wait --for=condition=Ready \
  cluster/seeker-postgres --timeout=20m
```

En génération automatique, vérifier le nom du secret applicatif, usuellement `seeker-postgres-app`, sans afficher son contenu. Un secret Kubernetes est encodé en base64 et n’est pas chiffré par ce seul mécanisme ; activer le chiffrement au repos et des RBAC adaptés. [Source](https://cloudnative-pg.github.io/docs/1.26/security/) · [Source](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

### Création SQL sur VM

Ne pas exécuter cette variante après le bootstrap CNPG : celui-ci crée déjà le propriétaire et la base. Sur une instance administrée manuellement, utiliser un compte DBA et `psql` interactif ; la commande `\password` évite d’écrire le secret en clair dans le SQL du guide.

```sql
CREATE ROLE seeker LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE;
\password seeker

CREATE DATABASE seekerdb
  OWNER seeker
  TEMPLATE template0
  ENCODING 'UTF8'
  LC_COLLATE 'en_US.UTF-8'
  LC_CTYPE 'en_US.UTF-8';

REVOKE CONNECT ON DATABASE seekerdb FROM PUBLIC;
GRANT CONNECT ON DATABASE seekerdb TO seeker;
\connect seekerdb
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
GRANT USAGE, CREATE ON SCHEMA public TO seeker;
ALTER ROLE seeker IN DATABASE seekerdb SET search_path = public;
```

Créer la base avec `OWNER seeker` est plus précis qu’un simple `GRANT ALL PRIVILEGES ON DATABASE` qui ne règle pas à lui seul tous les droits de schéma. La syntaxe propriétaire/encodage/locale est documentée par PostgreSQL ; le schéma choisi doit correspondre au `search_path` attendu par Seeker. [Source](https://www.postgresql.org/docs/current/sql-createdatabase.html) · [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/postgresql-database-requirements.html)

### Vérification SQL et TLS

Dans un poste de diagnostic autorisé, utiliser une invite de mot de passe `-W` ou un fichier d’identification protégé, jamais `PGPASSWORD=<secret>` passé dans l’historique. Monter la CA dans l’environnement de diagnostic et vérifier que le nom DNS choisi figure dans les SAN du certificat.

```bash
psql "host=<DNS_SERVICE_PRIMAIRE> port=5432 dbname=seekerdb \
user=seeker sslmode=verify-full sslrootcert=/certs/ca.crt" -W
```

```sql
SELECT version(), current_database(), current_user;
SHOW server_encoding;
SHOW lc_collate;
SHOW lc_ctype;
SHOW search_path;
SHOW shared_buffers;
SHOW work_mem;
SHOW max_connections;
SELECT ssl, version, cipher
FROM pg_stat_ssl WHERE pid = pg_backend_pid();
```

`verify-full` vérifie la chaîne de confiance et le nom du serveur ; `require` seul ne doit pas être présenté comme équivalent. Ce contrôle `psql` prouve la configuration du client de diagnostic, pas encore celle des clients Seeker : vérifier séparément leurs sessions effectives. [Source](https://www.postgresql.org/docs/current/libpq-ssl.html)

## 7. Installation Seeker sur Kubernetes

### Obtenir le package officiel

Utiliser l’installeur Linux correspondant à la version et à l’architecture retenues, fourni par Black Duck. L’option Kubernetes/Docker Compose télécharge et extrait un répertoire `seeker_docker` contenant notamment les éléments de construction des images et le chart ; il ne faut pas inventer un dépôt Helm public Seeker ou un nom d’image à partir d’un exemple. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html)

```bash
# Sur le bastion de construction autorisé, avec un compte non-root :
sh ./seeker-server-linux64-<VERSION_SEEKER>.sh
# Sélectionner le type et l’option de déploiement Kubernetes proposés.
# Vérifier ensuite le contenu exact du package extrait.

cd <CHEMIN_PACKAGE>/seeker_docker/images
export DOCKER_REGISTRY=<REGISTRE_INTERNE>
./build.sh
```

Le script `build.sh` construit et publie les images selon le package officiel. Pour une plateforme sans accès Internet, préparer les téléchargements et les artefacts en amont ; le seul fait de disposer de l’installeur ne garantit pas que toute la chaîne de construction est hors ligne. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html)

**Contrôles proposés :** noter les empreintes de l’installeur et du package, analyser les images, relever leurs tags/digests réels, vérifier leurs architectures et tester le pull depuis les workers. Ne pas modifier les Dockerfiles avant d’avoir un déploiement de référence fonctionnel.

### Examiner les valeurs Helm

Dans le package fourni :

```bash
cd <CHEMIN_PACKAGE>/seeker_docker/orchestrators/kubernetes/seeker
helm show chart .
helm show values . > /repertoire-protege/values-package.yaml
```

La procédure publique documente les clés `registry`, `seekerManagedDatabase.enabled`, `externalDatabaseHost`, `externalDatabasePort`, `externalDatabaseName`, `externalDatabaseUsername`, `externalDatabasePasswordSecret` et `nginxHttpsCertSecret`. Vérifier leur présence et leur rendu dans le chart livré ; les autres noms de paramètres ne doivent pas être devinés. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html)

```yaml
# values-onprem.yaml — valeurs non sensibles
registry: "<REGISTRE_INTERNE>"
seekerManagedDatabase:
  enabled: false
externalDatabaseHost: seeker-postgres-rw.seeker-db.svc
externalDatabasePort: 5432
externalDatabaseName: seekerdb
externalDatabaseUsername: seeker
externalDatabasePasswordSecret: seeker-dbpass
nginxHttpsCertSecret: seeker-certs
```

La désactivation de la base gérée par Seeker évite de créer une seconde base par inadvertance. Le nom DNS de service doit être celui effectivement créé par CNPG et validé par le certificat PostgreSQL ; ne pas utiliser une IP de pod comme adresse stable.

### Fournir le secret PostgreSQL

Un pod du namespace `seeker` ne peut pas référencer directement le secret du namespace `seeker-db`. La cible proposée contient donc un secret `seeker-dbpass` dans `seeker`, avec une clé `dbpass`, attendue par le chart documenté ; synchroniser sa valeur depuis le coffre-fort ou le secret applicatif PostgreSQL. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html) · [Source](https://kubernetes.io/docs/concepts/configuration/secret/)

```yaml
# Schéma uniquement : à matérialiser par le gestionnaire de secrets.
apiVersion: v1
kind: Secret
metadata:
  name: seeker-dbpass
  namespace: seeker
type: Opaque
stringData:
  dbpass: "<MOT_DE_PASSE_DU_ROLE_SEEKER>"
```

**OBLIGATOIRE :** ne pas commiter cette version en clair, ne pas afficher le secret dans les logs CI et ne pas placer le mot de passe dans les arguments `helm --set`. Un changement de secret Kubernetes ne garantit pas à lui seul une rotation du rôle PostgreSQL ni un rechargement des clients Seeker : tester la séquence complète.

### Configurer TLS et ingress

Pour le frontal NGINX, la procédure Seeker attend un secret contenant les clés `seeker-server-cert.pem` et `seeker-server.key`. Ce format diffère du secret TLS Kubernetes standard utilisé par beaucoup de contrôleurs ingress ; prévoir deux ressources distinctes si leurs formats attendus sont différents. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html) · [Source](https://kubernetes.io/docs/concepts/services-networking/ingress/)

```bash
kubectl -n seeker create secret generic seeker-certs \
  --from-file=seeker-server-cert.pem=/chemin-protege/fullchain.pem \
  --from-file=seeker-server.key=/chemin-protege/private-key.pem
```

| Segment | Configuration cible proposée | Preuve à recueillir |
|---|---|---|
| Agent / navigateur → ingress | HTTPS avec CA d’entreprise | Chaîne valide et bon FQDN |
| Ingress → NGINX Seeker | HTTPS avec validation du certificat backend | Configuration et test du contrôleur |
| NGINX → Server / Sensor | Configuration du package, renforcée selon support | Manifests et flux inspectés |
| Server / Sensor → PostgreSQL | TLS avec authentification serveur qualifiée | Sessions observées côté PostgreSQL |
| PostgreSQL → stockage objet | HTTPS et CA reconnue | Test d’archivage et restauration |

Un Ingress standard avec une section `tls` termine le TLS à l’ingress ; il ne chiffre pas automatiquement le segment vers le backend. Les annotations de rechiffrement et les réglages WebSocket sont spécifiques au contrôleur : aucune annotation universelle n’est proposée ici. [Source](https://kubernetes.io/docs/concepts/services-networking/ingress/)

**OBLIGATOIRE :** vérifier que le chart n’expose pas de NodePort ou de LoadBalancer indésirable, que les endpoints agent/API passent par le bon frontal et que les WebSockets restent fonctionnels. Ne pas limiter le routage aux seules pages visibles dans le navigateur.

### Rendre et déployer

Les fichiers rendus peuvent contenir des secrets ou des informations sensibles selon le chart ; conserver les sorties dans un espace protégé. Comparer la topologie rendue au package officiel avant personnalisation.

```bash
helm lint . -f values-onprem.yaml
helm template seeker . --namespace seeker \
  -f values-onprem.yaml \
  > /repertoire-protege/seeker-rendered.yaml

# Ajouter le fichier de dimensionnement approprié uniquement après lecture.
helm upgrade --install seeker . \
  --namespace seeker \
  -f values-onprem.yaml \
  --wait --timeout 20m

kubectl -n seeker get deployments,pods,services,pvc
kubectl -n seeker get events --sort-by=.metadata.creationTimestamp
```

La documentation propose des fichiers de taille comme `medium.yaml`. Vérifier l’ordre de fusion et les valeurs effectives si un fichier de taille et un fichier local sont utilisés ensemble ; ne pas considérer le succès Helm comme la preuve du bon fonctionnement de l’agent ou de PostgreSQL. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/deploy-seeker-server-on-kubernetes.html)

### Initialisation applicative

Configurer l’URL publique interne du serveur et le mot de passe du compte intégré `administrator` dans l’assistant initial. Cette URL doit être joignable par les agents, les utilisateurs et les intégrations ; privilégier un nom DNS stable et un mot de passe conforme à la politique d’entreprise. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/set-up-the-seeker-enterprise-server.html)

1. Charger la licence obtenue auprès de Black Duck et relever sa date d’expiration.
2. Valider l’URL, la chaîne de certificats et le trajet depuis un pod applicatif.
3. Créer les comptes/groupes selon le moindre privilège ; tester l’annuaire ou le SSO retenu.
4. Conserver un compte de secours protégé et tester sa procédure d’accès.
5. Créer un projet pilote et documenter son propriétaire, son environnement et sa convention de nommage.
6. Vérifier le fonctionnement d’un agent avant d’intégrer une politique bloquante CI/CD.

Les étapes de gouvernance ci-dessus sont une proposition locale ; les intitulés précis des menus et les possibilités d’authentification doivent être confirmés dans la release installée. L’architecture documentée prévoit notamment LDAP/LDAPS et SAML 2.0. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/seeker-architecture-and-communications.html)

## 8. Intégration des agents

### Procédure commune

**OBLIGATOIRE :** utiliser le **Connect Agent Wizard** du projet pour obtenir la distribution et les instructions de la version retenue. Préparer d’abord une application et une suite de tests qui fonctionnent sans agent, afin de distinguer un défaut d’application d’un défaut d’instrumentation. [Source](https://docs.blackduck.com/r/seeker/latest/release-notes/release-notes-2025.4.0.html)

1. Relever runtime, version, framework, architecture, serveur et agents APM existants.
2. Télécharger l’agent depuis une source autorisée et figer son artefact dans le processus de livraison.
3. Appliquer la méthode de chargement propre au runtime ; ne pas supposer une méthode unique pour tous les langages.
4. Configurer l’URL Seeker, l’association au projet et les éventuels paramètres de confiance TLS.
5. Redémarrer l’application instrumentée dans l’environnement de test.
6. Vérifier l’attachement, la connexion et l’activité pendant une requête connue.
7. Exécuter les tests ; rapprocher les résultats de la version de l’application et du créneau d’exécution.
8. Mesurer latence, consommation mémoire, erreurs et volume de résultats.

### Variables courantes

La communauté officielle documente `SEEKER_SERVER_URL` pour la localisation du serveur, et `SEEKER_PROJECT_KEY` pour l’association au projet. Pour l’agent Java, les variables d’environnement prennent priorité sur les propriétés correspondantes ; une configuration répartie sur plusieurs mécanismes peut donc surprendre lors du diagnostic. [Source](https://community.blackduck.com/s/article/Seeker-Attaching-an-Agent) · [Source](https://community.blackduck.com/s/article/Using-Environment-Variables-for-Seeker-Agent-Configuration)

```yaml
# Extrait conceptuel à intégrer au Deployment de l’application testée.
# Il ne constitue pas une instrumentation complète.
env:
  - name: SEEKER_SERVER_URL
    value: "https://<FQDN_SEEKER>"
  - name: SEEKER_PROJECT_KEY
    valueFrom:
      configMapKeyRef:
        name: seeker-agent-config
        key: project-key
```

La clé de projet est ici un identifiant de routage ; elle ne doit pas être présentée comme un mot de passe PostgreSQL ou comme la preuve d’une authentification agent. Si la release ou la politique interne la classe sensible, la placer dans un secret plutôt que dans une ConfigMap ; qualifier séparément les éventuels contrôles d’authentification disponibles.

### Java : exemple conceptuel

```bash
# Le nom et le chemin du JAR proviennent du package de l’agent.
export SEEKER_SERVER_URL="https://<FQDN_SEEKER>"
export SEEKER_PROJECT_KEY="<CLE_PROJET>"
java -javaagent:/opt/seeker/<AGENT_JAR_FOURNI> -jar application.jar
```

Ce schéma illustre le chargement d’un agent JVM ; adapter la commande au lancement réel et aux instructions du Wizard. Si `JAVA_TOOL_OPTIONS` est retenu, vérifier qu’il ne remplace pas une valeur existante et qu’il n’instrumente pas des processus JVM non visés ; comparer les interactions avec l’APM en environnement pilote. [Source](https://community.blackduck.com/s/article/Seeker-Attaching-an-Agent)

### Node.js et autres runtimes

Pour Node.js, suivre les instructions correspondant au mode CommonJS ou ESM ; des releases récentes documentent `--import`, alors que des exemples historiques utilisent d’autres options. Pour .NET, Python, PHP ou Go, conserver les instructions de la distribution réellement utilisée plutôt que de transposer l’exemple Java. [Source](https://docs.blackduck.com/r/seeker/latest/release-notes/release-notes-2025.4.0.html) · [Source](https://docs.blackduck.com/r/seeker/latest/release-notes/release-notes-2025.9.0.html)

### Critères d’intégration réussie

| Vérification | Critère proposé |
|---|---|
| Démarrage applicatif | Pas d’erreur de chargement ni de régression fonctionnelle |
| Agent | Visible, connecté et associé au bon projet |
| Réseau | DNS, HTTPS, CA, proxy et WebSocket validés depuis le pod |
| Tests | Parcours connus exécutés avec des comptes représentatifs |
| Données observées | Activité attribuable à la bonne application et au bon créneau |
| Vulnérabilité de laboratoire | Détection attendue sur un cas contrôlé et isolé |
| Remédiation | Correctif retesté et traitement du résultat documenté |
| Ressources | Mesures avant/après acceptées par les équipes |

## 9. CI/CD et politiques

### Pipeline de référence

```text
Build -> tests unitaires + SAST/SCA
      -> livraison environnement de test + agent Seeker
      -> attente application ET agent
      -> tests API / integration / E2E
      -> attente de collecte et controles de completude
      -> collecte des resultats / politique AppSec
      -> gate -> archivage des preuves autorisees -> nettoyage
```

La documentation Seeker décrit des intégrations au SDLC et des API REST pour automatiser notamment la gestion des projets et les exports. Les endpoints, l’authentification et les filtres doivent être pris dans la référence API de l’instance ; aucun endpoint hypothétique n’est fourni dans ce guide. [Source](https://community.blackduck.com/s/article/Seeker-Using-the-API)

### Exemple de contrat interne

Le pseudo-code suivant décrit les fonctions à développer, pas un outil fourni par Black Duck :

```text
ci/deploy-test-environment
ci/wait-application-ready
ci/check-seeker-agent
ci/run-api-and-e2e-tests
ci/wait-seeker-results
ci/export-seeker-results
ci/evaluate-appsec-policy
ci/archive-approved-evidence
ci/remove-ephemeral-environment
```

**OBLIGATOIRE :** le contrôle de complétude doit distinguer « aucun résultat de vulnérabilité » de « aucun agent », « tests non exécutés », « collecte inachevée » ou « API inaccessible ». Dans ces derniers cas, le résultat proposé est **inconclusif**, jamais « conforme » par défaut.

### Politique progressive proposée

| Phase | Politique | Condition de passage |
|---|---|---|
| Découverte | Non bloquante | Agent et couverture de base validés |
| Stabilisation | Alertes sur les nouveaux résultats prioritaires | Triage partagé et règles comprises |
| Gate pilote | Blocage des nouveaux risques répondant à une politique approuvée | Exceptions et procédure d’urgence disponibles |
| Généralisation | Extension application par application | Charge, qualité et support maîtrisés |

Ne pas bloquer aveuglément tout un historique à la première intégration. Définir la notion de « nouveau », le traitement des alertes non vérifiées, les délais de correction, les exceptions et leur expiration ; éviter aussi d’ignorer systématiquement une alerte uniquement parce qu’elle n’a pas le tag `Seeker-Verified`.

### Preuves et indicateurs

| Indicateur proposé | Utilité | Précaution |
|---|---|---|
| Applications / agents raccordés | Avancement du déploiement | Ne mesure pas la couverture fonctionnelle |
| Parcours et routes testés | Confiance dans la campagne | Dénominateur issu du produit / OpenAPI |
| Nouveaux résultats prioritaires | Triage opérationnel | Dédupliquer et contextualiser |
| Délai de prise en charge / correction | Fluidité de remédiation | Distinguer correction et acceptation |
| Campagnes inconclusives | Santé de l’intégration | Résoudre la cause avant d’assouplir le gate |
| Coût runtime | Capacité et stabilité | Mesure avant/après comparable |

**RECOMMANDÉ :** conserver l’identifiant du pipeline, le commit, la version de l’agent, le projet, l’environnement et la fenêtre de test. Vérifier les mécanismes d’attribution réellement disponibles dans la release avant de promettre un cloisonnement automatique par build.

## 10. Administration et exploitation

### Routines d’exploitation

| Fréquence proposée | Contrôles | Responsable |
|---|---|---|
| Chaque jour ouvré | Santé des pods, collecte agents, sauvegardes, erreurs et espace disque | Exploitation |
| Chaque semaine | Tendances, latence, capacité, connexions DB et exceptions AppSec | Lead / DBA / AppSec |
| Chaque mois | Comptes, droits, secrets, certificats, licence, versions et correctifs | Lead / sécurité |
| Selon politique PRA | Exercice de restauration chronométré et preuve exploitable | DBA / plateforme |
| Avant changement majeur | Sauvegarde vérifiée, qualification, rollback documenté | Responsable du changement |

### Observabilité

**RECOMMANDÉ :** surveiller PostgreSQL (disponibilité, connexions, latence disque, espace, WAL, réplication, échec d’archivage), Kubernetes (restarts, OOM, scheduling, PVC), ingress (codes d’erreur, timeouts, TLS) et Seeker (agents connectés, collecte, traitement, disponibilité UI/API). Les seuils sont à établir à partir du pilote et des objectifs de service, pas à partir d’une valeur universelle.

CloudNativePG possède ses propres flux de contrôle et possibilités de monitoring ; préserver les accès nécessaires à l’opérateur, à l’API et aux métriques lors du durcissement. Ne pas publier le port de métriques sur un réseau non maîtrisé. [Source](https://cloudnative-pg.github.io/docs/1.26/security/)

### Sauvegarde PostgreSQL

**Choix proposé :** sauvegardes physiques régulières et archivage continu des WAL vers un stockage objet interne qualifié, plus copie indépendante du domaine de panne primaire. Une stratégie complète inclut aussi les valeurs de déploiement, les secrets récupérables depuis le coffre-fort, les certificats et les volumes applicatifs Seeker réellement requis ; la sauvegarde PostgreSQL seule ne décrit pas tout le PRA.

Le plugin Barman Cloud fournit sauvegardes physiques, archivage WAL et restauration, y compris PITR. Il nécessite CloudNativePG 1.26 ou supérieur, la documentation recommandant au moins 1.27 pour les améliorations de gestion des erreurs ; la matrice complète de la version de plugin retenue reste à vérifier. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/intro/)

Le chart du plugin s’installe dans le même namespace que l’opérateur et attend cert-manager opérationnel. Les versions du chart, du plugin, de CNPG et de PostgreSQL doivent être figées dans la nomenclature ; ne pas installer automatiquement la dernière version en production sans qualification. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/installation/)

```bash
# cert-manager doit déjà être installé et validé.
cmctl check api

helm upgrade --install plugin-barman-cloud \
  cnpg/plugin-barman-cloud \
  --namespace cnpg-system \
  --version <CHART_VERSION_BARMAN_VALIDEE> \
  --wait --timeout 10m

kubectl -n cnpg-system get deployments,pods
kubectl get crd objectstores.barmancloud.cnpg.io
```

### ObjectStore et archivage WAL

Créer le bucket, sa politique, un compte technique dédié et les secrets avant d’activer le plugin. Pour MinIO ou un autre fournisseur compatible S3, la compatibilité doit être testée avec les versions effectivement retenues ; « compatible S3 » ne dispense pas de tester sauvegarde, rétention et restauration. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/)

```yaml
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata:
  name: seeker-backups
  namespace: seeker-db
spec:
  retentionPolicy: "30d"
  configuration:
    destinationPath: s3://<BUCKET>/seeker/postgresql
    endpointURL: https://<ENDPOINT_S3_INTERNE>
    endpointCA:
      name: backup-endpoint-ca
      key: ca.crt
    s3Credentials:
      accessKeyId:
        name: backup-s3-creds
        key: ACCESS_KEY_ID
      secretAccessKey:
        name: backup-s3-creds
        key: ACCESS_SECRET_KEY
    wal:
      compression: gzip
```

`retentionPolicy` s’applique au niveau de `spec` de l’ObjectStore. Une fenêtre de récupération de 30 jours n’est pas la promesse que tous les objets de plus de 30 jours seront immédiatement supprimés : la conservation doit préserver la base et les WAL nécessaires au point récupérable. La valeur de 30 jours est ici un exemple à faire approuver. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/retention/)

Ajouter ensuite au `spec` du cluster PostgreSQL, via le fichier déclaratif complet et une revue Git :

```yaml
plugins:
  - name: barman-cloud.cloudnative-pg.io
    isWALArchiver: true
    parameters:
      barmanObjectName: seeker-backups
```

Cette section active l’archivage et les sauvegardes via le plugin. Ne pas définir `serverName` dans l’ObjectStore ; lors d’une restauration, le nom du serveur source se configure dans les paramètres du plugin de l’`externalCluster`. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/)

### Sauvegarde planifiée

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: seeker-postgres-daily
  namespace: seeker-db
spec:
  schedule: "0 0 2 * * *"
  backupOwnerReference: self
  cluster:
    name: seeker-postgres
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
```

L’expression utilise six champs, dont les secondes en premier : cet exemple demande une sauvegarde à 02:00:00 dans le fuseau effectivement utilisé par l’ordonnanceur. Vérifier explicitement ce fuseau et le comportement attendu lors des changements d’heure ; ne pas interpréter l’expression comme un cron Linux à cinq champs. [Source](https://cloudnative-pg.io/docs/1.27/backup/)

Pour une première sauvegarde à la demande :

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: seeker-postgres-qualification-001
  namespace: seeker-db
spec:
  cluster:
    name: seeker-postgres
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
```

```bash
kubectl apply -f 20-objectstore.yaml
# Appliquer ensuite le Cluster incluant la section plugins.
kubectl apply -f 10-postgresql-cluster.yaml
kubectl apply -f 21-scheduled-backup.yaml
kubectl apply -f 22-backup-qualification.yaml

kubectl -n seeker-db get backups.postgresql.cnpg.io
kubectl -n seeker-db describe backup seeker-postgres-qualification-001
```

La ressource `Backup` avec `method: plugin` et `pluginConfiguration` est la méthode documentée pour les sauvegardes à la demande. Vérifier la réussite réelle, les objets présents et l’archivage WAL ; la seule création de la ressource n’est pas une preuve de sauvegarde utilisable. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/)

**OBLIGATOIRE :** coordonner les droits de suppression, la rétention Barman, les règles lifecycle et l’éventuel Object Lock. Une règle qui efface trop tôt des WAL peut rendre le PITR impossible ; une rétention immuable peut au contraire empêcher le nettoyage et augmenter le stockage.

### Exercice de restauration

L’exemple suivant crée un **nouveau** cluster, sans remplacer le cluster source. Pour la première qualification, le garder isolé du Seeker de production et ne pas activer l’archivage vers le préfixe source. La procédure documentée utilise `bootstrap.recovery` et un `externalCluster` faisant référence au plugin. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/)

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: seeker-postgres-restore-test
  namespace: seeker-db
spec:
  instances: 1
  imageName: "<IMAGE_POSTGRES_COMPATIBLE_AVEC_LA_SAUVEGARDE>"
  bootstrap:
    recovery:
      source: seeker-source
  externalClusters:
    - name: seeker-source
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters:
          barmanObjectName: seeker-backups
          serverName: seeker-postgres
  storage:
    storageClass: "<STORAGE_CLASS>"
    size: 100Gi
```

Ce manifeste illustre une récupération à partir des sauvegardes disponibles, pas une procédure PITR avec cible temporelle préconfigurée. Qualifier séparément la cible temporelle, le fuseau, la présence d’une sauvegarde antérieure et la continuité WAL ; utiliser une image PostgreSQL physiquement compatible avec la sauvegarde. [Source](https://cloudnative-pg.io/plugin-barman-cloud/docs/usage/)

**Recette proposée :** mesurer la durée de provisionnement, de restauration et de remise en service ; comparer les données attendues ; démarrer, si nécessaire, un Seeker isolé de même version en neutralisant agents réels, notifications et intégrations. Valider l’authentification, les projets et les résultats avant de déclarer l’objectif RTO atteint.

### Mise à jour et retour arrière

1. Lire les notes de version, la matrice de compatibilité et les éventuels chemins de mise à jour imposés.
2. Répéter la mise à jour en préproduction avec un jeu de données représentatif et autorisé.
3. Sauvegarder la base et les autres données nécessaires ; s’assurer qu’une restauration a déjà été testée.
4. Planifier l’arrêt ou la coordination des traitements selon la procédure de la release.
5. Déployer les artefacts figés, surveiller les migrations et réaliser les tests de santé et de collecte.
6. En cas d’échec, appliquer la procédure de restauration approuvée ; ne pas supposer qu’un `helm rollback` annule les migrations SQL.

Le guide éditeur recommande de sauvegarder avant une mise à jour et d’arrêter le serveur avant les opérations concernées. Ses procédures Kubernetes anciennes comportent des avertissements liés à des releases historiques ; elles ne doivent pas être recopiées comme une procédure universelle pour la version actuelle. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/upgrade-seeker.html)

### Rotation de secrets

**Séquence proposée :** inventorier tous les consommateurs, préparer une fenêtre et une méthode de retour, modifier le rôle PostgreSQL selon la procédure DBA, synchroniser le secret Seeker, recharger/redémarrer les consommateurs si nécessaire et tester les nouvelles connexions Server et Sensor. Révoquer l’ancien secret et contrôler son absence dans les logs, les artefacts et les environnements de diagnostic.

## 11. Sécurité et durcissement

### Données et identités

Les résultats de sécurité, traces de code, paramètres et exports doivent être traités comme des données sensibles. Seeker offre un suivi de données sensibles et des diagnostics ; contrôler les règles de collecte, le masquage disponible, la rétention et les personnes habilitées avant d’instrumenter une application contenant des données personnelles. [Source](https://docs.blackduck.com/r/seeker/latest/server-installation-guide/features-and-capabilities.html)

| Contrôle proposé | Application |
|---|---|
| Moindre privilège | Comptes nominaux, rôles par fonction et accès projets |
| Secrets | Coffre-fort, chiffrement Kubernetes au repos et rotation testée |
| Compte DB | Pas de superutilisateur pour l’usage normal |
| Certificats | Inventaire, renouvellement et tests de confiance |
| Exports | Accès restreint et conservation limitée |
| Jeux de tests | Données synthétiques ou anonymisées et usages réels neutralisés |
| Intégrations | Comptes techniques dédiés et accès explicitement autorisés |
| Registre | Images approuvées, versions figées et contrôle des vulnérabilités |

Kubernetes recommande notamment le chiffrement des secrets au repos, des RBAC minimaux et des contrôles réseau ; une ConfigMap ne doit pas contenir de mot de passe. La simple création d’un objet `Secret` n’apporte pas toutes ces garanties. [Source](https://kubernetes.io/docs/concepts/security/security-checklist/) · [Source](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

### Réseau : éviter le blocage aveugle

Les NetworkPolicies dépendent d’un CNI qui les applique ; une stratégie « deny all egress » bloque aussi DNS si aucune autorisation adaptée n’est ajoutée. Les règles sont à construire à partir du tableau des flux et à tester avant généralisation. [Source](https://kubernetes.io/docs/reference/kubernetes-api/networking/network-policy-v1/) · [Source](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

**Ordre proposé :** inventorier les flux, observer le pilote, autoriser les échanges indispensables, tester le service et les modes dégradés, puis appliquer l’isolation par défaut. Préserver particulièrement opérateur CNPG → port 8000, réplication, DNS, API Kubernetes, endpoints de sauvegarde et flux agent via le frontal.

### Contexte d’exécution

CloudNativePG documente des conteneurs PostgreSQL non root, sans privilèges et avec système de fichiers racine en lecture seule. Ces propriétés ne doivent pas être attribuées automatiquement aux images Seeker : inspecter les images et manifests fournis, puis qualifier les adaptations de sécurité sans casser l’installeur, les permissions ou les sondes. [Source](https://cloudnative-pg.github.io/docs/1.26/security/)

**OPTIONNEL :** segmentation réseau avancée, mesh, coffre de secrets externe et stockage immuable peuvent renforcer la plateforme, mais ajoutent des contraintes d’exploitation. Les adopter seulement si les équipes savent superviser et restaurer l’ensemble.

## 12. Montée en compétence et recette

### Parcours pédagogique proposé

| Étape | Exercice | Critère de maîtrise |
|---|---|---|
| Comprendre l’IAST | Expliquer une requête, son instrumentation et son résultat | Distinguer IAST, SAST, SCA et DAST |
| Déployer la plateforme | Installer le package et connecter PostgreSQL | Savoir expliquer chaque composant et son flux |
| Instrumenter | Intégrer un agent sur l’application pilote | Observer activité et association au projet |
| Tester | Lancer API/E2E avec plusieurs profils | Identifier ce qui a réellement été couvert |
| Analyser | Triage d’un cas de laboratoire | Expliquer sévérité, tag et contexte |
| Corriger | Modifier le code puis retester | Produire une preuve de remédiation |
| Automatiser | Ajouter un gate d’abord non bloquant | Distinguer conformité et campagne inconclusive |
| Exploiter | Restaurer et diagnostiquer une panne | Appliquer le runbook sans improviser |

Les parcours Black Duck Academy « Seeker From Install to Results » et « Seeker for Server Admins » peuvent compléter ces exercices, selon les accès de formation disponibles. [Source](https://blackduck.skilljar.com/path/seeker-from-install-to-results) · [Source](https://blackduck.skilljar.com/path/seeker-for-server-admins?reg=1)

### Plan de recette technique

| Test | Résultat attendu | Preuve |
|---|---|---|
| DNS et HTTPS | FQDN joignable, chaîne valide | Résultat depuis navigateur et pod |
| WebSocket / agent | Connexion stable et activité visible | Journaux expurgés et observation console |
| Base externe | Server et Sensor utilisent la bonne base | Configuration et sessions PostgreSQL |
| TLS PostgreSQL | Connexions réelles chiffrées et serveur authentifié | Configuration cliente + contrôles côté DB |
| Mauvais certificat | Échec maîtrisé de connexion | Test négatif isolé |
| Identités | Droits conformes aux rôles | Tests positifs et négatifs |
| Tests applicatifs | Parcours connus, plusieurs profils | Rapports QA et activité instrumentée |
| Détection / correction | Résultat de laboratoire puis retest | Dossier de triage |
| Sauvegarde | Backup réussi et WAL archivés | Statut et contrôle du stockage objet |
| Restauration | Données et service contrôlés | Compte rendu chronométré |
| Incident d’instance | Comportement conforme à la conception | Chronologie et mesures |
| Mise à jour | Migration et recette réussies | Procédure répétée en préproduction |

**OBLIGATOIRE :** ne pas exécuter un exercice de failover, une suppression de pod primaire ou une restauration invasive en production sans changement approuvé. Un test de panne doit identifier ses impacts attendus, ses critères d’arrêt et sa procédure de rétablissement.

## 13. Annexes et runbooks

### Arborescence documentaire proposée

```text
seeker-platform/
  README.md
  architecture/
    decisions.md
    flux-reseau.md
    qualification-versions.md
  deploy/
    namespaces.yaml
    postgresql-cluster.yaml
    objectstore.yaml
    scheduled-backup.yaml
    values-onprem.yaml
  runbooks/
    incident-agent.md
    incident-postgresql.md
    sauvegarde-restauration.md
    rotation-secrets.md
    mise-a-jour.md
  validation/
    recette.md
    mesures-capacite.md
    exercices-pra.md
```

Ne conserver dans Git que les versions sans secrets ni données de vulnérabilités sensibles. Les manifests contenant des marqueurs non remplacés ne constituent pas une version de production ; la branche déployable doit être validée par revue et tests.

### Commandes de diagnostic de base

```bash
kubectl -n seeker get pods -o wide
kubectl -n seeker get deployments,services,ingresses,pvc
kubectl -n seeker get events --sort-by=.metadata.creationTimestamp
kubectl -n seeker describe pod <POD>

# Limiter la collecte et expurger avant partage.
kubectl -n seeker logs <POD> -c <CONTENEUR> --since=15m --tail=200
kubectl -n seeker logs <POD> -c <CONTENEUR> --previous --tail=200

kubectl -n seeker-db get clusters.postgresql.cnpg.io
kubectl -n seeker-db get pods,pvc,services
kubectl -n seeker-db get backups.postgresql.cnpg.io
kubectl -n cnpg-system get deployments,pods
```

Utiliser les noms réellement présents et les droits du compte d’exploitation. Ne pas joindre automatiquement les sorties de `get secret`, les fichiers de valeurs sensibles ou un diagnostic complet à un ticket non habilité.

### Agent absent ou inactif

1. Vérifier l’application sans agent et le runtime attendu.
2. Vérifier le chargement de l’agent dans le processus réel, pas seulement la présence du fichier.
3. Vérifier l’URL effective, la clé projet et les priorités de configuration.
4. Tester DNS, connectivité, chaîne de confiance, proxy, ingress et WebSocket depuis l’environnement applicatif.
5. Vérifier le frontal, Server et Sensor, puis la licence et la compatibilité de versions.
6. Rejouer une requête connue et vérifier l’activité plutôt que la seule présence d’une ligne dans la console.

Une URL absente ou mal configurée peut entraîner une tentative de connexion vers localhost pour les agents concernés. Les variables d’environnement Java peuvent aussi remplacer les propriétés configurées dans un fichier ; vérifier la configuration effective sans afficher de secret. [Source](https://community.blackduck.com/s/article/Seeker-Attaching-an-Agent) · [Source](https://community.blackduck.com/s/article/Using-Environment-Variables-for-Seeker-Agent-Configuration)

### Base inaccessible

| Symptôme | Vérification prioritaire | Action prudente |
|---|---|---|
| Résolution DNS impossible | Service primaire et DNS depuis le pod client | Corriger le routage/DNS, pas l’adresse vers une IP de pod |
| Refus / timeout | Endpoints, politiques réseau, port et santé du cluster | Restaurer les flux requis |
| Erreur d’authentification | Identifiant, secret utilisé, rotation et règles `pg_hba` | Ne pas donner superuser pour contourner |
| Erreur TLS | CA, SAN, protocole et configuration cliente | Corriger la confiance ; ne pas désactiver la vérification |
| Trop de connexions | Sessions, traitements et dimensionnement | Analyser avant d’augmenter `max_connections` |
| Disque / WAL saturé | PVC, archivage, rétention et réplication | Traiter la cause ; ne pas supprimer manuellement des WAL |
| Bootstrap en attente | Locale, image, stockage, ressources et anti-affinité | Examiner événements et logs opérateur |

### Aucun résultat après les tests

**Diagnostic proposé :** confirmer l’activité de l’agent, la technologie supportée, l’environnement, le projet et la fenêtre de test ; vérifier que le parcours arrive réellement aux opérations pertinentes. Examiner ensuite le filtre de résultats, les règles de collecte et la fin du traitement ; un rapport vide n’est pas une preuve suffisante d’absence de risque.

### Sauvegarde en échec

Vérifier successivement le statut de la ressource, le plugin et son sidecar, les credentials, le bucket/préfixe, l’endpoint et la CA, puis l’espace, les règles d’immutabilité et les journaux expurgés. Après correction, relancer une sauvegarde et réaliser un test de restauration ; ne pas clore l’incident uniquement parce que le dernier pod est redevenu `Running`.

### Glossaire

| Terme | Définition pratique |
|---|---|
| IAST | Analyse de sécurité à l’intérieur d’une application instrumentée pendant son exécution |
| Agent | Composant qui instrumente le runtime de l’application |
| Checker | Règle de détection appliquée par l’outil |
| Source / sink | Entrée de donnée et opération sensible observées dans le flux |
| WAL | Journal PostgreSQL utilisé notamment pour réplication et récupération |
| PITR | Récupération à un point dans le temps avec une sauvegarde de base et les WAL requis |
| RPO | Perte de données maximale acceptable exprimée en temps |
| RTO | Durée maximale acceptable de remise en service |
| CSI | Interface de stockage pour Kubernetes |
| CNPG | CloudNativePG, opérateur PostgreSQL pour Kubernetes |
| Faux positif | Résultat ne correspondant pas à une vulnérabilité confirmée après analyse |
| Inconclusif | Campagne insuffisamment complète ou fiable pour produire une décision de conformité |

## 14. Checklist Go-Live

Les 15 points suivants sont à renseigner avec une preuve, un responsable et une date dans le dossier de recette ; un point critique non validé bloque le passage en production.

- [ ] **1. Versions :** Seeker, agents, PostgreSQL, CNPG, plugin et Kubernetes compatibles, maintenus et figés.
- [ ] **2. Licence et support :** droits d’usage, échéances, accès aux packages et procédure de support disponibles.
- [ ] **3. Artefacts :** package officiel, images internes, empreintes et provenance documentés.
- [ ] **4. Capacité :** ressources, concurrence, stockage, croissance et anti-affinité qualifiés.
- [ ] **5. Réseau :** tous les flux nécessaires fonctionnent ; aucune exposition DB ou NodePort indésirable.
- [ ] **6. TLS :** certificats valides et renouvelables, WebSocket fonctionnel, clients DB réellement qualifiés.
- [ ] **7. Secrets :** aucun secret en clair dans Git ou les logs ; chiffrement et rotation testés.
- [ ] **8. Base :** propriétaire, schéma, locale, encodage et paramètres conformes aux exigences retenues.
- [ ] **9. Identités :** comptes nominatifs, rôles, accès projets et compte de secours contrôlés.
- [ ] **10. Agents :** application pilote instrumentée, activité visible et compatibilité APM validée.
- [ ] **11. Tests :** parcours représentatifs, profils utilisateurs et couverture documentés.
- [ ] **12. Triage :** politique de priorité, gate progressif et exceptions approuvés.
- [ ] **13. Sauvegardes :** base, WAL, configuration et autres données requises couverts et supervisés.
- [ ] **14. PRA :** restauration isolée réussie, RPO/RTO mesurés et copie indépendante disponible.
- [ ] **15. Exploitation :** alertes, runbooks, mise à jour, retour arrière et transfert de compétence validés.
