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
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.0/manifests/install.yaml

# Wait for rollout
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

After installation, access the Argo CD UI:

```bash
# Port-forward to the Argo CD server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Navigate to `https://localhost:8080` in your browser.

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Log in with username `admin` and the password you just retrieved.

## Step 2: Install Dapr Using Argo CD

Create an Argo CD application for Dapr:

```bash
# Apply the Dapr application manifest
kubectl apply -f gitops/dapr/application.yaml
```

This will automatically install Dapr in the `dapr-system` namespace using Helm.

Verify the installation:

```bash
kubectl get pods -n dapr-system
```

## Step 3: Install Redis Using Argo CD

Create an Argo CD application for Redis:

```bash
# Apply the Redis application manifest
kubectl apply -f gitops/redis/application.yaml
```

This will install Redis in the `redis` namespace.

Verify the installation:

```bash
kubectl get pods -n redis
```

## Step 4: Configure Dapr Components

Create the Dapr pub/sub component for Redis:

```bash
# Apply the components application manifest
kubectl apply -f gitops/pubsub-components/application.yaml
```

This will create the necessary Dapr component to use Redis as a pub/sub broker.

## Step 5: Deploy the Sample Application

Deploy the publisher and subscriber applications:

```bash
# Apply the application manifest
kubectl apply -f app/kubernetes/application.yaml
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