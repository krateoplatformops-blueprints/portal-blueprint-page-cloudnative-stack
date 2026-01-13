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
helm inspect values marketplace/portal-blueprint-page-cloudnative-stack --version 0.0.3 > ~/portal-blueprint-page-cloudnative-stack-values.yaml
```

Modify the *portal-blueprint-page-cloudnative-stack-values.yaml* file as the following example:

```yaml
blueprint:
  repo: cloudnative-stack
  url: https://marketplace.krateo.io
  version: 0.4.2
  hasPage: true
  credentials: {}
form:
  widgetData:
    schema: {}
    objectFields:
      - path: frontend.deployments
        displayField: name
      - path: backend.deployments
        displayField: name
      - path: kafka.topics
        displayField: topicName
      - path: hazelcast.clusters
        displayField: name
      - path: mongodb.instances
        displayField: clusterName
      - path: cloudnativepg.instances
        displayField: databaseName
    submitActionId: submit-action-from-string-schema
    actions:
      rest:
        - id: submit-action-from-string-schema
          onEventNavigateTo:
            eventReason: "CompositionCreated"
            url: '${ "/compositions/" + .response.metadata.namespace + "/" + (.response.kind | ascii_downcase) + "/" + .response.metadata.name }'
            urlIfNoPage: "/blueprints"
            timeout: 50
          successMessage: '${"Successfully deployed " + .response.metadata.name + " composition in namespace " + .response.metadata.namespace }'
          resourceRefId: composition-to-post
          type: rest
          payloadToOverride:
            - name: spec
              value: ${ .json | del(.composition) }
            - name: metadata.name
              value: ${ .json.composition.name }
            - name: metadata.namespace
              value: ${ .json.composition.namespace }
          headers:
            - "Content-Type: application/json"
  widgetDataTemplate:
    - forPath: stringSchema
      expression: >
        ${
          .getNotOrderedSchema["values.schema.json"] as $schema
          | .allowedNamespaces[] as $allowedNamespaces
          | .featuresFrontend as $featuresFrontend
          | .featuresBackend as $featuresBackend
          | "\"properties\": " | length as $keylen
          | ($schema | index("\"properties\": ")) as $idx
          | ($schema[0:$idx + $keylen]) as $prefix
          | ($schema[$idx + $keylen:]) as $rest
          | {
              composition: {
                type: "object",
                properties: {
                  name: {
                    type: "string"
                  },
                  namespace: {
                    type: "string",
                    enum: $allowedNamespaces
                  }
                },
                required: ["name", "namespace"]
              }
            } | tostring as $injected
          | ($prefix + $injected[:-1] + "," + $rest[1:]) as $withComposition
          | (
              if ($withComposition | test("\n  \"required\": \\[")) then
                if ($withComposition | test("\n  \"required\": \\[[^\\]]*\"composition\"")) then
                  $withComposition
                else
                  ($withComposition
                  | gsub("\n  \"required\": \\["; "\n  \"required\": [\"composition\", "))
                end
              else
                ($withComposition | index("\n  \"type\": \"object\"")) as $tidx
                | ($withComposition[0:$tidx+1]) as $p2
                | ($withComposition[$tidx+1:]) as $r2
                | $p2 + "  \"required\": [\"composition\"],\n" + $r2
              end
            ) as $schemaWithRequired
          | $featuresFrontend as $ff
          | ($ff | map("\"" + . + "\"") | join(", ")) as $feEnumItems
          | ("[" + $feEnumItems + "]") as $frontendEnum
          | $featuresBackend as $fb
          | ($fb | map("\"" + . + "\"") | join(", ")) as $beEnumItems
          | ("[" + $beEnumItems + "]") as $backendEnum
          | (
              if ($schemaWithRequired
                    | test("(?s)\"frontend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"")) then
                ($schemaWithRequired
                | gsub(
                    "(?s)(?<prefix>\"frontend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"\\s*:\\s*)\\[[^]]*\\]"
                    ;
                    "\(.prefix)\($frontendEnum)"
                  ))
              else
                ($schemaWithRequired
                | gsub(
                    "(?s)(?<prefix>\"frontend\"[\\s\\S]*?\"features\"[\\s\\S]*?\"items\"[\\s\\S]*?\"type\": \"string\")"
                    ;
                    "\(.prefix),\n                \"enum\": \($frontendEnum)"
                  ))
              end
            ) as $schemaAfterFrontend
          | (
              if ($schemaAfterFrontend
                    | test("(?s)\"backend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"")) then
                ($schemaAfterFrontend
                | gsub(
                    "(?s)(?<prefix>\"backend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"\\s*:\\s*)\\[[^]]*\\]"
                    ;
                    "\(.prefix)\($backendEnum)"
                  ))
              else
                ($schemaAfterFrontend
                | gsub(
                    "(?s)(?<prefix>\"backend\"[\\s\\S]*?\"features\"[\\s\\S]*?\"items\"[\\s\\S]*?\"type\": \"string\")"
                    ;
                    "\(.prefix),\n                \"enum\": \($backendEnum)"
                  ))
              end
            )
        }

  resourcesRefsTemplate:
    iterator: ${ .allowedNamespacesWithResource[] }
    template:
      id: composition-to-post
      apiVersion: composition.krateo.io/v0-4-2
      namespace: ${ .namespace }
      resource: ${ .resource }
      verb: POST

  apiRef:
    name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-schema-override-allows-ns'
    namespace: ""

  hasPage: false

  instructions: |
    # cloudnative-stack

    A Cloud Native Stack based on Frontend / Backend / Kafka / Hazelcast / MongoDB / CloudNativePG

    ## Overview
    
    This Krateo blueprint deploys a complex, multi-tier application stack on Kubernetes. It is designed to be highly configurable and relies on best-practice Kubernetes operators for managing stateful components like databases and message queues.
    The blueprint will provision the following components:

    -   A **Frontend** pod (e.g., Nginx) to serve a user interface.
    -   A **Backend** pod (e.g., Spring Boot) for business logic.
    -   A **Kafka Topic** managed by the Strimzi operator.
    -   A **Hazelcast** in-memory data grid cluster managed by the Hazelcast Platform Operator.
    -   A **MongoDB** replica set managed by the MongoDB Community Kubernetes Operator.
    -   A **PostgreSQL** cluster managed by the CloudNative-PG operator.

    ## Requirements

    For this blueprint to function correctly, your Kubernetes cluster **must** have the following operators installed and running. Please follow their official documentation for installation instructions.

    1. **Strimzi (for Kafka):**
        -   **Purpose:** Manages Kafka clusters and topics.
        -   **Installation Guide:** [Strimzi Deployment Documentation](https://strimzi.io/docs/operators/latest/deploying)

    2.  **Hazelcast Platform Operator:**
        -   **Purpose:** Manages Hazelcast in-memory computing clusters.
        -   **Installation Guide:** [Hazelcast Operator Deployment](https://docs.hazelcast.com/operator/5.14/)

    3.  **MongoDB Kubernetes Operator:**
        -   **Purpose:** Manages MongoDB Community replica sets and clusters.
        -   **Installation Guide:** [https://github.com/mongodb/mongodb-kubernetes](https://github.com/mongodb/mongodb-kubernetes)

    4.  **CloudNative-PG (for PostgreSQL):**
        -   **Purpose:** Manages the full lifecycle of a PostgreSQL cluster.
        -   **Installation Guide:** [CloudNative-PG Installation](https://cloudnative-pg.io/documentation/1.27/)
panel:
  markdown: Click here to deploy a **{{ .Values.global.compositionName }}** composition
  apiRef:
    name: "{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-compositiondefinition-schema-ns"
    namespace: ""
  widgetData:
    clickActionId: blueprint-click-action
    footer:
      - resourceRefId: blueprint-panel-button
    tags:
      - '{{ .Release.Namespace }}'
      - '{{ .Values.blueprint.version }}'
    icon:
      name: fa-cloud
    items:
      - resourceRefId: blueprint-panel-markdown
    title: Cloud Native Stack
    actions:
      openDrawer:
        - id: blueprint-click-action
          resourceRefId: blueprint-form
          type: openDrawer
          size: large
      navigate:
        - id: blueprint-click-action
          path: '/{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-blueprint'
          type: navigate
  widgetDataTemplate:
    - forPath: icon.color
      expression: >
        ${
          (
            (
              .getCompositionDefinition?.status?.conditions // []
            )
            | map(
                select(.type == "Ready")
                | if .status == "False" then
                    "orange"
                  elif .status == "True" then
                    "green"
                  else
                    "grey"
                  end
              )
            | first
          )
          // "grey"
        }
    - forPath: headerLeft
      expression: >
        ${
          (
            (
              .getCompositionDefinition?.status?.conditions // []
            )
            | map(
                select(.type == "Ready")
                | if .status == "False" then
                    "Ready:False"
                  elif .status == "True" then
                    "Ready:True"
                  else
                    "Ready:Unknown"
                  end
              )
            | first
          )
          // "Ready:Unknown"
        }
    - forPath: headerRight
      expression: >
        ${
          .getCompositionDefinition // {}
          | .metadata.creationTimestamp
          // "In the process of being created"
        }
  resourcesRefs:
    items:
      - id: blueprint-panel-markdown
        apiVersion: widgets.templates.krateo.io/v1beta1
        name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-panel-markdown'
        namespace: ''
        resource: markdowns
        verb: GET
      - id: blueprint-panel-button
        apiVersion: widgets.templates.krateo.io/v1beta1
        name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-panel-button-cleanup'
        namespace: ''
        resource: buttons
        verb: DELETE
      - id: blueprint-form
        apiVersion: widgets.templates.krateo.io/v1beta1
        name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-form'
        namespace: ''
        resource: forms
        verb: GET
restActions:
  - name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-compositiondefinition-schema-ns'
    namespace: ""

    api:
      - name: getCompositionDefinition
        path: "/apis/core.krateo.io/v1alpha1/namespaces/{{ .Release.Namespace }}/compositiondefinitions/{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}"
        verb: GET
        headers:
          - "Accept: application/json"

      - name: getCompositionDefinitionResource
        path: "/apis/core.krateo.io/v1alpha1/namespaces/{{ .Release.Namespace }}/compositiondefinitions/{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}"
        verb: GET
        headers:
          - "Accept: application/json"
        filter: ".getCompositionDefinitionResource.status.resource"

      - name: getOrderedSchema
        path: >
          ${ "/apis/apiextensions.k8s.io/v1/customresourcedefinitions/" + (.getCompositionDefinitionResource) + ".composition.krateo.io" }
        verb: GET
        headers:
          - "Accept: application/json"
        dependsOn:
          name: getCompositionDefinitionResource
        filter: >
          .getOrderedSchema.spec.versions[]
          | select(.name == "v{{ .Values.blueprint.version | replace "." "-" }}")
          | .schema.openAPIV3Schema.properties.spec

      - name: getNotOrderedSchema
        path: >
          ${ "/api/v1/namespaces/{{ .Release.Namespace }}/configmaps/" + (.getCompositionDefinitionResource) + "-v{{ .Values.blueprint.version | replace "." "-" }}-jsonschema-configmap" }
        verb: GET
        headers:
          - "Accept: application/json"
        dependsOn:
          name: getCompositionDefinitionResource
        filter: ".getNotOrderedSchema.data"

      - name: getNamespaces
        path: "/api/v1/namespaces"
        verb: GET
        headers:
          - "Accept: application/json"
        filter: "[.getNamespaces.items[] | .metadata.name]"

    filter: >
      {
        getOrderedSchema: .getOrderedSchema,
        getNotOrderedSchema: .getNotOrderedSchema,
        getCompositionDefinition: .getCompositionDefinition,
        getCompositionDefinitionResource: .getCompositionDefinitionResource,
        possibleNamespacesForComposition:
        [
          .getNamespaces[] as $ns |
          {
            resource: .getCompositionDefinitionResource,
            namespace: $ns
          }
        ]
      }

  - name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-schema-override-allows-ns'
    namespace: ""

    api:
      - name: getCompositionDefinitionSchemaNs
        path: "/call?apiVersion=templates.krateo.io/v1&resource=restactions&name={{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-compositiondefinition-schema-ns&namespace={{ .Release.Namespace }}"
        verb: GET
        endpointRef:
          name: snowplow-endpoint
          namespace: '{{ default "krateo-system" (default dict .Values.global).krateoNamespace }}'
        headers:
          - "Accept: application/json"
        continueOnError: true
        errorKey: getCompositionDefinitionSchemaNs
        exportJwt: true

      - name: allowedNamespaces
        continueOnError: true
        dependsOn:
          name: getCompositionDefinitionSchemaNs
          iterator: .getCompositionDefinitionSchemaNs.status.possibleNamespacesForComposition
        path: "/apis/authorization.k8s.io/v1/selfsubjectaccessreviews"
        verb: POST
        headers:
          - "Content-Type: application/json"
        payload: |
          ${
            {
              "kind": "SelfSubjectAccessReview",
              "apiVersion": "authorization.k8s.io/v1",
              "spec": {
                "resourceAttributes": {
                  "namespace": .namespace,
                  "verb": "create",
                  "group": "composition.krateo.io",
                  "version": "v{{ .Values.blueprint.version | replace "." "-" }}",
                  "resource": .resource
                }
              }
            }
          }
      - name: featuresFrontend
        verb: GET
        headers:
        - 'Accept: application/vnd.github+json'
        - 'X-GitHub-Api-Version: 2022-11-28'
        path: "/search/repositories?q=org:krateoplatformops-test+topic:feature+topic:frontend"
        endpointRef:
          name: github-api
          namespace: '{{ default "krateo-system" (default dict .Values.global).krateoNamespace }}'
        filter: |
          .featuresFrontend.items | map((.name | sub("-feature$"; "")))

      - name: featuresBackend
        verb: GET
        headers:
        - 'Accept: application/vnd.github+json'
        - 'X-GitHub-Api-Version: 2022-11-28'
        path: "/search/repositories?q=org:krateoplatformops-test+topic:feature+topic:backend"
        endpointRef:
          name: github-api
          namespace: '{{ default "krateo-system" (default dict .Values.global).krateoNamespace }}'
        filter: |
          .featuresBackend.items | map((.name | sub("-feature$"; "")))

    filter: >
      {
        getOrderedSchema: .getCompositionDefinitionSchemaNs.status.getOrderedSchema,
        getNotOrderedSchema: .getCompositionDefinitionSchemaNs.status.getNotOrderedSchema,
        getCompositionDefinitionResource: .getCompositionDefinitionSchemaNs.status.getCompositionDefinitionResource,
        featuresFrontend: .featuresFrontend,
        featuresBackend: .featuresBackend,        
        allowedNamespaces: [
          [.allowedNamespaces[] | .status.allowed] as $allowed
          | [.getCompositionDefinitionSchemaNs.status.possibleNamespacesForComposition[] | .namespace]
          | to_entries
          | map(select($allowed[.key] == true) | .value)
        ],
        allowedNamespacesWithResource: [
          [.allowedNamespaces[] | .status.allowed] as $allowed
          | .getCompositionDefinitionSchemaNs.status.getCompositionDefinitionResource as $resource
          | [.getCompositionDefinitionSchemaNs.status.possibleNamespacesForComposition[] | .namespace]
          | to_entries
          | map(
              select($allowed[.key] == true)
              | {
                  namespace: .value,
                  resource: $resource
                }
            )
        ]
      }
```

Install the Blueprint using, as a release name, the *Blueprint* name (the Helm Chart name of the blueprint):

```sh
helm install portal-blueprint-page-cloudnative-stack portal-blueprint-page-cloudnative-stack \
  --repo https://marketplace.krateo.io \
  --namespace demo-system \
  --create-namespace \
  -f ~/portal-blueprint-page-cloudnative-stack-values.yaml \
  --version 0.0.3 \
  --wait
```

### Install using Krateo Composable Operation

Install the CompositionDefinition for the *Blueprint*:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: portal-blueprint-page
  namespace: krateo-system
spec:
  chart:
    repo: portal-blueprint-page-cloudnative-stack
    url: https://marketplace.krateo.io
    version: 0.0.3
EOF
```

Install the following endpoint:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: "v1"       
kind: Secret
metadata:                                                    
 name: github-api
 namespace: krateo-system
stringData:
 server-url: https://api.github.com
EOF
```

Install the Blueprint using, as metadata.name, the *Blueprint* name (the Helm Chart name of the blueprint):

```sh
cat <<'EOF' | kubectl apply -f -
apiVersion: composition.krateo.io/v0-0-3  
kind: PortalBlueprintPageCloudnativeStack
metadata:
  name: cloudnative-stack
  namespace: demo-system
spec:
  blueprint:
    repo: cloudnative-stack
    url: https://marketplace.krateo.io
    version: 0.4.2
    hasPage: true
    credentials: {}
  form:
    widgetData:
      schema: {}
      objectFields:
        - path: frontend.deployments
          displayField: name
        - path: backend.deployments
          displayField: name
        - path: kafka.topics
          displayField: topicName
        - path: hazelcast.clusters
          displayField: name
        - path: mongodb.instances
          displayField: clusterName
        - path: cloudnativepg.instances
          displayField: databaseName
      submitActionId: submit-action-from-string-schema
      actions:
        rest:
          - id: submit-action-from-string-schema
            onEventNavigateTo:
              eventReason: "CompositionCreated"
              url: '${ "/compositions/" + .response.metadata.namespace + "/" + (.response.kind | ascii_downcase) + "/" + .response.metadata.name }'
              urlIfNoPage: "/blueprints"
              timeout: 50
            successMessage: '${"Successfully deployed " + .response.metadata.name + " composition in namespace " + .response.metadata.namespace }'
            resourceRefId: composition-to-post
            type: rest
            payloadToOverride:
              - name: spec
                value: ${ .json | del(.composition) }
              - name: metadata.name
                value: ${ .json.composition.name }
              - name: metadata.namespace
                value: ${ .json.composition.namespace }
            headers:
              - "Content-Type: application/json"
    widgetDataTemplate:
      - forPath: stringSchema
        expression: >
          ${
            .getNotOrderedSchema["values.schema.json"] as $schema
            | .allowedNamespaces[] as $allowedNamespaces
            | .featuresFrontend as $featuresFrontend
            | .featuresBackend as $featuresBackend
            | "\"properties\": " | length as $keylen
            | ($schema | index("\"properties\": ")) as $idx
            | ($schema[0:$idx + $keylen]) as $prefix
            | ($schema[$idx + $keylen:]) as $rest
            | {
                composition: {
                  type: "object",
                  properties: {
                    name: {
                      type: "string"
                    },
                    namespace: {
                      type: "string",
                      enum: $allowedNamespaces
                    }
                  },
                  required: ["name", "namespace"]
                }
              } | tostring as $injected
            | ($prefix + $injected[:-1] + "," + $rest[1:]) as $withComposition
            | (
                if ($withComposition | test("\n  \"required\": \\[")) then
                  if ($withComposition | test("\n  \"required\": \\[[^\\]]*\"composition\"")) then
                    $withComposition
                  else
                    ($withComposition
                    | gsub("\n  \"required\": \\["; "\n  \"required\": [\"composition\", "))
                  end
                else
                  ($withComposition | index("\n  \"type\": \"object\"")) as $tidx
                  | ($withComposition[0:$tidx+1]) as $p2
                  | ($withComposition[$tidx+1:]) as $r2
                  | $p2 + "  \"required\": [\"composition\"],\n" + $r2
                end
              ) as $schemaWithRequired
            | $featuresFrontend as $ff
            | ($ff | map("\"" + . + "\"") | join(", ")) as $feEnumItems
            | ("[" + $feEnumItems + "]") as $frontendEnum
            | $featuresBackend as $fb
            | ($fb | map("\"" + . + "\"") | join(", ")) as $beEnumItems
            | ("[" + $beEnumItems + "]") as $backendEnum
            | (
                if ($schemaWithRequired
                      | test("(?s)\"frontend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"")) then
                  ($schemaWithRequired
                  | gsub(
                      "(?s)(?<prefix>\"frontend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"\\s*:\\s*)\\[[^]]*\\]"
                      ;
                      "\(.prefix)\($frontendEnum)"
                    ))
                else
                  ($schemaWithRequired
                  | gsub(
                      "(?s)(?<prefix>\"frontend\"[\\s\\S]*?\"features\"[\\s\\S]*?\"items\"[\\s\\S]*?\"type\": \"string\")"
                      ;
                      "\(.prefix),\n                \"enum\": \($frontendEnum)"
                    ))
                end
              ) as $schemaAfterFrontend
            | (
                if ($schemaAfterFrontend
                      | test("(?s)\"backend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"")) then
                  ($schemaAfterFrontend
                  | gsub(
                      "(?s)(?<prefix>\"backend\"[\\s\\S]*\"features\"[\\s\\S]*\"enum\"\\s*:\\s*)\\[[^]]*\\]"
                      ;
                      "\(.prefix)\($backendEnum)"
                    ))
                else
                  ($schemaAfterFrontend
                  | gsub(
                      "(?s)(?<prefix>\"backend\"[\\s\\S]*?\"features\"[\\s\\S]*?\"items\"[\\s\\S]*?\"type\": \"string\")"
                      ;
                      "\(.prefix),\n                \"enum\": \($backendEnum)"
                    ))
                end
              )
          }

    resourcesRefsTemplate:
      iterator: ${ .allowedNamespacesWithResource[] }
      template:
        id: composition-to-post
        apiVersion: composition.krateo.io/v0-4-2
        namespace: ${ .namespace }
        resource: ${ .resource }
        verb: POST

    apiRef:
      name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-schema-override-allows-ns'
      namespace: ""

    hasPage: false

    instructions: |
      # cloudnative-stack

      A Cloud Native Stack based on Frontend / Backend / Kafka / Hazelcast / MongoDB / CloudNativePG

      ## Overview
      
      This Krateo blueprint deploys a complex, multi-tier application stack on Kubernetes. It is designed to be highly configurable and relies on best-practice Kubernetes operators for managing stateful components like databases and message queues.
      The blueprint will provision the following components:

      -   A **Frontend** pod (e.g., Nginx) to serve a user interface.
      -   A **Backend** pod (e.g., Spring Boot) for business logic.
      -   A **Kafka Topic** managed by the Strimzi operator.
      -   A **Hazelcast** in-memory data grid cluster managed by the Hazelcast Platform Operator.
      -   A **MongoDB** replica set managed by the MongoDB Community Kubernetes Operator.
      -   A **PostgreSQL** cluster managed by the CloudNative-PG operator.

      ## Requirements

      For this blueprint to function correctly, your Kubernetes cluster **must** have the following operators installed and running. Please follow their official documentation for installation instructions.

      1. **Strimzi (for Kafka):**
          -   **Purpose:** Manages Kafka clusters and topics.
          -   **Installation Guide:** [Strimzi Deployment Documentation](https://strimzi.io/docs/operators/latest/deploying)

      2.  **Hazelcast Platform Operator:**
          -   **Purpose:** Manages Hazelcast in-memory computing clusters.
          -   **Installation Guide:** [Hazelcast Operator Deployment](https://docs.hazelcast.com/operator/5.14/)

      3.  **MongoDB Kubernetes Operator:**
          -   **Purpose:** Manages MongoDB Community replica sets and clusters.
          -   **Installation Guide:** [https://github.com/mongodb/mongodb-kubernetes](https://github.com/mongodb/mongodb-kubernetes)

      4.  **CloudNative-PG (for PostgreSQL):**
          -   **Purpose:** Manages the full lifecycle of a PostgreSQL cluster.
          -   **Installation Guide:** [CloudNative-PG Installation](https://cloudnative-pg.io/documentation/1.27/)
  panel:
    markdown: Click here to deploy a **{{ .Values.global.compositionName }}** composition
    apiRef:
      name: "{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-compositiondefinition-schema-ns"
      namespace: ""
    widgetData:
      clickActionId: blueprint-click-action
      footer:
        - resourceRefId: blueprint-panel-button
      tags:
        - '{{ .Release.Namespace }}'
        - '{{ .Values.blueprint.version }}'
      icon:
        name: fa-cloud
      items:
        - resourceRefId: blueprint-panel-markdown
      title: Cloud Native Stack
      actions:
        openDrawer:
          - id: blueprint-click-action
            resourceRefId: blueprint-form
            type: openDrawer
            size: large
        navigate:
          - id: blueprint-click-action
            path: '/{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-blueprint'
            type: navigate
    widgetDataTemplate:
      - forPath: icon.color
        expression: >
          ${
            (
              (
                .getCompositionDefinition?.status?.conditions // []
              )
              | map(
                  select(.type == "Ready")
                  | if .status == "False" then
                      "orange"
                    elif .status == "True" then
                      "green"
                    else
                      "grey"
                    end
                )
              | first
            )
            // "grey"
          }
      - forPath: headerLeft
        expression: >
          ${
            (
              (
                .getCompositionDefinition?.status?.conditions // []
              )
              | map(
                  select(.type == "Ready")
                  | if .status == "False" then
                      "Ready:False"
                    elif .status == "True" then
                      "Ready:True"
                    else
                      "Ready:Unknown"
                    end
                )
              | first
            )
            // "Ready:Unknown"
          }
      - forPath: headerRight
        expression: >
          ${
            .getCompositionDefinition // {}
            | .metadata.creationTimestamp
            // "In the process of being created"
          }
    resourcesRefs:
      items:
        - id: blueprint-panel-markdown
          apiVersion: widgets.templates.krateo.io/v1beta1
          name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-panel-markdown'
          namespace: ''
          resource: markdowns
          verb: GET
        - id: blueprint-panel-button
          apiVersion: widgets.templates.krateo.io/v1beta1
          name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-panel-button-cleanup'
          namespace: ''
          resource: buttons
          verb: DELETE
        - id: blueprint-form
          apiVersion: widgets.templates.krateo.io/v1beta1
          name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-form'
          namespace: ''
          resource: forms
          verb: GET
  restActions:
    - name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-compositiondefinition-schema-ns'
      namespace: ""

      api:
        - name: getCompositionDefinition
          path: "/apis/core.krateo.io/v1alpha1/namespaces/{{ .Release.Namespace }}/compositiondefinitions/{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}"
          verb: GET
          headers:
            - "Accept: application/json"

        - name: getCompositionDefinitionResource
          path: "/apis/core.krateo.io/v1alpha1/namespaces/{{ .Release.Namespace }}/compositiondefinitions/{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}"
          verb: GET
          headers:
            - "Accept: application/json"
          filter: ".getCompositionDefinitionResource.status.resource"

        - name: getOrderedSchema
          path: >
            ${ "/apis/apiextensions.k8s.io/v1/customresourcedefinitions/" + (.getCompositionDefinitionResource) + ".composition.krateo.io" }
          verb: GET
          headers:
            - "Accept: application/json"
          dependsOn:
            name: getCompositionDefinitionResource
          filter: >
            .getOrderedSchema.spec.versions[]
            | select(.name == "v{{ .Values.blueprint.version | replace "." "-" }}")
            | .schema.openAPIV3Schema.properties.spec

        - name: getNotOrderedSchema
          path: >
            ${ "/api/v1/namespaces/{{ .Release.Namespace }}/configmaps/" + (.getCompositionDefinitionResource) + "-v{{ .Values.blueprint.version | replace "." "-" }}-jsonschema-configmap" }
          verb: GET
          headers:
            - "Accept: application/json"
          dependsOn:
            name: getCompositionDefinitionResource
          filter: ".getNotOrderedSchema.data"

        - name: getNamespaces
          path: "/api/v1/namespaces"
          verb: GET
          headers:
            - "Accept: application/json"
          filter: "[.getNamespaces.items[] | .metadata.name]"

      filter: >
        {
          getOrderedSchema: .getOrderedSchema,
          getNotOrderedSchema: .getNotOrderedSchema,
          getCompositionDefinition: .getCompositionDefinition,
          getCompositionDefinitionResource: .getCompositionDefinitionResource,
          possibleNamespacesForComposition:
          [
            .getNamespaces[] as $ns |
            {
              resource: .getCompositionDefinitionResource,
              namespace: $ns
            }
          ]
        }

    - name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-schema-override-allows-ns'
      namespace: ""

      api:
        - name: getCompositionDefinitionSchemaNs
          path: "/call?apiVersion=templates.krateo.io/v1&resource=restactions&name={{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-compositiondefinition-schema-ns&namespace={{ .Release.Namespace }}"
          verb: GET
          endpointRef:
            name: snowplow-endpoint
            namespace: '{{ default "krateo-system" (default dict .Values.global).krateoNamespace }}'
          headers:
            - "Accept: application/json"
          continueOnError: true
          errorKey: getCompositionDefinitionSchemaNs
          exportJwt: true

        - name: allowedNamespaces
          continueOnError: true
          dependsOn:
            name: getCompositionDefinitionSchemaNs
            iterator: .getCompositionDefinitionSchemaNs.status.possibleNamespacesForComposition
          path: "/apis/authorization.k8s.io/v1/selfsubjectaccessreviews"
          verb: POST
          headers:
            - "Content-Type: application/json"
          payload: |
            ${
              {
                "kind": "SelfSubjectAccessReview",
                "apiVersion": "authorization.k8s.io/v1",
                "spec": {
                  "resourceAttributes": {
                    "namespace": .namespace,
                    "verb": "create",
                    "group": "composition.krateo.io",
                    "version": "v{{ .Values.blueprint.version | replace "." "-" }}",
                    "resource": .resource
                  }
                }
              }
            }
        - name: featuresFrontend
          verb: GET
          headers:
          - 'Accept: application/vnd.github+json'
          - 'X-GitHub-Api-Version: 2022-11-28'
          path: "/search/repositories?q=org:krateoplatformops-test+topic:feature+topic:frontend"
          endpointRef:
            name: github-api
            namespace: '{{ default "krateo-system" (default dict .Values.global).krateoNamespace }}'
          filter: |
            .featuresFrontend.items | map((.name | sub("-feature$"; "")))

        - name: featuresBackend
          verb: GET
          headers:
          - 'Accept: application/vnd.github+json'
          - 'X-GitHub-Api-Version: 2022-11-28'
          path: "/search/repositories?q=org:krateoplatformops-test+topic:feature+topic:backend"
          endpointRef:
            name: github-api
            namespace: '{{ default "krateo-system" (default dict .Values.global).krateoNamespace }}'
          filter: |
            .featuresBackend.items | map((.name | sub("-feature$"; "")))

      filter: >
        {
          getOrderedSchema: .getCompositionDefinitionSchemaNs.status.getOrderedSchema,
          getNotOrderedSchema: .getCompositionDefinitionSchemaNs.status.getNotOrderedSchema,
          getCompositionDefinitionResource: .getCompositionDefinitionSchemaNs.status.getCompositionDefinitionResource,
          featuresFrontend: .featuresFrontend,
          featuresBackend: .featuresBackend,        
          allowedNamespaces: [
            [.allowedNamespaces[] | .status.allowed] as $allowed
            | [.getCompositionDefinitionSchemaNs.status.possibleNamespacesForComposition[] | .namespace]
            | to_entries
            | map(select($allowed[.key] == true) | .value)
          ],
          allowedNamespacesWithResource: [
            [.allowedNamespaces[] | .status.allowed] as $allowed
            | .getCompositionDefinitionSchemaNs.status.getCompositionDefinitionResource as $resource
            | [.getCompositionDefinitionSchemaNs.status.possibleNamespacesForComposition[] | .namespace]
            | to_entries
            | map(
                select($allowed[.key] == true)
                | {
                    namespace: .value,
                    resource: $resource
                  }
              )
          ]
        }
EOF
```