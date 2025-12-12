# VM Migration for VMWare to KubeVirt with Forklift

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
```sh
cat <<EOF | kubectl apply -f -
apiVersion: composition.krateo.io/v0-1-1
kind: VmMigration
metadata:
  name: ovh-rosa-composition
  namespace: openshift-mtv
spec:
  esxi:
    user: # username
    password: # password
EOF
```