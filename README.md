# kubernetes-in-practice

## Prerequisites

### minikube
Install [minikube](https://minikube.sigs.k8s.io/docs/start).

```shell
# brew
brew install minikube
```

Make sure you have a minikube driver such as Docker or Podman. See [here](https://minikube.sigs.k8s.io/docs/drivers/) for a full overview of supported drivers.

Then, start minikube.
```shell
# [optional] for a full reset, first delete any pre-existing stuff
minikube delete

minikube start
```

### kubectl
Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/).

```shell
# brew
brew install kubectl
```

### helm (kubernetes package manager)
Install [helm](https://helm.sh/docs/intro/install/).

```shell
# brew
brew install helm
```

### [Optional] lens (kubernetes GUI)
Install [lens](https://k8slens.dev/download).

```shell
# brew
brew install --cask lens
```

## Recommended workflow
### kubectl apply
I would recommend working with `kubectl apply` and `.yaml` files over using specfic `kubectl` commands.

For example, if you would like to create a new deployment, you could use `kubectl create deployment ...`. However, I would recommend to first create a `deployment.yaml` file, copy and paste a deployment template from the web, and then use `kubectl apply -f deployment.yaml` to deploy it instead.

### GUI or kubectl
To answer the questions in the exercises you can use a GUI or use kubectl. Do what feels best for you!

## Exercises

### 01. Jobs
#### 01.A. Deploy the job
Deploy the example job to kubernetes.
```shell
kubectl apply -f examples/jobs/job.yaml
```

Observe the status of the job.
```shell
kubectl get jobs
```

Then, let's inspect the job itself
```shell
# you can use describe for a quick overview
kubectl describe job/hello

# but personally I prefer the .yaml output
kubectl get job/hello -o yaml

# you can also inspect events
kubectl events --for job/hello
```

The Job started a Pod, can you find the name of the Pod?

Can you find the output logs of the Pod (hint: `kubectl logs`)?

#### 01.B. CronJob
Can you create a CronJob that runs the hello job every minute?

#### 01.C. ConfigMap
Can you create a ConfigMap resource and use it configure the output string of the Job?

#### 0.1.D. Dealing with failures
Try changing the command of the (Cron)Job so it fails (e.g. make it `asdf`). It should retry automatically a few times. Can you change the number of retries? Can you turn it off?

### 02. Deployments

#### 02.A. Deploy and scale the Deployment
There is an example Deployment in `examples/deployments/deployment.yaml`. Deploy it.

Increase the number of replicas and redeploy the Deployment. What happens?

Now, reduce the number of replicas. What happens?

Then, delete some pods and observe what happens.
```shell
kubectl get pods
kubectl delete pod/<name_of_pod>
```
The Pod controller will automatically start new Pods to meet the desired replica count. This is why kubernetes is considered self-healing and why we denote resources as 'records of intent'.

#### 02.B Updating the Deployment
Before continuing, make sure the replica count is at 4.

The nginx image used in this example is old. Can you update it to the latest version?

Observe what happens. Hint: do not forget about `kubectl events`.

This behavior is caused by the default RollingUpdate strategy.

Change the Deployment strategy to Recreate and repeat the exercise. What is the difference?

#### 02.C Connect a Service to a Deployment
Let's create a new deployment with the following image `crccheck/hello-world`. It should be exposed to port 8000.

Add a Service resource that connects to the newly created Deployment. Remember that a Service points to a Deployment via selector labels.

Port-forward the created Service to your localhost.
```shell
# via minikube
minikube service <name_of_service>

# via kubectl
kubectl port-forward service/<name_of_service> <localhost_port>:<service_port>
```

Go to the forward port in your browser. What do you see? Can you also find the request in the logs of the Pod?

Increase the number of replicas of the service. Then, refresh the page a few times. Does your request get send to a single Pod or does it reach multiple Pods?

### 03. StatefulSet
A database is best deployed as StatefulSet. Let's deploy Postgres to kubernetes.

Get into the *Vibe*. Use your favorite LLM to help you get started! But don't forget to deploy a Service to expose your database. You probably also need some type of PersistentVolume, and maybe a ConfigMap or Secret for configuration.

### 04. Helm
#### 04.A. Using helm as package manager
As you maybe noticed already writing all the yaml files manually can be quite a hassle. Helm acts a type of package manager for yaml files.

Let's deploy pgAdmin, a Postgres web interface, with helm.
```shell
helm repo add runix https://helm.runix.net
# helm install <local_name> <repo>/<chart>
helm install pgadmin4 runix/pgadmin4
```

#### 04.B. [Optional] Get pgAdmin to work
Executing this command installed a bunch of resources on your cluster. Which ones?

Forward the Service using your favorite method. And visit the page on your browser. Login with the default credentials:
- email: chart@domain.com
- password: SuperSecret

Now try to add the PG server you deployed in the previous exercise. As hostname you can simply specify the name of the Service resource. Please note that depending on what you did during that exercise the username and password might be different from the defaults.

#### 0.4.C. Using helm to build a chart yourself

Let's create a helm chart ourselves.
```shell
helm create awesome-chart
```
This created a bunch of example stuff in the `awesome-chart` directory.

To install it, we can again use helm.
```shell
helm install awesome-release ./awesome-chart
```

What happens behind the curtains is that helm generates manifests based on the templates in the `awesome-chart/templates` folder. It uses values from `awesome-chart/values.yaml` and functions as specified in the `_helpers.tpl`. This is powered by Helms own little [templating language](https://helm.sh/docs/chart_template_guide/function_list/).

Adjust the `replicaCount` in `values.yaml`. Then update your helm installation.
```shell
helm upgrade awesome-release ./awesome-chart
```
Helm really is a powerful way to work with large groups of manifests.

### 05. CustomResourceDefinitions
To unlock even more power of kubernetes let's introduce some custom resources. We are going to install [Strimzi](https://strimzi.io/quickstarts/) and setup a Kafka cluster.

Install Strimzi.
```shell
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

With the installation of Strimzi you also installed new custom kubernetes objects (via [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)).

To see this in action, copy the resources [here](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/kafka/kafka-single-node.yaml) to setup a single-node Kafka cluster.

Then create a new Topic (use the same example repo for inspiration).

Can you deploy a resource that produces some messages to this topic? hint: google 'bin/kafka-console-producer.sh'. Can you also deploy a resource that receives messages from the topic?

### 06. ArgoCD
[ArgoCD](https://argo-cd.readthedocs.io/en/stable/) is a cool tool that you can use to sync a Git repo with a helm chart to your kubernetes platform. This is one of the way to achieve Continous Deployment in a GitOps way-of-working.

Install it.
```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Also install the CLI.
```shell
# brew
brew install argocd
```

Then, login the [webui](https://localhost:8080/). Potentially you get a certificate warning, ignore this.
```shell
# find the temporary password
argocd admin initial-password -n argocd

# forward the service
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

With the installation of ArgoCD you also installed new custom kubernetes objects (via [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)).

> The Application CRD is the kubernetes resource object representing a deployed application instance in an environment.

Let's try an example.
```shell
kubectl apply -f examples/argo/application.yaml
```

What do you see happening in the webui?

On default Applications do not sync automatically. Click the 'sync' button to sync it. What happens?

Can you change the Application spec to enable automatic syncing? Can you enable automatic pruning as well?

Enable [Resource Monitoring](https://localhost:8080/settings/projects/default). Now ArgoCD also tracks orphaned resources (all resources not linked to an Application). This is useful maintain clean namespaces.

Can you sync the [Grafana helm chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana) to your cluster using ArgoCD?


### 07. Other stuff
I have no more concrete exercises. But down below you can find a list of some more advanced stuff that might be interesting to dive into.
- [Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- Custom kubernetes controller / operators
   - https://kopf.readthedocs.io/en/stable/concepts/
   - https://github.com/kube-rs/controller-rs
