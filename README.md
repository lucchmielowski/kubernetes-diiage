# DIIAGE kubernetes course

## Installing the kind cluster

To follow this example you'll need a kubernetes cluster. You can create create one using `kind` ([how to install](https://kind.sigs.k8s.io/docs/user/quick-start/))
You'll also need to install the [kubectl CLI](https://kubernetes.io/docs/tasks/tools/) for your distribution

Once the CLI installed, you can create the cluster by running the following command :

```shell
 kind create cluster --config cluster.yaml
```

We dont need to understand what each line of the `kind-cluster.yml` does for now

## Creating your first pod

Let's create your first pod ! To do that, you can run the following command:

```shell
kubectl run nginx --image=nginx:latest --port=80
```

## About the kubectl CLI

The `kubectl` CLI is one the main tools with which you're going to interact with your kubernetes cluster. All of its
commands have a pretty similar syntax to `docker`.

eg:
```shell
# List running pods
kubectl get pods

# Create a pod nginx
kubectl run nginx --image=nginx:latest

# Execute an ls command on the nginx pod created previously
kubectl exec -it nginx -- ls /

# Get all machines from the kubernetes cluster
kubectl get nodes

# And much more ...
```

## Ok I have my pod running, what now ?



Let's try something more complicated, we're gonna deploy a `wordpress` application inside our `kubernetes` cluster

We want it to look like this: ![tutorial wordpress schema](./docs/images/wordpress_schema.png)

### Deployments

_All files must be applied using the following command:_

```bash
kubectl apply -f <yaml_file_path> # (or <directory>)
```

Let's first create our wordpress pods ! The proper way to deploy pods in kubernetes is by using 
an object called `Deployments` since they come with a lot of features that are going to be usefull to us

Let's create our first one in `wordpress/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:4.8-apache
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_PASSWORD
              value: toto123
          ports:
            - containerPort: 80
              name: wordpress
```

We'll need to create the mysql one as well inside `mysql/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: toto123
          ports:
            - containerPort: 3306
              name: mysql
```


### Services

You can see that both pods are created but only the mysql is staying "up". 
The wordpress is restarting and should probably be in `CrashLoopBackOff` state (meaning it's failing and restarting in a loop).
Why's that ? 

*Because the wordpress pod cannot access the db pod !*

Because of it's highly distributed nature, kubernetes forces us to use a network abstraction called `Service` to make 
containers talk together. (Otherwise we'd have to deal with the ever-changing IP address of the pods 
to be able to connect to them !)

Let's create 2 services then

```yaml
# First one to make the DB available
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```

```yaml
# Another one for wordpress, since we want to access it as well
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend

```

Note: services are using `selector` to match a pod / list of pods and know where to re-route the
traffic to. So you should make sure that the service's `selector` matches some of your pods' `labels`.


At this point your `wordpress` pods should be able to discuss together ! 

To test that it works you can connect to the `wordpress` pods on the port 80 (pod that we're publishing in the deployment)
with the following command: 

```shell
# List pods and find the one called wordpress-<some_id>
kubectl get pods
# Forwards port 80 of the container to your localhost:9090
kubectl port-forward wordpress-<some_id> 9090:80
```

Now by going to your [localhost](http://localhost:9090) you should be able to see the wordpress config page
![wordpress config page](./docs/images/wordpress-config.png)

### Persistent storage

Now, if you've followed the previous steps, you should have a working wordpress configuration ...
almost.

Let's configure wordpress completely. Once done go to your console and do a :

```shell
# Delete the mysql pod
kubectl delete pods wordpress-mysql-<some_id>
# Restart wordpress for good measure (since it has lost connection)
kubectl delete pods wordpress<some_id>
```

If you retry to access the `wordpress` container with the `port-foward` command, you'll notice that the app is now back
at the configuration-level 😭.

This happens because we're missing *persistence*. By default, in containers, we're storing data on the host on which 
the pod is running ... which is problematic when you never know which pod is going to go where ! That's why kubernetes 
has the concept of `PersistentVolumeClaim` which acts as a volume that you mount on your container and that can be used
to store data and that will never disapear even when a pod restarts !

Since we want to save or data in both the DB and wordpress (images, ...) Let's create 2 PVC:

```yaml
# PVC for wordpress pods, notice the storage request ?
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20G
```

```yaml
# Same thing for mysql
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

You should be able to see your 2 PVC created and `Pending`. They are in `Pending` state because they are waiting to be
attached to a given pod / set of pods, so let's attach them !

To do that we need to modify our deployments by adding `volumes` and `volumeMounts` to our config:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: toto123
          ports:
            - containerPort: 3306
              name: mysql
          # NEW
          volumeMounts:
            - name: mysql-persistent-storage # Must match the name of a volume in volumes
              mountPath: /var/lib/mysql
      # NEW
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            # Name of the pvc created in pvc.yaml
            claimName: mysql-pv-claim
```

Same thing should be done for wordpress: 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:4.8-apache
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_PASSWORD
              value: toto123
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/www/html # Different path
      volumes:
        - name: wordpress-persistent-storage
          persistentVolumeClaim:
            claimName: wp-pv-claim
```

and re-apply your config by doing: 
```shell
kubectl apply -f mysql/deployment.yaml -f wordpress/deployment.yaml
```

Your volumes should now be in `Bound` state and your pods `Running`

If your try to look on localhost again, you should be able to re-configure your wordpress, kill it like last time but this time, data
should have been persisted and you should be able to see your config even when the pod restarted !


### Secrets

Now that the app works, we start working on improving our application. First step is to improve security !

You've probably noticed that we're passing a string `toto123` in clear text for both the `wordpress` and `mysql`. 
This creates 2 issues:
- First, you have a password in clear-text in the code
- Second, your password is not shared by applications, if the mysql operator decides to change the DB password he'll 
have to change it everywhere by hand !

To avoid both issues, kubernetes has a `Secret` object that store a list of keys of your choice and that can either be 
mounted or used as an environment variable. We're going to do the latter and replace our toto123 value by a shared secret !

In kubernetes, secrets are by default using a `base64` encoding to make sure the value is not in clear-text (not the most secure way of doing things though)
``

We can create the secret with kubectl and, as we did with our volumes, bind it inside our deployments:

```shell
# Create a secret with a new password
kubectl create secret generic mysql-pass --from-literal=password='MY_PASSWORD_123'
```

```yaml
# mysql/deployment.yqml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              # NEW
              valueFrom:
                secretKeyRef:
                  key: password # matches on of the key inside our secret  
                  name: mysql-pass # matches the metadata.name of our secret
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - image: wordpress:4.8-apache
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: wordpress-mysql
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: mysql-pass
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-persistent-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-persistent-storage
          persistentVolumeClaim:
            claimName: wp-pv-claim

```
To make it work we need to delete the existing mysql `Deployment` and `PVC` and reapply them from 0 (since we changed the password). Then the only thing to do is to re-apply the deployments and the app should still work but now more secure ! 

```shell
 kubectl delete pvc mysql-pv-claim
 kubectl delete deploy wordpress-mysql
 kubectl apply -f mysql/pvc.yaml -f mysql/deployment.yaml -f wordpress/deployment.yaml
 kubectl rollout restart deployment wordpress
```

### Ingress

Congratz our application is now working ! One last thing though, is that it's not accessible from outside of the kubernetes cluster,
this is why you still have to run `port-forward` commands to access your `wordpress` pods. But your users cannot do that !
Let's give them access to our wordpress pod.

This is done by using an `Ingress` object. Ingresses are pods running in your cluster that are forwarding traffic from the outside thanks to a "special" type of `Service`
(either `NodePort` in kind's case or `LoadBalancer` when in the cloud)

Ingresses controllers (the tool that's handling ingresses creation) are `addons` that need to be added to a cluster by the operators, we have to install one first ! The most common one is 
`NGINX` (but [a lot of solutions](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) exists )

```shell
# Install the NGINX ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Now we can create an ingress that will redirect a call to `http://localhost:8080/` to our wordpress pods

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
spec:
  rules:
    - http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: wordpress
              port:
                number: 80

```

You should now be able to access your application from outside the cluster ! 🎉

![congratulations](https://media.giphy.com/media/jJQC2puVZpTMO4vUs0/giphy.gif)


## Other things this tutorial did not show

I voluntarily kept some notions out of this tutorial to keep it dead simple, but there are some must-haves resources to 
know when working with kube:

- `Statefulsets` are used fox`r stateful applications such as databases that have specific needs (boot order, 1 volume per pod, ...)
- `LoadBalancer services` (can be used to expose an application without using and additional ingress controller)
- `ConfigMap`, like secrets they store configuration but configmaps don't cypher the content (eg: useful to store config files)
- `ReplicaSets` are what allowing you to create more pods in parallel, they can be configured through a `Deployment` by specifying a `replicas: <number>`
- `HorizontalPodAutoscaler`: allows your pods to scale up or down depending on their usages
