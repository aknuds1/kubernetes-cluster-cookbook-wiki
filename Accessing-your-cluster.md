### Accessing your cluster

The easiest method for accessing your cluster API functions (ETCD and Kube API server) are via localhost on one of your master nodes themselves. However, this may not be practical- and is honestly far from optimal. You should configure the client binaries to work from your desired location remotely.

-etcdctl: etcdctl is the client binary for accessing etcd API functions. It allows you to browse ETCD much like a filesystem. you can list keys using "ls", display their values using "get", write values using "put", create keys using "mk", and create directories using "mkdir". The default behavior for the client is to access an insecure etcd instance with an exposed endpoint at 127.0.0.1:2379/127.0.0.1:4001. Most typical Kubernetes functions do not require you to access etcd, but it is still good to be able to read key/values when needed. Configuration of the etcd client only happens via command line switches or environment variables at the moment. Using the appropriate switches you should be able to access the ETCD cluster as needed.

| Switch  | Environment variable  | Function  |
|---|---|---|
| --endpoint  | ETCDCTL_ENDPOINT  | Override the endpoint to access  |
| --cert-file  | ETCDCTL_CERT_FILE  | Specify SSL/TLS client certificate |
| --key-file  | ETCDCTL_KEY_FILE  | Specify private key matching client certificate  |
| --ca-file  | ETCDCTL_CA_FILE  | Specify ETCD client CA cert for verification |

-kubectl: kubectl is the client binary for accessing Kubernetes API functions. It provides the capabilities to create services, add users, create replication controllers, view statuses and logs, as well as almost any other operational task you would need for Kubernetes. This is your bread and butter- it will be your deployment mechanism and your means for status updates. The good thing about kubectl, is that you can create a config file to tell kubectl how to access your API server. As with etcdctl, kubectl defaults to connecting to http://127.0.0.1:8080. In order to set TLS/SSL certificate information, as well as changing the endpoints, you will create a kube.config file and specify that with the --kubeconfig command line switch.

The below yaml is a sample kube.config file. Replace data as needed- SSL/TLS certificate information is only needed in a TLS secured cluster, set the server to the API server endpoint you wish to access.
```yaml
current-context: default
apiVersion: v1
clusters:
- cluster:
    api-version: v1
    server: https://kubernetes-master-1.example.com:8443
    certificate-authority: /path/to/client/ca
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    namespace: default
    user: operator
  name: default
kind: Config
preferences:
  colors: true
users:
- name: operator
  user:
    client-certificate: /path/to/client/cert
    client-key: /path/to/client/key
```

With a good kube.config you should now be able to access your cluster. Test it with the following command; it should output the hostnames of your cluster members:

    kubectl --kubeconfig=/path/to/kube.config get nodes

If that is successful, you are now ready to proceed! [Try deploying an application!](https://github.com/bloomberg/kubernetes-cluster-cookbook/wiki/Deploying-your-application)