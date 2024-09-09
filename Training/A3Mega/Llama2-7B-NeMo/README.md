# Instructions for Training Llama2-7B NeMo on A3Mega

This README provides step-by-step instructions for setting up and running the Llama2-7B NeMo model training on an A3Mega cluster. It includes explanations for each step to help you understand the process better.

## Prerequisites

- [Google Cloud Platform account with necessary permissions](https://console.cloud.google.com/?hl=en-au)
- [Gcloud CLI tool installed](https://cloud.google.com/sdk/docs/install)
- [Kubectl installed](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_kubectl)
- [Gke-gcloud-auth-plugin installed](https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke)
- [Docker installed (if building the image locally)](https://docs.docker.com/engine/install/)

## Step 1: Clone the Repository

First, clone the repository containing the necessary files:

```bash
git clone git@github.com:gclouduniverse/reproducibility.git
```

This step ensures you have all the required configuration files and scripts for the training process.

## Step 2: Set Up Google Cloud Environment

### Login to Google Cloud

Authenticate your Google Cloud account:

```bash
gcloud auth application-default login
```

This step ensures that you have the necessary credentials to interact with Google Cloud services.

### Set the Project

Set the Google Cloud project you'll be working in:

```bash
gcloud config set project <project_id>
```

Replace `<project_id>` with your actual Google Cloud project ID. This step is crucial as it determines which project's resources you'll be using.

## Step 3: Configure Kubernetes Cluster Access

Obtain the credentials for your Kubernetes cluster:

```bash
gcloud container clusters get-credentials <cluster_name> --zone <cluster_zone>
```

Example:
```bash
gcloud container clusters get-credentials a3plus-benchmark --zone australia-southeast1
```

This command updates your kubeconfig file with the necessary credentials to interact with your Kubernetes cluster.

## Step 4: Prepare the Training Environment

Navigate to the Llama2-7B-NeMo directory:

```bash
cd /path/to/reproducibility/Training/A3Mega/Llama2-7B-NeMo
```

## Step 5: Set Up Docker Authentication (if using Google Cloud Container Registry)

If you're using Google Cloud Container Registry, configure Docker to use Google Cloud credentials:

```bash
gcloud auth configure-docker <registries>
```

Example:
```bash
gcloud auth configure-docker us-docker.pkg.dev
```

This step allows Docker to push and pull images from your Google Cloud Container Registry. If you are not familiar with docker container registry, you can [read more here.](https://cloud.google.com/artifact-registry/docs/docker/authentication#standalone-helper)

## Step 6: Build and Push the Docker Image

Build the Docker image with the NeMo environment:

```bash
cd docker
docker build -t <registry_path_image_name>:<image_tag> -f nemo_example.Dockerfile .
```

Push the built image to your container registry:

```bash
docker push <registry_path_image_name>:<image_tag>
```

Example:
```bash
cd docker
docker build -t us-east4-docker.pkg.dev/supercomputer-testing/reproducibility/nemo_test:24.05 -f nemo_example.Dockerfile .
docker push us-east4-docker.pkg.dev/supercomputer-testing/reproducibility/nemo_test:24.05
cd ..
```

These steps create a Docker image with all the necessary dependencies and push it to your container registry for later use in the Kubernetes cluster. It will take a while to build the docker image. Please make sure you have enough space left on device to build the docker image.

## Step 7: Install Helm

Helm is a package manager for Kubernetes. Install it using the following commands:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Make the Helm binary executable:

```bash
sudo chmod +x /usr/local/bin/helm
```

## Step 8: Configure the Training Job

### Update Helm Values

Edit the `/helm_context/values.yaml` file:

```bash
vi helm-context/values.yaml
```

Make the following changes:
- Modify the `workload.image` string to match the Docker image you built earlier.
- Update the `workload.gcsBucketForDataCataPath` to point to an existing Google Cloud Storage bucket in the same region as your cluster. Or [create a new bucket](https://cloud.google.com/storage/docs/creating-buckets#console) and use it. 
- Optionally, adjust settings like the number of GPUs or other workload-specific parameters.

### Update Workload Configuration

Edit the workload configuration file:

```bash
vi helm-context/nemo-configurations/llama2-7b-16gpus-bf16.yaml
```

Adjust any model-specific parameters if needed.

### Set the Configuration File

Create a `selected-configuration.yaml` file in the `helm-context` directory:

```bash
cd helm-context
cp nemo-configurations/llama2-7b-16gpus-bf16.yaml ./selected-configuration.yaml
cd ..
```

This step is crucial for the workflow to function correctly.

## Step 9: Schedule the Training Job

Use Helm to package and schedule the job:

```bash
helm install <username_workload_job_name> helm-context/
```

Example:
```bash
helm install stingram-llama2-7b-nemo-16gpus helm-context/
```

This command creates and starts the Kubernetes job for training.

## Step 10: Monitor the Training Job

### Check Pod Status

To find the name of your training pod:

```bash
kubectl get pods | grep "<some_part_of_username_workload_job_name>"
```

Example:
```bash
kubectl get pods | grep "stingram-llama2-7b-nemo"
```

### Check Job Status

To see the overall job status:

```bash
kubectl get jobs | grep "<some_part_of_username_workload_job_name>"
```

Example:
```bash
kubectl get jobs | grep "stingram-llama2-7b-nemo"
```

### View Logs

To see the training logs:

```bash
kubectl logs "<pod_name>"
```

Replace `<pod_name>` with the name of your pod from the previous step.

## Step 11: Check Results

After the training is complete, you can find the Nsight logs in your specified Google Cloud Storage bucket:

```
/path/to/your/bucket/<run_id>/rank-0.nsys-rep
```

Example:
```
gs://benchmarks/nemo-experiments/stingram-llama2-7b-nemo-16gpus-1721164368-fwuz/rank-0.nsys-rep
```

These logs provide detailed performance information about your training run.

## Troubleshooting

If you encounter issues:
1. Double-check all configuration files for typos or misconfigurations.
2. Ensure your Google Cloud account has the necessary permissions.
3. Verify that your Kubernetes cluster has sufficient resources for the job.
4. Check the Kubernetes events and logs for any error messages.

## Conclusion

This README provides a detailed guide for setting up and running Llama2-7B NeMo training on an A3Mega cluster. Remember to clean up your resources after training to avoid unnecessary costs. If you have any questions or encounter issues, please refer to the project's documentation or seek support from the maintainers.