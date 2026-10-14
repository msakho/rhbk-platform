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
operators/
  rhbk/base/                     Namespace, OperatorGroup, Subscription (canal stable-v26.6)
  rhbk/overlays/prod|dev/        namespace, nom de l'OperatorGroup existant
  eso/                           Namespace, OperatorGroup, Subscription (canal stable-v1)
eso/
  externalsecretsconfig.yaml     démarre ESO + NetworkPolicy de sortie vers Vault
  clustersecretstore.yaml        vault-store : ESO -> Vault (auth Kubernetes)
vault/
  values.yaml                    Vault 1 nœud, stockage Raft sur PVC, route
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
| 7 | Dépôt `rhbk-gitops` : bootstrap applicatif | voir son guide |

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
5. Le chart Vault contient un pod de test Helm (`helm.sh/hook: test`), jamais créé par l'installation
   actuelle. Si ArgoCD le fait apparaître lors de la première synchro, il est sans effet sur Vault.

## Vault : étapes manuelles

Ce qui n'est pas déclaratif dans ce dépôt (détail des commandes : guide de `rhbk-gitops`, annexe A) :

| Étape | Quand | Guide `rhbk-gitops` |
|---|---|---|
| Initialiser (clé de descellement + token root → `vault-init.json`, ignoré par git) | une fois, cluster neuf | annexe A.4 |
| Desceller | **après chaque redémarrage du pod ou du cluster** | annexe A.4, section 15.5 |
| Configurer : KV v2 `secret/`, auth Kubernetes, politique `rhbk-read`, rôle `external-secrets` | une fois | annexe A.5 |
| Ranger les valeurs d'un environnement (`secret/rhbk/<env>/...`) | par environnement | annexe A.7 |

**Sauvegarder `vault-init.json`** : sans la clé de descellement, les données de Vault sont
irrécupérables.

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
