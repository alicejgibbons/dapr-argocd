# Dapr + Argo CD + Redis PubSub Example

This repository demonstrates how to deploy Dapr and Redis using Argo CD, along with a sample Go application that uses Dapr's pub/sub building block with Redis as the message broker.

## Repository Structure

- `/gitops`: Contains all Argo CD, Dapr, and Redis configuration files
- `/app`: Contains the sample Go publisher and subscriber applications

## Prerequisites

- Kubernetes cluster (v1.21+ recommended)
- kubectl configured to access your cluster
- Helm v3
- Docker (for building application images)
- Git

## Step 1: Install Argo CD

First, you need to install Argo CD on your Kubernetes cluster:

```bash
# Create a namespace for Argo CD
kubectl create namespace argocd

# Apply the Argo CD installation manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Then, install the Argo CD cli:

```bash
# Install Argo CD CLI first if you haven't already
brew install argocd
```

After installation, access the Argo CD UI:

```bash
# Port-forward to the Argo CD server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Navigate to `https://localhost:8080` in your browser.

Get the initial admin password:

```bash
argocd admin initial-password -n argocd
```

Log in with username `admin` and the password you just retrieved.

```bash
argocd admin <initial-password> -n argocd
```

Using the username admin and the password from above, login to Argo CD's IP or hostname (in our case, we are running it on `localhost:8080`):

```bash
argocd login localhost:8080
```

For a detailed step-by-step process on how to install Argo CD, follow the [official documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/).

## Step 2: Install Dapr Using the Argo CD cli

An Argo CD application is a custom Kubernetes resource that defines how an application should be deployed and managed using the GitOps methodology. It specifies the source configuration (usually a Git repository, Helm chart, or directory path), the destination cluster and namespace, sync policies for automation, and special synchronization options. This resource acts as the bridge connecting your desired state (defined in Git) with the actual state in your Kubernetes cluster.

When deployed, Argo CD continuously monitors both your Git repository and Kubernetes cluster to ensure they remain synchronized. Any deviation triggers either an alert or an automatic reconciliation based on your configuration. The application can be created and managed through Argo CD's web UI, CLI commands, YAML manifests, or API calls, making it a flexible foundation for implementing continuous delivery in Kubernetes environments.

Here, the Argo CD cli is used to create the Application for our Dapr deployment:

```bash
 argocd app create dapr --repo https://github.com/rochabr/dapr-argocd.git --path gitops/dapr --dest-server https://kubernetes.default.svc --dest-namespace default
```

Sync the application and verify the installation:

```bash
argocd app sync dapr
argocd app get dapr
kubectl get pods -n dapr-system
```

## Step 3: Install Redis Using Argo CD

Create an Argo CD application for Redis:

```bash
argocd app create redis --repo https://github.com/rochabr/dapr-argocd.git --path gitops/redis --dest-server https://kubernetes.default.svc --dest-namespace default
```

Sync the application and verify the installation:

```bash
argocd app sync redis
argocd app get redis
kubectl get pods -n redis
```

## Step 4: Configure Dapr Components

Create the Dapr pub/sub component application:

```bash
argocd app create dapr-components \
  --repo https://github.com/yourusername/dapr-argocd-pubsub.git \
  --path gitops/pubsub-components \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace pubsub-demo \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --directory-recurse \
  --directory-exclude "*.yaml" \
  --directory-include "redis-pubsub.yaml"
```

Sync the application:

```bash
argocd app sync dapr-components
argocd app get dapr-components
```

## Step 5: Deploy the Sample Application

Deploy the publisher and subscriber applications:

```bash
argocd app create pubsub-demo \
  --repo https://github.com/yourusername/dapr-argocd-pubsub.git \
  --path app/kubernetes \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace pubsub-demo \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

This will deploy both the publisher and subscriber services in the `pubsub-demo` namespace.

Verify the deployments:

```bash
kubectl get pods -n pubsub-demo
```

You should see both the publisher and subscriber pods running with their Dapr sidecars.

## Step 6: Verify the Pub/Sub Communication

Check the logs of the subscriber application to verify it's receiving messages:

```bash
# Get the subscriber pod name
SUBSCRIBER_POD=$(kubectl get pod -n pubsub-demo -l app=subscriber -o jsonpath='{.items[0].metadata.name}')

# View the logs
kubectl logs -n pubsub-demo $SUBSCRIBER_POD -c subscriber
```

You should see messages like:
```
Received message: Message 1 from publisher
Received message: Message 2 from publisher
...
```

Check the publisher logs:

```bash
# Get the publisher pod name
PUBLISHER_POD=$(kubectl get pod -n pubsub-demo -l app=publisher -o jsonpath='{.items[0].metadata.name}')

# View the logs
kubectl logs -n pubsub-demo $PUBLISHER_POD -c publisher
```

You should see messages like:
```
Published message: Message 1
Published message: Message 2
...
```

## Understanding the Application

The sample application demonstrates the pub/sub pattern using Dapr:

1. **Publisher**: Sends messages every 5 seconds to the `messages` topic
2. **Subscriber**: Subscribes to the `messages` topic and processes incoming messages

The communication between the services is facilitated by Dapr, which uses Redis as the message broker behind the scenes.

## How It Works

1. Dapr injects sidecars into both applications
2. The publisher uses Dapr's HTTP/gRPC API to publish messages
3. The subscriber registers an endpoint that Dapr calls when messages arrive
4. Dapr handles all the message routing and delivery guarantees

## Customization

To modify this example for your own use:

1. Update the Docker image references in the deployment files
2. Modify the publisher/subscriber code for your specific use case
3. Adjust the Dapr component configuration for production settings

## Clean Up

To remove everything:

```bash
# Delete the application
kubectl delete -f app/kubernetes/application.yaml

# Delete the Dapr components
kubectl delete -f gitops/pubsub-components/application.yaml

# Delete Redis
kubectl delete -f gitops/redis/application.yaml

# Delete Dapr
kubectl delete -f gitops/dapr/application.yaml

# Delete Argo CD
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.0/manifests/install.yaml

# Delete namespaces
kubectl delete namespace argocd dapr-system redis pubsub-demo
```