# Plan d'industrialisation Black Duck Seeker IAST

Nov 13, 2026 · @EDENE

## 0. Synthèse, conventions et cadrage

Recommandation : un serveur Seeker unique dimensionné « Large », avec un serveur de secours (standby) et une base PostgreSQL externe en haute disponibilité. Authentification par SAML 2.0 (SSO) pour les humains et LDAPS pour la résolution des groupes. Droits gérés uniquement par groupes AD mappés vers des groupes Seeker. Agents injectés par les pipelines dans les environnements hors production uniquement. Un compte local de secours (break-glass) est conservé.

D'après la documentation Black Duck, [un seul serveur Seeker supporte 10 000 agents simultanés et 2 000 projets](https://demo.seeker.synopsys.com/internal/help/en/topics/c_bp_deplsetup.html). L'éditeur recommande d'y ajouter un serveur de secours, qui sert aussi à tester les mises à jour. La haute disponibilité se joue donc surtout sur la reprise rapide, pas sur la répartition de charge.

**Légende utilisée dans tout le document**

| Code | Signification |
| --- | --- |
| **\[OBL\]** | Obligatoire avant mise en production |
| **\[REC\]** | Recommandé, à planifier dans les 3 premiers mois |
| **\[OPT\]** | Optionnel, selon maturité ou contexte |

**Avertissement de version.** Les écrans, chemins d'API et variables cités viennent de l'aide en ligne publique de Seeker. Validez-les dans l'aide intégrée à votre instance (`https://<seeker>/internal/help`) et dans le menu **Web API**, qui reflètent votre version exacte. Les éléments marqués « à vérifier » ne sont pas documentés publiquement.

### Questions de cadrage

Les réponses à ces questions ajustent le dimensionnement, les modèles de configuration et le planning. En attendant, la colonne « Hypothèse retenue » sert de configuration type.

| # | Question | Impact | Hypothèse retenue |
| --- | --- | --- | --- |
| Q1 | Version de Seeker et mode d'installation (installeur Linux, Windows, Docker Compose, Kubernetes) ? | Architecture, procédure de mise à jour | Seeker 2025.x, installeur Linux sur VM RHEL 9 |
| Q2 | On-premise, cloud privé ou cloud public (Azure, AWS, GCP) ? | Réseau, sauvegarde, PostgreSQL managé | On-premise, datacenter principal + site de secours |
| Q3 | Nombre d'applications, d'équipes et d'agents simultanés au pic ? | Taille de déploiement | 150 applications, 40 équipes, 600 agents simultanés |
| Q4 | Technologies applicatives (Java, .NET, .NET Core, Node.js, PHP, Python, Go) ? | Types d'agents, images de base | Java 60 %, .NET Core 25 %, Node.js 15 % |
| Q5 | Plateformes d'exécution des applications testées (VM, Kubernetes, OpenShift, PaaS) ? | Méthode d'injection des agents | Kubernetes pour 70 %, VM pour le reste |
| Q6 | Fournisseur d'identité SSO et annuaire (Okta, Entra ID, ADFS ; AD mono ou multi-forêt) ? | Modèles LDAP et SAML | Entra ID pour le SSO, AD mono-forêt pour les groupes |
| Q7 | Outils CI/CD utilisés ? | Modèles de pipeline | GitLab CI (majoritaire), Jenkins, Azure DevOps |
| Q8 | Outil de suivi des anomalies et de communication ? | Intégrations | Jira, Microsoft Teams, ServiceNow pour les incidents |
| Q9 | SIEM et outil de supervision ? | Collecte de logs, alertes | Splunk (SIEM), Prometheus + Grafana (supervision) |
| Q10 | Coffre-fort de secrets ? | Injection des tokens | HashiCorp Vault, ou variables masquées CI |
| Q11 | PKI interne et proxy sortant ? | TLS, accès Internet | PKI interne, proxy HTTP authentifié |
| Q12 | Exigences RPO / RTO et réglementaires (DORA, PCI DSS, ISO 27001, RGPD) ? | Sauvegarde, rétention, audit | RPO 24 h, RTO 8 h, ISO 27001 |
| Q13 | Seeker est-il autorisé en préproduction ou en production ? | Stratégie d'agents | Jusqu'à la préproduction, jamais en production |
| Q14 | Qui porte la plateforme (équipe AppSec, plateforme DevOps, production) ? | Gouvernance, RACI | Équipe Plateforme DevSecOps, sponsor RSSI |

## 1. Architecture et dimensionnement

Cible : une instance Seeker active derrière un reverse proxy d'entreprise, une base PostgreSQL externe répliquée, et un serveur standby prêt à reprendre. Seeker embarque déjà un NGINX, un processus serveur, un processus « sensor » et PostgreSQL ; en industriel on externalise la base et on place un proxy/WAF devant.

### 1.1 Composants

| Composant | Rôle | Recommandation | Niveau |
| --- | --- | --- | --- |
| Reverse proxy / load balancer d'entreprise (F5, HAProxy, Traefik) | Point d'entrée unique `seeker.<entreprise>.com:443`, terminaison TLS ou passthrough, bascule actif/passif | VIP vers le nœud actif, health check HTTPS, pas de répartition actif/actif | \[OBL\] |
| NGINX Seeker (intégré) | Reçoit UI, API, WebSockets des agents (8080/8443) | HTTP 8080 désactivé ou fermé au pare-feu, 8443 seul ouvert | \[OBL\] |
| Processus Server (9911) et Sensor (8082) | Logique applicative, ingestion des événements agents | Écoute localhost uniquement (défaut) | \[OBL\] |
| PostgreSQL | Vulnérabilités, configuration, utilisateurs | Base externe managée ou cluster Patroni (1 primaire + 1 réplica synchrone), conforme aux « PostgreSQL Database Requirements » de votre version | \[REC\] (intégrée acceptable en pilote) |
| Serveur standby | Reprise après incident, tests de montée de version | Même version, même taille, Seeker arrêté, pointant sur la réplique ou restauré depuis sauvegarde | \[OBL\] |
| Stockage | Base, logs, sauvegardes | SSD, volume dédié pour `data/`, volume séparé ou NFS/objet pour les sauvegardes | \[OBL\] |
| Instance hors production (staging) | Recette des mises à jour et des configurations SSO | Petite taille, IdP de test | \[REC\] |

### 1.2 Options de déploiement

- **VM Linux (installeur)** \[REC pour démarrer\] : le plus simple à exploiter ; sauvegarde par `sk_ctl backup` et snapshots VM.
- **Kubernetes (chart Helm fourni)** \[OPT\] : Seeker fournit des Dockerfiles et un chart Helm, avec des fichiers de taille (`medium.yaml`, etc.) et une option base externe (`seekerManagedDatabase.enabled=false`). Pertinent si vous avez une plateforme K8s mature avec ingress, PV et sauvegarde Velero.
- **Cloud public** \[OPT\] : VM + PostgreSQL managé (Azure Database for PostgreSQL, Amazon RDS) dans un sous-réseau privé, exposition par load balancer interne.

### 1.3 Haute disponibilité et reprise

| Scénario | Mécanisme | RTO cible |
| --- | --- | --- |
| Panne du nœud Seeker | Bascule VIP vers le standby (démarrage manuel ou scripté), base inchangée | < 1 h |
| Panne de la base primaire | Promotion du réplica PostgreSQL (Patroni/managé), redémarrage Seeker | < 30 min |
| Corruption logique ou erreur humaine | Restauration de la dernière sauvegarde à chaud | < 4 h |
| Perte du site | Restauration sur le site de secours depuis une sauvegarde répliquée hors site | < 8 h |

Un nœud actif seul suffit : Seeker est un outil de test, une interruption bloque au pire les quality gates. Prévoyez un mode dégradé dans les pipelines (gate en « warning » si Seeker ne répond pas, voir section 6).

### 1.4 Dimensionnement indicatif

La taille dépend du nombre d'**agents en cours d'exécution simultanée**, pas du nombre total d'applications. Valeurs issues des [Sizing Guidelines Seeker](https://demo.seeker.synopsys.com/internal/help/en/topics/c_sizing_guide.html) (machine unique) :

| Taille | Agents simultanés | vCPU | RAM | Disque PostgreSQL | Bande passante sortante/jour |
| --- | --- | --- | --- | --- | --- |
| Small | jusqu'à 100 | 4 | 16 Go | 100 Go | < 1 Go |
| Medium | jusqu'à 500 | 8 | 32 Go | 150 Go | < 5 Go |
| Large | jusqu'à 1 000 | 12 | 32 Go | 200 Go | < 10 Go |
| Extra Large | jusqu'à 7 000 | 32 | 64 Go | 300 Go | < 15 Go |

**Estimer vos agents simultanés :** (applications instrumentées) × (environnements instrumentés) × (réplicas ou instances par application) + (jobs CI concurrents instrumentés au pic). Avec l'hypothèse Q3 : 150 × 2 × 1,5 + 150 jobs CI ≈ 600 agents → taille **Large**, avec 30 % de marge. Ajoutez +50 % de disque si vous conservez plus de 12 mois d'historique ou de nombreuses versions par projet. Le volume d'événements et de builds se mesure en phase pilote (CPU sensor, croissance de la base par semaine) avant de figer la taille.

### 1.5 Flux réseau et ports

| Source | Destination | Protocole | Port | Niveau |
| --- | --- | --- | --- | --- |
| Agents, navigateurs, API, runners CI/CD | Load balancer → NGINX Seeker | HTTPS + WebSockets | 443 → 8443 | \[OBL\] |
| NGINX | Processus Sensor | HTTP | 8082 (localhost) | \[OBL\] |
| NGINX | Processus Server | HTTP | 9911 (localhost) | \[OBL\] |
| Server, Sensor | PostgreSQL | TCP (TLS) | 5432 | \[OBL\] |
| Server | Annuaire AD | LDAPS | 636 (éviter 389 en clair) | \[OBL\] si LDAP |
| Navigateur ↔ IdP SAML | IdP (Entra ID, Okta, ADFS) | HTTPS | 443 | \[OBL\] si SSO |
| Server | Relais SMTP | SMTPS / STARTTLS | 465 ou 587 (25 par défaut) | \[REC\] |
| Server | Jira, ServiceNow, outils de tickets | HTTPS | 443 | \[REC\] |
| Server | Black Duck SCA / BDBA, Secure Code Warrior | HTTPS | 443 | \[OPT\] |
| Supervision | Seeker (endpoint de santé), hôte (node\_exporter) | HTTPS / HTTP | 443 / 9100 | \[REC\] |
| Bastion d'administration | Hôte Seeker | SSH | 22 | \[OBL\] |

Ports et flux détaillés : [Seeker Architecture and Communications](https://demo.seeker.synopsys.com/internal/help/en/topics/c_arch_comm.html). Le flux SAML ne part pas du serveur : c'est le navigateur qui porte l'assertion vers l'URL ACS.

### 1.6 Prérequis système, TLS et DNS

- **OS** \[OBL\] : une distribution de la matrice de compatibilité de votre version (RHEL 8/9 recommandé) ; NTP synchronisé (indispensable pour SAML) ; compte de service non root dédié.
- **DNS** \[OBL\] : `seeker.<entreprise>.com` (VIP) ; enregistrements des nœuds `seeker-01`, `seeker-02` en interne. Ne jamais coder l'IP dans les agents.
- **Certificats** \[OBL\] : certificat de la PKI interne, SAN = nom VIP + noms de nœuds, clé RSA 2048+ ou ECDSA P-256, TLS 1.2 minimum (1.3 recommandé). La chaîne de la PKI doit être dans les truststores des agents (JVM `cacerts`, magasin Windows, `NODE_EXTRA_CA_CERTS`).
- **Accès sortants** \[OBL\] : seulement ceux listés ci-dessus, via proxy si exigé. Les agents peuvent passer par `SEEKER_PROXY_URL`.
- **Navigateurs** : Chrome ou Firefox récents.

## 2. Authentification et gestion des identités

Recommandation : SAML 2.0 pour la connexion des humains, LDAPS pour les groupes, et deux comptes locaux « Super Administrator » de secours dont les mots de passe sont au coffre. Aucun droit n'est attribué nominativement : tout passe par des groupes.

### 2.1 Compte administrateur initial et comptes locaux

- &#91;OBL\] Changer le mot de passe de l'administrateur prédéfini dès la première connexion (≥ 20 caractères, généré), le stocker dans le coffre (Vault, CyberArk) avec double contrôle.
- &#91;OBL\] Créer un second compte local `bg-seeker-admin-02` (type Seeker, case **Super Administrator**) pour le break-glass, lui aussi au coffre. Ces deux comptes ne servent qu'en cas d'incident IdP/LDAP ; toute connexion déclenche une alerte SIEM.
- &#91;OBL\] Les administrateurs du quotidien utilisent leur compte nominatif SSO, membre du groupe AD des administrateurs plateforme.
- &#91;REC\] Comptes techniques pour l'API : pas de compte humain partagé. Utiliser des **access tokens** de type API, un par usage (pipeline, intégration Jira, reporting), nommés et datés.
- &#91;REC\] Revue trimestrielle des comptes locaux et des tokens, suppression des inutilisés.

### 2.2 Intégration LDAP / Active Directory

Configuration dans **Settings > User Management > LDAP/Active Directory** (libellés exacts à vérifier dans votre version). Fonctionnement documenté : un utilisateur d'un groupe LDAP autorisé qui se connecte pour la première fois est créé automatiquement dans Seeker. Il reçoit alors les rôles des groupes Seeker mappés à ses groupes LDAP, plus le groupe `everyone`.

| Paramètre | Valeur type | Bonne pratique |
| --- | --- | --- |
| URL | `ldaps://ldap.corp.example.com:636` | Nom DNS d'un pool de DC (pas une IP), LDAPS obligatoire ; éviter le Global Catalog (3268/3269) sauf multi-domaines |
| Bind DN | `CN=svc-seeker-ldap,OU=ServiceAccounts,OU=IT,DC=corp,DC=example,DC=com` | Compte de service dédié, lecture seule, mot de passe non expirant mais tourné tous les 6 mois, « Log on to » restreint |
| Mot de passe de bind | depuis le coffre | Jamais dans un ticket ou un wiki ; rotation planifiée (section 8) |
| Base DN utilisateurs | `OU=Users,OU=Corp,DC=corp,DC=example,DC=com` | Le plus précis possible pour la performance |
| Filtre utilisateurs | `(&(objectCategory=person)(objectClass=user)(sAMAccountName={0})(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` | Exclut les comptes désactivés |
| Base DN groupes | `OU=Seeker,OU=Groups,DC=corp,DC=example,DC=com` | Une OU dédiée aux groupes `GG-SEEKER-*` |
| Filtre groupes | `(&(objectClass=group)(cn=GG-SEEKER-*))` | Limite les groupes visibles à ceux prévus pour Seeker |
| Attribut identifiant | `sAMAccountName` (ou `userPrincipalName`) | Doit correspondre au NameID SAML si vous combinez les deux |
| Attributs mappés | `mail`, `displayName`, `givenName`, `sn`, `memberOf` | `mail` indispensable pour les notifications |
| Groupes imbriqués | `memberOf:1.2.840.113556.1.4.1941:=` si supporté | À vérifier ; sinon utiliser des groupes plats |
| Certificat | Chaîne de la PKI interne importée dans le truststore du serveur Seeker | Tester l'expiration du certificat des DC |

**Test de connexion** \[OBL\] : bouton de test de l'écran LDAP, puis `ldapsearch` depuis l'hôte Seeker (modèle en livrable L3), puis connexion réelle d'un utilisateur de chaque groupe pilote.

### 2.3 SSO SAML 2.0

Seeker joue le rôle de Service Provider. Écran : **Settings > User Management > SAML 2.0**. La procédure se fait en parallèle dans l'IdP et dans Seeker ([doc SAML Seeker](https://demo.seeker.synopsys.com/internal/help/en/topics/t_config_saml.html)).

**Côté IdP (Entra ID, Okta, ADFS)**

1. Créer une application SAML « Seeker IAST » (Entra : Enterprise application non-gallery ; ADFS : Relying Party Trust).
2. Copier depuis l'écran Seeker : **Service Provider Entity ID** → Identifier / Audience URI ; **SAML Assertion Consumer Service URL** → Reply URL / Single sign-on URL.
3. NameID : format EmailAddress, valeur `user.mail` (ou UPN, identique à l'identifiant LDAP).
4. Claims : `email`, `givenname`, `surname`, et un attribut de groupes, par exemple `GROUP_MEMBERSHIP`.
5. Groupes : n'émettre que les groupes affectés à l'application (Entra : « Groups assigned to the application », valeur `sAMAccountName` ou nom cloud ; éviter les GUID). Dans Okta, déclarer le *Group Attribute Statement* avec un filtre `GG-SEEKER-`.
6. Signature : assertion signée SHA-256. Noter la date d'expiration du certificat de signature de l'IdP.
7. Affecter l'application aux groupes `GG-SEEKER-*` uniquement.

**Côté Seeker**

1. Renseigner l'IdP : **Single sign-on URL**, **Issuer ID**, **X.509 Certificate** (certificat complet avec les lignes BEGIN/END).
2. **Group membership attribute name** = `GROUP_MEMBERSHIP` (doit être identique au nom du claim).
3. **Enable Automatic User Creation** : activé si vous n'utilisez pas LDAP. Si LDAP est configuré, la création suit la configuration LDAP, quelle que soit cette option.
4. **Enable Force-Authentication** : \[OPT\], force une réauthentification à l'IdP (utile pour les administrateurs).
5. Enregistrer, se déconnecter, tester **Sign in with SSO** avec un compte de chaque groupe.

**Stratégie de secours si l'IdP est indisponible** \[OBL\] : la connexion locale reste accessible sur la page de login (la doc indique d'utiliser une fenêtre de navigation privée pour contourner le SSO). Les comptes break-glass 2.1 servent alors. Les agents et pipelines ne dépendent pas de l'IdP (tokens), donc la collecte continue.

### 2.4 LDAP, SAML ou les deux ?

| Option | Avantages | Limites | Pour qui |
| --- | --- | --- | --- |
| LDAP seul | Simple, mapping de groupes natif, pas d'IdP | Mot de passe AD saisi dans Seeker, pas de MFA | Petites équipes, réseau isolé |
| SAML seul | SSO, MFA et accès conditionnel de l'IdP, aucun mot de passe dans Seeker | Groupes transmis à chaque connexion seulement ; dépendance à l'IdP | Entreprises « cloud identity » |
| **SAML + LDAP (recommandé)** | MFA via SSO, groupes AD fiables via LDAP, création des utilisateurs pilotée par LDAP | Deux configurations à maintenir, identifiants à aligner | Grande entreprise avec AD |

### 2.5 Éviter le verrouillage administrateur

- &#91;OBL\] Ne jamais modifier SAML ou LDAP sans une session Super Administrator locale ouverte dans un autre navigateur.
- &#91;OBL\] Tester chaque changement sur l'instance staging, puis appliquer en production avec un plan de retour (configuration précédente notée, export JSON via API si disponible).
- &#91;OBL\] Surveiller l'expiration des certificats (IdP, LDAPS, TLS Seeker) à J-60, J-30, J-7.
- &#91;REC\] Garder au moins deux personnes membres du groupe AD des administrateurs plateforme, et un groupe AD de secours distinct.
- &#91;REC\] Procédure documentée de réactivation de l'authentification locale si l'accès UI est perdu (à obtenir du support Black Duck pour votre version).

## 3. Modèle d'autorisations et RBAC

Le RBAC Seeker repose sur quatre objets : **rôles globaux** (fonctions de la plateforme), **rôles projet** (droits sur un projet), **groupes** (porteurs des rôles, mappés aux groupes LDAP/SAML) et **appartenance aux projets** (un groupe reçoit un rôle projet sur un projet donné). Seeker fournit trois rôles prédéfinis, DevSecOps, Auditor et Developer, à dupliquer plutôt qu'à modifier. La case Super Administrator d'un utilisateur donne tous les droits et se réserve aux comptes break-glass.

### 3.1 Rôles proposés

| Rôle | Type | Mission | Basé sur |
| --- | --- | --- | --- |
| `R-PLATFORM-ADMIN` | Global | Exploite la plateforme : paramètres système, LDAP/SAML, SMTP, licences, mises à jour | Nouveau (tous droits globaux) |
| `R-SECURITY-ADMIN` | Global + projet (tous) | Définit politiques, seuils, statuts ; valide exceptions ; vue consolidée | DevSecOps |
| `R-PROJECT-CREATOR` | Global | Crée projets et composites selon la convention de nommage (équipe Plateforme ou automate) | Nouveau |
| `R-APP-OWNER` | Projet | Responsable applicatif : gère membres, versions, triage, priorisation | DevSecOps réduit |
| `R-DEVELOPER` | Projet | Consulte et corrige, change le statut (sauf exception/faux positif) | Developer |
| `R-QA` | Projet | Consulte, relance des tests, vérifie les corrections | Developer réduit |
| `R-DEVOPS` | Projet | Configure agents (version, tags), génère les tokens agents de son périmètre | Nouveau |
| `R-AUDITOR` | Global (lecture) + projet (tous) | Lecture seule, exports et rapports de conformité | Auditor |
| `R-VIEWER` | Projet | Lecture seule d'un périmètre (management, PO) | Auditor réduit, sans export |

### 3.2 Permissions par rôle

Les permissions ci-dessous sont fonctionnelles ; associez-les aux interrupteurs On/Off de **Settings > User Management > Roles** (la liste exacte dépend du type Global ou Projet et de votre version).

| Permission | PLATFORM-ADMIN | SECURITY-ADMIN | PROJECT-CREATOR | APP-OWNER | DEVELOPER | QA | DEVOPS | AUDITOR | VIEWER |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Consulter les vulnérabilités | ✓ | ✓ | – | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Changer statut / commentaire | – | ✓ | – | ✓ | ✓ | ✓ | – | – | – |
| Marquer faux positif / exception | – | ✓ (valide) | – | ✓ (demande) | – | – | – | – | – |
| Créer / supprimer un projet | ✓ | – | ✓ (créer) | – | – | – | – | – | – |
| Configurer projet (versions, réglages) | ✓ | ✓ | – | ✓ | – | – | ✓ | – | – |
| Configurer agents (version, activation) | ✓ | – | – | – | – | – | ✓ | – | – |
| Gérer les membres du projet | ✓ | ✓ | – | ✓ | – | – | – | – | – |
| Gérer politiques / conformité | – | ✓ | – | – | – | – | – | – | – |
| Exporter rapports | ✓ | ✓ | – | ✓ | ✓ | ✓ | – | ✓ | – |
| Gérer intégrations (Jira, SMTP) | ✓ | ✓ (projet) | – | – | – | – | – | – | – |
| Gérer utilisateurs, groupes, rôles | ✓ | – | – | – | – | – | – | – | – |
| Gérer tokens d'accès | ✓ | – | – | – | – | – | ✓ (agents) | – | – |
| Administration système (LDAP, SAML, licence) | ✓ | – | – | – | – | – | – | – | – |

Principe de séparation des tâches : celui qui demande une exception (APP-OWNER) n'est pas celui qui la valide (SECURITY-ADMIN), et l'administrateur plateforme n'a pas de droit de triage.

### 3.3 Mapper les groupes AD / IdP vers Seeker

1. **Dans AD** : créer des groupes `GG-SEEKER-<FONCTION>[-<BU>-<APP>]` dans l'OU dédiée (convention en 4.2). Les groupes globaux sont peu nombreux ; les groupes projet existent par application ou par équipe.
2. **Dans Seeker** : créer un groupe Seeker de même nom pour chaque groupe AD, et le relier au groupe LDAP correspondant (et au nom de groupe reçu dans le claim SAML).
3. **Rôles globaux** : affecter au groupe Seeker son rôle global (ex. `GG-SEEKER-PLATFORM-ADMIN` → `R-PLATFORM-ADMIN`).
4. **Projets** : dans chaque projet, ajouter les groupes membres avec leur rôle projet (ex. `GG-SEEKER-DEV-PAY-CHECKOUT` → `R-DEVELOPER` sur `PAY-CHECKOUT-*`).
5. **Automatiser** : appliquer ces liens par script via l'API (`/api/{ver}/groups`, `/api/{ver}/users`, appartenance projet) depuis un fichier YAML versionné (« RBAC as code »), revu en merge request.
6. **Contrôler** : le groupe `everyone` ne porte aucun rôle, pour qu'un utilisateur sans groupe ne voie rien.

La matrice prête à l'emploi « rôle / groupe / permissions / périmètre » est en livrable L2.

## 4. Organisation multi-équipes et multi-applications

Règle de base : **un projet Seeker par composant déployable** (service, API, front), regroupés dans un **projet composite par application métier**. L'environnement est porté par les **tags d'agents**, la release par la **version du projet**. Un projet par environnement multiplierait les doublons et casserait le suivi d'une vulnérabilité de la QA à la préprod.

### 4.1 Structure

| Niveau métier | Objet Seeker | Exemple | Qui le crée |
| --- | --- | --- | --- |
| Entité / Business Unit | Préfixe de nommage + groupe AD BU (pas d'objet Seeker dédié) | `pay` | – |
| Application métier | Projet composite | `cmp-pay-checkout` | Plateforme (automate d'onboarding) |
| Composant / microservice | Projet | `pay-checkout-api` | Plateforme, ou création automatique via `SEEKER_COMPOSITE_PROJECT_KEY` |
| Release / build | Version du projet (synchronisée avec la version détectée) | `2.4.1` | Pipeline |
| Environnement | Tag d'agent `SEEKER_AGENT_CUSTOM_TAGS` | `env_qa` | Pipeline / manifeste |
| Équipe | Groupe AD + appartenance au projet | `GG-SEEKER-DEV-PAY-CHECKOUT` | IAM, sur demande APP-OWNER |

Cas particulier \[OPT\] : un projet séparé par environnement se justifie seulement si les données de préprod doivent être cloisonnées (ex. données réelles masquées sous contrainte réglementaire).

### 4.2 Conventions de nommage

| Objet | Convention | Exemple | Règles |
| --- | --- | --- | --- |
| Clé de projet | `<bu>-<app>-<composant>` | `pay-checkout-api` | minuscules, chiffres, tirets ; ≤ 40 caractères ; immuable |
| Nom affiché du projet | `<BU> · <Application> · <Composant>` | `PAY · Checkout · API` | Libre, lisible |
| Projet composite | `cmp-<bu>-<app>` | `cmp-pay-checkout` | Un par application métier (code CMDB en description) |
| Version (branche principale, release) | SemVer de l'application | `2.4.1` | Identique à l'artefact déployé |
| Version (merge request, branche) | `<semver>-mr<id>` ou `<semver>-<branche-courte>` | `2.5.0-mr1842` | Purge automatique après 30 jours (via API) |
| Tags d'agent | `env_<env>,bu_<bu>,team_<équipe>,ci_<outil>` | `env_qa,bu_pay,team_checkout,ci_gitlab` | Valeurs d'env fixes : `dev`, `int`, `qa`, `rec`, `ppd` |
| Groupes AD | `GG-SEEKER-<FONCTION>[-<BU>-<APP>]` | `GG-SEEKER-APPOWNER-PAY-CHECKOUT` | Majuscules, une OU dédiée |
| Rôles Seeker | `R-<FONCTION>` | `R-DEVELOPER` | Jamais modifier les rôles prédéfinis |
| Tokens d'accès | `<type>-<périmètre>-<usage>-<aaaamm>` | `agent-pay-checkout-gitlab-202611` | Date de création pour la rotation |

### 4.3 Isoler les équipes tout en consolidant

- &#91;OBL\] **Cloisonnement** : un utilisateur ne voit que les projets dont son groupe est membre ; `everyone` sans rôle ; pas de rôle projet global pour les développeurs.
- &#91;OBL\] **Tokens agents par projet** : un token d'authentification agent restreint aux clés de projet de l'équipe, pour qu'une équipe ne puisse pas alimenter le projet d'une autre.
- &#91;REC\] **Vue consolidée sécurité** : `GG-SEEKER-SECURITY-ADMIN` et `GG-SEEKER-AUDITOR` ajoutés automatiquement à chaque nouveau projet par le script d'onboarding (ou rôle global de lecture si votre version le permet).
- &#91;REC\] **Vue management par BU** : groupe `GG-SEEKER-VIEWER-<BU>` membre des composites de la BU ; tableau de bord BU construit à partir des composites.
- &#91;OPT\] **Reporting transverse** : extraction quotidienne via API vers l'entrepôt de données ou Power BI (vulnérabilités ouvertes par BU, âge moyen, respect des SLA).

## 5. Déploiement des agents Seeker

Principe : l'agent n'est jamais intégré à l'image applicative livrée en production. Il est injecté au déploiement dans les environnements de test (init container, admission controller ou option JVM), avec une version figée par projet et un token d'authentification par périmètre.

### 5.1 Options par plateforme

| Plateforme | Méthode | Recommandation | Niveau |
| --- | --- | --- | --- |
| Kubernetes / OpenShift | **Admission controller** Seeker (webhook de mutation) qui instrumente les pods selon une annotation ou un label | Cible pour les clusters de test : aucune modification des manifestes applicatifs | \[REC\] |
| Kubernetes | **Init container** qui télécharge l'agent dans un volume `emptyDir` partagé | Simple, sans composant cluster ; modèle en livrable L4 | \[REC\] |
| Kubernetes | Agent sur volume persistant | Pour clusters sans accès sortant vers Seeker au démarrage | \[OPT\] |
| Docker / Docker Compose | Image de test dérivée (`FROM app:x.y` + agent) ou volume monté | Ne jamais pousser l'image instrumentée dans le registre de production | \[OPT\] |
| VM Java (Tomcat, JBoss, Spring Boot) | `-javaagent:/opt/seeker/seeker-agent.jar` dans `JAVA_OPTS` ou `setenv.sh` | Installé par Ansible, version pilotée par variable | \[REC\] |
| VM Windows / IIS (.NET Framework) | Service « Seeker .NET Agents » installé en administrateur, configuration `dotnet-agent.config` | Désactiver l'autoupdate et piloter la version | \[REC\] |
| .NET Core / .NET 6+ | Variables CLR profiler (fournies par l'installeur) + `SEEKER_*` | Init container sur Linux, script sur VM | \[REC\] |
| Node.js | Module agent chargé au démarrage (`node -r` ou `NODE_OPTIONS`) | Init container ou dépendance de build de test uniquement | \[REC\] |
| Azure App Service | Procédure dédiée .NET Agent | Slot de test uniquement | \[OPT\] |
| Autres (PHP, Python, Go) | Selon guide technologique de votre version | Valider en pilote avant engagement | \[OPT\] |

Téléchargement des binaires par API : `GET <seeker>/rest/api/latest/installers/agents/binaries/{JAVA|DOTNETCORE|NODEJS}` avec `projectKey` (version définie pour le projet) ou `agentVersion`. Les exemples de l'éditeur utilisent `curl -k` ou `--no-check-certificate` : **à proscrire**, fournir le certificat de la PKI (`--cacert`).

### 5.2 Variables, secrets et certificats à injecter

| Paramètre | Variable / propriété | Source | Secret ? | Niveau |
| --- | --- | --- | --- | --- |
| URL du serveur | `SEEKER_SERVER_URL` / `-Dseeker.server.url` | Variable CI de groupe | Non | \[OBL\] |
| Clé de projet | `SEEKER_PROJECT_KEY` / `-Dseeker.project.key` | Fichier `seeker.yml` du dépôt ou variable CI | Non | \[OBL\] |
| Clé composite (création auto) | `SEEKER_COMPOSITE_PROJECT_KEY` | Variable CI | Non | \[OPT\] |
| Token d'authentification agent | Token de type « Agent authentication » (variable d'agent ou `accessTokens` pour .NET) — nom exact de la variable à vérifier | Vault → secret Kubernetes / variable CI masquée protégée | **Oui** | \[OBL\] |
| Token API (quality gate) | En-tête `Authorization` des appels REST | Vault, variable masquée protégée | **Oui** | \[OBL\] |
| Tags d'environnement | `SEEKER_AGENT_CUSTOM_TAGS` / `-Dseeker.agent.custom.tags` | Pipeline (`env_qa,...`) | Non | \[OBL\] |
| Proxy | `SEEKER_PROXY_URL` | Variable de plateforme | Non (sauf identifiants) | \[OPT\] |
| Version d'agent | Version définie dans le projet Seeker, ou `agentVersion` au téléchargement | Pipeline / projet | Non | \[OBL\] |
| Chaîne de certificats PKI | `cacerts` JVM, magasin Windows, `NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE` | Image de base durcie | Non | \[OBL\] |

Ordre de priorité des réglages : propriété JVM ou fichier de configuration, puis variable d'environnement, puis configuration du projet dans l'UI ([Appendix A, variables Seeker](https://demo.seeker.synopsys.com/internal/help/en/topics/r_env_var.html)).

### 5.3 Stratégie par environnement

| Environnement | Agent | Vérification active | Quality gate | Commentaire |
| --- | --- | --- | --- | --- |
| Développement (poste, dev) | \[OPT\] à la demande | Activée | Aucun | Feedback rapide, données fictives |
| Intégration (CI, tests automatisés) | \[OBL\] | Activée | **Bloquant** sur nouvelles Critical/High | Cœur du dispositif |
| QA / tests fonctionnels | \[OBL\] | Activée | Bloquant sur Critical, alerte High | Meilleure couverture fonctionnelle |
| Recette (UAT) | \[REC\] | Selon accord métier | Informatif | Tests manuels = couverture complémentaire |
| Préproduction | \[OPT\] | **Désactivée** | Informatif | Uniquement si données masquées, accord RSSI et test de charge préalable |
| Production | **Non recommandé** | – | – | Seeker est un outil de test : surcoût, requêtes de vérification, données sensibles |

La vérification active envoie des requêtes de test vers l'application ; elle se désactive par projet (`PUT /api/{ver}/projects/{key}/settings/features/activeverification`).

### 5.4 Maîtriser l'impact sur les performances

- &#91;OBL\] Mesurer en pilote : temps de réponse p95, CPU, mémoire, temps de démarrage, avec et sans agent, sur le même jeu de tests. Seuil d'acceptation proposé : +20 % de latence p95 et +15 % de mémoire au maximum.
- &#91;OBL\] Augmenter les limites de ressources des pods instrumentés (+25 % CPU et mémoire, +256 Mo de heap JVM au départ) et les sondes de démarrage (`startupProbe`), le démarrage étant plus long.
- &#91;REC\] Ne pas instrumenter les tests de performance ou de charge : résultats faussés.
- &#91;REC\] Exclure les processus non pertinents (batchs, workers techniques) via les options de filtrage de processus de l'agent.
- &#91;REC\] Prévoir un interrupteur de désactivation : variable `SEEKER_ENABLED=false` dans vos scripts d'injection (convention interne) pour couper l'agent sans redéployer l'image.

### 5.5 Gestion des versions d'agents

- &#91;OBL\] Le serveur n'est **pas compatible avec des agents plus récents que lui** : toujours mettre à jour le serveur d'abord.
- &#91;OBL\] Fixer la version d'agent par projet (par défaut « Latest ») une fois le pilote validé ; une variable globale `SEEKER_DEFAULT_AGENT_VERSION` donne la version par défaut côté serveur.
- &#91;OBL\] Désactiver l'autoupdate des agents .NET (service Windows) pour éviter des montées de version non maîtrisées.
- &#91;REC\] Cycle de montée : serveur staging → projets canaris (5 projets volontaires, 1 semaine) → vague générale ; décalage maximal autorisé : N-2 versions d'agent par rapport au serveur.
- &#91;REC\] Tableau de bord des versions d'agents connectés, revu mensuellement (API d'administration des agents).

## 6. Intégration CI/CD et DevSecOps

Seeker ne « scanne » pas : il observe l'application pendant les tests. Le pipeline doit donc déployer l'application instrumentée, exécuter les tests automatisés, puis interroger l'API Seeker pour décider. Seeker propose un plugin Jenkins (étape **Seeker Build Status**, qui fait échouer le build en cas de Critical ou High) ; pour les autres outils, la décision passe par un appel REST `/api/{ver}/vulnerabilities` dans un script.

### 6.1 Étapes du pipeline

1. **Build** de l'artefact et de l'image (inchangée, sans agent).
2. **Tests unitaires + SAST/SCA** (hors Seeker).
3. **Déploiement instrumenté** dans l'environnement éphémère ou d'intégration : init container ou `-javaagent`, avec `SEEKER_PROJECT_KEY`, version = version de l'artefact, tags `env_int,ci_gitlab,pipeline_<id>`.
4. **Attente de connexion de l'agent** (polling API des agents connectés, timeout 5 min).
5. **Tests automatisés** : API (Postman/Newman, Karate, RestAssured), E2E (Playwright, Selenium). La couverture des tests fait la couverture Seeker.
6. **Temporisation de traitement** : 60 à 120 s pour laisser la vérification et l'ingestion se terminer.
7. **Récupération des résultats** : appel API filtré sur projet + version, export JSON/SARIF en artefact de pipeline.
8. **Quality gate** : comparaison aux seuils (6.2), sortie 0 ou 1.
9. **Publication** : commentaire de merge request, tickets Jira, notification Teams.
10. **Nettoyage** : suppression de l'environnement éphémère ; les vulnérabilités restent dans Seeker.

Mode dégradé \[OBL\] : si Seeker est injoignable, la gate passe en « warning » (exit 0 + alerte), sauf sur les branches de release où elle reste bloquante. Le modèle de pipeline complet (GitLab CI, GitHub Actions, Azure DevOps, Jenkins) est en livrable L4.

### 6.2 Seuils de qualité et de sécurité

| Sévérité | Merge request / intégration | Branche de release | Délai de remédiation (SLA) | Exception possible ? |
| --- | --- | --- | --- | --- |
| Critical | **Bloquant** (toute nouvelle occurrence) | **Bloquant** (toute occurrence ouverte) | 7 jours | Oui, RSSI, 30 jours max |
| High | **Bloquant** (nouvelle) | **Bloquant** (ouverte, hors exception) | 30 jours | Oui, Security Admin, 90 jours max |
| Medium | Avertissement | Avertissement au-delà de 10 ouvertes | 90 jours | Oui, App Owner + Security Admin |
| Low / Info | Informatif | Informatif | 180 jours / best effort | Oui, App Owner |

**Montée en puissance** \[REC\] : 4 premières semaines en mode informatif pour chaque application, puis bloquant sur Critical, puis Critical + High à S+8. Les seuils sont versionnés dans un fichier central (`seeker-policy.yml`) pour qu'ils ne soient pas modifiables par chaque équipe.

**Faux positifs et exceptions** \[OBL\] :

- Un faux positif est marqué dans Seeker avec un commentaire obligatoire (raison technique) et revu par la sécurité sous 5 jours ouvrés.
- Une exception (risque accepté) a un propriétaire, une date d'expiration, une mesure compensatoire et un ticket de référence ; elle est réévaluée à expiration.
- Indicateurs : taux de faux positifs par règle (cible < 10 %), nombre d'exceptions actives par BU, dépassements de SLA.

### 6.3 Intégrations outils

| Outil | Méthode | Usage recommandé | Niveau |
| --- | --- | --- | --- |
| Jira | Intégration native « Bug Tracking » de Seeker (livraison automatique des détections) | Ticket auto pour Critical/High, projet Jira de l'équipe, lien vers Seeker, déduplication par ID de vulnérabilité | \[REC\] |
| ServiceNow (SecOps / VR) | Script planifié lisant l'API Seeker et créant des enregistrements via l'API ServiceNow | Suivi des SLA et des exceptions au niveau entreprise | \[OPT\] |
| Microsoft Teams / Slack | Webhook entrant appelé par l'étape « Publication » du pipeline, ou résumé quotidien par script | Alerte au canal de l'équipe en cas de gate bloquée ; canal sécurité pour les Critical | \[REC\] |
| SonarQube | Conversion du résultat Seeker au format « Generic Issue Import » puis `sonar.externalIssuesReportPaths` | Vue unique pour le développeur (code + runtime) | \[OPT\] |
| GitLab / GitHub | Rapport SARIF ou commentaire de merge request via l'API de la forge | Visibilité dans la revue de code | \[REC\] |
| Black Duck SCA / BDBA | Intégration native côté serveur | Corréler composants vulnérables et usage réel | \[OPT\] |
| Secure Code Warrior | Intégration native | Formation contextualisée liée à la vulnérabilité | \[OPT\] |

## 7. Sécurisation de la plateforme Seeker

Seeker concentre la liste des vulnérabilités exploitables de toute l'entreprise, avec les requêtes qui les prouvent : c'est un actif classifié **Confidentiel**, à protéger comme un coffre de secrets. La checklist détaillée est en livrable L5.

### 7.1 Mesures de durcissement

| Domaine | Mesure | Niveau |
| --- | --- | --- |
| TLS | HTTPS seul sur 443/8443, TLS 1.2+ (1.3 si possible), suites fortes, HSTS sur le proxy ; port 8080 fermé ; TLS vers PostgreSQL et LDAPS | \[OBL\] |
| Certificats | PKI interne, renouvellement automatisé ou alerté à J-60, inventaire des certificats (Seeker, IdP, DC, PostgreSQL) | \[OBL\] |
| Réseau | Seeker dans une zone « outillage sécurité » ; entrant 443 depuis réseaux de test, CI et postes internes uniquement ; aucune exposition Internet ; base accessible du seul serveur Seeker | \[OBL\] |
| Pare-feu hôte | `firewalld` : 8443 depuis le load balancer, 22 depuis le bastion, rien d'autre | \[OBL\] |
| Système | CIS Benchmark niveau 1, SELinux enforcing, compte de service sans shell, correctifs mensuels, EDR | \[OBL\] |
| Secrets | Mots de passe base, bind LDAP, tokens dans Vault ; aucun secret dans les dépôts ou les logs de pipeline ; variables CI masquées et protégées | \[OBL\] |
| Rotation | Bind LDAP et base : 6 mois ; tokens API et agents : 12 mois max (90 jours pour les tokens d'administration) ; immédiate au départ d'un détenteur | \[OBL\] |
| Accès | SSO + MFA imposé par l'IdP, accès conditionnel (poste géré) pour les administrateurs ; revue trimestrielle des groupes | \[OBL\] |
| Journalisation | Logs applicatifs, NGINX, système et base envoyés au SIEM ; horloge NTP | \[OBL\] |
| Sauvegardes | Chiffrées (AES-256), stockées hors de l'hôte, copie immuable hors site | \[OBL\] |
| Vulnérabilités de la plateforme | Scan mensuel de l'hôte et de l'image ; veille sur les bulletins Black Duck ; mise à jour Seeker au moins trimestrielle | \[OBL\] |
| WAF / proxy | Règles de base, limitation de débit sur `/rest/api` et la page de login | \[REC\] |
| Données | Pas de données personnelles réelles dans les environnements instrumentés ; purge des projets inactifs > 12 mois | \[REC\] |
| Pentest | Test d'intrusion de la plateforme avant généralisation, puis annuel | \[REC\] |

### 7.2 Politique de sauvegarde

Seeker propose une sauvegarde **à chaud** (serveur en marche, à la demande en CLI ou planifiée) et une sauvegarde **à froid** (serveur arrêté). Commande Linux : `sk_ctl backup <base> <utilisateur> <répertoire> <nom> tar.gz false` ; restauration : `sk_ctl restore <base> <utilisateur> <fichier>`, qui arrête puis redémarre le serveur ([Hot Backup and Restore](https://demo.seeker.synopsys.com/internal/help/en/topics/r_hotbackup_restore.html)).

| Élément | Fréquence | Rétention | Emplacement |
| --- | --- | --- | --- |
| Sauvegarde à chaud Seeker (données + configuration) | Quotidienne, 01:00 | 14 jours | Volume de sauvegarde local, puis copie |
| Copie hors site chiffrée et immuable | Quotidienne | 35 jours quotidiennes, 12 mensuelles | Stockage objet (S3 compatible, verrouillage WORM) |
| PostgreSQL externe : base + WAL (PITR) | Continue + complète hebdomadaire | 14 jours | Outil du SGBD (pgBackRest, sauvegarde managée) |
| Sauvegarde à froid / snapshot VM | Avant chaque mise à jour | 3 dernières | Snapshot hyperviseur |
| Configuration (fichiers, certificats, scripts RBAC-as-code, valeurs Helm) | À chaque changement | Historique Git | Dépôt Git privé + Vault |

**Objectifs** : RPO 24 h (15 min avec PITR PostgreSQL), RTO 8 h (1 h en bascule sur standby). **Test de restauration** \[OBL\] : trimestriel, sur le serveur standby, avec contrôle du nombre de projets, de vulnérabilités et d'une connexion SSO ; compte rendu archivé.

### 7.3 Journaux à collecter et alertes SIEM

**Sources à collecter** \[OBL\] : logs du serveur et du sensor Seeker (répertoire de logs de l'installation), logs d'accès et d'erreurs NGINX, journal d'audit applicatif si votre version en dispose (à vérifier), logs système (`auditd`, `secure`, `journald`), logs PostgreSQL (connexions, erreurs), logs du proxy/load balancer, journaux de connexion de l'IdP pour l'application Seeker.

| Alerte | Condition | Sévérité | Destinataire |
| --- | --- | --- | --- |
| Connexion d'un compte break-glass | Toute connexion d'un compte local Super Administrator | Haute | SOC + responsable plateforme |
| Force brute | > 10 échecs de connexion en 5 min pour un compte ou une IP | Haute | SOC |
| Changement de configuration d'authentification | Modification LDAP, SAML, rôles ou groupes globaux | Haute | SOC + RSSI |
| Création de token API | Token créé hors fenêtre de changement | Moyenne | Responsable plateforme |
| Export massif | Export de rapports > N par heure pour un utilisateur | Moyenne | SOC |
| Accès API anormal | Pic de 401/403 sur `/rest/api`, IP hors plages CI | Moyenne | SOC |
| Accès direct à la base | Connexion PostgreSQL depuis une autre source que Seeker | Haute | SOC |
| Certificat proche d'expiration | J-30 | Moyenne | Responsable plateforme |

## 8. Exploitation, supervision et support

L'équipe Plateforme DevSecOps opère Seeker comme un service interne avec un engagement de disponibilité de 99,5 % en heures ouvrées, un support N1 par le service desk et N2/N3 par l'équipe plateforme, puis le support Black Duck.

### 8.1 Métriques à surveiller

| Métrique | Source | Seuil d'alerte | Niveau |
| --- | --- | --- | --- |
| Disponibilité de l'UI et de l'API | Sonde synthétique HTTPS (login page + appel API authentifié) | 2 échecs consécutifs à 1 min | \[OBL\] |
| Latence API p95 | Logs NGINX / sonde | > 2 s pendant 10 min | \[OBL\] |
| Taux d'erreurs HTTP 5xx | Logs NGINX | > 2 % sur 5 min | \[OBL\] |
| CPU / RAM hôte | node\_exporter | CPU > 85 % 15 min ; RAM > 90 % | \[OBL\] |
| Disque `data/` et sauvegardes | node\_exporter | > 75 % (alerte), > 85 % (critique) | \[OBL\] |
| PostgreSQL : connexions, taille, retard de réplication | postgres\_exporter | Retard > 60 s ; croissance > 10 Go/semaine | \[OBL\] |
| Agents connectés (nombre, versions) | API Seeker d'administration des agents | Chute > 30 % en 15 min en heures ouvrées | \[REC\] |
| Agents en erreur ou déconnectés | API agents + logs sensor | > 5 % des agents attendus | \[REC\] |
| Projets actifs / projets sans activité > 90 j | API projets | Rapport mensuel | \[REC\] |
| Volume de vulnérabilités (nouvelles, ouvertes, hors SLA) | API vulnérabilités | Hors SLA en hausse 2 semaines de suite | \[REC\] |
| Échecs LDAP / SAML | Logs serveur, logs IdP | > 5 échecs/min ou toute erreur de certificat | \[OBL\] |
| Erreurs CI/CD de la gate (timeouts, 401) | Logs des jobs, métriques CI | > 5 % des jobs Seeker en échec technique | \[REC\] |
| Expiration des certificats et tokens | Inventaire + script | J-60 / J-30 / J-7 | \[OBL\] |
| Âge de la dernière sauvegarde réussie | Script de contrôle | > 26 h | \[OBL\] |

### 8.2 Runbook d'exploitation

| Procédure | Étapes clés | Fenêtre / durée |
| --- | --- | --- |
| **Démarrage / arrêt** | `sk_ctl stop` puis `sk_ctl start` (ou `systemctl` selon installation) ; vérifier l'UI, l'API et la reconnexion des agents ; prévenir les canaux CI si arrêt > 15 min | 10 min |
| **Mise à jour Seeker** | 1. Lire les release notes ; 2. tester sur staging ; 3. sauvegarde à froid + snapshot ; 4. arrêt, installeur, démarrage ; 5. tests de validation (L6) ; 6. mettre à jour les agents par vagues | Fenêtre mensuelle, 2 h |
| **Sauvegarde à la demande** | `sk_ctl backup inline seeker /backup seeker-$(date +%F) tar.gz false` ; vérifier taille et somme de contrôle ; copier hors site | 15 à 60 min |
| **Restauration** | Arrêter le trafic au load balancer ; `sk_ctl restore inline seeker /backup/<fichier>` ; contrôler projets, vulnérabilités, SSO ; rouvrir le trafic | 1 à 4 h |
| **Rotation de secrets** | Générer le nouveau secret (Vault) ; mettre à jour Seeker (bind LDAP) ou les consommateurs (tokens) ; tester ; révoquer l'ancien après 7 jours de recouvrement pour les tokens | 30 min |
| **Renouvellement de certificat** | CSR depuis l'hôte ; installer le certificat (proxy et NGINX Seeker) ; redémarrer ; tester avec `openssl s_client` ; vérifier la connexion d'un agent Java et d'un agent Node.js | 30 min |
| **Incident SSO** | Basculer sur un compte break-glass ; vérifier certificat IdP, horloge NTP, Entity ID et ACS ; comparer l'assertion (outil SAML-tracer) ; communiquer aux utilisateurs | P2, rétablissement < 4 h |
| **Incident LDAP** | `ldapsearch` depuis l'hôte ; vérifier compte de bind (verrouillé, expiré), certificat des DC, pare-feu 636 ; les sessions SSO existantes restent valides | P2, < 4 h |
| **Incident agent** | Logs agent (application), version agent vs serveur, connectivité 443, token, truststore ; désactiver l'agent (`SEEKER_ENABLED=false`) si l'application est impactée | P3, contournement immédiat |
| **Incident base de données** | Arrêter Seeker ; vérifier espace disque, verrous, réplication ; promouvoir le réplica ou restaurer (PITR) ; redémarrer Seeker | P1, < 1 h en bascule |

### 8.3 Gouvernance (RACI)

| Activité | Plateforme DevSecOps | Équipe AppSec | RSSI | App Owner | Développeurs | IAM | SOC |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Administration et MCO de Seeker | **R/A** | C | I | – | – | – | I |
| Politique de sécurité, seuils, SLA | C | **R** | **A** | C | I | – | – |
| Validation des exceptions Critical | – | R | **A** | C | I | – | – |
| Validation des exceptions High/Medium | – | **R/A** | I | C | I | – | – |
| Gestion des accès (groupes AD) | I | C | – | **A** (demande) | – | **R** | – |
| Onboarding d'une application | **R** | C | – | **A** | C | C | – |
| Remédiation des vulnérabilités | – | C | I | **A** | **R** | – | – |
| Pilotage des indicateurs et comité mensuel | C | **R** | **A** | C | – | – | – |
| Réception des alertes plateforme | **R** | I | – | – | – | – | **R** (sécurité) |

R = réalise, A = approuve et rend compte, C = consulté, I = informé. Rituel proposé : comité AppSec mensuel (vulnérabilités hors SLA, exceptions à échéance, couverture des applications).

## 9. Plan de déploiement progressif

Durée totale estimée : 6 à 7 mois jusqu'à la généralisation, avec un pilote opérationnel à S+8. Chaque phase se termine par un point de passage (go/no-go) validé par le responsable plateforme et l'équipe AppSec.

| Phase | Durée | Livrables | Tests de validation | Critères de passage | Risques principaux |
| --- | --- | --- | --- | --- | --- |
| **P1 — Installation** | S1–S2 | VM ou cluster, PostgreSQL externe, certificats, DNS, load balancer, standby | UI en HTTPS via le nom VIP ; ports conformes à 1.5 (scan nmap) ; sauvegarde à chaud OK | Checklist L1 « Infrastructure » verte | Délais d'ouverture de flux et de certificats |
| **P2 — Configuration initiale** | S2–S3 | Compte admin sécurisé, break-glass au coffre, SMTP, rôles R-\*, conventions, scripts RBAC-as-code | Envoi de mail de test ; création de projet par API ; restauration testée sur standby | Restauration réussie ; rôles créés | Configuration manuelle non reproductible |
| **P3 — Intégration LDAP / SSO** | S3–S5 | Configuration LDAPS et SAML, groupes AD `GG-SEEKER-*`, mapping groupes → rôles | Checklist L6 « SSO/LDAP » ; un utilisateur par rôle voit exactement son périmètre ; test break-glass | 100 % des cas L6 OK ; aucun accès non prévu | Verrouillage administrateur ; claims de groupes mal formatés |
| **P4 — Pilote (2 à 3 applications)** | S5–S8 | Une application Java sur K8s, une .NET Core, une Node.js ; mesures de performance ; premiers résultats triés | Surcoût ≤ seuils 5.4 ; vulnérabilités remontées et confirmées ; taux de faux positifs mesuré | Retour positif des 3 équipes ; faux positifs < 15 % ; aucun incident applicatif | Couverture de tests trop faible ; surcoût mémoire |
| **P5 — Industrialisation CI/CD** | S8–S12 | Templates GitLab, Jenkins, Azure DevOps (L4), script de gate, intégration Jira et Teams, tableau de bord | Gate bloque un build de test avec une vulnérabilité injectée ; mode dégradé testé (Seeker coupé) | Templates publiés et documentés ; pilote en mode bloquant | Pipelines allongés ; tokens mal protégés |
| **P6 — Déploiement progressif (vagues)** | S12–S20 | Vague 1 : 15 applications volontaires ; vague 2 : 40 applications critiques (exposition Internet) ; kit d'onboarding, formation | Checklist d'onboarding par application ; supervision des ressources serveur | Charge serveur < 60 % ; SLA support tenus ; satisfaction ≥ 7/10 | Saturation de l'équipe plateforme ; résistance des équipes |
| **P7 — Généralisation** | S20–S28 | Toutes les applications du périmètre ; seuils bloquants Critical + High ; reporting par BU | Couverture mesurée ; tests de reprise (bascule standby) | ≥ 80 % des applications du périmètre instrumentées ; test de PRA réussi | Dette de vulnérabilités historiques ; dimensionnement |
| **P8 — Exploitation (run)** | Continu | Runbooks, RACI, comité mensuel, revue des accès trimestrielle, plan de montée de version | Restauration trimestrielle ; revue d'accès ; audit annuel | – | Dérive des versions d'agents ; exceptions qui s'accumulent |

Ordre de priorité des vagues : d'abord les applications exposées sur Internet et manipulant des données sensibles, à condition qu'elles aient des tests automatisés API ou E2E (sans tests, Seeker ne voit rien).

## 10. Livrables

Six livrables prêts à l'emploi, à adapter avec les réponses aux questions de cadrage. Les valeurs `example.com`, les noms de groupes et les clés de projet sont des exemples.

### L1 — Checklist technique avant mise en production

**Infrastructure**

- [ ] &#91;OBL\] Taille de déploiement validée (agents simultanés estimés + 30 % de marge)
- [ ] &#91;OBL\] OS de la matrice de compatibilité, durci CIS niveau 1, NTP synchronisé
- [ ] &#91;OBL\] Nom DNS `seeker.<entreprise>.com` sur la VIP ; certificat PKI avec SAN corrects
- [ ] &#91;OBL\] Flux 1.5 ouverts et seulement ceux-là (scan de vérification archivé)
- [ ] &#91;OBL\] Port 8080 fermé ; 8443 joignable uniquement depuis le load balancer
- [ ] &#91;REC\] PostgreSQL externe en HA, TLS activé, compte dédié
- [ ] &#91;OBL\] Serveur standby installé à la même version
- [ ] &#91;OBL\] Supervision 8.1 active (sonde, ressources, disque, base, certificats)

**Identités et accès**

- [ ] &#91;OBL\] Mot de passe de l'admin initial changé et au coffre
- [ ] &#91;OBL\] Compte break-glass créé, testé, au coffre ; alerte SIEM associée
- [ ] &#91;OBL\] LDAPS configuré et testé (L3, L6)
- [ ] &#91;OBL\] SAML configuré et testé avec chaque rôle (L3, L6)
- [ ] &#91;OBL\] Rôles R-\* et groupes créés selon L2 ; `everyone` sans rôle
- [ ] &#91;REC\] RBAC-as-code versionné et appliqué par script

**Données et continuité**

- [ ] &#91;OBL\] Sauvegarde à chaud planifiée quotidiennement, copie hors site chiffrée
- [ ] &#91;OBL\] Restauration testée sur le standby, compte rendu archivé
- [ ] &#91;OBL\] RPO / RTO validés par le métier et le RSSI

**Intégration**

- [ ] &#91;OBL\] Tokens API et agents créés par périmètre, stockés dans Vault / variables masquées
- [ ] &#91;OBL\] Template de pipeline (L4) validé sur le pilote, mode dégradé testé
- [ ] &#91;REC\] Intégration Jira active sur les projets pilotes
- [ ] &#91;REC\] Notifications Teams/Slack configurées

**Exploitation**

- [ ] &#91;OBL\] Runbooks 8.2 relus par l'équipe d'astreinte
- [ ] &#91;OBL\] RACI 8.3 validé ; contacts support Black Duck et numéro de contrat connus
- [ ] &#91;OBL\] Logs envoyés au SIEM, alertes 7.3 testées
- [ ] &#91;OBL\] Checklist de sécurité L5 entièrement verte (ou écarts acceptés par écrit)

### L2 — Matrice RBAC prête à l'emploi

| Rôle Seeker | Groupe AD / IdP | Type | Permissions clés | Périmètre | Approbateur de l'accès |
| --- | --- | --- | --- | --- | --- |
| Super Administrator (case utilisateur) | aucun (comptes locaux `bg-seeker-admin-01/02`) | Local | Tous droits | Plateforme | RSSI (usage tracé) |
| `R-PLATFORM-ADMIN` | `GG-SEEKER-PLATFORM-ADMIN` | Global | Système, LDAP/SAML, SMTP, licences, utilisateurs/groupes/rôles, tokens, création de projets ; pas de triage | Plateforme | Responsable plateforme + RSSI |
| `R-SECURITY-ADMIN` | `GG-SEEKER-SECURITY-ADMIN` | Global + projet | Politiques, conformité, validation faux positifs/exceptions, exports, gestion des membres | Tous les projets | RSSI |
| `R-PROJECT-CREATOR` | `GG-SEEKER-PROJECT-CREATOR` (+ compte technique d'onboarding) | Global | Création de projets et composites | Plateforme | Responsable plateforme |
| `R-AUDITOR` | `GG-SEEKER-AUDITOR` | Global lecture + projet | Lecture, exports, rapports de conformité | Tous les projets | RSSI / Audit interne |
| `R-APP-OWNER` | `GG-SEEKER-APPOWNER-<BU>-<APP>` | Projet | Lecture, triage, demande d'exception, versions, membres, exports | Projets de l'application (`<bu>-<app>-*`, `cmp-<bu>-<app>`) | Responsable de la BU |
| `R-DEVELOPER` | `GG-SEEKER-DEV-<BU>-<APP>` | Projet | Lecture, changement de statut, commentaires, exports | Projets de l'application | App Owner |
| `R-QA` | `GG-SEEKER-QA-<BU>-<APP>` | Projet | Lecture, changement de statut (vérification), exports | Projets de l'application | App Owner |
| `R-DEVOPS` | `GG-SEEKER-DEVOPS-<BU>-<APP>` | Projet | Configuration agents et projet, tokens agents de son périmètre | Projets de l'application | App Owner + Responsable plateforme |
| `R-VIEWER` | `GG-SEEKER-VIEWER-<BU>` | Projet | Lecture seule, sans export | Composites de la BU | Responsable de la BU |
| (aucun rôle) | `everyone` (intégré) | – | Aucune | – | – |

Règles d'usage : un utilisateur cumule les rôles de tous ses groupes ; aucun rôle n'est attribué directement à un utilisateur nominatif ; l'accès est demandé dans l'outil IAM, approuvé par l'approbateur de la colonne de droite, et revu tous les trimestres.

Exemple de fichier RBAC-as-code (à appliquer par script via l'API groupes, utilisateurs et membres de projet) :

```yaml
# rbac/pay-checkout.yml
application: cmp-pay-checkout
bu: pay
projects:
  - pay-checkout-api
  - pay-checkout-web
members:
  - group: GG-SEEKER-APPOWNER-PAY-CHECKOUT
    role: R-APP-OWNER
  - group: GG-SEEKER-DEV-PAY-CHECKOUT
    role: R-DEVELOPER
  - group: GG-SEEKER-QA-PAY-CHECKOUT
    role: R-QA
  - group: GG-SEEKER-DEVOPS-PAY-CHECKOUT
    role: R-DEVOPS
  # ajoutés automatiquement à tous les projets
  - group: GG-SEEKER-SECURITY-ADMIN
    role: R-SECURITY-ADMIN
  - group: GG-SEEKER-AUDITOR
    role: R-AUDITOR
  - group: GG-SEEKER-VIEWER-PAY
    role: R-VIEWER
```

### L3 — Modèles de configuration LDAP et SAML

**Fiche de paramétrage LDAP / Active Directory** (à reporter dans l'écran Seeker ; noms de champs à aligner sur votre version)

```yaml
# seeker-ldap.yml — fiche de référence, versionnée sans secret
ldap:
  type: ActiveDirectory
  url: ldaps://ldap.corp.example.com:636      # pool de DC, jamais une IP
  bind_dn: CN=svc-seeker-ldap,OU=ServiceAccounts,OU=IT,DC=corp,DC=example,DC=com
  bind_password: vault:secret/seeker/ldap#password   # saisi depuis le coffre
  user_base_dn: OU=Users,OU=Corp,DC=corp,DC=example,DC=com
  user_filter: "(&(objectCategory=person)(objectClass=user)(sAMAccountName={0})(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
  user_id_attribute: sAMAccountName
  attributes:
    email: mail
    display_name: displayName
    first_name: givenName
    last_name: sn
    member_of: memberOf
  group_base_dn: OU=Seeker,OU=Groups,DC=corp,DC=example,DC=com
  group_filter: "(&(objectClass=group)(cn=GG-SEEKER-*))"
  group_name_attribute: cn
  nested_groups: false                         # groupes plats recommandés
  tls:
    truststore: PKI-Corp-Root-CA + PKI-Corp-Issuing-CA
  referrals: ignore                            # mono-forêt
```

Tests depuis l'hôte Seeker avant de saisir la configuration :

```bash
# 1. Certificat LDAPS présenté par les DC
openssl s_client -connect ldap.corp.example.com:636 -showcerts </dev/null | openssl x509 -noout -subject -enddate

# 2. Bind + recherche d'un utilisateur
LDAPTLS_CACERT=/etc/pki/ca-trust/source/anchors/corp-root.pem \
ldapsearch -H ldaps://ldap.corp.example.com:636 -x \
  -D "CN=svc-seeker-ldap,OU=ServiceAccounts,OU=IT,DC=corp,DC=example,DC=com" -W \
  -b "OU=Users,OU=Corp,DC=corp,DC=example,DC=com" \
  "(sAMAccountName=jdupont)" mail displayName memberOf

# 3. Liste des groupes Seeker visibles
ldapsearch -H ldaps://ldap.corp.example.com:636 -x -D "<bind_dn>" -W \
  -b "OU=Seeker,OU=Groups,DC=corp,DC=example,DC=com" "(cn=GG-SEEKER-*)" cn
```

**Fiche de paramétrage SAML 2.0**

| Côté | Champ | Valeur type |
| --- | --- | --- |
| Seeker → IdP | Service Provider Entity ID | Copié depuis Settings > User Management > SAML 2.0 (ex. `https://seeker.corp.example.com/...`) |
| Seeker → IdP | SAML Assertion Consumer Service URL | Copié depuis le même écran |
| IdP → Seeker | Single sign-on URL | Entra : `https://login.microsoftonline.com/<tenant-id>/saml2` |
| IdP → Seeker | Issuer ID | Entra : `https://sts.windows.net/<tenant-id>/` |
| IdP → Seeker | X.509 Certificate | Certificat de signature (Base64), lignes BEGIN/END incluses |
| Seeker | Group membership attribute name | `GROUP_MEMBERSHIP` |
| Seeker | Enable Automatic User Creation | Oui si SAML seul ; sans effet si LDAP actif |
| Seeker | Enable Force-Authentication | Non (oui pour une instance dédiée admin) |
| IdP | NameID | Format EmailAddress, valeur `user.mail` |
| IdP | Claims | `email` = mail, `givenname`, `surname`, `GROUP_MEMBERSHIP` = groupes affectés |
| IdP | Signature | Assertion signée, SHA-256 |
| IdP | Affectation | Groupes `GG-SEEKER-*` uniquement |

**Entra ID** : Enterprise application → Single sign-on SAML ; *Attributes & Claims* → Add a group claim → « Groups assigned to the application », source `sAMAccountName` (groupes synchronisés depuis AD), *Customize the name of the group claim* = `GROUP_MEMBERSHIP`. Équivalent manifeste :

```json
{
  "groupMembershipClaims": "ApplicationGroup",
  "optionalClaims": {
    "saml2Token": [
      { "name": "groups", "additionalProperties": ["sam_account_name"] }
    ]
  }
}
```

**Okta** : App SAML 2.0, Name ID format `EmailAddress`, *Group Attribute Statements* : Name `GROUP_MEMBERSHIP`, filtre « Starts with » `GG-SEEKER-`.

**ADFS** : Relying Party Trust avec l'Entity ID et l'ACS de Seeker, puis ces règles de revendication :

```text
@RuleName = "LDAP attributes"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory",
    types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
             "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
             "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname"),
    query = ";mail,givenName,sn;{0}", param = c.Value);

@RuleName = "NameID email"
c:[Type == "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"]
 => issue(Type = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier",
    Issuer = c.Issuer, OriginalIssuer = c.OriginalIssuer, Value = c.Value, ValueType = c.ValueType,
    Properties["http://schemas.xmlsoap.org/ws/2005/05/identity/claimproperties/format"] = "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress");

@RuleName = "Groupes Seeker"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => add(store = "Active Directory", types = ("http://schemas.xmlsoap.org/claims/Group"), query = ";tokenGroups;{0}", param = c.Value);
c:[Type == "http://schemas.xmlsoap.org/claims/Group", Value =~ "^GG-SEEKER-"]
 => issue(Type = "GROUP_MEMBERSHIP", Value = c.Value);
```

Extrait attendu de l'assertion (contrôle avec SAML-tracer) :

```xml
<saml:Attribute Name="GROUP_MEMBERSHIP">
  <saml:AttributeValue>GG-SEEKER-DEV-PAY-CHECKOUT</saml:AttributeValue>
  <saml:AttributeValue>GG-SEEKER-VIEWER-PAY</saml:AttributeValue>
</saml:Attribute>
```

### L4 — Modèle de pipeline CI/CD avec Seeker

Trois briques réutilisables : l'injection de l'agent (manifeste Kubernetes), le script de quality gate, et les templates par outil CI. **À vérifier avant usage** dans le menu Web API de votre instance : chemin exact de l'endpoint vulnérabilités, noms des paramètres de filtre, format de l'en-tête d'authentification et noms des champs JSON.

**1. Injection de l'agent Java par init container (Kubernetes, environnement de test)**

```yaml
# k8s/overlays/int/seeker-patch.yaml
spec:
  template:
    spec:
      volumes:
        - name: seeker-agent
          emptyDir: {}
        - name: corp-ca
          configMap: { name: corp-ca-bundle }
      initContainers:
        - name: seeker-agent-download
          image: registry.corp.example.com/base/curl:8
          command: ["/bin/sh", "-c"]
          args:
            - >-
              curl -fsS --cacert /etc/corp-ca/ca.pem
              -o /tmp/agent.zip
              "${SEEKER_SERVER_URL}/rest/api/latest/installers/agents/binaries/JAVA?projectKey=${SEEKER_PROJECT_KEY}"
              && unzip -o /tmp/agent.zip -d /seeker
          env:
            - { name: SEEKER_SERVER_URL, value: "https://seeker.corp.example.com" }
            - { name: SEEKER_PROJECT_KEY, value: "pay-checkout-api" }
          volumeMounts:
            - { name: seeker-agent, mountPath: /seeker }
            - { name: corp-ca, mountPath: /etc/corp-ca }
      containers:
        - name: app
          env:
            - { name: JAVA_TOOL_OPTIONS, value: "-javaagent:/seeker/seeker-agent.jar" }
            - { name: SEEKER_SERVER_URL, value: "https://seeker.corp.example.com" }
            - { name: SEEKER_PROJECT_KEY, value: "pay-checkout-api" }
            - { name: SEEKER_AGENT_CUSTOM_TAGS, value: "env_int,bu_pay,team_checkout,ci_gitlab" }
            # Token d'authentification agent : nom de variable à vérifier pour votre version
            - name: SEEKER_AGENT_ACCESS_TOKEN
              valueFrom: { secretKeyRef: { name: seeker-agent-token, key: token } }
          resources:
            requests: { cpu: "500m", memory: "1280Mi" }   # +25 % vs non instrumenté
            limits:   { cpu: "1500m", memory: "2Gi" }
          volumeMounts:
            - { name: seeker-agent, mountPath: /seeker, readOnly: true }
```

**2. Script de quality gate**

```bash
#!/usr/bin/env bash
# ci/seeker-gate.sh — sortie 0 = OK, 1 = gate bloquée, 0 + warning = Seeker injoignable (hors release)
set -euo pipefail
: "${SEEKER_SERVER_URL:?}" "${SEEKER_PROJECT_KEY:?}" "${SEEKER_PROJECT_VERSION:?}" "${SEEKER_API_TOKEN:?}"
BLOCK_SEVERITIES="${BLOCK_SEVERITIES:-CRITICAL,HIGH}"   # depuis seeker-policy.yml
STRICT="${SEEKER_STRICT:-false}"                     # true sur les branches de release
CA="${SEEKER_CA_FILE:-/etc/ssl/certs/corp-ca.pem}"
OUT=seeker-results.json

sleep "${SEEKER_SETTLE_SECONDS:-90}"   # laisser l'ingestion et la vérification se terminer

HTTP=$(curl -sS --cacert "$CA" -o "$OUT" -w '%{http_code}' \
  -H "Authorization: ${SEEKER_API_TOKEN}" \
  "${SEEKER_SERVER_URL}/rest/api/latest/vulnerabilities?format=JSON&projectKeys=${SEEKER_PROJECT_KEY}&projectVersions=${SEEKER_PROJECT_VERSION}" \
  || echo 000)

if [ "$HTTP" != "200" ]; then
  echo "::warning:: Seeker injoignable (HTTP $HTTP)"
  [ "$STRICT" = "true" ] && exit 1 || exit 0
fi

# Compte les vulnérabilités ouvertes aux sévérités bloquantes (adapter les noms de champs)
COUNT=$(jq --arg sev "$BLOCK_SEVERITIES" '
  ($sev | ascii_upcase | split(",")) as $block
  | [ .[] | select(((.Severity // .severity // "") | ascii_upcase) as $s | $block | index($s))
          | select(((.Status // .status // "") | ascii_upcase) | test("FIXED|IGNORED|FALSE") | not) ]
  | length' "$OUT")

echo "Seeker: $COUNT vulnérabilité(s) bloquante(s) [$BLOCK_SEVERITIES] sur $SEEKER_PROJECT_KEY $SEEKER_PROJECT_VERSION"
[ "$COUNT" -eq 0 ] || exit 1
```

**3. GitLab CI (template central inclus par les projets)**

```yaml
# templates/seeker.gitlab-ci.yml
variables:
  SEEKER_SERVER_URL: "https://seeker.corp.example.com"
  # SEEKER_PROJECT_VERSION est calculé dans default.before_script (APP_VERSION fourni par le projet)
  SEEKER_AGENT_CUSTOM_TAGS: "env_int,ci_gitlab,pipeline_${CI_PIPELINE_ID}"

stages: [build, test, deploy-iast, iast-tests, seeker-gate, report]

default:
  before_script:
    - export SEEKER_PROJECT_VERSION="${CI_COMMIT_TAG:-${APP_VERSION}-mr${CI_MERGE_REQUEST_IID}}"

deploy-iast:
  stage: deploy-iast
  environment: { name: "int-$CI_COMMIT_REF_SLUG", on_stop: cleanup-iast }
  script:
    - helm upgrade --install "$APP-$CI_COMMIT_REF_SLUG" chart/ -n iast-int
        -f chart/values-int.yaml -f chart/values-seeker.yaml
        --set seeker.projectKey="$SEEKER_PROJECT_KEY"
        --set seeker.version="$SEEKER_PROJECT_VERSION"
        --set seeker.tags="$SEEKER_AGENT_CUSTOM_TAGS"
    - kubectl rollout status deploy/"$APP-$CI_COMMIT_REF_SLUG" -n iast-int --timeout=300s

iast-tests:
  stage: iast-tests
  script:
    - newman run tests/api.postman_collection.json -e tests/int.env.json
    - npx playwright test --project=e2e-int
  artifacts: { when: always, paths: [reports/] }

seeker-gate:
  stage: seeker-gate
  image: registry.corp.example.com/base/ci-tools:latest   # curl + jq + CA d'entreprise
  variables:
    SEEKER_STRICT: "false"
  rules:
    - if: $CI_COMMIT_TAG
      variables: { SEEKER_STRICT: "true" }
    - when: on_success
  script:
    - ci/seeker-gate.sh   # SEEKER_API_TOKEN = variable masquée protégée (ou Vault via id_tokens)
  artifacts: { when: always, paths: [seeker-results.json] }

notify-teams:
  stage: report
  when: on_failure
  script:
    - 'curl -fsS -H "Content-Type: application/json" -d "{\"text\":\"Gate Seeker bloquée: $CI_PROJECT_PATH $SEEKER_PROJECT_VERSION $CI_PIPELINE_URL\"}" "$TEAMS_WEBHOOK_URL"'

cleanup-iast:
  stage: report
  when: manual
  environment: { name: "int-$CI_COMMIT_REF_SLUG", action: stop }
  script: [ "helm uninstall $APP-$CI_COMMIT_REF_SLUG -n iast-int" ]
```

**4. GitHub Actions (job équivalent)**

```yaml
  seeker-gate:
    needs: iast-tests
    runs-on: [self-hosted, linux, corp]
    env:
      SEEKER_SERVER_URL: https://seeker.corp.example.com
      SEEKER_PROJECT_KEY: pay-checkout-api
      SEEKER_PROJECT_VERSION: ${{ github.ref_type == 'tag' && github.ref_name || format('{0}-pr{1}', vars.APP_VERSION, github.event.number) }}
      SEEKER_STRICT: ${{ github.ref_type == 'tag' }}
      SEEKER_API_TOKEN: ${{ secrets.SEEKER_API_TOKEN }}
    steps:
      - uses: actions/checkout@v4
      - run: ci/seeker-gate.sh
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: seeker-results, path: seeker-results.json }
```

**5. Azure DevOps (étape)**

```yaml
- task: Bash@3
  displayName: Seeker quality gate
  inputs: { filePath: ci/seeker-gate.sh }
  env:
    SEEKER_SERVER_URL: https://seeker.corp.example.com
    SEEKER_PROJECT_KEY: pay-checkout-api
    SEEKER_PROJECT_VERSION: $(Build.BuildNumber)
    SEEKER_API_TOKEN: $(SEEKER_API_TOKEN)   # groupe de variables lié à Key Vault
```

**6. Jenkins** : installer le plugin fourni par le serveur (`<seeker>/rest/ui/installers/integrations/jenkins`, fichier `SeekerJenkinsPlugin.hpi`), déclarer l'URL et le token dans *Configure System > Seeker Configuration*, puis ajouter l'étape **Seeker Build Status** avec les clés de projet. En pipeline déclaratif, appeler `ci/seeker-gate.sh` dans un `stage('Seeker gate')` avec `withCredentials([string(credentialsId: 'seeker-api-token', variable: 'SEEKER_API_TOKEN')])`.

### L5 — Checklist de sécurité et de durcissement

**Réseau et TLS**

- [ ] &#91;OBL\] Seeker non exposé sur Internet ; accès entrant limité aux réseaux CI, test et postes internes
- [ ] &#91;OBL\] TLS 1.2+ uniquement, suites faibles désactivées (contrôle `testssl.sh` archivé)
- [ ] &#91;OBL\] HTTP 8080 fermé ; redirection 80 → 443 au proxy
- [ ] &#91;OBL\] Processus internes (8082, 9911) et PostgreSQL (5432) non joignables depuis le réseau
- [ ] &#91;OBL\] LDAPS et TLS PostgreSQL actifs, certificats validés (pas de `-k`, pas de `trustAll`)
- [ ] &#91;REC\] HSTS, limitation de débit et règles WAF de base au proxy

**Système**

- [ ] &#91;OBL\] CIS niveau 1, SELinux enforcing, EDR installé, correctifs à jour
- [ ] &#91;OBL\] Accès SSH par bastion, clés nominatives, pas de connexion root
- [ ] &#91;OBL\] Compte de service Seeker non privilégié ; permissions `data/` en 700
- [ ] &#91;OBL\] Scan de vulnérabilités de l'hôte sans Critical/High ouverte

**Identités et secrets**

- [ ] &#91;OBL\] MFA imposé par l'IdP ; accès conditionnel pour les administrateurs
- [ ] &#91;OBL\] Comptes break-glass au coffre, testés, alertés
- [ ] &#91;OBL\] Aucun rôle sur `everyone` ; aucun rôle nominatif
- [ ] &#91;OBL\] Mots de passe base et bind LDAP au coffre, rotation planifiée
- [ ] &#91;OBL\] Tokens API/agents par périmètre, datés, avec propriétaire et date de rotation
- [ ] &#91;OBL\] Secrets masqués dans les logs CI (vérifié sur un job réel)

**Données, journaux, continuité**

- [ ] &#91;OBL\] Sauvegardes chiffrées, copie hors site immuable, restauration testée
- [ ] &#91;OBL\] Logs Seeker, NGINX, système, PostgreSQL et IdP dans le SIEM ; alertes 7.3 actives
- [ ] &#91;OBL\] Classification « Confidentiel » inscrite dans la CMDB et le registre des traitements si besoin
- [ ] &#91;REC\] Pas de données personnelles réelles dans les environnements instrumentés
- [ ] &#91;REC\] Purge des projets inactifs et des versions de merge request expirées

**Cycle de vie**

- [ ] &#91;OBL\] Procédure de veille et de mise à jour Seeker (au moins trimestrielle)
- [ ] &#91;OBL\] Revue des accès trimestrielle tracée
- [ ] &#91;REC\] Test d'intrusion de la plateforme avant généralisation

### L6 — Plan de rollback et checklist de validation

**Plan de rollback par type de changement**

| Changement | Déclencheur du rollback | Action | Durée |
| --- | --- | --- | --- |
| Mise à jour du serveur Seeker | Échec d'un test de validation ci-dessous, ou erreurs 5xx > 2 % pendant 15 min | Arrêt, restauration du snapshot VM ou de la sauvegarde à froid pré-mise à jour, redémarrage ; agents déjà mis à jour : revenir à la version précédente (le serveur n'accepte pas d'agents plus récents) | 1 h |
| Configuration SAML | Un rôle ne peut plus se connecter | Session break-glass ; restaurer les valeurs précédentes (fiche L3 versionnée) ; en dernier recours désactiver SAML et laisser LDAP | 15 min |
| Configuration LDAP | Connexions ou créations d'utilisateurs en échec | Session break-glass ; restaurer la fiche L3 précédente ; vérifier le compte de bind | 15 min |
| RBAC (rôles, groupes, membres) | Accès excessif ou manquant constaté | `git revert` du fichier RBAC-as-code et réapplication du script | 15 min |
| Version d'agent (projet ou global) | Surcoût > seuils 5.4, plantage applicatif, agent non connecté | Remettre la version précédente dans le projet ; redéployer ; ou `SEEKER_ENABLED=false` | 30 min |
| Template de pipeline | Taux d'échec technique de la gate > 5 % | Repointer l'`include` sur le tag précédent du template ; gate en mode informatif temporaire | 15 min |
| Migration PostgreSQL | Incohérence de données, lenteurs | Repointer Seeker sur l'ancienne base (conservée 7 jours en lecture seule) | 1 h |

Règle : aucun changement sans sauvegarde préalable vérifiée, ticket de changement et créneau annoncé aux équipes CI.

**Checklist de validation SSO / LDAP**

- [ ] Connexion locale break-glass fonctionnelle (page de login, navigation privée)
- [ ] Connexion SSO d'un utilisateur de chaque groupe `GG-SEEKER-*` ; redirection vers l'accueil Seeker
- [ ] Utilisateur créé automatiquement à la première connexion, avec e-mail et nom corrects
- [ ] Rôles reçus = rôles attendus (L2) ; un développeur ne voit que ses projets
- [ ] Utilisateur sans groupe Seeker : refusé par l'IdP (non affecté) ou sans aucune visibilité
- [ ] Retrait d'un groupe AD : perte des droits à la connexion suivante (délai mesuré et documenté)
- [ ] Compte AD désactivé : connexion impossible
- [ ] Déconnexion et nouvelle connexion OK ; MFA demandé selon la politique IdP
- [ ] Logs de connexion visibles dans le SIEM (succès et échecs)

**Checklist de validation des agents**

- [ ] Agent Java, .NET Core et Node.js connectés, visibles dans le projet attendu avec les bons tags
- [ ] Connexion en HTTPS avec certificat validé (aucun `--no-check-certificate`)
- [ ] Token agent requis : un agent sans token ou avec le token d'un autre périmètre est refusé
- [ ] Version d'agent = version fixée pour le projet
- [ ] Vulnérabilité de test (application de démonstration type WebGoat) détectée et vérifiée
- [ ] Surcoût de performance mesuré sous les seuils 5.4
- [ ] Coupure de Seeker : l'application continue de fonctionner normalement
- [ ] Gate CI : bloque sur la vulnérabilité de test, passe sans, passe en warning si Seeker est coupé (hors release)
- [ ] Ticket Jira et notification Teams créés pour la vulnérabilité de test

## Sources

Aide en ligne publique Seeker (Black Duck), consultée en novembre 2026. À recouper avec l'aide intégrée de votre version.

- [Seeker Architecture and Communications](https://demo.seeker.synopsys.com/internal/help/en/topics/c_arch_comm.html) — flux et ports
- [Sizing Guidelines](https://demo.seeker.synopsys.com/internal/help/en/topics/c_sizing_guide.html) — dimensionnement
- [Server Deployment and Initial Setup](https://demo.seeker.synopsys.com/internal/help/en/topics/c_bp_deplsetup.html) — capacité d'un serveur, standby
- [Server System Requirements](https://demo.seeker.synopsys.com/internal/help/en/topics/c_systemreq.html)
- [Configure SAML 2.0-based SSO](https://demo.seeker.synopsys.com/internal/help/en/topics/t_config_saml.html)
- [Managing User Access](https://demo.seeker.synopsys.com/internal/help/en/topics/r_bp_mng_access.html) — RBAC, mapping LDAP
- [Manage Roles](https://demo.seeker.synopsys.com/internal/help/en/topics/t_manage_roles.html) · [Manage Users](https://demo.seeker.synopsys.com/internal/help/en/topics/t_manage_users.html) · [Manage Groups](https://demo.seeker.synopsys.com/internal/help/en/topics/t_manage_groups.html)
- [Application Onboarding Checklist](https://demo.seeker.synopsys.com/internal/help/en/topics/r_bp_onboardapp.html)
- [Appendix A. Seeker Environment Variables](https://demo.seeker.synopsys.com/internal/help/en/topics/r_env_var.html)
- [Connecting and Upgrading Agents](https://demo.seeker.synopsys.com/internal/help/en/topics/c_conn_upgr_agents.html) · [Configuring and Customizing Agents](https://demo.seeker.synopsys.com/internal/help/en/topics/t_config_agents.html)
- [Deploying Agents on Kubernetes](https://demo.seeker.synopsys.com/internal/help/en/topics/r_deploy_kuber.html) · [Deploy Seeker Server on Kubernetes](https://demo.seeker.synopsys.com/internal/help/en/topics/t_deploy_server_kuber.html)
- [Configure Access Tokens](https://demo.seeker.synopsys.com/internal/help/en/topics/t_config_api_tokens.html) · [Common .NET Agent Settings](https://demo.seeker.synopsys.com/internal/help/en/topics/r_config_netagent_common.html)
- [Install and Configure Seeker Jenkins Plugin](https://demo.seeker.synopsys.com/internal/help/en/topics/t_config_jenplugin.html)
- [Backup and Restore](https://demo.seeker.synopsys.com/internal/help/en/topics/c_backup_restore_ovw.html) · [Hot Backup and Restore](https://demo.seeker.synopsys.com/internal/help/en/topics/r_hotbackup_restore.html)
- [API Scenarios](https://demo.seeker.synopsys.com/internal/help/en/topics/r_api_scenarios.html)
