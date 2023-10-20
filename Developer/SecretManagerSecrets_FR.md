# Créer et accéder à un secret à l'aide du Secret Manager

## Vue d'ensemble

Le stockage de données sensibles en dehors du cluster Kubernetes réduit le risque d'accès non autorisé aux données. Les fournisseurs de stockage de secrets externes tels que [Google Secret Manager] (https://cloud.google.com/secret-manager) fournissent un service dédié au stockage externe de données sensibles.

Les applications déployées sur les clusters Google Kubernetes Engine (GKE) peuvent accéder aux secrets stockés dans Secret Manager à l'aide de Workload Identity.

Chaque nœud d'un cluster GKE est associé à un compte de service de gestion des identités et des accès (IAM). Il s'agit d'un compte de service personnalisé provisionné spécifiquement pour les nœuds de la grappe. Le compte de service a reçu les autorisations nécessaires pour accéder aux services de Google Secret Manager dans le projet d'application. Cela permet aux applications déployées dans le cluster GKE d'accéder aux secrets.

Références :
[Secrets Manager Documentation](https://cloud.google.com/secret-manager/docs/overview)

## Création d'un secret

Pour créer un secret à partir d'un fichier :

```bash
gcloud secrets create my-secret --data-file=/tmp/secret --replication-policy=user-managed --locations=northamerica-northeast1
```

Pour créer un secret avec la valeur `s3cr3t`

```bash
printf "s3cr3t" | gcloud secrets create my-secret --data-file=- --replication-policy=user-managed --locations=northamerica-northeast1
```

Reférences:
[`gcloud secrets` CLI reference](https://cloud.google.com/sdk/gcloud/reference/secrets)

## Accéder à un secret dans les applications

Pour accéder à un secret dans une application, l'application doit récupérer par programme le secret à partir de Google Cloud Secret Manager.  Pour obtenir des exemples de mise en œuvre dans différents langages de programmation, reportez-vous à [la documentation officielle] (https://cloud.google.com/secret-manager/docs/reference/libraries).

Références:
[Accès aux secrets stockés en dehors des clusters GKE à l'aide de Workload Identity](https://cloud.google.com/kubernetes-engine/docs/tutorials/workload-identity-secrets)