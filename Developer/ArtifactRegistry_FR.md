# Artifact Registry

## Vue d'ensemble

Google Cloud Artifact Registry constitue une solution fondamentale pour la gestion des images Docker et le déploiement d'applications sur Google Kubernetes Engine (GKE). Artifact Registry s'intègre à GKE et fournit un référentiel centralisé pour le stockage et la distribution des images de conteneurs. En plus du stockage des images, Artifact Registry permet de gérer les versions des images et d'effectuer une analyse avancée des vulnérabilités des conteneurs.

## Provisionnement

Artifact registry est une ressource de niveau 4 qui peut être approvisionnée à l'aide de la ressource Config Connector [Artifact Registry Repository] (https://cloud.google.com/config-connector/docs/reference/resource-docs/artifactregistry/artifactregistryrepository). Un cas d'utilisation typique pour un registre d'image de conteneur peut être provisionné en utilisant :

```yml
apiVersion: artifactregistry.cnrm.cloud.google.com/v1beta1
kind: ArtifactRegistryRepository
metadata:
  name: artifactregistryrepository-sample
spec:
  format: DOCKER
  location: northamerica-northeast1
```

Reférences:
[ArtifactRegistryRepository Config Connector Documentation](https://cloud.google.com/config-connector/docs/reference/resource-docs/artifactregistry/artifactregistryrepository)

## Gestion des versions

Un référentiel peut contenir de nombreuses images de conteneurs, et ces images peuvent avoir des versions différentes. Pour identifier une version spécifique d'une image, vous pouvez spécifier le résumé de l'image ou le tag.

**Digest**
Un condensé d'image est un hachage généré automatiquement de l'index de l'image ou du manifeste de l'image. Chaque résumé d'image est un identifiant unique pour une version d'image et ne peut pas être modifié. Le condensé est la valeur de hachage sha256 du contenu de l'image.

**Tag**
Un Tag d'image est un balise et est souvent une chaîne lisible par l'humain telle que `v1.1` ou `development`. Un tag ne peut pointer que vers une seule version d'une image. Dans Artifact Registry, vous pouvez configurer un dépôt Docker pour autoriser les tags mutables ou imposer les tags immuables.

- **Mutable Tags**: Un Tag pointe vers une seule version d'une image, mais le résumé spécifique qu'elle référence peut changer.

Une approche commune est de marquer les images avec un identifiant de version, comme `v1.1` au moment de la construction. Lorsque la compilation pousse plusieurs versions de l'image dans le registre avec la même balise `v1.1`, la balise référence le résumé de la dernière version poussée dans le registre. Bien que les balises mutables fournissent un moyen pratique d'étiqueter les versions, elles ont un inconvénient dans la mesure où une balise ne spécifie pas de manière unique une version particulière d'une image.

- **Immutable Tags**: Dans le référentiel, une balise pointe toujours vers le même résumé d'image

Si un référentiel du registre d'artefacts est configuré pour des balises immuables, les actions suivantes ne sont pas autorisées :
- Supprimer une image marquée. La suppression d'images non balisées est toujours autorisée.
- Supprimer une balise d'une image.
- Pousser une image avec une balise qui est déjà utilisée par une autre version de l'image dans le référentiel.


Dans les scénarios de production, il peut être souhaitable d'appliquer des balises immuables pour fournir une spécificité stricte aux artefacts déployés.

Pour activer les balises immuables sur un registre d'artefacts, utilisez le champ `dockerConfig.immutableTags` :

``yml
apiVersion : artifactregistry.cnrm.cloud.google.com/v1beta1
kind : ArtifactRegistryRepository
metadata :
  name : artifactregistryrepository-sample
spec :
  format : DOCKER
  location : northamerica-northeast1
  dockerConfig :
    immutableTags : true
```

Reférences:
[Repository and image names](https://cloud.google.com/artifact-registry/docs/docker/names)

## Pousser des images de conteneurs

1. S'authentifier sur le dépôt :

``bash
gcloud auth configure-docker northamerica-northeast1-docker.pkg.dev
```

2. Marquez l'image locale avec le nom du dépôt :

``bash
docker tag image-name northamerica-northeast1-docker.pkg.dev/project-id/artifactregistryrepository-sample/image-name:tag
```

3. Pousser l'image étiquetée vers le registre d'artefacts :

``bash
docker push northamerica-northeast1-docker.pkg.dev/project-id/artifactregistryrepository-sample/image-name:tag
```

Reférences:
[Push and pull images](https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling)


## Déploiement de l'image de l'application

Pour déployer une application dans un cluster Kubernetes, il suffit de spécifier la référence à l'image du conteneur dans le champ `image` de l'objet `containers` dans le manifeste pour la charge de travail Kubernetes (par exemple, `Deployment`, `StatefulSet`, `DeamonSet`, `CronJob`).  Voici un exemple simple de déploiement :

```yml
piVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: northamerica-northeast1-docker.pkg.dev/project-id/artifactregistryrepository-sample/image-name:tag
        ports:
        - containerPort: 80
```

Reférences:
[Deployments Kubernetes Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

## Gestion de l'accès au cluster

Chaque nœud d'une grappe GKE est associé à un compte de service de gestion des identités et des accès (IAM).  Il s'agit d'un compte de service personnalisé provisionné spécifiquement pour les nœuds de la grappe. Le compte de service a reçu les autorisations nécessaires pour accéder aux référentiels du registre des artefacts dans le projet d'application.  Cela permet au cluster GKE d'extraire des images du référentiel d'artefacts et d'exécuter des applications conteneurisées.

## Analyse des vulnérabilités

L'analyse d'artefact fournit des informations sur les vulnérabilités des images de conteneurs dans le registre d'artefact.  Cette fonctionnalité est automatiquement activée lorsque le projet d'application GKE est provisionné et configuré.  Aucune autre action n'est requise de la part des équipes d'application.

Pour afficher les vulnérabilités d'une image :

```bash
gcloud artifacts docker images list --show-occurrences northamerica-northeast1-docker.pkg.dev/project-id/artifactregistryrepository-sample/image-name
```

Pour afficher les vulnérabilités d'une balise d'image :

```bash
gcloud artifacts docker images describe --show-package-vulnerability northamerica-northeast1-docker.pkg.dev/project-id/artifactregistryrepository-sample/image-name
```

Reférences:
[Scan images for OS vulnerabilities automatically](https://cloud.google.com/artifact-analysis/docs/os-scanning-automatically)