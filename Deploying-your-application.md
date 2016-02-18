Kubernetes biggest winning point is the ability to deploy and manage [twelve factor methodology applications](http://12factor.net/) seamlessly and with little to no effort. However, Kubernetes can be used for less agile and lightweight applications- using local mounts and including complex dependencies- and once you set up the application correctly the deployment process becomes repeatable regardless. There are a few concepts that we will discuss quickly so you understand what the deployment process looks like:
***

###Pods, not containers:

While strictly speaking, Kubernetes does handle the deployment of containers- it would be more accurate to say that Kubernetes deploys pods. Pods are logically grouped sets of containers that either serve the same application, or are depending on each other to accomplish their task. For example- If you were to deploy an application that has a frontend like Apache httpd, and a backend message queue like RabbitMQ- you would not build those into the same containers. You build them into separate containers so you can maintain fine grained logging, resource control, and monitoring. The problem with this is that you now have to deploy two containers instead of one. Enter: pods. A pod by default has open network communication between its member containers, so you wont have to set up any special networking link- its like they are on the same virtual machine. The pod also ties the deployment of these containers together, meaning that you are only deploying one object. When you delete the pod, both containers go away. Technically, a pod could consist of a single container, and that is perfectly acceptable to do- but related containers which depend on each other to complete a task should be created as a pod when possible.

###Replication controllers control replication (and other things):

Replication controllers dictate what your pod contains, how many copies of it there are, and what resources the pod will have. This allows you to dictate your entire application deployment in one simple method. A great way of managing your replication controllers is to check the replication controller YAML in to source control with the application itself, if possible. This way, you can manage the deployment method for your application- and the application itself in one place! Replication controllers are very powerful, you can set health checking, replication numbers, containers, exposed ports, volume mounts... almost everything that doesn't involve network routing.

###Services:

Services dictate how you will access containers within your pods. Remember that by default network communication within a pod is open? Well by default, any communication reaching to a pod is closed. This means that users outside the cluster can't reach in to your containers. Here is where services come in. A service dictates how you access the containers, and there are multiple ways of dictating this. You can expose a container on a port on all the members of your cluster (nodePort), or even give your service its own unique IP address withing the cluster "Service network". Node ports require no extra configuration- you will simply be able to access your container on that port on any of the minions in your cluster- whether the container is on that minion or not! Using service network IP however, will require you to have a proxy within the cluster that will route traffic accordingly- this generally is hosted on the master, or is another service within the cluster with an exposed node port!

***

Now with the basic concepts out of the way- lets work with some examples. This example is assuming that you already know how to build a Docker container. I suggest starting with something simple- perhaps a simple nodeJS front end, or an HTTP static content server. This will minimize the confusion and the number of things that can go wrong. We will go over deploying a *single container per pod replication controller*. Parenthetical comments are descriptive and should not be included in the actual YAML.

```yaml
kind: "ReplicationController"
apiVersion: "v1"
metadata:
  name: "mytestapp-controller"        (Displayed name for the RC)
spec:
  replicas: 2                         (Number of copies of the pod to have running at any given time)
  selector:
    name: "mytestapp-rc"
  template:
    spec:
      containers:
        - name: "mytestapp-http"      (Display name for the container)
          image: "mytestapp:1.0.0"    (Docker image to create container with)
          ports:
            - containerPort: 8080     (Container port to expose)
          livenessProbe:
            httpGet:                  (Use http healthcheck to verify health of container)
              path: /                 (URL to verify health at)
              port: 8080              (Port to connect to via HTTP)
            initialDelaySeconds: 5    (how long to wait before starting healthcheck)   
            timeoutSeconds: 2         (How long to wait before healthcheck fails)
    metadata:
      labels:
        app: "mytestapp"              (Metadata label to set for application for RC in ETCD)
  labels:
    name: "mytestapp-rc"              (Metadata label to set for name for RC in ETCD)
```

The above replication controller yaml only touches on the power of replication controllers. This is simply specifying the container in your pod, a health check, name for your replication controller and app, and an exposed port on the container. You can also mount local volumes, nfs volumes, and amazon EBS volumes. You can specify multiple containers, as well.

    kubectl create -f /path/to/rc.yaml
    kubectl get rc

You now have a replication controller, being deployed and managed by kubernetes. This means your container is deployed! But it is not accessible.

***

As discussed above, a POD is externally inaccessible without a service exposing a method for access! This may be intentional, for example if you want a container that just spins up to do work and then dies- or even a container that stays in the cluster for some other reason- like pushing events or messages. Lets take a look at a sample service yaml:

```yaml
Kubernetes:
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mytestapp
  name: mytestapp-service
  namespace: default
spec:
  type: NodePort           (The method of exposing access)
  ports:
  - port: 8080             (The port of the container to be accessed)
    protocol: TCP
    nodePort: 30095        (The port on the cluster to expose)
  selector:
    app: mytestapp         (A selector matching the replication controller pod label)
```

Above, I use NodePort as my method for service exposure. This means you can access this application on the specified nodePort on the IP or FQDN of the actual host itself. Meaning I can now access this application on http://minion-1.example.com:30095. The big advantage of this is that I don't have to keep tight tracking of what node the container is actually on- Kubernetes will actually route the traffic as needed. So I can also access it on http://minion-2.example.com:30095 and http://minion-3.example.com:30095 as well- even if the application is only running on one or two of them! Using this setup, I can point a load balancer at the entire cluster, and I don't have to worry about discovery, I would just update the load balancer to expose 30095 and my application is easily accessible!

    kubectl create -f /path/to/svc.yaml
    kubectl get svc
    curl http://minion-1.example.com:30095

Congratulations! If all went well, your first application is now deployed!