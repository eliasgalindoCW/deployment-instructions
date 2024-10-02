
# Deployment Instructions for Staging and Production Environments

First, make sure your application is working correctly and has all necessary dependencies.

Once this is done, you will need to create a GitHub Workflow to deploy the application after a predefined action. To do this, simply create a folder named ".github" at the root of the project, and inside it create `.yml` files with the desired name.

Basically, you will need the tags: `name`, `on`, `env`, and `jobs`, where:

- `name`: is the name of the Workflow.
- `on`: the action that will trigger the Workflow.
- `env`: containing the environment variables of the project.
- `jobs`: the tasks that will be executed by the Workflow.

### Example of GH Workflow:

```Workflow
name: Deploy to Production

on:
  push:

env:
  ENVIRONMENT: 'production'
  GOPRIVATE: github.com/cloudwalk

jobs:
  build:
    permissions:
      contents: 'write'
      id-token: 'write'
    runs-on: cloudwalk-k8s-runner
    steps:
    - uses: actions/checkout@v4
      with:
        token: ${{ secrets.GHA_PAT }}
    - id: auth
      uses: google-github-actions/auth@v2
      with:
        access_token_lifetime: 1800s
        token_format: access_token
        workload_identity_provider: 'projects/39758643659/locations/global/workloadIdentityPools/github-workload-identity-pool/providers/github-actions-identity-provider'
        service_account: 'gke-github-actions@infinitepay-production.iam.gserviceaccount.com'
    - id: gcloud-setup
      name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v2
    - id: github-builder
      uses: cloudwalk/github-builder@v2
      env:
        GHA_PAT: ${{ secrets.GHA_PAT }}
      with:
        environment: 'production'
        gha_pat: ${{ secrets.GHA_PAT }}
        docker_buildkit: true
        docker_secrets: '["id=GHA_PAT"]'
```

## Dockerfile

After creating the Workflow responsible for sending the files to the Staging and Production environment in Argo CD, it's time to create a Dockerfile.

A Dockerfile is a text file that contains a series of instructions that Docker uses to create a Docker image. An image is like a "snapshot" of a complete environment where the application will run. Each instruction in the Dockerfile represents a step to configure that environment. Here's a basic explanation of the main components of a Dockerfile:

### Example of Dockerfile
```Dockerfile
# Choose the base image
FROM ubuntu:20.04

# Set the working directory inside the container
WORKDIR /app

# Copy files from the local directory to the container directory
COPY . /app

# Install dependencies
RUN apt-get update && apt-get install -y python3

# Set the command that will be executed when starting the container
CMD ["python3", "app.py"]
```

## YAML Manifest Files

After that, you will need to create the YAML manifests inside the `./config` folder. You can create the `staging.yaml` and `production.yaml` files for the respective environments.

Here is a detailed example:

### Example of Kubernetes YAML file:

This section defines general settings and information about the deployment environment and repositories used.

```Kubernetes
- Kubernetes:
- sourceRepo: #Defines the application's repository.
  - name: infinitepay-project #The name of the application's repository.
  - org: cloudwalk #The name of the organization hosting the repository (on GitHub or another provider).

- monoRepo: #Configures the monorepo and configuration files:
  - repository: monorepo-gitops #Repository where the GitOps infrastructure is stored (probably used with ArgoCD).
  - appsFolder: apps-config #Folder where the application's configuration files are maintained in the monorepo.
  - argoCDValues: helms/argocd/values.yaml #Helm values file used by ArgoCD to manage deployment.

- containerRegistry: #Where the application's container images are stored:
  - url: gcr.io/infinitepay-production #The Google Container Registry (GCR) used to store the application's container images.

- deployConfig: #Deployment settings for the application:
  - cluster: infinitepay-production #The name of the Kubernetes production cluster where the application will be deployed.
  - environment: production #The environment where the application will be deployed (production).
  - steps:
    - push #Sends the container images to the registry.
    - versioning #Defines versions for the images or the deployment.
    - build #Builds the container images.
    - deploy #Deploys to Kubernetes.

- service:
  - asmRevision: asm-managed-stable #This suggests that the environment is using Anthos Service Mesh (ASM) from Google Cloud for managing services and networking.

- name: infinitepay-project #Application name.

- imageRepoName: infinitepay-project #The name of the container image repository for this application.

- kind: deployment #The type of Kubernetes resource to be deployed (in this case, a Deployment that controls the execution of the application's pods).

- replicas: 1 #Defines that only 1 replica (pod) of the application will be executed.

- secretsManager: #Configurations for secret management:
  - enabled: true #Enables the use of secrets.
  - provider: gcpsm #The secret management provider is the Google Cloud Secret Manager (GCPSM).
  - secretsName: production-infinitepay-project #The name of the secret to be used in the production environment.

- resources: #Configures the resource limits and requests for the application:
  - limits: #Defines the maximum use of resources:
    - cpu: 100m
    - memory: 128Mi
  - requests: #Defines the minimum resources needed:
    - cpu: 50m
    - memory: 64Mi

- hpa: #Configures the Horizontal Pod Autoscaler (HPA), which dynamically adjusts the number of replicas based on CPU usage:
  - resource.name: cpu #HPA will monitor CPU usage.
  - resource.threshold: 30 #Horizontal scaling will occur when CPU usage exceeds 30%.
  - maxReplicas: 1 #The maximum number of replicas is 1 (no scaling, as the limit is 1).
  - minReplicas: 1 #The minimum number of replicas is 1.

- livenessProbe: #Defines how Kubernetes will check if the application is "alive" and working:
  - failureThreshold: 60 #The number of consecutive failures that can occur before the pod is considered "unhealthy".
  - periodSeconds: 5 #Interval between each liveness check attempt (every 5 seconds).
  - timeoutSeconds: 15 #Maximum time for the liveness check response.
  - httpGet: #Sends an HTTP request to check the health of the application:
    - path: / #Path of the application to be checked.
    - port: 8000 #The port to be used (8000).
    - scheme: HTTP #Uses HTTP to check the application.

- readinessProbe: #Checks if the application is ready to receive traffic:- Configurations similar to livenessProbe to check if the application is ready.

- service: #Service (network) settings:
  - istio: true #Indicates that Istio is enabled to manage the application's network communication.
  - ports: #Defines the port exposed by the container
    - containerPort: 8000 #The application is running on port 8000.
    - name: project #Service name.
    - appProtocol: http #The communication protocol is HTTP.
  - virtualservice:
    - gateway: internal #The service is available internally only.
    - hosts: #List of allowed hosts:
      - infinitepay-project.services.production.cloudwalk.network #The host that will access the service on the production network.
```

These are general and basic instructions for deploying an application to staging and production environments. You will need to adapt your application to its specific needs.
