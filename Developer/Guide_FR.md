# Guide du développeur

## Renseignements généraux

Services partagés Canada offre la zone d'accueil GCP en tant que service. Cette zone d'accueil respecte les exigences de sécurité ITSG-33 tout en mettant en œuvre les meilleures pratiques GCP. Ce service peut être utilisé par tous les ministères et organismes du gouvernement du Canada pour héberger leurs applications.

## Objet

Ce guide fournira les informations nécessaires à tout développeur ou opérateur d'applications pour héberger avec succès une charge de travail dans la zone d'accueil du SSC et rationaliser son processus ATO.

## Aperçu de la technologie

SSC a choisi ACM [Anthos Config Management] (https://cloud.google.com/anthos/config-management) pour gérer l'infrastructure en tant que code. ACM utilise le manifeste du connecteur de configuration de Kubernetes pour contrôler les ressources déployées dans GCP. La plupart des développeurs ont une certaine expérience du format de fichier manifeste Kubernetes et ne devraient pas se sentir perdus lorsqu'ils regardent, pour la première fois, une définition de ressource Kubernetes [config connector](https://cloud.google.com/config-connector/docs/reference/overview).

### Les composants d'Anthos Config Management sont les suivants

![ACM Components](img/acm-components.png)

### 1. Config Sync

Les fichiers de manifeste sont stockés et contrôlés par version dans un dépôt git.

Un opérateur Kubernetes fonctionnant sur le cluster Kubernetes de l'ACM, appelé "Config Sync", observe ce dépôt git et réconcilie tous les manifestes en suivant un processus GitOps.

![Config-Sync](img/config-sync.png)
![GitOps](img/gitops.png)

Le processus de déploiement de Git se déroule comme suit
![Git Process](img/git-deployment-process.png)

### 2. Policy Controller

![Policy Controller](img/policy-controller.png)

Example of policy use cases

![use cases](img/policy-use-cases.png)

### 3. Config Controller

![config controller](img/config-controller.png)

## Shared Services Canada - GCP Landing zone (Zone d'accueil)

SSC a mis en place 4 organisations GCP distinctes afin d'isoler chaque environnement les uns des autres. Les environnements sont l'expérimentation, DEV, PREPROD et PROD.

![organizations](img/organizations.png)

## Expérimentation / Sanbox

Cet environnement offre un grand niveau de flexibilité et d'autonomie aux développeurs d'applications. L'objectif est vraiment de vous permettre d'expérimenter n'importe quel service GCP sans nécessiter l'implication de l'administrateur de la plateforme.

  Il répond aux exigences du [TBS Cloud Usage Profile 1](https://canada-ca.github.io/cloud-guardrails/EN/00_Applicable-Scope.html) et autorise uniquement les charges de travail **Unclassified**.

### Tarification

  Ce projet est lié à un compte de facturation GCP appartenant à **"your organization"**.

### Permissions

  Votre équipe se voit attribuer le rôle `Editor` sur le projet GCP.

### Travailler avec GCP

L'interaction avec GCP est possible via la [cloud console](https://console.cloud.google.com/) ou en utilisant [Gcloud SDK](https://cloud.google.com/sdk/docs/) à partir de [Cloud Shell](https://cloud.google.com/shell/docs/) ou de tout autre ordinateur Linux ou Windows, vous pouvez également interagir avec l'API GCP directement. Pour ce faire, vous devez vous authentifier avec votre compte utilisateur `user` de Google Cloud Identity.

### Mise en réseau

  Votre projet est livré avec une configuration réseau existante. Cette configuration répond aux exigences de 30 jours [Guardrails] (https://github.com/canada-ca/cloud-guardrails-gcp) pour l'expérimentation (Guardrails 1,2,4,8,12). Le rôle `Editor` vous permet de créer, par exemple, une règle de pare-feu.

  Voici la liste des ressources réseau qui ont été déployées dans votre projet d'expérimentation :

- VPC (journaux de flux activés)
- Sous-réseaux (régions de Montréal et de Toronto)
  - PAZ
  - APPRZ
  - DATARZ
- Route vers l'internet (les ressources nécessitent un `network tag`)
- Cloud NAT
- DNS logging policy (Politique d'enregistrement DNS)

  ![experimentation networking](img/experimentation-networking.png)

### AutoPilot Cluster Creation

Dans le cas où un client souhaite créer un cluster AutoPilot pour la zone d'accueil d'expérimentation, les configurations suivantes doivent être suivies.

*Note : Ces instructions ne s'appliquent pas au déploiement d'une grappe de contrôleurs de configuration en utilisant `gcloud anthos config controller create`*.

Connectez-vous à [https://console.cloud.google.com/](https://console.cloud.google.com/).


1. Créez votre propre sous-réseau

- Sélectionnez le projet et allez à "VPC Network"
- Cliquez sur "global-vpc1-vpc"
- Cliquez sur "ADD SUBNET"
  - Name: **{user defined}-snet**
  - Region: **northamerica-northeast1**
  - IPv4 range: **10.10.0.0/24**
  - Private Google Access: **On**
  - Flow Logs: **On**
- Cliquez sur "ADD"

2. Créez le cluster avec les paramètres suivants

- Allez à "Kubernetes Engine".
- Activez "Kubernetes Engine API" (if not already enabled)
- Cliquez sur "Create"
  - Cluster Basics
    - Name: **{user defined}**
    - Region:  **northamerica-northeast1**
  - Cliquez sur "Next: Networking"
  - Networking
    - Networking
      - Network: **global-vpc1-vpc**
      - Node subnet: **{new subnet from step 1 above}**
    - Cliquez sur **Private Cluster**
      - Control Plane IP range: **172.16.0.0/28**
      - Cluster default Pod address range: **/17**
      - Service address range: **/22**
    - Auto-provisioning network tags
      - **internet-egress-route**
  - Cliquez sur "CREATE"

### Création d'un cluster standard

Pour déployer un cluster standard, créez le même sous-réseau qu'à l'étape 1 ci-dessus.  Créez le cluster GKE standard à partir du menu Kubernetes Engine. Sélectionnez toutes les configurations dont vous avez besoin.

Veillez à définir les éléments suivants :

- Principes de base du cluster
  - Location Type:  **Regional**
  - Region: **northamerica-northeast1**
- Networking
  - Utilisez les mêmes valeurs de configuration qu'à l'étape 2.
- Cliquez sur "CREATE"  lorsque vous avez terminé.

## DEV, PREPROD et PROD (EN COURS DE CONSTRUCTION)

L'environnement DEV est celui dans lequel vous développerez votre application. PREPROD et PROD doivent être utilisés pour la promotion du code avec le pipeline CD de votre choix.

Comme vous pouvez vous y attendre, le nombre de contrôles de sécurité implémentés pour DEV, PREPROD et PROD est beaucoup plus important que pour l'environnement d'expérimentation. Les 12 `30 days guardrails` et de nombreux contrôles ITSG 33 supplémentaires sont appliqués dans ces environnements afin que vous puissiez rationaliser le processus ATO de vos applications en tirant parti de l'ATO de la zone d'accueil SSC GCP.

### Tarification

Vos projets sont liés à un compte de facturation GCP appartenant à **"votre organisation "**.

### Permissions

Votre équipe (développeurs) se voit attribuer le rôle `Viewer` sur le projet GCP dans l'organisation Dev uniquement. Pour assurer la séparation des tâches, c'est l'opérateur de l'application qui se voit attribuer ce même rôle pour les organisations PREPROD et PROD.

### Travailler avec GCP

En tant que développeur/opérateur, vous aurez accès aux dépôts git. Ces dépôts sont observés par ACM - ConfigSync ([voir ci-dessus](#1-config-sync)) de sorte que le manifeste Kubernetes existant sera automatiquement appliqué par ACM. Les ressources GCP (GCE, GCS, etc.) dont vous avez besoin pour votre application doivent être définies dans un fichier manifeste en suivant le connecteur config [schema](https://cloud.google.com/config-connector/docs/reference/overview).


La zone d'accueil utilise 3 niveaux différents de dépôts git pour gérer les ressources GCP. Le diagramme ci-dessous montre quel type de ressource est défini dans chaque niveau ainsi que les rôles et permissions configurés pour chacun d'entre eux.

![repo-structure](img/repo-structure.png)

### Tier 2

Comme vous l'avez vu dans ce diagramme, les ressources pour votre application sont incluses dans deux niveaux appelés "niveau 2" et "niveau 3". C'est au niveau 2 que cette zone d'accueil se distingue des autres services traditionnels en essayant de simplifier autant que possible la gestion de la sécurité de l'application tout en maintenant un contrôle très fort et sécurisé sur l'environnement. Vous disposez d'un référentiel de niveau 2 pour chaque référentiel de niveau 3 (relation de un à un). En tant que développeur/opérateur d'application, vous avez un contrôle total (contributeur + réviseur) sur votre référentiel de niveau 3, mais vous avez également un rôle de contributeur sur le référentiel de niveau 2, ce qui permet de mettre en œuvre ce processus simplifié de gestion des changements. C'est l'administrateur de la plateforme et l'administrateur de la sécurité qui examineront et approuveront le PR.

Ainsi, lorsque vous voulez qu'un changement soit implémenté sur votre repo de niveau 2, vous avez deux options :

1. créer une branche sur le repo de niveau 2, faire et livrer vos changements et ensuite, ouvrir une demande d'extraction.
2. contacter l'administrateur de la plateforme et trouver une solution

Une fois que cette modification est fusionnée dans la branche principale, un nouveau tag est créé sur le repo. Ce tag est un numéro de version qui suit le [standard](https://semver.org/) major.minor.patch. TODO : mise à jour avec les exigences du processus de marquage (changelog, etc.)

ConfigSync n'est pas configuré pour observer la branche principale, à la place, il est configuré pour observer un tag spécifique sur le repo `Infra`. Si vous voulez que votre nouvelle version soit effective pour DEV, PREPROD ou PROD, vous devez changer ce tag à l'intérieur du repo `ConfigSync`.

Ce processus complet est illustré dans le diagramme suivant

![gitops](img/gitops-ssc.png)

TODO: Fournir un exemple

### Nomenclature des dépôts

Le diagramme ci-dessous décrit la convention de nommage des dépôts et utilise na, en tant que client, pour fournir des exemples.

![naming](img/naming-repos.png)

### Mise en réseau

Ci-dessous se trouve la conception de haut niveau du réseau.
TODO : fournir plus de détails une fois que la conception du réseau est finalisée.

![hld](img/networking-hld-env.png)

## Formation

- GCP
  - Principes de base de Google Cloud Platform
  - Développer des applications avec Google Cloud

- CCCS
  - L'infonuagique au sein du GC : le processus SA&A

## Addendum aux ressources de la zone d'accueil GCP

[Resources Addendum](../Architecture/Addendum.md)