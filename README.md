# VM Migration for VMWare to KubeVirt with Forklift

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
    version: 0.1.0
EOF