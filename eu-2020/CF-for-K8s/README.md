## Introduction

In this hands-on lab, attendees will learn how to deploy Cloud Foundry on a Kubernetes (cf-for-k8s) project to a Kubernetes cluster, push source code apps, deep dive into cf push workflow, inspect various cluster resources created by cf-for-k8s. 

### Target Audience

This lab is targeted towards the audience who would like to use Cloud Foundry for packaging and deploying cloud native applications with Kubernetes as the underlying infrastructure.

### Learning Objectives

You will be performing the following tasks in this lab :

- Install cf-for-k8s
- Inspect app workloads, routing, and logging
- Overlays with `ytt`
- Delete cf-for-k8s

**WE RECOMMEND YOU READ EACH STEP IN ITS ENTIRETY, SO THAT YOU HAVE CLEAR UNDERSTANDING OF HOW CF-FOR-K8S WORKS**

### Prerequisites

Students must have basic knowledge of Cloud Foundry and Kubernetes.

### Google Cloud Shell window size
We recommend you increase the size of the cloudshell window to the largest size available since this lab will exclusively work in console.

## Setup Environment

Let's set up your environment by running the following command in your console. 

    eval "$(./setup-env.sh)"

#### What it's doing

- The script will install cf-cli, Carvel tools (`ytt`, `kapp`), plus a few other helpful tools for the lab session. 

- The script also setups up your `kubeconfig` by connecting to an existing k8s cluster. 

- The script installs bosh CLI to generate self-signed certificates and credentials. It is a matter of convenience and in the future, it will be replaced by tools such as CredHub.

### Verify CLIs exists

Running the following command will print versions for each CLI. 

    cf version
    ytt version
    kapp version

### Verify connection with K8s cluster

Running the following command will list the default namespaces.

    kubectl get namespaces

Your output should display 4 namespaces that come with most K8s clusters.
  
## Clone project

Clone the cf-for-k8s project from the source repository.

    git clone https://github.com/cloudfoundry/cf-for-k8s.git
    cd cf-for-k8s
    git checkout v1.0.0-rc.2

## Create a values file

Let's create a values file using `generate-values.sh`. Copy and paste the following command in your console.

    ./hack/generate-values.sh -d $CF_DOMAIN > cf-values.yml
    cat<<EOF >> cf-values.yml
    load_balancer:
      static_ip: "$(host api.${CF_DOMAIN} | awk '{print $NF}')"
    EOF

#### What it's doing

- The `generate-values.sh` script generates the necessary self-signed certificates and credentials (via bosh cli). 
- Also, it sets the Kubernetes loadbalancer IP to a reserved IP that comes with the lab session. It reduces the overall time to install the cluster.


### Peek into values

Lets quickly look into the `cf-values.yml` by running the following command,

    head cf-values.yml

- Notice the `system_domain` and `apps_domain` properties are the same as `$CF_DOMAIN` variable.
- `cf_admin_password` password was generated by bosh cli. 
- `system_certificate` is a self-signed certificate generated via bosh cli. As mentioned above, it will be replaced by tools such as CredHub in the future.

## Setup docker registry
Before we can push source code apps, we need to setup a docker registry. We pre-created a `labs-values.yml` file for you to use in the lab.

    cat ../labs-values.yml >> cf-values.yml

#### What it's doing

It appends the docker registry configuration to your `cf-values.yml`. You can check it out by running `tail cf-values.yml` to see the docker registry configuration.

## Render with ytt
Run the following command to create the final K8s config file `cf-for-k8s-rendered.yml`,

    ytt -f config -f cf-values.yml > cf-for-k8s-rendered.yml

#### What it's doing

`ytt` consumes all the template k8s files under folder `config` (and also any templates under subdirectories) and uses `cf-values.yml` to generate the final K8s config file. You can peek into the file by running `head -n 20 cf-for-k8s-rendered.yml`. You will notice it's just a regular K8s yml file.

## Install with kapp

Let's now install cf-for-k8s by running the following command,

    kapp deploy -a cf \
        -f cf-for-k8s-rendered.yml -y
 
#### What it's doing

`kapp` will delegate the creation of k8s resources to the native `kubectl` API and it will watch on those resources until they are available. The `kapp deploy` should take about ~8-10 minutes to finish.

## Connect to CF CLI
Verify that you can connect to the foundation using the CF CLI,

    cf api --skip-ssl-validation https://api.$CF_DOMAIN

If successful, the output should print the API version and endpoint. In the next step, we will log in to CF.

## Login to CF
Login using the admin credentials in `cf-values.yml`

    cf auth admin $(yq -r .cf_admin_password cf-values.yml)
 
`yq` returns the value for `cf_admin_password` from `cf-values.yml` and feeds into `cf auth` command. You're now ready to create orgs and spaces.

## Create org/space for your app
Next, create orgs and spaces

    cf create-org labs-org
    cf create-space -o labs-org labs-space
    cf target -o labs-org -s labs-space

We are so close. In the next step, we will push an app.

## Deploy the source code app

We will push two instances of the app (`-i 2`) and we will give a path (`- p`) to an existing app `test-node-app`. The app, when deployed successfully, will return `Hello World`.

Run the following command and pay attention to the logs it prints.

    cf push node-app -i 2 -p ./tests/smoke/assets/test-node-app/

### Watch the logs

Notice how your source code is transformed into an actual running app. At a high level, the steps worth noting are,

1. push app source code from directory to the app registry
1. run through detection to identify the app language
1. pull the necessary base images for the given language (in this case `Node Engine Buildpack`)
1. build an **OCI compliant** app image with the above base image
1. Push the app image to the registry
1. create an HTTP route to the app
1. schedule the app with the given # of instances
1. report the app status and any metrics

In the next step, we will verify if the app is reachable via curl.

## Curl the app
The final moment we have been waiting. Run the following command. It should print `Hello world`!

    curl -k https://node-app.apps.$CF_DOMAIN

Congratulations!! You have done it. 

### App information
It's worth looking into the app status and logs. 

### App status

    cf app node-app

The command will print node-app with `running` status and app metrics if available. 

### App logs

    cf logs --recent node-app

The command will print app related logs. There is a known issue with app logs where it collects and prints `Envory` related logs. Future versions will only show app related events.

### App routes

    cf routes

The above command will print all routes in the org.

In the next section, we will go on a journey to understand how an app is deployed, how it's built, system components involved in creating the app and much more. Let's start!!!

## Inspecting App pods
Let's look at our apps from Kubernetes point of view. The app is deployed as a Deployment pod to `cf-workloads` namespace (more on namespaces in the next sections). Run the command to see the pods.

    kubectl get pods -n cf-workloads

Notice that there are two pods. The # of pods corresponds to the # of app instances that you specified above (`-i 2`) during `cf push`.

## Inspecting App route services
For every app route, cf-for-k8s creates route CRD and a Kubernetes native `Service` that serves the app instances. Let's see the services of the app.

    kubectl get svc -n cf-workloads

Let's look at the individual service. **Pick the service guid from the above and replace `<service guid>` below.**

    kubectl describe svc/<service guid> -n cf-workloads

Notice the `Annotations` and `route-fqdn` value. It is the same URL that you used to access the app above.

### Create another route for the same app
We will create a route `node-another-app` using `cf map-route` command.

    cf map-route node-app --hostname node-another-app apps.$CF_DOMAIN

You should now see 2 services

    kubectl get svc -n cf-workloads

Pick the `Service` that was recently created (HINT: Look at the **Age** column). **Replace the `<service guid>` with the service guid in the command below**. Inspect its output `Annotations.route-fqdn` property. Notice the URL you just created.

    kubectl describe svc/<name of the service guid from the above> -n cf-workloads |  grep Annotations

### Route Controller
The `route-controller` is responsible for creating the `route` CRD, which in turn creates the actual k8s `Service`

    kubectl get routes -n cf-workloads

Notice two routes under the `cf-workloads` namespace.

To summarize, for every app route, there exists a `route` CRD that maps to a Kubernetes `service`.

## Inspect Ingress gateway
To access the app from an external network, you still need a mechanism to connect the incoming traffic to the right app `service`. Ingress gateway fills the role of a traffic director.
    
    kubectl get gateway -n cf-system

The gateway is responsible for directing the traffic to the apps or CF API. The domain you setup above is fed into the gateway as the allowed hostnames.

    kubectl describe gateway/ingressgateway -n cf-system

## Building the image
As you noticed, cf-for-k8s builds an **OCI compliant image** from your source code. Let's inspect the pod responsible for creating the app image.

    kubectl get pods -n cf-workloads-staging
    
There should be a single build pod with the status `Completed` in the `cf-workloads-staging` namespace. This pod was responsible for creating the app image from source and then pushing to the docker registry (which we configured in `labs-values.yml` file when we installed cf-for-k8s).

**Replace the `<pod-guid>` with the pod `guid` from the above output**.

    kubectl describe pod/<pod-guid> -n cf-workloads-staging | grep Events -A 20

Notice the events that it emits during the build stage. You probably saw them during `cf push` streaming logs above. In the next section, we will take a closer look at the language buildpacks.

## Buildpacks

During `cf-push`, the source code goes through a detection phase to find the right language buildpack image to build the app from source code.

    kubectl describe clusterstores/cf-buildpack-store | grep "Buildpackage" -A 10 | grep node -A 5 -B 5

The above command shows a list of node language buildpacks. cf-for-k8s uses the new rebranded paketo buildpacks, which are based on the cloud native buildpack spec.

    kubectl describe clusterstores/cf-buildpack-store | grep Order: -A 20 | grep node -B 5 -A 5

The above command highlights the detection ordering within the node language buildpackage. You may have noticed it during `cf push` logs.

## Stacks
Apps need a root file system to run. A stack provides the buildpack lifecycle with build-time and run-time environments in the form of images. You can see stack images in `cf-workloads` namespace.

    kubectl get clusterstacks -n cf-workloads-staging

You will `cflinuxfs3-stack` is the default stack used in cf-for-k8s.

    kubectl describe clusterstacks/bionic-stack -n cf-workloads-staging | grep Spec -A 6

Notice the build and run image entries. In most cases, both are the same images.


## Control plane components
The control plane components run in `cf-system` namespace

    kubectl get pods -n cf-system

Notable pods are the CAPI components that provide the `cf push` experience, uaa components provide authentication and authorization sevices, logging and metrics components provide observability, and eirini is responsible for scheduling & managing the app workloads and finally route-controller is responsible for the app routes. 

Also, notice `fluentd` pods in `cf-system`. It's a `DaemonSet` type running on every node to collect and filter logs for CF.

## Inspect namespaces

    kubectl get namespaces

It will return sever namespaces created by cf-for-k8s (the rest are namespaces created by K8s).

### Statefulsets

    kubectl get pvc -n cf-db
    kubectl get pvc -n cf-blobstore

`cf-db` and `cf-blobstore` namespaces run Postgres database and Minio blobstore stateful-sets. CAPI uses the blobstore to store the source code bits, the database to store state of the foundation (like orgs/space..) and app data. UAA uses the database for its authentication/authorization needs.

### kpack

`kpack` namespace contains kpack controller, which is responsible for building, packaging and pushing the app images to the docker registry (see Staging apps and App lang detection sections above).

    kubectl get pods -n kpack

### Istio

Let's look into the Istio namespace.

    kubectl get pods -n istio-system

`istio-system` namespace contains istio control plane components. Istio is responsible for ingress gateway, ingress encryption and encrypted communication between components - aka the service mesh.

`istio` injects side-cars into pods, which encrypt all communication between the containers running on the pod and other pods in the cluster (who are also running the side-car. The side-car injection is enabled at namespace level, so every pod within that namespace is injected with a side-car. We can see `cf-system` is side-car enabled by running this command,

    kubectl describe ns/cf-system | grep istio-injection

Let's check out the side car proxy in a CAPI pod.

    kubectl get pods -n cf-system | grep cf-api-server

**Pick any one pod from the above and replace `<pod id>` below** and run the command

    kubectl describe pod/<pod id> -n cf-system | grep "istio-proxy:" -A 5 -B 5

Notice the `istio-proxy` is a container running alongside the `cf-api-server` container.


## Overlays with `ytt`
In this exercise, we will **scale the `uaa` components from one to two pods** using a `ytt` overlay. `ytt` is a powerful yml templating tool that cf-for-k8s uses extensively. 


### Overlay yml
Let's peek into the overlay yml to understand ytt features.

    cat ../scale-cluster.yml

- The first line `load("@ytt:overlay", "overlay")` loads `ytt` overlay libraries.
- Line 2 `overlay/match by=overlay.subset({"kind":"Deployment","metadata":{"name":"uaa"}})` looks for a match of `kind: Deployment` by `name: uaa`. 
- If found, replace the `spec.replicas` in the target yml with everything after line 4. In this case, **set UAA replicas to two**.

### Render with ytt
Run the following command by including the `scale-cluster.yml` to create the final K8s config file `cf-for-k8s-rendered.yml`,

    ytt -f config -f ../scale-cluster.yml -f cf-values.yml > cf-for-k8s-rendered.yml

Note, you can pass multiple any number of templates/folders with `-f` flag but values `cf-values.yml` needs to be the last one.

### Redeploy with kapp

#### Confirm current UAA replicas
Before we scale UAA to 2 pods, let's confirm the current count.

    kubectl get pods -n cf-system | grep uaa

You should see just one pod count.

Redeploy cf-for-k8s by running the following command,

    kapp deploy -a cf \
    -f cf-for-k8s-rendered.yml -y

Note that `kapp` will check and display the differences between previous and current deploy and will prompt you for confirmation. We are passing `-y` to skip the confirmation.
 
Once `kapp` is finished, verify the `uaa` is infact scaled to two instances.
 
    kubectl get pods -n cf-system | grep uaa

### Verify cf push and app is still reachable
Verify app is still reachable by running this command,

    curl -k https://node-app.apps.$CF_DOMAIN

Now verify that cf push still works by running this command,

    cf push node-app -i 2 -p ./tests/smoke/assets/test-node-app/

## Delete cf-for-k8s
Finally, its time to say good bye. Run the following command to delete cf-for-k8s

    kapp delete -a cf -y

Check to make sure cf-for-k8s resources are compeletly deleted from the cluster

    kubectl get namespaces

## Beyond the Lab
I hope you enjoyed the lab session. If you want to get involved, here are few resources

- cf-for-k8s repository: https://github.com/cloudfoundry/cf-for-k8s
- cf-for-k8s slack channel: https://cloudfoundry.slack.com/archives/CH9LF6V1P
- ytt playground: https://get-ytt.io/
- kapp: https://get-kapp.io/