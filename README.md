[![Build Status](https://travis-ci.org/ballerina-guides/ballerina-gcp-deployment.svg?branch=master)](https://travis-ci.org/ballerina-guides/ballerina-gcp-deployment)

# Ballerina Deployment with Google Cloud Platoform Kubernetes Engine

[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/) is a powerful cluster manager and orchestration system for running your Docker containers

> In this guide you will learn about building a Ballerina service and deploying it on Google Kubernetes Enginer (GKE).

The following are the sections available in this guide.

- [What you'll build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
- [Deployment](#deployment)
- [Testing](#testing)

## What you’ll build

In this guide, you will build a simple Ballerina Hello World service, and you will deploy this service on GKE.

## Compatibility

| Ballerina Language Version
| --------------------------
| 0.990.0

## Prerequisites

- [Ballerina Distribution](https://ballerina.io/learn/getting-started/)
- A Text Editor or an IDE
- [Docker](https://docs.docker.com/engine/installation/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Docker Hub Account](https://hub.docker.com/)
- [Google Cloud Platform account](https://cloud.google.com/)

### Optional requirements

- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina))

## Implementation

As the first step, you can build a Ballerina service that gives an UUID as the output. You can simply create a file `hello_world_service.bal` and add the following content to the service code.

```ballerina
import ballerina/http;
import ballerina/log;
import ballerinax/kubernetes;

listener http:Listener httpListener = new(9090);

// By default, Ballerina exposes a service via HTTP/1.1.
service hello on httpListener {

    // Invoke all resources with arguments of server connector and request.
    resource function sayHello(http:Caller caller, http:Request req) {
        http:Response res = new;

        // Use a util method to set a string payload.
        res.setPayload("Hello, World!");

        // Send the response back to the caller.
        var result = caller->respond(res);
        if (result is error) {
            log:printError("Error sending response", err = result);
        }
    }
}
```

Now you can add the Kubernetes annotations that are required to generate the Kubernetes deployment artifacts.

```ballerina
import ballerina/http;
import ballerina/log;
import ballerinax/kubernetes;

@kubernetes:Service {
    name:"hello-world",
    serviceType:"LoadBalancer",
    port:80
}
listener http:Listener httpListener = new(9090);

@kubernetes:Deployment {
    enableLiveness:true,
    image:"<username>/hello_world_service:latest",
    push:true,
    username:"<username>",
    password:"<password"
}
// By default, Ballerina exposes a service via HTTP/1.1.
service hello on httpListener {

    // Invoke all resources with arguments of server connector and request.
    resource function sayHello(http:Caller caller, http:Request req) {
        http:Response res = new;

        // Use a util method to set a string payload.
        res.setPayload("Hello, World!");

        // Send the response back to the caller.
        var result = caller->respond(res);
        if (result is error) {
            log:printError("Error sending response", err = result);
        }
    }
}

```
We will be building a Docker image here and publishing it to Docker Hub. This is required, since we cannot simply have the Docker image in the local registry, and run the Kubernetes applicates in GKE, where it needs to have access to the Docker image in a globally accessible location. For this, an image name should be given in the format $username/$image_name in the "image" property, and "username" and "password" properties needs to contain the Docker Hub account username and password respectively. The property "push" is set to "true" to signal the build process to push the build Docker image to Docker Hub.

You can build the Ballerina service using `$ ballerina build hello_world_service.bal`. You should be able to see the following output.

```bash
$ ballerina build src/hello_world_service.bal
Compiling source
    hello_world_service.bal
Generating executable
    hello_world_service.balx
        @kubernetes:Service                      - complete 1/1
        @kubernetes:Deployment                   - complete 1/1
        @kubernetes:Docker                       - complete 2/3
        @kubernetes:Docker                       - complete 3/3
        @kubernetes:Helm                         - complete 1/1

        Run the following command to deploy the Kubernetes artifacts:
        kubectl apply -f /home/manurip/Documents/Work/Repositories/ballerina-gcp-deployment/kubernetes/

        Run the following command to install the application using Helm:
        helm install --name hello-world-service-deployment /home/manurip/Documents/Work/Repositories/ballerina-gcp-deployment/kubernetes/hello-world-service-deployment
```

After the build is complete, the Docker image is created and pushed to Docker Hub. The Kubernetes deployment artifacts are generated as well.

## Deployment

- Configuring GKE environment

Before deploying the service on GKE, you will need to setup the GKE environment to create the Kubernetes cluster and deploy an application.

Let's start by installing the Google Cloud SDK in our local machine. Please refer to [Google Cloud SDK Installation](https://cloud.google.com/sdk/install) in finding the steps for the installation.

Next step is gcloud configuration and creating a Google Cloud Platform project.
You can begin with `gcloud init` command.

```bash
>$ gcloud init
Welcome! This command will take you through the configuration of gcloud.

Your current configuration has been set to: [default]

You can skip diagnostics next time by using the following flag:
  gcloud init --skip-diagnostics

Network diagnostic detects and fixes local network connection issues.
Checking network connection...done.
Reachability Check passed.
Network diagnostic passed (1/1 checks passed).

You must log in to continue. Would you like to log in (Y/n)?

Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/auth?redirect_uri=http%3A%2F%2Flocalhost%3A8085%2F&prompt=select_account&response_type=code&client_id=32555940559.apps.googleusercontent.com&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fappengine.admin+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Faccounts.reauth&access_type=offline

```

Once you click on the provided link, you will be prompted to login to your Google account and authorize.

Once authentication flow completes, you will be prompted to create a new project or choose an existing project as the
current project. Here, we are creating a project named "BallerinaDemo"

```bash
This account has no projects.

Would you like to create one? (Y/n)?

Enter a Project ID. Note that a Project ID CANNOT be changed later.
Project IDs must be 6-30 characters (lowercase ASCII, digits, or
hyphens) in length and start with a lowercase letter. BallerinaDemo
Your current project has been set to: [BallerinaDemo].

Not setting default zone/region (this feature makes it easier to use
[gcloud compute] by setting an appropriate default value for the
--zone and --region flag).
See https://cloud.google.com/compute/docs/gcloud-compute section on how to set
default compute region and zone manually. If you would like [gcloud init] to be
able to do this for you the next time you run it, make sure the
Compute Engine API is enabled for your project on the
https://console.developers.google.com/apis page.

Created a default .boto configuration file at [/home/kasun/.boto]. See this file and
[https://cloud.google.com/storage/docs/gsutil/commands/config] for more
information about configuring Google Cloud Storage.
Your Google Cloud SDK is configured and ready to use!
```

With the following command you can list the projects.

```bash
$ gcloud projects list
PROJECT_ID                NAME              PROJECT_NUMBER
ballerinademo-225007      BallerinaDemo     1036334079773
```

- Create the Kubernetes cluster

Next step is creating a kubernetes cluster in the project we just created.
With a command similar to below, you can create a cluster with minimal resources.

```bash
gcloud container clusters create ballerina_demo_cluster --zone us-central1 --machine-type g1-small --disk-size 30GB --max-nodes-per-pool 1
```

With the following command you can verify the cluster is running.

```bash
$ gcloud container clusters list

ballerina-demo-cluster  us-central1  1.9.7-gke.11    35.239.235.173  g1-small      1.9.7-gke.11  3          RUNNING
```

Also, with `kubectl get nodes` commands you can verify the connection to the cluster. In next steps we'll be using
`kubectl` in order to create our kubernetes deployment.
Please note that when you create a cluster using gcloud container clusters create, an entry is automatically added to the kubeconfig in your environment, and the current context changes to that cluster. Therefore, we don't have to do any manual
configuration to make it possible for `kubectl` to talk to the cluster.

```bash
$ kubectl get nodes
NAME                                                  STATUS    ROLES     AGE       VERSION
gke-ballerina-demo-clust-default-pool-70ca2fd4-jv97   Ready     <none>    4h        v1.9.7-gke.11
gke-ballerina-demo-clust-default-pool-77c556be-x42n   Ready     <none>    4h        v1.9.7-gke.11
gke-ballerina-demo-clust-default-pool-8a9f3889-l6ks   Ready     <none>    4h        v1.9.7-gke.11
```


- Deploying the Ballerina service in GKE

Since the Kubernetes artifacts were automatically built in the earlier Ballerina application build, we simply have to run the following command to deploy the Ballerina service in GKE:

```bash
$ kubectl apply -f /home/manurip/Documents/Work/Repositories/ballerina-gcp-deployment/kubernetes/
service "hello-world" created
deployment.extensions "hello-world-service-deployment" created
```

When you list the pods in Kubernetes, it shows that the current application was deployed successfully.

```bash
$ kubectl get pods
NAME                                                     READY     STATUS    RESTARTS   AGE
hello-world-service-deployment-6ccccb4dd8-wd78n          1/1       Running   0          3h
```

After verifying that the pod is alive, we can list the services to see the status of the Kubernetes service created to represent our Ballerina service:

```bash
$ kubectl get svc
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
hello-world      LoadBalancer   10.19.255.12    104.154.64.112   80:30736/TCP   3h
kubernetes       ClusterIP      10.19.240.1     <none>           443/TCP        4h
```

## Testing

You've just deployed your first Ballerina service in GKE!. You can test out the service using a web browser with the URL [http://$EXTERNAL-IP/hello/sayHello](http://$EXTERNAL-IP/hello/sayHello), or by running the following cURL command:

```bash
$ curl http://$EXTERNAL-IP/hello/sayHello
Hello, World!
```


