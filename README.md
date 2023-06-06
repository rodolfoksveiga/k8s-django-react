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

---

In the following sections we'll define the necessary resources to deploy database, backend, and frontend to Kubernetes. We'll store our environment variables in _ConfigMaps_ or _Secrets_, according to the needs. All our persistent data will be managed by a _PersistentVolume (PV)_, which will be attached to a _Pod_ through a _PersistentVolumeClaims (PVC)_. We'll deploy our containers using _Deployments_. A _Deployment_ is an abstract layer wrapping _Pods_, the Kubernetes' smallest deployable units of computing. _Deployments_ regularly check if their _Pods_ are healthy and create or delete _Pods_ whenever demanded. We'll finally expose our _Pods_ internally using _ClusterIP Services_ and externally using _Ingresses_.

That was all we need to prepare. **Let's get started!**

### Stage 1 - Database (PostgreSQL)

In the database section we'll setup the following resources: _Secret_, _PV_, _PVC_, _Deployment_, and _Service_. Note that we don't need to deploy an _Ingress_ for our database _Service_, because we want the database to be exposed only to the backend, which lives within the cluster. For that, a _ClusterIP Service_ is enough and safer than an _Ingress_.

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
                - `containers[0].envFrom.secretRef`
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

### Stage 2 - Backend (Django)

In the second section we'll setup the Django API and its resources, which are: _Secret_, _Deployment_, _Service_, and _Ingress_. This time we need an _Ingress_ to externally expose our API. The _Ingress_ will map our cluster IP to a human-readable URL. Be sure that you went through the steps described at the [requirements section](#requeriments), so you can later access the API from your browser.

1. `~/mayflower/infra/backend/secret.yaml`

    ```yaml
    apiVersion: v1
    data:
      DJANGO_SUPERUSER_EMAIL: YWRtaW5AbWF5Zmxvd2VyLmNvbQ==
      DJANGO_SUPERUSER_PASSWORD: bWF5Zmxvd2Vy
      DJANGO_SUPERUSER_USERNAME: YWRtaW4=
      PSQL_HOST: ZGF0YWJhc2Utc2VydmljZQ==
      PSQL_NAME: cG9zdGdyZXM=
      PSQL_PASSWORD: cGFzc3dvcmQ=
      PSQL_PORT: NTQzMg==
      PSQL_USER: YWRtaW4=
    kind: Secret
    metadata:
      name: backend-secret
    ```

    Here we had to define all the nine environment variables necessary to run our Docker image. These variables are essential to properly setup communication between Django's ORM and the PostgreSQL database. A detailed description of these variables can be found in the [previous blog article](https://blog.mayflower.de/13652-containerizing-django-react-docker.html). To check the value of the variables, you can use the command `echo $VARIABLE_VALUE | base64 --decode`, as mentioned in the [database section](#stage-1-database-postgresql).

---

2. `~/mayflower/infra/backend/deployment.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: backend
      name: backend-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: backend
      template:
        metadata:
          labels:
            app: backend
        spec:
          containers:
            - image: rodolfoksveiga/django-react_django:new
              name: django
              ports:
                - containerPort: 8000
              envFrom:
                - secretRef:
                    name: backend-secret
              volumeMounts:
                - name: backend-logs
                  mountPath: /var/log
          volumes:
            - name: backend-logs
              hostPath:
                path: /var/log
    ```

    - Description of the _Deployment's_ specification:
        - `spec.replicas = 1`
        - `spec.selector.matchLabels = app=backend`
        - `spec.template`
            - `template.metadata.labels = app=backend`
            - `template.spec.containers`
                - `containers[0].image = rodolfoksveiga/django-react_django:latest`
                    - The image was generated using the backend `Dockerfile` created on the [last tutorial](https://blog.mayflower.de/13652-containerizing-django-react-docker.html?cookie-state-change=1683468755671).
                - `containers[0].ports.containerPort = 8000`
                    - The container exposes the default port of the official Django image.
                - `containers[0].envFrom.secretRef`
                    - Inject all the secrets from the _Secret_ store `backend-secret` as environment variables of the container.
                - `containers[0].volumeMounts`
                    - The container mounts just one volume on path `/var/log` (container), where it writes Django API logs.
            - `template.volumes`
                - Map the volume mounted on the container to the `/var/log` (host machine).

---

3. `~/mayflower/infra/backend/service.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: backend-service
    spec:
      type: ClusterIP
      selector:
        app: backend
      ports:
        - name: 8000-8000
          port: 8000
          targetPort: 8000
    ```

    - Description of the _Service's_ specification:
        - `type = ClusterIP`
        - `selector = app=backend`
        - `ports`
            - `ports[0].port = 8000` and `ports[0].targetPort === 8000`

---

4. `~/mayflower/infra/backend/ingress.yaml`

    As we already discussed, a _Pod_ exposed by a _Service_ can only be reached from inside the cluster. Since we want to call the API from the frontend, which will run on our browser, we also need to deploy an _Ingress_.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: backend-ingress
    spec:
      rules:
        - host: api.mayflower.de
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: backend-service
                    port:
                      number: 8000
    ```

    - Description of the _Ingress's_ specification:
        - `rules`: list of hosts to expose routes associated to services
            - `rules[0].host = api.mayflower.com`
                - The mapped URL of our Django API. This URL matches the URL added to our hosts' file on the [foundation section](#stage-0-foundation).
            - `rules.http.paths`: list of routes to expose spicific service ports
                - `paths[0].path = /`
                    - The root path of the host is mapped the root route endpoint of Django API.
                - `paths[0].pathType = Prefix`
                    - The value `Prefix` of the `pathType` key means that the children routes of our _Service_ will also be mapped to this host. For example, we'll be able to access the backend endpoint `/admin` on `https://api.mayflower.com/admin` as well as the endpoint `/students` on `https://api.mayflower.com/students`.
                - `paths[0].backend.service.name = backend-service`
                    - The _Ingress_ will look for a _Service_ called `backend-service`.
                - `paths[0].backend.service.port.number = 8000`
                    - We mapped the _Service's_ port `8000` to the root route.

---

To deploy our backend in we can execute `kubectl create -f ~/mayflower/infra/backend`, and "voilà", our Django API is accessible through the URL `https://api.mayflower.com`. **Isn't it cool!?**

Play around with your API, add some students to our database, so you can see it later on our frontend URL.

![Django Admin](https://github.com/rodolfoksveiga/k8s-django-react/blob/main/imgs/django_admin.png)
![Student's Endpoint](https://github.com/rodolfoksveiga/k8s-django-react/blob/main/imgs/students_endpoint.png)

### Stage 3 - Frontend (React APP)

Last but not least, we'll deploy the React APP using the following resources: _ConfigMap_, _Deployment_, _Service_, and _Ingress_. Note that this time we opted for a _ConfigMap_ instead of a _Secret_, because the only variable we will store in it isn't that important and can be exposed to other people.

1. `~/mayflower/infra/frontend/config-map.yaml`

    ```yaml
    apiVersion: v1
    data:
      REACT_APP_API_URL: https://api.mayflower.de
    kind: ConfigMap
    metadata:
      name: frontend-config-map
    ```

    In this _ConfigMap_ manifest we set just one environment variable called `REACT_APP_API_URL`. This variable is used to print the Django Admin URL as a link in the frontend.

---

2. `~/mayflower/infra/frontend/deployment.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: frontend
      name: frontend-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: frontend
      template:
        metadata:
          labels:
            app: frontend
        spec:
          containers:
            - image: rodolfoksveiga/django-react_react:latest
              name: react
              envFrom:
                - configMapRef:
                    name: frontend-config-map
    ```

    - Description of the _Deployment's_ specification:
        - `spec.replicas = 1`
        - `spec.selector.matchLabels = app=frontend`
        - `spec.template`
            - `template.metadata.labels = app=frontend`
            - `template.spec.containers`
                - `containers[0].image = rodolfoksveiga/django-react_django:latest`
                    - The image was generated using the frontend `Dockerfile` created on the [previous tutorial](https://blog.mayflower.de/13652-containerizing-django-react-docker.html?cookie-state-change=1683468755671).
                - `containers[0].ports.containerPort = 3000`
                    - The container exposes the default port of the official React image.
                - `containers[0].envFrom.configMapRef`
                    - Inject all the configurations from the _ConfigMap_ store `frontend-config-map` as environment variables of the container.

---

4. `~/mayflower/infra/frontend/service.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend-service
    spec:
      type: ClusterIP
      selector:
        app: frontend
      ports:
        - name: 3000-3000
          port: 3000
          targetPort: 3000
    ```

    - Description of the _Service's_ specification:
        - `type = ClusterIP`
        - `selector = app=frontend`
        - `ports`
            - `ports[0].port = 3000` and `ports[0].targetPort === 3000`

---

5. `~/mayflower/infra/frontend/ingress.yaml`

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress
    spec:
      rules:
        - host: app.mayflower.de
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: frontend-service
                    port:
                      number: 3000
    ```

    - Description of the _Ingress's_ specification:
        - `rules`
            - `rules[0].host = app.mayflower.com`
            - `rules.http.paths`
                - `paths[0].path = /`
                - `paths[0].pathType = Prefix`
                - `paths[0].backend.service.name = frontend-service`
                - `paths[0].backend.service.port.number = 3000`

---

Finally we can execute `kubectl create -f ~/mayflower/infra/frontend`, and shortly our React APP will be available on our browser through the URL `https://app.mayflower.com`. If you can see in the frontend the data you have created before in backend URL, it means you did everthing right and the services are properly connected to each other. The backend manage the database and the frontend prints the data gathered from the backend. *It wasn't that hard, right!?*

![React with data](https://github.com/rodolfoksveiga/k8s-django-react/blob/main/imgs/react.png)

**That was quick, but it's indeed everything you need to get started with Kubernetes. Now you can use this Kubernetes cluster as you will. You can play around with it, extend it, and perhaps use it as a baseline to create your future customer's application.**

## Conclusion

Following through this tutorial, you learned how to serve your PostgreSQL, Django, and React containers from a Minikube Kubernetes cluster, which mimics a real cloud server. Since you already learned in the [previous tutorial](https://blog.mayflower.de/13652-containerizing-django-react-docker.html) how to package application in containers, this was your second step into the cloud - and it was huge one!

You can reproduce many of the concepts you've learned here in a real world application, but there were still some limitations to consider before you publicly deploy your containers to Kubernetes. Among the limitations, it's worth it to point out once more that the _Secrets_ we defined are only "base64" encoded, therefore everyone can decode it using a simple a command. To really protect your data you must encrypt it at rest. Luckly Kubernetes has a [built-in feature](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) to achieve that. Check it out and implement it on your Kubernetes' cluster.

Well, you have now the whole power of Kubernetes at the tip of your fingers! You can seamlessly deploy new resources, expose them internally to the services you've already deployed or to external services, scale your containers using _Deployments_, so the demand of your application is optimally fulfilled, and much more... Kubernetes offers you the possibility to easily design the infrastructure to match your needs, so it's opportunity to go further and think your infrastructure your way. Use the concepts you've just learned, but don't limit yourself to them.
