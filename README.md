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
