# Controlling Access in Kubernetes with RBAC


#### Preparing Certificates for user

To find K8s certificate authority (CA), go to
```
$ etc/kubernetes/pki> ls -la
```
```
-rw-r--r-- 1 root root 1099 Jul  4 09:41 ca.crt
-rw------- 1 root root 1679 Jul  4 09:41 ca.key
```

Create and move to a directory where we work on the creation of the user certificate. Copy K8S CA cert and key in order to sign user certificate:
```
$ k8s-resources> sudo mkdir rbac
```
```
$ k8s-resources/rbac> sudo cp /etc/kubernetes/pki/ca.crt ca.crt
```
```
$ k8s-resources/rbac> sudo cp /etc/kubernetes/pki/ca.key ca.key
```
```
$ k8s-resources/rbac> ls -la
-rw-r--r-- 1 root      root      1099 Jul 12 10:29 ca.crt
-rw-r--r-- 1 root      root      1099 Jul 12 10:29 ca.key
```

the format of the ca.crt should start and end like this:
```
-----BEGIN RSA PRIVATE KEY-----
......
-----END RSA PRIVATE KEY-----
```
and the format for ca.key should start and end like this:
```
-----BEGIN CERTIFICATE-----
......
-----END CERTIFICATE-----
```

Make sure openssl is installed and create a certificate for user "dev".

```
$ k8s-resources/rbac> openssl genrsa -out dev.key 2048
```
```
Generating RSA private key, 2048 bit long modulus (2 primes)
....+++++
....+++++
e is 65537 (0x010001)
```
```
$ k8s-resources/rbac> ls -la
```
```
-rw-r--r-- 1 automaton automaton 1099 Jul 12 10:29 ca.crt
-rw-r--r-- 1 automaton automaton 1099 Jul 12 10:29 ca.key
-rw------- 1 automaton automaton 1675 Jul 12 10:40 dev.key
```

Create a certificate signing request (CSR). Make sure to specify the GROUP "dev" user belongs to, in this case I will use "Development". Take note of the CN value.

```
$ k8s-resources/rbac> openssl req -new -key dev.key -out dev.csr -subj "/CN=User-Dev/O=Development"
```
```
$ k8s-resources/rbac> ls -la
```
```
-rw-r--r-- 1 automaton automaton 1099 Jul 12 10:29 ca.crt
-rw-r--r-- 1 automaton automaton 1099 Jul 12 10:29 ca.key
-rw-rw-r-- 1 automaton automaton  920 Jul 12 10:45 dev.csr
-rw------- 1 automaton automaton 1675 Jul 12 10:40 dev.key
```

Use the CA to generate a signed certificate for user "dev" using the csr previously generated

```
$ k8s-resources/rbac> openssl x509 -req -in dev.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dev.crt -days 3650
```
```
Signature ok
subject=CN = User-Dev, O = Development
Getting CA Private Key
```


#### Creating a new user config file for K8S


Lets tell kubectl to look to a new config that does not exist yet, in this case "dev-config" file

```
$ export KUBECONFIG=~/.kube/dev-config
```

In order to start building that dev-config file, we need to pass cluster, credential and context information to the empty file "dev-config".

a) Cluster entry:
```
$ kubectl config set-cluster kubernetes --server=https://11.111.33.147:6443 --certificate-authority=ca.crt --embed-certs=true
```
```
Cluster "kubernetes" set.
```
```
$ cat ~/.kube/dev-config
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS00tCg==
    server: https://11.111.33.147:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

b) Setting user credentials

```
$ kubectl config set-credentials dev --client-certificate=dev.crt --client-key=dev.key --embed-certs=true
```
```
User "dev" set.
$ cat ~/.kube/dev-config
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS00tCg==
    server: https://11.111.33.147:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: dev
  user:
    client-certificate-data: LS00tCg==
    client-key-data: LS00tCg==
```


c) Setting Context with namespace

```
$ kubectl config set-context dev --cluster=kubernetes --namespace=testnamespace --user=dev
```
```
Context "dev" created.
```
```
$ cat ~/.kube/dev-config
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS00tCg==
    server: https://11.111.33.147:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: testnamespace
    user: dev
  name: dev
current-context: ""
kind: Config
preferences: {}
users:
- name: dev
  user:
    client-certificate-data: LS00tCg==
    client-key-data: LS00tCg==
```

d) Switching to context dev
```
$ kubectl config use-context dev
```
```
Switched to context "dev".
```
```
$ cat ~/.kube/dev-config
```
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS00tCg==
    server: https://11.111.33.147:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: testnamespace
    user: dev
  name: dev
current-context: dev
kind: Config
preferences: {}
users:
- name: dev
  user:
    client-certificate-data: LS00tCg==
    client-key-data: LS00tCg==
```


Note "current-context" key, it contains the current context "dev" active.

furthermore once using context "dev":
```
$ kubectl get pods
```
```
Error from server (Forbidden): pods is forbidden: User "User-Dev" cannot list resource "pods" in API group "" in the namespace "testnamespace"
```
The error above is because there is not ROLE or ROLEBINDING associated to this user "dev"


At this stage, would worth to pass the initial and new config file paths to the KUBECONFIG variable, in order to switch contexts accordingly. This will not persist this variable so to to this add this to e.g. ~/.bashrc

```
$ export KUBECONFIG="${HOME}/.kube/config:${HOME}/.kube/dev-config"
```
```
$ echo $KUBECONFIG
```
```
/home/automaton/.kube/config:/home/automaton/.kube/dev-config
```


#### Implementing Role and Rolebindings (RBAC)

From here we are assuming there is a kubernetes cluster with 2 context: kubernetes-admin@kubernetes and dev also the cluster has a namespace testnamespace and with one zookeeper pod.


Create a role spec file: 

```
$ k8s-resources> vi pod-reader-role.yml
```
Add the following to the file:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: testnamespace
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
```

Create the role:

Make sure you are K8s admin
```
$ kubectl config use-context kubernetes-admin@kubernetes
```
```
Switched to context "kubernetes-admin@kubernetes".
```
And then apply role to testnamespace

```
$ kubectl apply -f ~/k8s-resources/pod-reader-role.yml -n testnamespace
```

```
role.rbac.authorization.k8s.io/pod-reader created
```

Create the RoleBinding spec file:

```
$ k8s-resources> vi pod-reader-rolebinding.yml
```
Add the following to the file:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
  namespace: testnamespace
subjects:
- kind: User
  name: User-Dev # Should match CN in dev.crt
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Create the RoleBinding:

Make sure you are K8s admin

```
$ kubectl config use-context kubernetes-admin@kubernetes
```
```
Switched to context "kubernetes-admin@kubernetes".
```

And then apply rolebinding to testnamespace

```
$ kubectl apply -f ~/k8s-resources/pod-reader-rolebinding.yml -n testnamespace
```
```
rolebinding.rbac.authorization.k8s.io/pod-reader_rb created
```


Now change to dev context

```
$ kubectl config use-context dev
```
```
Switched to context "dev".
```

Test access again to verify you can successfully list pods:

```
$ kubectl get pods
```
Above namespace name is not needed as we are under dev context.
On the other hand if we are outside dev context, we can execute this:

```
$ kubectl get pods -n testnamespace --kubeconfig $HOME/k8s-resources/dev-config
```

This time, we should see a list of pods, there's just one.

```
NAME                              READY   STATUS    RESTARTS   AGE
zookeeper-depl-5fbcc55b7b-2b4xm   1/1     Running   0          5d22h
```

Verify the dev user can read pod logs:

```
$ kubectl logs zookeeper-depl-5fbcc55b7b-2b4xm
```

Above namespace name is not needed as we are under dev context.
On the other hand if we are outside dev context, we can execute this:

```
$ kubectl logs zookeeper-depl-5fbcc55b7b-2b4xm -n testnamespace --kubeconfig $HOME/k8s-resources/dev-config
```
We'll get an Auth processing... message.

```
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
2022-07-05 15:58:40,886 [myid:] - INFO  
...
```

Verify the dev user cannot make changes by attempting to delete a pod:

```
$ kubectl delete pod zookeeper-depl-5fbcc55b7b-2b4xm
```
Above namespace name is not needed as we are under dev context.
On the other hand if we are outside dev context, we can execute this:

```
$ kubectl delete pod zookeeper-depl-5fbcc55b7b-2b4xm -n testnamespace --kubeconfig $HOME/k8s-resources/dev-config
```

We'll get an error, which is what we want.

```
Error from server (Forbidden): pods "zookeeper-depl-5fbcc55b7b-2b4xm" is forbidden: User "User-Dev" cannot delete resource "pods" in API group "" in the namespace "testnamespace"
```
