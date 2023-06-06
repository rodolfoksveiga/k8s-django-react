# Deploying Django and React to Kubernetes

In a [previous blog article](https://blog.mayflower.de/13652-containerizing-django-react-docker.html) we explored the capabalities of Docker and created containers for a Django and React web application, regarding a development environment. While Docker is an excellent tool for packaging containerized applications, it might not be enough for deploying applications in a production environment. Managing multiple containers across multiple servers, load balancing, and scaling the application is out of Docker's scope and can quickly become a complex, time-consuming, and error-proning task. This is where container orchestration comes into play.

Container orchestration is a crucial tool for managing and deploying containerized applications at scale. With the increasing adoption of containerization, the need for orchestration has become critical to ensure a efficient management of infrastructures. Container orchestration frameworks enable development and operation teams to automate deployment and management of containerized applications. This technology also promotes high availability, resilience, and fault tolerance, making it a valuable tool to implement Continuous Integration/Containuous Deployment (CI/CD).

While there are many container orchestration tools available, Kubernetes has emerged as the most popular framework. Among many reasons why Kubernetes became so popular, there are its scalability, flexibility, and active open-source community. Kubernetes offers a comprehensive solution for orchestrating from small to large-scale applications, making it a viable choice for diverse enterprises. Besides that, Kubernetes provides built-in security features and robust monitoring capabilities, making it secure and reliable.

Since Kubernetes is designed to run applications in clusters, developers may benefit of deploying the Kubernetes cluster locally throughout the infrastructure development stage. Minikube is a tool that allows developers to create a local Kubernetes cluster, which mimics a production environment. Using Minikube they can test and debug their applications locally and ensure that the application will run correctly when deployed to a production cluster.

The goal of this blog article is to demonstrate how to deploy the previously developed Django and React application to a Kubernetes cluster using Minikube. By following this tutorial, you'll learn how to create a local Kubernetes cluster using Minikube and deploy the pre-built containerized applications to it. This tutorial covers topics such as creating Kubernetes _PersistentVolumes_, _Deployments_, _Services_, and much more. Stay tunned and take the best out of it!

In order to structure our learning, the tutorial is split into four stages:

- _Stage 0_: Foundation - [stage0-base](https://github.com/rodolfoksveiga/k8s-django-react/tree/stage0-base)
  - This branch is a copy of the [main branch](https://github.com/mayflower/k8s-django-react) of the previous tutorial
- _Stage 1_: PostgreSQL Database - [stage1-psql](https://github.com/rodolfoksveiga/k8s-django-react/tree/stage1-psql)
- _Stage 2_: Django API - [stage2-django](https://github.com/rodolfoksveiga/k8s-django-react/tree/stage2-django)
- _Stage 3_: React APP - [stage3-react](https://github.com/rodolfoksveiga/k8s-django-react/tree/stage3-react)

#### Requeriments

A basic understanding of Kubernetes and the following resources is required: _ConfigMap_, _Secret_, _Pod_, _Deployment_, _Service_, and _Ingress_. In case you haven't had contact with these resources, take your time to read the [Kubernetes documentation](https://kubernetes.io/docs/home/). Don't rush, the blog post will still be available for you later!

To follow the approach proposed in this tutorial, we must have the following packages installed in our system:

- Git
  - `git 2.40`
- Docker
  - `docker 23.0`
- Kubernetes
  - `kubectl 1.27`
  - `minikube 1.29`
    - Be sure that your Minikube cluster is up and running. To achieve that, execute `minikube start`. Check the status of your cluster with `minikube status`. You also need to deploy an _Ingress_ controller running `minikube addons enable ingress`. The status of your Minikube addons can be verified with `minikube addons list`. For debugging Minikube's setup, refer to their detailed [Documentation](https://minikube.sigs.k8s.io/docs/)

> Remember that you can still use newer version of the referred packages, but be aware that sometimes you may reproduce different outputs.

**That's all you need..!** After installing these packages and configuring Minikube, you're good to go!

### Stage 0 - Foundation

The first step of our Kubernetes journey is to download the data related to the containers developed in the [previous blog article](https://blog.mayflower.de/13652-containerizing-django-react-docker.html). Since the `main` branch of the previous tutorial was forked into the `stage0-base` branch of the current tutorial, we simply have to clone the [current tutorial's Git repository](https://github.com/rodolfoksveiga/k8s-django-react) into a local machine and switch the branch.

---

1. Create the local repository `~/mayflower`, clone the Git repository into it, and switch to the branch `stage0-base`.

    ```bash
    mkdir ~/mayflower
    cd ~/mayflower
    git clone https://github.com/rodolfoksveiga/k8s-django-react.git .
    git switch stage0-base
    ```

    At this point we should be able to run `docker-compose up` from `~/mayflower`. After successfully running the containers, we can open our browser and navigate to http://localhost:8000 (Django API) or http://localhost:3000 (React APP). If you want to have a better idea of the project's structure, feel free to investigate the files within this branch and test the application. By doing it so, you'll feel more confident in the comming sections of this tutorial.

    > If it's all still a bit confusing for you, I totally recommend you to step back and follow the [previous blog article](https://blog.mayflower.de/13652-containerizing-django-react-docker.html) through.

---

2. Add the hosts `api.mayflower.com` and `app.mayflower.com` to your hosts' file.

    To be able to access our application using the browser, we have to map Minikube's IP address to the target URL. So first we have to find out our Minikube's IP. To figure that out, we can execute `minikube ip` and check the command's output. Next we have to add two new line to our hosts' file containing our Minikube IP and the mapped hosts. To do that we can follow the example below:

    ```bash
    $MINIKUBE_IP_ADDRESS api.mayflower.de
    $MINIKUBE_IP_ADDRESS app.mayflower.de
    ```

    In this example we just have to substitute `$MINIKUBE_IP_ADDRESS` with the IP address printed out as we ran the command `minikube ip`.

    > The host file location depends on your operational system. At Linux, MacOS, and Windows the file can be found respectively on: `/etc/hosts`, `/private/etc/hosts`, and `c:\Windows\System32\Drivers\etc\hosts`.

In the following sections we'll define the necessary resources to deploy database, backend, and frontend to Kubernetes. We'll store our environment variables in _ConfigMaps_ or _Secrets_, according to the needs. All our persistent data will be managed by a _PersistentVolume (PV)_, which will be attached to a _Pod_ through a _PersistentVolumeClaims (PVC)_. We'll deploy our containers using _Deployments_. A _Deployment_ is an abstract layer wrapping _Pods_, the Kubernetes' smallest deployable units of computing. _Deployments_ regularly check if their _Pods_ are healthy and create or delete _Pods_ whenever demanded. We'll finally expose our _Pods_ internally using _ClusterIP Services_ and externally using _Ingresses_.

That was all we need to prepare. **Let's get started!**

### Stage 1 - Database (PostgreSQL)

In the database section we'll setup the following resources: _Secret_, _PV_, _PVC_, _Deployment_, and _Service_. Note that we don't need to deploy an _Ingress_ for our database _Service_, because we want the database to be exposed only to the backend, which lives within the cluster. For that, a _ClusterIP Service_ is enough and safer than an _Ingress_.

---

1. `~/mayflower/infra/database/secret.yaml`

    Since the environment variables used to deploy the database are sensitive and we don't want to expose their values to the world, we will encode them and store them in a _Secret_.

    ```yaml
    apiVersion: v1
    data:
      POSTGRES_DB: cG9zdGdyZXM=
      POSTGRES_PASSWORD: cGFzc3dvcmQ=
      POSTGRES_USER: YWRtaW4=
    kind: Secret
    metadata:
      name: database-secret
    ```

    As discussed in the [previous blog article](https://blog.mayflower.de/13652-containerizing-django-react-docker.html), we needed to define three environment variables to properly start our database container: `POSTGRES_DB`, `POSTGRES_USER`, and `POSTGRES_PASSWORD`. These variables were encoded using the "base64" schema before they were stored in the _Secret_. To get the decoded value of a variable, run `echo $VARIABLE_VALUE | base64 --decode`. For example the command `echo cG9zdGdyZXM= | base64 --decode` should print out the value of `POSTGRES_DB`, which happens to be `postgres`.

---

2. `~/mayflower/infra/database/persistent-volume.yaml`

    The _PV_ is a resource that assures, as the name suggests, that our data persists if the attached _Pod_ occasionally dies or isn't `Running` state.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: database-persistent-volume
    spec:
      capacity:
        storage: 200Mi
      hostPath:
        path: /data
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
    ```

    - Description of the _PV's_ specification:
        - `spec.capacity.storage = 200Mi`
            - The maximum storage capacity of `200Mi` assures that only _PVCs_ requiring `200Mi` or less will be able to attach to this _PV_.
        - `spec.hostPath.path = /data`
            - The `hostPath` indicates where the data will be stored in the host machine.
        - `spec.accessModes`: list containing values that describe under which conditions _Pods_ can connect to the _PV_
            - `accessModes[0] = ReadWriteOnce`
                - The value `ReadWriteOnce` restricts the _PV_ to be mounted by a single _Node_. The _PV_ can still be mounted by multiple _Pods_, as long as they live in the same _Node_.
        - `spec.persistentVolumeReclaimPolicy = Retain`
            - The value `Retain` assures that if the _Pod_ dies and the _PVC_ is detached from the _PV_, the _PV_ will persist and wait for a new instance of the _Pod_ to spin up and reattach to the _PVC_.

---

3. `~/mayflower/infra/database/persistent-volume-claim.yaml`

    A _PV_ can only be requested and mounted to a _Pod_ through a _PVC_. The _PVC_ describes the requirements that the _PV_ must fulfil in order to attach to the _PVC_. Remember that if the _PVC_ isn't referenced by a _Pod_, it won't attach to any _PV_.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: database-persistent-volume-claim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 200Mi
    ```

    All the requirements defined for this _PVC_ match the values previously described by our _PV_ manifest. By doing it, we guarantee that only one _Pod_ will store data in and consume data from the pre-defined _PV_, since the _Pod_ consumes all the storage available on the _PV_.

    > Note that the _PVC's_ access modes must also match the access modes defined in the _PV_. If they don't match the _PVC_ won't be attached to the _PV_ and naturally the volume won't mount to any _Pod_ referencing this _PVC_.

---

4. `~/mayflower/infra/database/deployment.yaml`

    A Deployment controls the lifecycle of the _Pods_ with labels matching the _Deployment's_ selector labels. We'll use a _Deployment_ to specify all the configurations of the database _Pod_ and its container.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: database-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: database
      template:
        metadata:
          labels:
            app: database
        spec:
          containers:
            - image: postgres:14.1-alpine
              name: database
              ports:
                - containerPort: 5432
              envFrom:
                - secretRef:
                    name: database-secret
              volumeMounts:
                - name: storage
                  mountPath: /var/lib/postgresql/data
          volumes:
            - name: storage
              persistentVolumeClaim:
                claimName: database-persistent-volume-claim
    ```

   - Description of the _Deployment's_ specification:
       - `spec.replicas = 1`
           - The _Deployment_ tries to always keep one _Pod_ in `Running` state.
       - `spec.selector.matchLabels = app=database`
           - The _Deployment_ is allowed to manage the _Pods_ with label `app` equals to `database`. Any other _Pod_ with such label key and value will also be taken into consideration by this _Deployment_.
       - `spec.template`: configuration of the _Pod_ to be created by the _Deployment_ whenever needed
           - `template.metadata.labels = app=database`
               - Set the label key `app` equals to `database` to this _Pod_, so the _Deployment_ can use it for future state management of its underlying _Pods_.
           - `template.spec.containers`: list of containers deployed in the _Pod_
               - `containers[0].image = postgres:14.1-alpine`
                   - The image used to deploy the _Pods_ is the official PostgreSQL image.
               - `containers[0].ports.containerPort = 5432`
                   - The container exposes the default port of the official PostgreSQL image.
               - `containers[0].envFrom`
                   - Inject all the secrets from the _Secret_ store `database-secret` as environment variables of the container.
               - `containers[0].volumeMounts`
                   - The container mounts just one volume on path `/var/lib/postgresql/data` (container), where it stores the data of our database.
           - `template.volumes`
               - Map the volume mounted on the container to the _PVC_ `database-persistent-volume-claim` created beforehand. Note that the _PVC_ will look for a _PV_ matching its requirements.

---

5. `~/mayflower/infra/database/service.yaml`

    Finally we want to expose our database _Deployment_ to the backend _Service_ we'll deploy next. Since the database and the backend live in the same cluster, we'll set this communication using a _ClusterIP Service_.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: database-service
    spec:
      type: ClusterIP
      selector:
        app: database
      ports:
        - name: 5432-5432
          port: 5432
          targetPort: 5432
    ```

- Description of the _Service's_ specification:
    - `type = ClusterIP`
    - `selector = app=database`
        - As well as the _Deployment_, the _Service_ use the `selector` to match the _Pods_ with the same labels. In this case, we want to match the database _Pod_, which has label key `app` equals to `database`. 
    - `ports`: list of ports that will be exposed by the cluster
        - `ports[0].port = 5432` and `ports[0].targetPort === 5432`
            - Port `5432` of the container (target) was mapped to the same port on the cluster.

    After setting it all up we can execute `kubectl create -f ~/mayflower/infra/database`. If everything went right, when we run `kubectl get all` we'll see the resources we just deployed. For example, if we run `kubectl get deploys` we should see a list containing our `database-deployment`.

> You probably have learned a lot in this section and what we'll do next will be pretty similar. To avoid repeting concepts, we'll keep the resources' description quite shorter in the comming sections. If you need, come back to this section to recap some stuff that may still be confusing.
