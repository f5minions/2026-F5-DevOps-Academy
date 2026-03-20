## Lab 진행을 위한 선수과정


Kubernetes Context 확인 (Client-vscode) 

* rahcner1 Context 에서 진행 

    ```Basic
    kubectl config current-context 
    rancher1

    ## Context 확인
    kubectl config get-contexts 
    ## Context 변경
    kubectl config use-context rahcner1
    ```
    



<!-- kubernetes Admin 설정 

    export KUBECONFIG=/etc/rancher/k3s/k3s.yaml


Helm Package 설치 필요 

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
    chmod 700 get_helm.sh
    ./get_helm.sh -->
