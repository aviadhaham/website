---
title: 'Implementing (Scalable) GitOps With Argo CD and ApplicationSets: A Case Study'
tags: [devops, argocd, gitops, kubernetes, applicationsets]
date: 2024-04-27
slug: implementing-gitops-with-argo-cd-and-applicationsets
toc: true
draft: true
---

In this blog post, I'll explore our approach to establishing a scalable and maintainable GitOps infrastructure using the Argo CD `ApplicationSets` CRD.

**I'll focus on how we:**
1. Created a powerful and declarative GitOps framework using `ApplicationSet`s that can be easily maintained and reconstructed
2. Manage many deployments across multiple environments
3. Integrate with both Kustomize and Helm, seamlessly.

## Introduction to Our GitOps Setup

Our GitOps framework is built around two main repositories:

### `argocd-management` repository:

Manages Argo CD's foundational configurations, including `AppProject`s and `ApplicationSet`s.

#### Example Directory Structure of `argocd-management` Repo

_**Later on, you will understand more, so please don't worry if you don't understand it now, and check it out later again!**_
*_**I provided real examples, from two teams, Octo and Datamine2, to give you a better understanding of how we structure our deployments.**_
   
```text
├── app-of-appprojects.yml
├── app-of-appsets.yml
├── appprojects
|   ├── [more team based directories]
│   ├── octo
│   │   ├── appproject-infra-non-prod.yml
│   │   ├── appproject-infra-prod.yml
│   │   ├── appproject-orphans-non-prod.yml
│   │   ├── appproject-orphans-prod.yml
│   │   └── kustomization.yaml
│   ├── sunflower
│   │   ├── appproject-datamine2-non-prod.yml
│   │   ├── appproject-datamine2-prod.yml
│   │   └── kustomization.yaml
│   └── kustomization.yaml
├── appsets
|   ├── [more team based directories]
│   ├── octo
│   │   ├── appset-infra-non-prod.yml
│   │   ├── appset-infra-prod.yml
│   │   ├── appset-orphans-non-prod.yml
│   │   ├── appset-orphans-prod.yml
│   │   └── kustomization.yaml
│   ├── sunflower
│   │   ├── appset-datamine2-non-prod.yml
│   │   ├── appset-datamine2-prod.yml
│   │   └── kustomization.yaml
│   └── kustomization.yaml
└── bootstrap.yml
```

### `gitops-deployments` repository:

Acts as the single source of truth for all declarative deployments within our organization.

#### Example Directory Structure of `gitops-deployments` Repo

_**Later on, you will understand more, so please don't worry if you don't understand it now, and check it out later again!**_
*_**I provided real examples, from two teams, Octo and Datamine2, to give you a better understanding of how we structure our deployments.**_

```text
deployments
│
├── [more team based directories]
├── octo
│    ├── infra
│    │   ├── devlake
│    │   │   └── envs
│    │   │       └── prod
│    │   │           ├── appset_config.json
│    │   │           ├── kustomization.yaml
│    │   │           ├── ui_deployment_patch.yml
│    │   │           └── values.yml
│    │   └── vault-auth
│    │       └── envs
│    │           └── prod
│    │               ├── appset_config.json
│    │               ├── kustomization.yaml
│    │               └── resources.yml
│    └── orphans
│        ├── aws-search
│        │   ├── base
│        │   │   ├── deployment.yml
│        │   │   ├── kustomization.yaml
│        │   │   └── service.yml
│        │   ├── envs
│        │   │   └── prod
│        │   │       ├── appset_config.json
│        │   │       └── kustomization.yaml
│        │   └── variants
│        │       └── prod
│        │           ├── deployment.yml
│        │           └── kustomization.yaml
│        └── another-app
│          ├── base
│          │   ├── deployment.yml
│          │   └── kustomization.yaml
│          └── envs
│              └── prod
│                    ├── appset_config.json
│                    ├── consul_configmap.yml
│                    ├── consul_deployment.yml
│                    └── kustomization.yaml
│
└── datamine2
     ├── datamine-amoeba-appprd
     │   ├── base
     │   │   ├── deployment.yml
     │   │   ├── kustomization.yaml
     │   ├── envs
     │   │   ├── non-prod-sandbox
     │   │   │   ├── appset_config.json
     │   │   │   ├── kustomization.yaml
     │   │   ├── non-prod-staging
     │   │   │   ├── kustomization.yaml
     │   │   └── prod
     │   │       ├── appset_config.json
     │   │       ├── kustomization.yaml
     │   └── variants
     │       ├── prod
     │       │   ├── env.yml
     │       │   ├── kustomization.yaml
     │       ├── sandbox
     │       │   ├── env.yml
     │       │   ├── kustomization.yaml
     │       └── staging
     │           ├── env.yml
     │           ├── kustomization.yaml
     └── datamine-amoeba-catalog
         ├── base
         │   ├── deployment.yml
         │   ├── kustomization.yaml
         ├── envs
         │   ├── non-prod-sandbox
         │   │   ├── appset_config.json
         │   │   └── kustomization.yaml
         │   ├── non-prod-staging
         │   │   └── kustomization.yaml
         │   └── prod
         │       ├── appset_config.json
         │       └── kustomization.yaml
         └── variants
             ├── prod
             │   ├── env.yml
             │   ├── kustomization.yaml
             ├── sandbox
             │   ├── env.yml
             │   ├── kustomization.yaml
             └── staging
                 ├── env.yml
                 ├── kustomization.yaml
```

## How We Bootstrap and Manage our Argo CD - The Root Factories

We start our setup with a bootstrap operation defined in the `bootstrap.yml` file, which contains all necessary configurations for initializing Argo CD.

*_At the time of writing this post, it only contains an `AppProject` manifest (named `management`), that we use for the two root YAMLs I will mention below._

In order to reconstruct our setup, **the only manual steps we need to perform** are the following:

1. **Apply the bootstrap configuration:**
   ```bash
   kubectl apply -f bootstrap.yml
   ```
2. **Set up management for AppProjects and ApplicationSets:**
   ```bash
   kubectl apply -f app-of-appprojects.yml
   kubectl apply -f app-of-appsets.yml
   ```

To make changes (add, delete, or edit), simply update the `appprojects` and `appsets` directories using the `kustomize` structure. Each change will be automatically detected by either the `app-of-appprojects` or `app-of-appsets` respectively.

**We utlize the power of the `Application` CRD to manage the `AppProjects`, and the `Application` factories, the `ApplicationSets`!**

### Diagram: How It's Working Together
![Diagram illustrating the whole operation](/posts-images/appsets-factories-automation-diagram.png)


## A Word About our Approach to ApplicationSets

* **Generators**: Using `generators`, the `ApplicationSet` constructs a dynamic list of Argo CD applications to be managed. These generators use patterns and parameters defined in configuration files (like `appset_config*.json` that you'll see below) to determine which applications should exist (or be deleted).
* **Scalability**: This approach enables the `ApplicationSet` to scale efficiently, as it can automatically adjust the number of managed applications based on the repository’s current state. New applications are created, and outdated ones are removed without manual intervention.

* **Additional Notes**:
  * In our case, we set the `path` of the `git` generator to a full path, and use the `**` globbing pattern to crawl all subdirectories and files within the specified directory until it finds the `appset_config*.json` files.
  * Any directory that won't have the `appset_config*.json` file or that won't match its appropriate `ApplicationSet`, will be ignored by the `ApplicationSet`.
  * It's also important to make sure that no other `ApplicationSet` is targeting the same directory, as it may lead to conflicts and unexpected behavior.

## Our ApplicationSets Strategies

### GitLab Subgroup-based `ApplicationSet`s
`ApplicationSet`s are primarily organized per GitLab subgroup, automatically generating Argo CD `Application`s for each service or application within the subgroup.

**Orphan `ApplicationSet`s** (**for applications/repos that do not belong to any subgroup**)
* As we have many applications that do not belong to any subgroup,
we handle them as “orphans” (both in terms of `ApplicationSet`s and in the `gitops-deployments` repo).

**Prod and Non-Prod Environments**
* The production environments are handled by `ApplicationSet`s that target directories with a `prod` prefix in their paths, and non-production environments with a `non-prod` prefix.

* The specific `ApplicationSet` “configuration” JSON files (`appset_config*.json`) within these directories provide configuration/parameters for the `ApplicationSet` templates.


**Real example `ApplicationSet` config that we use for our own DevOps team ("Octo")**:

* Infra:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: octo-infra-prod
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: git@git.company.com:octo/gitops-deployments.git
        revision: main
        files:
          - path: deployments/octo/infra/**/envs/prod*/appset_config*.json
  template:
    metadata:
      name: "octo-{{path[3]}}-{{cluster.name}}-{{env}}"
      finalizers:
        - resources-finalizer.argocd.argoproj.io
      namespace: argocd
    spec:
      project: octo-infra-prod
      source:
        repoURL: git@git.company.com:octo/gitops-deployments.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: "{{cluster.address}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
```

As you can see, the `ApplicationSet` is configured to watch the `infra` directory within the `octo` subgroup in the `gitops-deployments` repo, targeting the `prod` environment. The `appset_config*.json` files within the `envs/prod` directory provide the necessary configuration for the `ApplicationSet`.

* Orphans:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: octo-orphans-prod
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: git@git.company.com:octo/gitops-deployments.git
        revision: main
        files:
          - path: deployments/octo/orphans/**/envs/prod*/appset_config*.json
  template:
    metadata:
      name: "octo-{{path[3]}}-{{cluster.name}}-{{env}}"
      finalizers:
        - resources-finalizer.argocd.argoproj.io
      namespace: argocd
    spec:
      project: octo-orphans-prod
      source:
        repoURL: git@git.company.com:octo/gitops-deployments.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: "{{cluster.address}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
```

This `ApplicationSet` targets the `orphans` directory within the `octo` subgroup in the `gitops-deployments` repo, focusing on the `prod` environment. The `appset_config*.json` files within the `envs/prod` directory provide the necessary configuration for the `ApplicationSet`.


**Example `appset_config.json` file**:

* _taken from `deployments/sunflower/datamine2/datamine-amoeba-appprd/envs/prod`, from the `gitops-deployments` repo_
* _Of course, you can structure your `ApplicationSet` and your JSON file in any way that suits your organization's needs._

```json
{
  "env": "prod",
  "cluster": {
    "name": "dc1-november",
    "address": "https://10.1.1.1:6443",
    "dc": "dc1"
  }
}
```

# ########
# LAST STOPPED: HERE
# ########

To overcome that difference between the GitLab repositories heirarcy and the `gitops-deployments`, we implemented in the CI, a logic that checks if the repository is in the “root” / group level, and then adds to its path in `gitops-deployments`, the `orphans/` directory, so it will hit the right path in the gitops repo.

  Rationale

  1. Having Orphans Directory: We didn’t want to create an `ApplicationSet` that will watch the group’s root hierarchy, as it doesn’t behave well, we tried it. We prefered to separate the “orphans” to a different directory, and just handle the logic of path in the CI with simple bash code.
  2. Prod and Non-Prod Separation: Having prefixes for `prod*` and `non-prod*` allows us to manage each “type” of deployment in a different `ApplicationSet`, and also enforce the user to add these prefixes as needed, only when he wants them to be picked up / watched by the `ApplicationSet` and hence deployed.


### Example: CI/CD Integration

Our CI/CD pipelines update kustomization endpoints in the `gitops-deployments` repository, triggering deployments through Argo CD when changes are detected. This ensures a seamless flow where new applications are added and outdated ones are removed without manual intervention.

## Multi-Environment Management

Our strategy for managing multiple environments includes:

- **Distinct Paths:** Separate paths for production (`prod`) and non-production environments (`non-prod-staging`, `non-prod-sandbox`) allow us to create distinct sets of applications for each environment.
- **Efficient Deployment Management:** This structure ensures that deployments are consistently managed across all environments.

## Integrating Kustomize and Helm

Our deployment approach integrates Kustomize and Helm, allowing us to manage Helm-based and native Kubernetes deployments seamlessly. Key integration points include:

- **Enabling Helm in Kustomize:** We modify the `argocd-cm` ConfigMap to include the `--enable-helm` flag in the Kustomize build command.
- **Example Configuration:**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  helmCharts:
    - name: devlake
      repo: https://apache.github.io/incubator-devlake-helm-chart
      releaseName: devlake
      namespace: devlake
      version: 1.0.0-beta3
      valuesFile: values.yml
  ```

## Final Words

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed ac purus sit amet nisl tincidunt tincidunt

---

Scale Strategies with ApplicationSets

  Our architecture leverages ApplicationSets to facilitate scalability in the following ways:

  * Adding New Applications: Adding a new directory in the gitops repo, to a path crawled by a specific AppSet will make the AppSet add another crawl (thanks to the globbing ** in the path), and so, if the rest of the path is matching the regex/path in the rest of the path, it will your new application, including all its environments.
  * Environment Expansion: Adding a new JSON configuration file (matching the regexappset_config*.json) in the relevant envs directory automatically triggers the creation of a new Argo Application tailored to the specific parameters defined within the file.
    * This makes scaling out to additional clusters or deployment environments seamless.
  * Multi-environment Management: By defining separate paths for different environments (like prod, non-prod-staging, non-prod-sandbox), ApplicationSets can manage deployments across multiple environments efficiently, creating distinct sets of applications for each environment.

Example of Environments (envs) Structure

  The envs directory structure plays a crucial role in how ApplicationSets manage deployments:

```text
envs/
├── non-prod-sandbox/
│   ├── appset_config-a.json
│   ├── appset_config-b.json
│   ├── kustomization.yml
├── non-prod-staging/
│   ├── appset_config-c.json
│   ├── kustomization.yml
├── prod/
    ├── appset_config-d.json
    ├── kustomization.yml
```

Non-Prod AppSets:

  As mentioned, these watch the non-prod* prefixed directories, creating applications based on the appset_config*.json files found within. Each config file leads to the creation of a new application targeting the specific non-production environment.

Prod AppSets:

  Similarly, the production ApplicationSet targets the directories that are prefixed with prod*, generating applications as defined by the configuration files present.

AppProjects Strategy

Overview of AppProjects

AppProjects in Argo CD serve as a way to organize and provide boundaries for Argo CD Applications. They enable teams to define specific permissions, resource quotas, and other governance controls for groups of applications.

Simplified Approach to AppProjects

We are creating a prod and non-prod AppProject for each subgroup or category within a larger group. Here’s how this approach works in practice:

* As the management of the AppSets and AppProjects is subgroup-based, then each main group, such as “DMP”, “Sunflower”, etc., will have a prod and a  non-prod AppSets and correlated AppProjects for each of their subgroups (either defined ones, like “datamine2”, or to ones that are in the root hierarchy of the group, which are called “orphans”, as mentioned here).

Rationale

This simplified approach to AppProject management is based on the following considerations:

* Granularity: ensuring that individuals of groups of them only have access to the Argo Projects that they are part of.
* Clarity: 1-to-1 relationship between each ApplicationSet and AppProject. Each AppSet (either prod/non-prod) will need to have a correlated AppProject.



Helm Deployments

Helm deployments work in a similar way to normal applications, where the entrypoint is the environment’s kustomization.yml file. This file should include a helmCharts section like so:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: devlake
    repo: https://apache.github.io/incubator-devlake-helm-chart
    releaseName: devlake
    namespace: devlake
    version: 1.0.0-beta3
    valuesFile: values.yml

This enables flexibility by allowing Helm-based deployments to be patched and to create native K8s resources along the Helm releases.

How it works

Kustomize allows deploying Helm releases by adding the —enable-helm flag in the kustomize build command. This was done by patching the argocd-cm ConfigMap and the AVP plugin resources (adding this flag to the kustomize build command)
