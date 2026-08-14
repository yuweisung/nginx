# Nginx Practices
## Design


## Implementation

### 1. K8s with kubeadm

    The practice contains two parts, one is k8s (kubeadm) deployment to my home lab, and the other is nginx statefulset deployment. 

* Installing ansible

    Refer to ansible site to install ansible to your local desktop.
    ```
    brew install ansible
    ```

* Modifying the hosts file (inventory)

    Modify the hosts file to reflect your inventory.  I have 1 master and 4 workers (ubuntu servers) and have ssh-copy-id sending my ssh public key to those servers.  

    ```
    mv hosts.template hosts
    ```

* Deploying k8s cluster

    First, as k8s requirements, setup some kernel configs in each server.  The os/main.yaml has those actions.

    ```
    ansible-playbook os/main.yaml
    ```

    Second, run kubeadm/main.yaml to deploy k8s control/data planes. The kubeadm.yaml will be used in kubeadm init process.

    ```
    ansible-playbook kubeadm/main.yaml
    ```

    Once the actions finished, you can use kubectl to check the status because one of the task pulled the admin.conf to ~/.kube/config directly.  It should look like this:

    ```
    kubectl get nodes
    NAME            STATUS   ROLES           AGE   VERSION
    m1.home.lab     NotReady    control-plane   17h   v1.36.3
    ms1.home.lab    NotReady    <none>          17h   v1.36.3
    ms2.home.lab    NotReady    <none>          17h   v1.36.3
    nuc9.home.lab   NotReady    <none>          17h   v1.36.3
    p3.home.lab     NotReady    <none>          17h   v1.36.3
    w4.home.lab     NotReady    <none>          17h   v1.36.3
    ```

    Next step, we deploy the callico cni. You should see that nodes are READY after calico installed. The last action in the playbook is a for loop to approve kubelet csr. Check csr status before you move to next step.

    ```
    ansible-playbook kubectl/0_callico.yaml
    ...
    k get nodes
    NAME            STATUS   ROLES           AGE   VERSION
    m1.home.lab     Ready    control-plane   17h   v1.36.3
    ms1.home.lab    Ready    <none>          17h   v1.36.3
    ms2.home.lab    Ready    <none>          17h   v1.36.3
    nuc9.home.lab   Ready    <none>          17h   v1.36.3
    p3.home.lab     Ready    <none>          17h   v1.36.3
    w4.home.lab     Ready    <none>          17h   v1.36.3
    ``` 

    Continue installing metallb, rook-ceph and cert-manager by running following playbooks. Note that you will need a keypair (self-signed) to create a ca-issuer.  Google openssl self-sign for commands.

    ```
    ansible-playbook kubectl/2_metallb.yaml
    ansible-playbook kubectl/3_rook-ceph.yaml
    ansible-playbook kubectl/4_cert-manager.yaml
    ```
    

### 2. Nginx statefulset with TLS

* Creating private key and csr

    ```
    openssl genrsa -out user1.key 2048
    openssl req -new -key user1.key -out user1.csr -subj "/CN=user1/O=dev"
    ```

* Creating user csr manifest

    ```
    cat << EOF > user1-csr.yaml
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
    name: user1-csr
    spec:
    request: `cat user1.csr|base64|tr -d "\n"`
    signerName: kubernetes.io/kube-apiserver-client
    expirationSeconds: 864000
    usages:
        - client auth
    EOF
    ```

* Applying and Approving user csr to k8s

    ```
    kubectl apply -f user1-csr.yaml
    kubectl get csr
    kubectl certificate approve user1-csr
    kubectl get csr user1-csr -o jsonpath='{.status.certificate}' | base64 -d > user1.crt
    ```

* Checking the user certificate

    ```
    openssl x509 -in user1.crt -noout -text
    ```

* Creating dev namespace

    ```
    kubectl create ns dev
    ```

* Applying user1/argocd rbac

    ```
    cat <<EOF > user1-rbac.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: user1-role
      namespace: dev
    rules:
    - apiGroups: ["apps",""]
        resources: ["statefulsets","deployments", "pods", "services", "persistentvolumeclaims"]
        verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: user1-binding
      namespace: dev
    subjects:
    - kind: User
      name: user1
      namespace: dev
    - kind: ServiceAccount
      name: argocd-application-controler
      namespace: argocd
    roleRef:
      kind: Role
      name: user1-role
      apiGroup: rbac.authorization.k8s.io
    EOF

    kubectl apply -f user1-rbac.yaml
    ```

* Setting user1 context

    ```
    kubectl config set-credentials user1 --client-certificate=user1.crt --client-key=user1.key --embed-certs=true
    kubectl config set-context user1-context --cluster=kubernetes --user=user1
    kubectl config use-context user1-context
    ```


* Prepparing nginx config

    ```
    cat <<EOF >nginx-cm.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: nginx-config
    namespace: dev
    data:
    default.conf: |
        server {
        listen      443 ssl http2;
        ssl_certificate /etc/nginx/tls/tls.crt;
        ssl_certificate_key /etc/nginx/tls/tls.key;
        ssl_protocols	TLSv1.3;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
        }
    EOF
    ```

* Preparing nginx statefulset

    ```
    cat <<EOF >nginx-sts.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: nginx
    namespace: dev
    labels:
        app: nginx
    spec:
    ports:
    - port: 443
        name: https
    clusterIP: None
    selector:
        app: nginx
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
    name: web
    namespace: dev
    spec:
    serviceName: nginx
    replicas: 2
    selector:
        matchLabels:
        app: nginx
    template:
        metadata:
        labels:
            app: nginx
        spec:
        containers:
        - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 443
            name: https
            volumeMounts:
            - name: nginx-config
            mountPath: /etc/nginx/conf.d/default.conf
            subPath: default.conf
            - name: tls
            mountPath: /etc/nginx/tls
            readOnly: true
        volumes:
        - name: nginx-config
            configMap:
            name: nginx-config
        - name: tls
            csi:
            driver: csi.cert-manager.io
            readOnly: true
            volumeAttributes:
                csi.cert-manager.io/issuer-name: ca-issuer
                csi.cert-manager.io/issuer-kind: ClusterIssuer
                csi.cert-manager.io/dns-names: ${POD_NAME}.${POD_NAMESPACE}.svc.cluster.local,www.home.lab
                csi.cert-manager.io/ip-sans: 10.0.0.31
                csi.cert-manager.io/common-name: "${SERVICE_ACCOUNT_NAME}.${POD_NAMESPACE}"
    EOF
    ```

* Preparing nginx loadbalancer service

    ```
    cat <<EOF >nginx-ext-svc.yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: nginx-ext-service
    namespace: dev
    spec:
    selector:
        app: nginx
    ports:
        - protocol: TCP
        port: 443
        targetPort: 443
    type: LoadBalancer
    sessionAffinity: ClientIP
    sessionAffinityConfig:
        clientIP:
        timeoutSeconds: 10800
    EOF
    ```

# argocd
