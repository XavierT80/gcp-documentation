# Création de la zone d'accueil

- [Création de la zone d'accueil](#landing-zone-building)
  - [Prérequis](#requirements)
  - [1. Créer le monorepo `tier1`](#1-create-tier1-monorepo)
  - [2. Bootstrap le projet Config Controller](#2-bootstrap-the-config-controller-project)
  - [3. Construire le coeur de la zone d'accueil](#3-build-the-core-landing-zone)
    - [Détails du paquet](#package-details)
  - [4. Ajouter les certificat SSL de Cloud IDS and Compute au tier1/configcontroller/crds](#4-add-cloud-ids-and-compute-ssl-certificate-to-tier1configcontrollercrds)
  - [5. Effectuer les étapes d'après déploiement](#5-perform-the-post-deployment-steps)
  - [LA FIN](#the-end)
  - [Prochaines étapes](#next-step)

--------------------------------------

Services partagés Canada a automatisé une partie du processus de déploiement de [landing-zone-v2](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/blob/main/docs/landing-zone-v2/README .md#Organization) dans le dépôt Pubsec Toolkit de Google.

Ce document décrira comment les scripts automatisés peuvent être utilisés pour créer une zone d'atterrissage.

**Nous vous recommandons de lire l'intégralité de la procédure avant de la lancer**

**Important** SSC utilise les référentiels et pipelines Azure Devops comme solution git.

## Prérequis

Services partagés Canada utilise l'architecture "[Multiple GCP organizations](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/blob/main/docs/landing-zone-v2/README.md#multiple-gcp-organizations)".

Passez en revue les exigences énumérées [here](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/blob/main/docs/landing-zone-v2/README.md#requirements).

## 1. Créer le monorepo `tier1`

SSC utilise le déploie de [Gitops-Git](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/tree/main/docs/landing-zone-v2/README.md#gitops---git).
Comme illustré dans ce diagramme [Gitops](../Architecture/Repository%20Structure.md#Gitops), l'opérateur ConfigSync observe notre monorepos de déploiement.

Suivez la section « Create New Deployment Monorepo » dans [Repositories.md](./Repositories.md) pour créer l'un des deux monorepo de « niveau 1 » :

- `gcp-experimentation-tier1`: Si vous déployer la version d'expérimentation de la zone d'accueil.
- `gcp-env-tier1`: Si vous déployer la version dev, preprod or prod de la zone d'accueil.

## 2. Bootstrap le projet Config Controller

Le script automatisé crée un projet, les paramètres FW, un routeur cloud, un Cloud NAT, un point de terminaison de connexion de service privé et le cluster Anthos Config Controller. Il crée également un fichier root-sync.yaml.

Le script nécessite un fichier « .env » pour déployer l'environnement.

1. Démarrez une nouvelle modification pour votre monorepo `tier1`, suivez la section "Étape 1 - Configuration" de [Changing.md](./Changing.md)
    - Expérimentation

        monorepo name = `gcp-experimentation-tier1`
    - DEV, PREPROD, PROD

        monorepo name = `gcp-env-tier1`
2. Votre terminal devrait maintenant être à la racine de votre monorepo `tier1`, sur une nouvelle branche et avec le sous-module tools renseigné.
3. S'il n'existe pas, créez un répertoire `bootstrap/<env>`. Il sera utilisé pour sauvegarder les fichiers générés lors du démarrage.
4. Copiez le fichier example.env du dossier `tools/scripts/bootstrap`.

    ```shell
    cp tools/scripts/bootstrap/.env.sample bootstrap/<ENV>/.env
    ```

5. **Important** Personnalisez le nouveau fichier avec les valeurs appropriées pour la zone d'atterrissage que vous créez.

6. Exportez une variable `TOKEN`. Définissez sa valeur sur le PAT qui a un accès en lecture au monorepo `tier1`. Il sera utilisé par `setup-kcc.sh` pour créer un secret appelé `git-creds` dans le "namespace" `config-management-system`.

    ```shell
    export TOKEN='xxxxxxxxxxxxxxx'
    ```

7. Exécutez le script automatisé setup-kcc :

    ```shell
    bash tools/scripts/bootstrap/setup-kcc.sh [-af] <PATH TO .ENV FILE>
    ```

    >- -a: autopilot. Il déploiera un cluster de pilote automatique au lieu d'un cluster standard.
    >- -f: folder_opt. Cela amorcera la zone de destination dans un dossier plutôt qu'au niveau de l'organisation.

8. Une fois le script terminé, vous devez déplacer le fichier `root-sync.yaml` dans le dossier `bootstrap/<env>` du monorepo `tier1`.

    ```shell
    mv root-sync.yaml bootstrap/<ENV>
    ```

9. Validez le déploiement de Config Controller. Vous devriez voir une `root-sync` synchronisée avec votre monorepo `...<tier1 monorepo>/csync/deploy/<env>@main` ne contenant aucune ressource.

    ```shell
    nomos status --contexts gke_${PROJECT_ID}_northamerica-northeast1_krmapihost-${CLUSTER}
    ```

## 3. Construire le coeur de la zone d'accueil

Vous construirez la zone d'atterrissage principale en ajoutant une collection de packages au monorepo « tier1 ».
À un niveau élevé, le processus ci-dessous doit être complété pour chaque package :

1. Configurez votre modification, suivez l'étape 1 de [Changing.md](./Changing.md#step-1---setup)
2. Ajoutez un package, suivez l'étape 2A de [Changing.md](./Changing.md#a-add-a-package)
3. Générez des fichiers hydratés, suivez l'étape 3 de [Changing.md](./Changing.md#step-3---hydrate).
4. Publiez les modifications dans le référentiel, suivez l'étape 4 de [Changing.md](./Changing.md#step-4---publish).
5. Une fois le PR fusionné, notez la nouvelle version de la balise ou validez SHA. Cela sera nécessaire dans la section suivante.
6. Synchronisez et promouvez la configuration, suivez l'étape 5 de [Changing.md](./Changing.md#step-5---synchronize--promote-configs).

### Détails du paquet

Vous ajouterez les 2 packages ci-dessous à votre monorepo `tier1`.
> **!!! Il est important que toutes les étapes répertoriées ci-dessus soient effectuées pour chaque package avant de passer au package suivant. !!!**

Les détails ci-dessous sont requis lors de l'exécution de l'étape 2A « Ajouter un package » de [Changing.md](./Changing.md)

1. Le paquet gatekeeper-policies :

    - Pour l'expérimentation, vous déployez ceci [package](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/tree/main/solutions/gatekeeper-policies) dans le repo `gcp-experimentation-tier1`.
    - Pour Dev, PreProd et Prod, vous déployez ceci [package](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/tree/main/solutions/gatekeeper-policies) dans le repo `gcp-env-tier1`.
    - Détails du paquet (identitique pour tous les environnements) (quand vous exécutez [step 2A](Changing.md#a-add-a-package)):

        ```shell
        export TIER='tier1'

        export REPO_URI='https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit.git'

        export PKG_PATH='solutions/gatekeeper-policies'

        # the version to get, located in the package CHANGELOG.md, use 'main' if not available
        export VERSION=''

        export LOCAL_DEST_DIRECTORY=''
        ```

    - Customization (same for all environments):

        ```shell
        export FILE_TO_CUSTOMIZE='gatekeeper-policies/naming-rules/project/setters.yaml'
        ```

2. Le paquet du coeur de la zone d'accueil:
    - Pour l'expérimentation, vous déployez ce [paquet](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/tree/main/solutions/experimentation/core-landing-zone) dans le repo `gcp-experimentation-tier1`.
      - Détails du paquet (quand vous exécutez [step 2A](Changing.md#a-add-a-package)):

        ```shell
        export TIER='tier1'

        export REPO_URI='https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit.git'

        export PKG_PATH='solutions/experimentation/core-landing-zone'

        # the version to get, located in the package CHANGELOG.md, use 'main' if not available
        export VERSION=''

        export LOCAL_DEST_DIRECTORY=''
        ```

      - Customization:

          ```shell
          export FILE_TO_CUSTOMIZE='core-landing-zone/setters.yaml'
          ```

    - Pour Dev, PreProd et Prod, vous déployez ceci [package](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/tree/main/solutions/core-landing-zone) dans le repo `gcp-env-tier1`.

      - Détails du paquet (quand vous exécutez [step 2A](Changing.md#a-add-a-package)):

        ```shell
        export TIER='tier1'

        export REPO_URI='https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit.git'

        export PKG_PATH='solutions/core-landing-zone'

        # the version to get, located in the package CHANGELOG.md, use 'main' if not available
        export VERSION=''

        export LOCAL_DEST_DIRECTORY=''
        ```

      - Customization:

          ```shell
          export FILE_TO_CUSTOMIZE='core-landing-zone/setters.yaml'
          ```

## 4. Ajouter les certificat SSL de Cloud IDS and Compute au tier1/configcontroller/crds

Le CRDS suivant pour Cloud IDS et Compute SSL Certificate doit être stocké dans tier1/configcontroller/crds.

- [Cloud IDS](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/master/crds/cloudids_v1alpha1_cloudidsendpoint.yaml)

- [Compute Managed SSL Certificate](https://github.com/GoogleCloudPlatform/k8s-config-connector/blob/master/crds/compute_v1alpha1_computemanagedsslcertificate.yaml)


## 5. Effectuer les étapes d'après déploiement

1. Certaines ressources du package `core-landing-zone` ne pourront pas être déployées tant que le nouveau `projects-sa` n'aura pas obtenu le rôle `billing.user`.

    Effectuez l'étape 5 de cette [procédure](https://github.com/GoogleCloudPlatform/pubsec-declarative-toolkit/blob/main/docs/landing-zone-v2/README.md#5-perform-the-post-deployment -étapes) pour résoudre ce problème.

2. Vous devez implémenter [Policy Controller Metrics](https://cloud.google.com/anthos-config-management/docs/how-to/policy-controller-metrics) pour éviter de nombreuses erreurs IAM sur l'instance Config Controller.
    - La majeure partie de cette procédure est couverte par les ressources du package `core-landing-zone` mais vous devez quand même exécuter les commandes ci-dessous :

      1. Annotez le compte de service Kubernetes à l'aide de l'adresse e-mail du compte de service Google :

          ```shell
          export PROJECT_ID='<management project id>'

          kubectl annotate serviceaccount \
            --namespace gatekeeper-system \
            gatekeeper-admin \
            iam.gke.io/gcp-service-account=gatekeeper-admin-sa@${PROJECT_ID}.iam.gserviceaccount.com
          ```

      2. Redémarrer le pod gatekeeper-controller-manager :

          ```shell
          kubectl rollout restart deployment gatekeeper-controller-manager -n gatekeeper-system
          ```

## LA FIN

Toutes nos félicitations! Vous avez terminé le déploiement de votre zone d'atterrissage principale conformément à la mise en œuvre de SSC.

## Prochaines étapes

Suivez les étapes "client onboarding" [procedure](../Onboarding/Client.md).
