# VM Migration for VMWare to KubeVirt with Forklift


```sh

```

# *VM Migration from VMWare to KubeVirt with Forklift* Blueprint

## Overview

This Krateo blueprint orchestrates the migration of virtual machines from a **VMware ESXi** environment to a **KubeVirt** environment in a Kubernetes cluster.
It leverages the **Forklift operator** for the migration process, ensuring a smooth transition of VMs.

The blueprint will deploy the necessary resources needed to achieve the migration, including:
- Network and storage mappings
- Migration plans

## Requirements

For this blueprint to function correctly, your Kubernetes cluster **must** have the following operators installed and running on the target Kubernetes cluster. Please follow their official documentation for installation instructions.

1.  **KubeVirt Operator:**
    -   **Purpose:** Manages KubeVirt virtual machine resources within the Kubernetes cluster.

2.  **Forklift Operator:**
    -   **Purpose:** Orchestrates the migration of virtual machines from a source environment (like VMware) to a KubeVirt environment.

3. Create the following manifests:

```sh
cat <<EOF | kubectl apply -f -
kind: Secret
apiVersion: v1
metadata:
  name: esxi-secret
  namespace: openshift-mtv
  labels:
    createdForProviderType: vsphere
    createdForResourceType: providers
stringData:
  insecureSkipVerify: "true"
  password: < esxi.password >
  url: < esxi.url >
  user: < esxi.user >
type: Opaque
---
kind: Secret
apiVersion: v1
metadata:
  name: forklift-inventory-endpoint
  namespace: openshift-mtv
stringData:
  server-url: < forklift inventory endpoint >
---
apiVersion: v1
kind: Secret
metadata:
  name: esxi-restaction-sa
  namespace: openshift-mtv
  annotations:
    kubernetes.io/service-account.name: "esxi-restaction"
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: esxi-restaction
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: esxi-restaction-admin-restaction
subjects:
- kind: ServiceAccount
  name: esxi-restaction
  namespace: openshift-mtv
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: forklift.konveyor.io/v1beta1
kind: Provider
metadata:
  name: ovh-esxi
  namespace: openshift-mtv
spec:
  secret:
    name: esxi-secret
    namespace: openshift-mtv
  settings:
    sdkEndpoint: esxi
    vddkInitImage: < vddk init image>
  type: vsphere
  url: < esxi url >
EOF
```

## Usage

### Install the Helm Chart

Download Helm Chart values:

```sh
helm repo add marketplace https://marketplace.krateo.io
helm repo update marketplace
helm inspect values marketplace/vm-migration --version 0.1.1 > ~/vm-migration-values.yaml
```

Modify the *vm-migration-values.yaml* file as the following example:

```yaml

```

Install the Blueprint:

```sh
helm install <release-name> vm-migration \
  --repo https://marketplace.krateo.io \
  --namespace <release-namespace> \
  --create-namespace \
  -f ~/vm-migration-values.yaml \
  --version 0.1.1 \
  --wait
```

### Install using Krateo Composable Operation

Install the CompositionDefinition for the *Blueprint*:

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: vm-migration
  namespace: openshift-mtv
spec:
  chart:
    repo: vm-migration
    url: https://marketplace.krateo.io
    version: 0.1.1
EOF
```

Install the Blueprint using, as metadata.name, the *Composition* name (the Helm Chart name of the composition):

```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v0-1-1
kind: VmMigration
metadata:
  name: ovh-rosa-composition
  namespace: openshift-mtv
spec:
  plan:
    vms:
    - id: "1"
      name: test-migration-1
EOF
```

### Install using Krateo Composable Portal

```sh
cat <<EOF | kubectl apply -f -
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: portal-blueprint-page
  namespace: krateo-system
spec:
  chart:
    repo: portal-blueprint-page
    url: https://marketplace.krateo.io
    version: 1.1.1
EOF
```

Install the Blueprint using, as metadata.name, the *Blueprint* name (the Helm Chart name of the blueprint):

```sh
cat <<'EOF' | kubectl apply -f -
apiVersion: composition.krateo.io/v1-1-1
kind: PortalBlueprintPage
metadata:
  name: vm-migration
  namespace: openshift-mtv
spec:
  blueprint:
    repo: vm-migration
    url: https://marketplace.krateo.io
    version: 0.1.1
    hasPage: true
    credentials: {}
  form:
    widgetData:
      schema: {}
      objectFields: []
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

            # --- inject "composition" under top-level "properties" ---
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

            # --- ensure top-level "required" includes "composition" ---
            | (
                if ($withComposition | test("\n  \"required\": \\[")) then
                  if ($withComposition | test("\n  \"required\": \\[[^\\]]*\"composition\"")) then
                    # "composition" already present at top-level
                    $withComposition
                  else
                    # prepend "composition" to existing top-level required array
                    ($withComposition
                    | gsub("\n  \"required\": \\["; "\n  \"required\": [\"composition\", "))
                  end
                else
                  # no top-level required: insert before top-level "type": "object"
                  ($withComposition | index("\n  \"type\": \"object\"")) as $tidx
                  | ($withComposition[0:$tidx+1]) as $p2
                  | ($withComposition[$tidx+1:]) as $r2
                  | $p2 + "  \"required\": [\"composition\"],\n" + $r2
                end
              )
          }
    resourcesRefsTemplate:
      iterator: ${ .allowedNamespacesWithResource[] }
      template:
        id: composition-to-post
        apiVersion: composition.krateo.io/v0-1-1
        namespace: ${ .namespace }
        resource: ${ .resource }
        verb: POST
    apiRef:
      name: '{{ .Values.global.compositionKind | lower }}-{{ .Values.global.compositionName }}-restaction-schema-override-allows-ns'
      namespace: ""
    hasPage: false
    instructions: |
      # *VM Migration from VMWare to KubeVirt with Forklift* Blueprint

      ## Overview

      This Krateo blueprint orchestrates the migration of virtual machines from a **VMware ESXi** environment to a **KubeVirt** environment in a Kubernetes cluster.
      It leverages the **Forklift operator** for the migration process, ensuring a smooth transition of VMs.

      The blueprint will deploy the necessary resources needed to achieve the migration, including:
      - Source and target providers (e.g., VMware and cluster with KubeVirt)
      - Network and storage mappings
      - Migration plans

      ## Requirements

      For this blueprint to function correctly, your Kubernetes cluster **must** have the following operators installed and running on the target Kubernetes cluster. Please follow their official documentation for installation instructions.

      1.  **KubeVirt Operator:**
          -   **Purpose:** Manages KubeVirt virtual machine resources within the Kubernetes cluster.

      2.  **Forklift Operator:**
          -   **Purpose:** Orchestrates the migration of virtual machines from a source environment (like VMware) to a KubeVirt environment.
          
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
        name: fa-cubes
      items:
        - resourceRefId: blueprint-panel-markdown
      title: GitHub Scaffolding with Composition Page
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
        - name: getToken
          path: /api/v1/namespaces/openshift-mtv/secrets/esxi-restaction-sa
          verb: GET
          filter: .getToken.data.token | @base64d
          errorKey: getTokenError
        - name: getUidProvider
          dependsOn:
            name: getToken
          path: /providers
          verb: GET
          headers:
          - "${ \"Authorization: Bearer \" + .getToken }"
          endpointRef:
            name: forklift-inventory-endpoint
            namespace: openshift-mtv
          filter: >
            .getUidProvider.vsphere[] | select(.name=="{{ .Release.Name }}") | .id
          errorKey: getUidProviderError
        - name: vms
          dependsOn:
            name: getUidProvider
          endpointRef:
            name: forklift-inventory-endpoint
            namespace: openshift-mtv
          headers:
          - "${ \"Authorization: Bearer \" + .getToken }"
          path: ${ "/providers/vsphere/" + .getUidProvider + "/vms" }
          filter: >
            [ .vms.[] | {id: .id, name: .name} ]
          verb: GET
          errorKey: vmsError
      filter: >
        {
          vms: .vms,
          getOrderedSchema: .getCompositionDefinitionSchemaNs.status.getOrderedSchema,
          getNotOrderedSchema: .getCompositionDefinitionSchemaNs.status.getNotOrderedSchema,
          getCompositionDefinitionResource: .getCompositionDefinitionSchemaNs.status.getCompositionDefinitionResource,      
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