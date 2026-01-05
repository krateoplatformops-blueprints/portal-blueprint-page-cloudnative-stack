# Add Blueprint to *Portal Blueprint Page*

This repository contains a **Blueprint** designed for use with [Krateo PlatformOps](https://krateo.io), specifically targeting the **Composable Portal** feature.

## Purpose

The Blueprint enables users to **add a Blueprint to the *Blueprint* page** of the Krateo Composable Portal, making it accessible for self-service provisioning by developers and platform consumers.

## What It Does

- Registers a new Blueprint in the Krateo *Blueprint* page
- Makes the Blueprint selectable and usable via the Composable Portal UI
- Supports namespace scoping and user permissions via RBAC (if configured)

## Usage

### Install the Helm Chart

Download Helm Chart values:

```sh
helm repo add marketplace https://marketplace.krateo.io
helm repo update marketplace
helm inspect values marketplace/portal-blueprint-page-cloudnative-stack --version 1.1.0 > ~/portal-blueprint-page-cloudnative-stack-values.yaml
```

Modify the *portal-blueprint-page-cloudnative-stack-values.yaml* file as the following example:

```yaml
blueprint:
  url: https://marketplace.krateo.io
  version: 1.0.0 # this is the Blueprint version
  hasPage: false
form:
  alphabeticalOrder: false
  hasPage: true
  instructions: "This form allows to do something"
panel:
  title: GitHub Scaffolding
  icon:
    name: fa-cubes
```

Install the Blueprint using, as a release name, the *Blueprint* name (the Helm Chart name of the blueprint):

```sh
helm install github-scaffolding template \
  --repo https://marketplace.krateo.io \
  --namespace demo-system \
  --create-namespace \
  -f ~/portal-blueprint-page-cloudnative-stack-values.yaml
  --version 1.1.0 \
  --wait
```

### Install using Krateo Composable Operation

Install the CompositionDefinition for the *Blueprint*:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: portal-blueprint-page-cloudnative-stack
  namespace: krateo-system
spec:
  chart:
    repo: portal-blueprint-page-cloudnative-stack
    url: https://marketplace.krateo.io
    version: 1.1.0
EOF
```

Install the Blueprint using, as metadata.name, the *Blueprint* name (the Helm Chart name of the blueprint):

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v1-1-0
kind: PortalBlueprintPage
metadata:
  name: github-scaffolding	
  namespace: demo-system
spec:
  blueprint:
    url: https://marketplace.krateo.io
    version: 1.0.0 # this is the Blueprint version
    hasPage: false
  form:
    alphabeticalOrder: false
    hasPage: true
    instructions: "This form allows to do something"
  panel:
    title: GitHub Scaffolding
    icon:
      name: fa-cubes
EOF
```