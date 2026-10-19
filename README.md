# rhbk-platform

Dépôt GitOps **plateforme** (administrateurs) pour Red Hat build of Keycloak : opérateurs,
External Secrets Operator (ESO) et Vault.

Il complète le dépôt **applicatif** [`rhbk-gitops`](https://github.com/msakho/rhbk-gitops), qui
déploie les instances Keycloak (CR, routes, ExternalSecrets) avec des droits limités.

| | `rhbk-platform` (ce dépôt) | `rhbk-gitops` |
|---|---|---|
| Responsables | administrateurs | équipe applicative |
| Contenu | opérateurs RHBK et ESO, configuration ESO, `ClusterSecretStore`, Vault | CR Keycloak, routes, ExternalSecrets |
| AppProject ArgoCD | `platform` (ressources cluster autorisées) | `rhbk` (seulement `Namespace` au niveau cluster) |
| Synchro | **manuelle** pour toutes les Applications | dev automatique, prod manuelle |

## Arborescence

```
bootstrap/root-app.yaml          App of apps "platform-root", appliquée une seule fois à la main
argocd/
  appproject.yaml                Projet "platform"
  applications/
    rhbk-operator-prod.yaml      operators/rhbk/overlays/prod -> namespace rhbk
    rhbk-operator-dev.yaml       operators/rhbk/overlays/dev  -> namespace rhbk-dev
    eso-operator.yaml            operators/eso                -> namespace external-secrets-operator
    vault.yaml                   chart Helm hashicorp/vault 0.34.1 + vault/values.yaml -> namespace vault
    eso-config.yaml              eso/ (ExternalSecretsConfig + ClusterSecretStore)
    platform-monitoring.yaml     monitoring/ -> surveillance des projets utilisateur + alertes plateforme
    grafana-operator.yaml        operators/grafana -> namespace grafana (opérateur communautaire)
    grafana.yaml                 grafana/ -> instance Grafana, source Thanos, tableaux de bord Keycloak
    coo-operator.yaml            operators/coo -> Cluster Observability Operator (Red Hat)
    observability-console.yaml   observability-console/ -> plugins de la console (Perses, Logs) + vue d'ensemble RHBK
    logging-operator.yaml        operators/logging -> Red Hat OpenShift Logging (stable-6.6)
    loki-operator.yaml           operators/loki -> Loki Operator Red Hat (stable-6.6)
    logging.yaml                 logging/ -> bucket NooBaa, LokiStack, ClusterLogForwarder
operators/
  rhbk/base/                     Namespace, OperatorGroup, Subscription (canal stable-v26.6)
  rhbk/overlays/prod|dev/        namespace, nom de l'OperatorGroup existant
  eso/                           Namespace, OperatorGroup, Subscription (canal stable-v1)
eso/
  externalsecretsconfig.yaml     démarre ESO + NetworkPolicy de sortie vers Vault
  clustersecretstore.yaml        vault-store : ESO -> Vault (auth Kubernetes)
vault/
  values.yaml                    Vault 1 nœud, stockage Raft sur PVC, route
operators/grafana/               Namespace, OperatorGroup, Subscription (canal v5, community-operators)
operators/logging/               Namespace openshift-logging, OperatorGroup, Subscription (cluster-logging)
operators/loki/                  Namespace openshift-operators-redhat, OperatorGroup, Subscription (loki-operator, Red Hat)
logging/
  bucket.yaml                    ObjectBucketClaim NooBaa (bucket loki-rhbk)
  loki-s3-secret.yaml            conversion des identifiants du bucket au format LokiStack (ESO, fournisseur kubernetes)
  lokistack.yaml                 LokiStack logging-loki, taille 1x.demo, rétention 7 jours
  clusterlogforwarder.yaml       collecteur : logs application + openshift-gitops (ArgoCD) -> LokiStack
operators/coo/                   Namespace, OperatorGroup, Subscription (canal stable, redhat-operators)
observability-console/
  uiplugin-monitoring.yaml       UIPlugin "monitoring" (Perses activé) : tableaux Perses dans la console
  uiplugin-logging.yaml          UIPlugin "logging" : logs dans la console (Observe -> Logs)
  namespace.yaml                 projet rhbk-observability
  datasource.yaml                source par défaut du projet : Thanos Querier (jeton de l'utilisateur connecté)
  dashboard-rhbk-overview.yaml   vue d'ensemble RHBK (13 panneaux, généré en JSON)
grafana/
  grafana.yaml                   instance Grafana (image figée) + proxy OAuth OpenShift
  proxy-service-route.yaml       Service du proxy (certificat service-ca) + route reencrypt
  proxy-cookie-secret.yaml       secret de session du proxy, généré par ESO (générateur Password)
  datasource-serviceaccount.yaml ServiceAccount en lecture seule (cluster-monitoring-view) + jeton
  datasource.yaml                source de données Thanos Querier (plateforme + projets utilisateur)
  dashboards.yaml                dossier "Keycloak" + 2 tableaux de bord officiels (version 26.4.0)
monitoring/
  cluster-monitoring-config.yaml active la surveillance des projets utilisateur (enableUserWorkload)
  user-workload-monitoring-config.yaml  rétention 7 jours, PVC de 5 Gi par réplique Prometheus
  eso-servicemonitor.yaml        collecte des métriques ESO
  eso-alerts.yaml                ExternalSecretNotSynced, ClusterSecretStoreNotReady
  vault-alerts.yaml              VaultSealedOrDown (pod Vault non prêt = scellé)
  argocd-alerts.yaml             ArgoCDApplicationUnhealthy
  alertmanager-main.yaml         configuration d'Alertmanager : route des alertes RHBK vers "rhbk-team"
  alert-webhook-test/            récepteur de test (webhook qui écrit les alertes dans ses logs)
  examples/alertmanager-main-smtp.yaml   modèle SMTP (ExternalSecret, non déployé) pour passer à l'e-mail
```

## Choix

- **Opérateur RHBK : une installation par namespace.** Il ne supporte que OwnNamespace /
  SingleNamespace. Le canal est figé à la version mineure (`stable-v26.6`) : la version de l'opérateur
  doit correspondre à l'image Keycloak (26.6).
- **Un seul OperatorGroup par namespace.** Les overlays reprennent les noms des OperatorGroups créés
  par la console (`rhbk-5htxv`, `rhbk-dev-m6xt2`, `external-secrets-operator-xlnvn`). Un nouvel
  environnement utilise le nom de la base, `rhbk-operator`.
- **`installPlanApproval: Automatic`**, comme aujourd'hui. Pour valider chaque montée de version en
  prod, passer à `Manual` (patch dans `operators/rhbk/overlays/prod`) : les InstallPlans sont alors
  à approuver dans la console.
- **Vault par Helm via ArgoCD** (Application multi-source : chart HashiCorp + valeurs de ce dépôt).
  L'initialisation, le descellement et la configuration interne de Vault restent manuels (voir plus bas).
- **Pas de synchro automatique** : chaque changement de plateforme est relu (App Diff) puis synchronisé.

## Ordre de déploiement (cluster neuf)

| Étape | Application / action | Vérification |
|---|---|---|
| 1 | `oc apply -f bootstrap/root-app.yaml` | `oc get applications -n openshift-gitops` : les 5 Applications existent |
| 2 | Sync `eso-operator` | CSV `openshift-external-secrets-operator` en `Succeeded` |
| 3 | Sync `vault` | pod `vault-0` `Running` (non prêt : scellé) |
| 4 | **Manuel** : initialiser, desceller et configurer Vault, ranger les valeurs | voir « Vault : étapes manuelles » |
| 5 | Sync `eso-config` | `oc get clustersecretstore vault-store` → `Valid` |
| 6 | Sync `rhbk-operator-prod`, `rhbk-operator-dev` | CSV `rhbk-operator` en `Succeeded` dans chaque namespace |
| 7c | Sync `loki-operator`, `logging-operator`, puis `logging` | `oc get lokistack logging-loki -n openshift-logging` : Ready ; pods `collector-*` sur chaque nœud |
| 7b | Sync `coo-operator` puis `observability-console` | console : Observe → Dashboards (Perses) → `rhbk-observability` |
| 7a | Sync `grafana-operator` puis `grafana` | `https://grafana.apps.<domaine>` : page « Log in with OpenShift » |
| 7 | Sync `platform-monitoring` | pods `prometheus-user-workload-*` et `thanos-ruler-user-workload-*` en `Running` |
| 8 | Dépôt `rhbk-gitops` : bootstrap applicatif | voir son guide |

Les annotations `SkipDryRunOnMissingResource` permettent de créer les Applications avant que les CRD
(ESO) n'existent ; la synchro de `eso-config` doit simplement venir après `eso-operator`.

## Reprise de l'existant (cluster actuel)

Tout est déjà installé (à la main ou par la console). Les manifestes de ce dépôt reprennent les
**mêmes noms et les mêmes valeurs** : la première synchro de chaque Application n'ajoute que des labels
et des annotations (vérifié avec `oc diff -k`), et le rendu Helm de Vault est identique à l'installation
actuelle.

1. Créer le dépôt `rhbk-platform` sur GitHub, pousser `main`.
2. `oc apply -f bootstrap/root-app.yaml`.
3. Pour chaque Application, dans la console ArgoCD : **App Diff** (seulement labels / annotations),
   puis **Sync**.
4. Vault était installé avec la CLI Helm : ne plus utiliser `helm upgrade` ; ArgoCD gère maintenant ces
   ressources. Les Secrets `sh.helm.release.v1.vault.*` du namespace `vault` ne servent plus et peuvent
   être supprimés.
5. Le chart Vault contient un pod de test Helm (`helm.sh/hook: test`). ArgoCD l'ignore : l'Application
   `vault` est directement `Synced`.
6. **Namespaces repris d'un dépôt applicatif** : si une Application applicative gérait le namespace
   (`CreateNamespace` + `managedNamespaceMetadata`), retirer d'abord ces options de son côté, puis supprimer
   l'annotation qu'elle avait posée, sinon `rhbk-operator-<env>` reste `OutOfSync` sur le Namespace :
   ```bash
   oc annotate ns rhbk-dev argocd.argoproj.io/sync-options-
   ```

## Surveillance

`platform-monitoring` active la surveillance des projets utilisateur (sans elle, les ServiceMonitors et
PrometheusRules hors des namespaces `openshift-*` sont ignorés) et ajoute les alertes de la plateforme.
Les alertes Keycloak sont dans le dépôt applicatif (guide, sections 5.6 et 15.6). Alertes visibles dans
la console : **Observe → Alerting**.

**Notifications** : `alertmanager-main.yaml` route les alertes des namespaces `rhbk*`, `vault`,
`external-secrets` et `openshift-gitops` vers le destinataire `rhbk-team` (les destinataires par défaut
d'OpenShift sont conservés). Aujourd'hui `rhbk-team` est le **webhook de test** (`oc logs -n
alert-webhook-test deploy/alert-webhook`). Pour passer à l'e-mail : modèle
`monitoring/examples/alertmanager-main-smtp.yaml`, procédure dans le guide de `rhbk-gitops`,
section 15.7.2 (paramètres SMTP dans Vault, `secret/platform/alertmanager/smtp`).

- **Stockage** : les 2 Prometheus des projets utilisateur ont chacun un PVC de 5 Gi
  (`ocs-storagecluster-ceph-rbd`), rétention 7 jours (`retentionSize: 4GB` en garde-fou). Les métriques
  survivent aux redémarrages du cluster. Environ 4 000 séries aujourd'hui : quelques centaines de Mo
  sur 7 jours. Le volume peut être agrandi (la classe de stockage le permet).
- **Vault** n'expose pas de métriques : un Vault scellé est détecté par son pod non prêt.
- **ESO** : les NetworkPolicies de l'opérateur autorisent déjà la collecte depuis
  `openshift-user-workload-monitoring` ; le namespace de chaque ExternalSecret est dans le label
  `exported_namespace`.
- **ArgoCD** : `openshift-gitops` est un namespace plateforme ; sa règle est évaluée par le Prometheus de
  la plateforme.

## Logs centralisés (OpenShift Logging + Loki)

Console : **Observe → Logs** (ou onglet **Aggregated Logs** d'un pod).

| Élément | Détail |
|---|---|
| Collecte | Vector sur chaque nœud (`collector-*`), `ClusterLogForwarder collector` |
| Périmètre | logs **application** (tous les namespaces hors `openshift-*`, `kube-*`, `default` : Keycloak, PostgreSQL, Vault, ESO, Grafana…) + parmi l'**infrastructure**, seulement `openshift-gitops` (ArgoCD), grâce à un filtre `drop` |
| Stockage | `LokiStack logging-loki`, taille **1x.demo** (pas de haute disponibilité : l'avertissement « ingester replicas (1) » est attendu ; production : 1x.pico ou plus), rétention 7 jours |
| Stockage objet | bucket NooBaa `loki-rhbk` (ObjectBucketClaim). Les identifiants générés par NooBaa sont convertis au format attendu par Loki par un ExternalSecret (fournisseur `kubernetes` d'ESO) : rien dans git, rien à recopier |
| Tenants | `application` et `infrastructure` (mode `openshift-logging`) : chacun ne voit que les logs des namespaces auxquels il a accès |

Ordre : Sync `loki-operator` et `logging-operator`, puis `logging`, puis `observability-console`.

## Vue d'ensemble dans la console (Perses)

Console OpenShift → **Observe → Dashboards (Perses)** → projet **`rhbk-observability`** →
**RHBK - vue d'ensemble**. Fourni par le Cluster Observability Operator (Red Hat).

| Section | Panneaux |
|---|---|
| État | pods Keycloak et PostgreSQL prêts (prod / dev), Vault prêt (= descellé), ExternalSecrets en erreur, Applications ArgoCD non saines, alertes RHBK actives |
| Keycloak : trafic | requêtes/s, taux de 5xx, temps de réponse p95, connexions réussies et échouées (prod et dev superposés) |
| Keycloak : ressources | connexions à la base, mémoire JVM, CPU |

La source de données interroge Thanos Querier **avec le jeton de l'utilisateur de la console** : chacun ne
voit que les métriques qu'il a le droit de voir, sans ServiceAccount ni jeton à gérer.

Grafana (section suivante) garde les tableaux officiels détaillés de Keycloak ; Perses donne la synthèse.

## Grafana

URL : `https://grafana.apps.cluster-nnljw.nnljw.sandbox742.opentlc.com` → **Log in with OpenShift**.
Accès réservé aux comptes autorisés à lire les services du namespace `grafana` (administrateurs du
cluster), avec le rôle Grafana *Editor*. Pas de formulaire de connexion Grafana.

| Élément | Détail |
|---|---|
| Opérateur | Grafana Operator v5 (**communautaire**, non supporté par Red Hat) |
| Image Grafana | figée par digest dans `grafana/grafana.yaml` (l'opérateur l'écrit dans le CR) |
| Métriques | Thanos Querier, ServiceAccount `grafana-thanos` (`cluster-monitoring-view`), jeton généré par OpenShift |
| Tableaux de bord | officiels Keycloak (*Troubleshooting*, *Capacity planning*), version `26.4.0` du dépôt `keycloak/keycloak-grafana-dashboard`, téléchargés par l'opérateur ; filtre par `namespace` (`rhbk`, `rhbk-dev`) |
| Secrets | aucun dans git : jeton créé par OpenShift, secret de session du proxy généré par ESO |

Ordre : Sync `grafana-operator`, puis `grafana` (l'Application réessaie seule si l'instance est encore
en cours de démarrage).

**Identifiant de la source de données** : l'opérateur remplace l'entrée `DS_PROMETHEUS` des tableaux par
la valeur `datasourceName` (`Prometheus`) **dans le champ uid**. La source de données a donc l'uid fixe
`Prometheus` (`grafana/datasource.yaml`) ; sans cela, la variable `namespace` reste vide et les panneaux
n'affichent rien. `spec.uid` est immuable : pour le changer, supprimer le `GrafanaDatasource` puis
resynchroniser.

Mettre à jour les tableaux de bord : changer la version dans les URL de `grafana/dashboards.yaml`.
Les modifications faites dans l'interface sur ces tableaux sont écrasées par l'opérateur.

## Vault : étapes manuelles

Ce qui n'est pas déclaratif dans ce dépôt (détail des commandes : guide de `rhbk-gitops`, annexe A) :

| Étape | Quand | Guide `rhbk-gitops` |
|---|---|---|
| Initialiser (clé de descellement + token root → `vault-init.json`, ignoré par git) | une fois, cluster neuf | annexe A.4 |
| Desceller | **après chaque redémarrage du pod ou du cluster** | annexe A.4, section 15.5 |
| Configurer : KV v2 `secret/`, auth Kubernetes, politique `rhbk-read`, rôle `external-secrets` | une fois | annexe A.5 |
| Ranger les valeurs d'un environnement (`secret/rhbk/<env>/...`) | par environnement | annexe A.7 |

`vault-init.json` (clé de descellement et token root) est conservé **dans le dossier local de ce dépôt**
(`/usr/local/dev/rhbk-platform`), ignoré par git ; les commandes manuelles se lancent depuis ce dossier.
**Le sauvegarder** : sans la clé de descellement, les données de Vault sont irrécupérables.

## Ajouter un environnement (exemple : `qualif`)

1. Copier l'overlay de dev :
   ```bash
   cp -r operators/rhbk/overlays/dev operators/rhbk/overlays/qualif
   ```
2. Dans `operators/rhbk/overlays/qualif/kustomization.yaml` : `namespace: rhbk-qualif`,
   `environment: qualif`, la valeur `rhbk-qualif` dans le patch `targetNamespaces`, et **supprimer le
   patch qui renomme l'OperatorGroup** (nouveau namespace : le nom de la base, `rhbk-operator`, convient).
3. Créer `argocd/applications/rhbk-operator-qualif.yaml` (copie de `rhbk-operator-dev.yaml`), l'ajouter
   à `argocd/kustomization.yaml`, et ajouter `rhbk-qualif` aux `destinations` de `argocd/appproject.yaml`.
4. Commit, push, Sync de `rhbk-operator-qualif`.
5. Ranger les valeurs de qualif dans Vault (annexe A.7 du guide), créer la base PostgreSQL, puis suivre
   la section 15.3 du guide de `rhbk-gitops`.

## Valider en local

```bash
for d in operators/rhbk/overlays/prod operators/rhbk/overlays/dev operators/eso eso argocd; do oc kustomize $d > /dev/null && echo "ok $d"; done
```

```bash
helm template vault hashicorp/vault --version 0.34.1 -n vault -f vault/values.yaml > /dev/null && echo "ok vault"
```

Écart avec le cluster (doit se limiter aux labels / annotations) :

```bash
oc diff -k operators/rhbk/overlays/prod
```
