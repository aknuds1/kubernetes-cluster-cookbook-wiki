Welcome to the kubernetes-cluster cookbook Beginners Guide! This will walk you through all the steps you will need to follow in order to set up and run a Kubernetes cluster using this cookbook! The first thing you will need to do is procure some systems to install your cluster on. What you will need will vary on use case. Below is a simple table for what you should use for various situations, but ultimately it is up to you. Your actual needs will vary depending on application types being deployed. Keep in mind that scaling the minion numbers to be larges is extremely quick and easy with this cookbook. And masters should exist in odd numbers to allow for simple master elections. I would recommend you start with a minimal number of Minions and scale as needed.

***
| Purpose  | #Masters  | #Minions  | Master CPU  | Master Memory  | Minion CPU  | Minion Memory  |
|---|---|---|---|---|---|---|
| Development  | 1  | 1  | 1  | 1GB | 1 | 1GB |
| POC  | 3  | 3  | 2  | 4GB  | 2  | 4GB  |
| Production  | 3+  | 3+  | 2  | 4GB  | 4  | 8GB  |
***

Next, make sure you have access to all the software you will need. The only dependency that is needed that cannot be found in the standard Enterprise Linux YUM repositories (satellite or otherwise) is generally Chef. Pull down Chef from the [Chef.io provided download](https://opscode-omnibus-packages.s3.amazonaws.com/el/7/x86_64/chef-12.4.0-1.el7.x86_64.rpm)

***

Now you need to decide if you are going to implement TLS security for your cluster. Without TLS security, everything is wide open if it has network access. You can still hit the APIs if you setup AUTHN/AUTHZ with just user tokens- and client and peer communication is still fairly open as well- with unencrypted connections. TLS security allows for A very simple method of adding additional minions, as well as giving certificate based access to the Kubernetes cluster itself on top of encrypting all connections. The initial setup can be very confusing if you do not have TLS/SSL experience, and it adds a lot of potential issues. A brief outline of how TLS in a Kubernetes cluster is as follows:

-Peer Certificate Authority: The Peer Certificate Authority is used to sign and verify peer-to-peer communication (Masters). This CA certificate only has to reside on the masters.

-Peer Certificate Authority Key: This key is used only for signing Peer Server Certificates. DO NOT distribute this certificate anywhere on the cluster.

-Peer Server Certificates: ETCD master to master security is managed via SSL peer certificates. Each of these peer certificates is unique for each master, and is signed by the "Peer Certificate Authority". These certificates also have Subject Alt Names (SANs) for the node IP address and 127.0.0.1 (localhost). If these SANs do not exist, the certificates will not work. These Certificates will only go on the masters.

-Peer Server Certificate Keys: This key is used for establishing secure connections using the associated Server Certificate. Each peer key will go on the appropriate associated master.

-Client Certificate Authority: The Client Certificate Authority is used to sign and verify client-to-master communication (Masters). This CA certificate will reside on every node in the cluster- as well as getting distributed to every user who will have API access to Kubernetes or ETCD. This certificate Authority can theoretically be the same as the Peer Certificate Authority.

-Client Certificate Authority Key: This key is used only for signing Client Server Certificates. DO NOT distribute this certificate anywhere on the cluster.

-Client Server Certificates: Client to master security is managed via SSL client certificates. Each of these peer certificates is unique for each node in the cluster- as well as each unique user accessing the ETCD or Kubernetes APIs from outside the cluster, and is signed by the "Client Certificate Authority". These certificates also have Subject Alt Names (SANs) for the node IP address and 127.0.0.1 (localhost). If these SANs do not exist, the certificates will not work for cluster members, but will still work for user API access. These Certificates will go on the associated host within the cluster, and to the appropriate users.

-Client Server Certificate Keys: This key is used for establishing secure connections using the associated Server Certificate. Each client key will go on the appropriate associated node on the cluster or user who will be accessing the API. These are sensitive private keys, be careful with storage and distribution.

***

With that out of the way- I will briefly go over TLS certificate generation. The best way to manage this is to either use your own certificate signing service- or to use something like CFSSL, you can install it using:

    curl -s -L -o ~/bin/cfssl https://pkg.cfssl.org/R1.1/cfssl_linux-amd64
    curl -s -L -o ~/bin/cfssljson https://pkg.cfssl.org/R1.1/cfssljson_linux-amd64

Now, moving forward:

    mkdir kube-certs && cd kube-certs

Create a file called ca-config.json with the following contents: (change expiry as appropriate)

    {
        "signing": {
            "default": {
                "expiry": "43800h"
            },
            "profiles": {
                "client-server": {
                    "expiry": "43800h",
                    "usages": [
                        "signing",
                        "key encipherment",
                        "server auth",
                        "client auth"
                    ]
                }
            }
        }
    }

Now, create a file called ca-csr.json: (Change information in "names" as appropriate)

    {
        "CN": "Kubernetes CA",
        "hosts": [
            "example.net",
            "www.example.net"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
                "C": "Country",
                "L": "Location",
                "O": "MyOrg",
                "ST": "State",
                "OU": "MyOrgUnit"
            }
        ]
    }

Now, create your Certificate authority cert and key. This will be used as your Certificate Authority cert that will be used on your nodes. Remember, if you want a UNIQUE certificate authority for both peer and client TLS you will need this twice. This will create a ca.pem and ca-key.pem for your CA accordingly:

    cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

Now, create a file for the certificates for each node called kube-<hostname>.json where <hostname> is the hostname of the host. Replace data in <> angled brackets as appropriate and match the ites in the "names" section with the CA csr above:

    {
        "CN": "<FQDN-OF-NODE>",
        "hosts": [
            "<IP of node>",
            "127.0.0.1"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
                "C": "US",
                "L": "Location",
                "ST": "State"
            }
        ]
    }

Now, sign those cert requests! Remember, each CSR needs to get signed by each appropriate CA. The nodes that are your masters will need to be signed by BOTH your client and peer CA, and the nodes that will be minions will only need to be signed by your client CA. Any TLS certificates for users accessing the APIs will need to also be signed only by the client CA.

    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client-server kube-<hostname>.json | cfssljson -bare kube-<hostname>

These certs will need to be translated into a format that can be inserted into a chef node attribute. This means that each newline must be changed into a "\n" delimiter.

    sed ':a;N;$!ba;s/\n/\\n/g' kube-<hostname>.pem
    sed ':a;N;$!ba;s/\n/\\n/g' kube-<hostname>-key.pem

***

On each node, you will install chef, and do some prep work to get chef ready.

    yum install -y git
    rpm -i chef-12.4.0-1.el7.x86_64.rpm
    mkdir /var/chef
    mkdir /var/chef/cookbooks
    cd /var/chef/cookbooks
    git clone https://github.com/bloomberg/kubernetes-cluster-cookbook

You are now ready to write out your solo.json that Chef-solo will ingest to converge the cookbook! This json will contain node attributes that specify options exposed by this cookbook- and will also optionally include your TLS certificates and other information. This file you will need to create will be /etc/chef/solo.json.

NON TLS Example solo.json for master. ['kubernetes']['etcd']['members'] will contain the fqdn of all your masters- 
```json
{
  "kubernetes": {
    "etcd": {
      "members": ["master1.example.com", "master2.example.com", "master3.example.com"]
    }
  },
  "run_list": ["recipe[kubernetes-cluster::master]"]
}
```

NON TLS Example solo.json for minions. ['kubernetes']['master']['fqdn'] will contain the fqdn of all your masters- 
```json
{
  "kubernetes": {
    "master": {
      "fqdn": ["master1.example.com", "master2.example.com", "master3.example.com"]
    }
  },
  "run_list": ["recipe[kubernetes-cluster::minion]"]
}
```