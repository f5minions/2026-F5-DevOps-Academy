
## CIS Deploy 


### Helm Repo Add

    helm repo add f5-stable https://f5networks.github.io/charts/stable

### CIS Install

    helm install -f values.yaml f5-cis-deploy f5-stable/f5-bigip-ctlr  --skip-crds