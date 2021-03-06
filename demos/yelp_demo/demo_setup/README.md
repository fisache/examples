# Instructions for Demo Setup

To setup your environment for running the demo, the following steps are
required. These steps should only need to be completed once.

1. [Install tools locally](#1-install-tools-locally)
1. [Set environment variables](#2-set-environment-variables)
1. [Setup GCP project permissions](#3-setup-gcp-project-permissions)
1. [Create a minikube cluster](#4-create-a-minikube-cluster)
1. [Create a GKE cluster](#5-create-a-gke-cluster)
1. [Prepare the ksonnet app](#6-prepare-the-ksonnet-app)
1. [Generate and store artifacts](#7-generate-and-store-artifacts)
1. [Troubleshooting](#8-troubleshooting)

## 1. Install tools locally

Ensure that you have at least the below versions of these tools (latest as of
2018-05-26). If so, skip to the [next step](#2-set-environment-variables).

* [docker](#install-docker) v18.03.1-ce
* [gcloud](#install-gcloud) v202.0.0
* [ksonnet](#install-ksonnet) v0.11.0
* [kubectl](#install-kubectl) v1.10.3
* [miniconda](#install-miniconda) v4.4.10
* [minikube](#install-minikube) v0.27.0
* [tensorflow](#install-tensorflow) v1.7.0
* [tensor2tensor](#install-tensor2tensor) v1.6.3
* [VirtualBox](#install-virtualbox) v5.2.12

### Install docker

The latest version for MacOS can be found
[here](https://store.docker.com/editions/community/docker-ce-desktop-mac).

### Install gcloud

The Google Cloud SDK can be found
[here](https://cloud.google.com/sdk/downloads).

### Install ksonnet

Download the correct binary based on your OS distro. The latest release can be found
[here](https://github.com/ksonnet/ksonnet/releases/tag/v0.11.0).

```
#export KS_VER=ks_0.11.0_linux_amd64
# MacOS
export KS_VER=ks_0.11.0_darwin_amd64
wget -O /tmp/$KS_VER.tar.gz https://github.com/ksonnet/ksonnet/releases/download/v0.11.0/$KS_VER.tar.gz
mkdir -p ${HOME}/bin
tar -xvf /tmp/$KS_VER.tar.gz -C ${HOME}/bin
export PATH=$PATH:${HOME}/bin/$KS_VER
```

### Install kubectl

After installing the Google Cloud SDK, install the kubectl CLI by running this command:

```
gcloud components install kubectl
```

### Install miniconda

Installation of [Miniconda](https://conda.io/docs/user-guide/install/index.html) for MacOS:

```
INSTALL_FILE=Miniconda2-latest-MacOSX-x86_64.sh
wget -O /tmp/${INSTALL_FILE} https://repo.continuum.io/miniconda/${INSTALL_FILE}
chmod 744 /tmp/${INSTALL_FILE}
bash -c /tmp/${INSTALL_FILE}
```

Installation of [conda](https://conda.io/docs/user-guide/install/index.html) for Ubuntu:

```
curl -O https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh
sudo apt-get install -y bzip2 # Not installed by default on GCP VMs
chmod 744 Anaconda3-5.0.1-Linux-x86_64.sh
bash -c ./Anaconda3-5.0.1-Linux-x86_64.sh
```

Create a new python2.7 environment:

```
conda create -y -n kfdemo python=2 pip scipy gevent sympy
source activate kfdemo
```

### Install minikube

For troubleshooting tips, see the [Kubeflow user
guide](https://github.com/kubeflow/kubeflow/blob/master/user_guide.md#minikube).

The below instructions install [Minikube](https://github.com/kubernetes/minikube):

#### Linux

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

#### macOS

```
brew cask install minikube
```

### Install VirtualBox

[VirtualBox](https://www.virtualbox.org/wiki/Downloads) is required for
minikube. To install, follow the instructions for your OS distro in the link.

### Install Tensorflow

These instructions install Tensorflow. Choose a version based on whether you have
GPUs locally:

```
pip install tensorflow==1.7.0 | tensorflow-gpu==1.7.0
```

### Install Tensor2Tensor

These instructions install tensor2tensor from master:

```
git clone git@github.com:tensorflow/tensor2tensor.git
cd tensor2tensor
git checkout tags/v1.6.3
python setup.py install
```

## 2. Set environment variables

Create a bash file with all the environment variables for this setup:

```
export DEMO_PROJECT=<your-project-name>
echo "source kubeflow-demo-base.env" >> ${DEMO_PROJECT}.env
```

Overwrite any environment variables from `kubeflow-demo-base.env` and add any
additional as required, then source the file:

```
source ${DEMO_PROJECT}.env
```

If you have not set the `GITHUB_TOKEN` environment variable, follow these
instructions to prevent rate-limiting errors by the GitHub API when installing ksonnet packages.
Using a personal access token authorizes you as an individual rather than anonymous user,
activating higher API limits.

Navigate to [https://github.com/settings/tokens](https://github.com/settings/tokens) and
generate a new token with no permissions. Save it somewhere safe. If you lose it, you will
need to delete and create a new one. Set the `GITHUB_TOKEN` environment variable,
preferably in your `.bash_profile` file:

```
export GITHUB_TOKEN=<token>
```

## 3. Setup GCP project permissions

The GCP project
[kubeflow-demo-base](https://console.cloud.google.com/kubernetes/list?project=kubeflow-demo-base)
has been created as part of `kubeflow.org`. It has quota for GPUs and TPUs and
should be sufficient for most purposes. To access this project, [create an
issue](https://github.com/kubeflow/examples/issues/new?title=[kubeflow-demo-base_access]:&labels=community/question)
in this repo and tag any of the [approvers](OWNERS).

To create an entirely new project of your own, complete the following steps:

1. [Create a Google group](#create-a-google-group)
1. [Create an owners project](#create-an-owners-project)
1. [Create a demo project](#create-a-demo-project)
1. [Setup GKE service account permissions](#setup-gke-service-account-permissions)
1. [Setup minikube service account permissions](#setup-minikube-service-account-permissions)

### Create a Google group

To easily maintain access to GCP resources, create a Google group. Members can
be added and removed over time as needed.

Set the following environment variables:

```
export GROUP_NAME=kubeflow-demos
export ORG_NAME=<your-org-name>
export DEMO_OWNERS_PROJECT_NAME=<unique_project_name>
```

Using the [GAM cli](https://github.com/jay0lee/GAM/wiki), execute the following
command:

```
~/bin/gam/gam create group ${GROUP_NAME}@${ORG_NAME} who_can_join \
  invited_can_join name ${GROUP_NAME} \
  description"Group members with access to demos" \
  allow_external_members true
```

### Create an owners project

Create a master project that allows creation of new projects with Deployment
Manager.

```
gcloud projects create ${DEMO_OWNERS_PROJECT_NAME} \
  --organization=${ORG_NAME}
```

#### Add permissions to an owners project

Grant access to the Gooogle group so that only members of
`${GROUP_NAME}@${ORG_NAME}` can create projects with Deployment Manager and
register DNS records:

```
gcloud projects add-iam-policy-binding ${DEMO_OWNERS_PROJECT_NAME} \
  --member group:${GROUP_NAME}@${ORG_NAME} \
  --role=roles/deploymentmanager.editor

gcloud projects add-iam-policy-binding ${DEMO_OWNERS_PROJECT_NAME} \
  --member group:${GROUP_NAME}@${ORG_NAME} \
  --role=roles/kubeflow-dns
```

### Create a demo project

Use Deployment Manager to easily create new projects for the demo.

To create a new project for use during demos:

1. Create a config file
```
cp project_creation/config-kubeflow-demo-base.yaml project_creation/config-${DEMO_PROJECT}.yaml
```
  * For `${DEMO_PROJECT}` use whatever name you want that isn't already taken. This
name must be unique across all organizations, not just kubeflow.org.

1. Modify `project_creation/config-${DEMO_PROJECT}.yaml`

  * Change resources.name to `${DEMO_PROJECT}`
  * Populate resources.properties.organization-id or resources.properties.parent-folder-id
  * Populate resources.properties.billing-account-name
  * Populate resources.properties.iam-policy-patch.add.members (both array elements)

1. Run

```
cd project_creation
gcloud deployment-manager deployments create ${DEMO_PROJECT} \
  --project=${DEMO_OWNERS_PROJECT_NAME} \
  --config=config-${DEMO_PROJECT}.yaml
```

After creating the deployment, it can be changed later with this command:

```
gcloud deployment-manager deployments update ${DEMO_PROJECT} \
  --project=${DEMO_OWNERS_PROJECT_NAME} \
  --config=config-${DEMO_PROJECT}.yaml
```

#### Update Resource Quotas for the Project

Currently this has to be done via the UI. Change the project name in this
[URL](https://console.cloud.google.com/iam-admin/quotas?project=kubeflow-demo-base&metric=Backend%20services,CPUs,CPUs%20(all%20regions%29,Health%20checks,NVIDIA%20K80%20GPUs,Persistent%20Disk%20Standard%20(GB%29&location=GLOBAL,us-central1,us-east1).

Suggested quota usages:

* In regions us-east1 & us-central1
* 100 CPUs per region
* 200 CPUs (All Region)
* 100000 Gb PDs in each region
* 10 K80s in each region
* 10 backend services
* 100 health checks

Usually the resource grants are auto-approved pretty quickly.

### Setup minikube service account permissions

To run from a cluster outside of GKE such as minikube or Docker EE, kubeflow
needs access to credentials for a service account. To create a service account,
issue the following command:

```
SERVICE_ACCOUNT=minikube@${DEMO_PROJECT}.iam.gserviceaccount.com
gcloud iam service-accounts create ${SERVICE_ACCOUNT} --display-name=${SERVICE_ACCOUNT}
```

Issue permissions to the service account:

```
gcloud projects add-iam-policy-binding ${DEMO_PROJECT} \
  --member=serviceAccount:${SERVICE_ACCOUNT} \
  --role=roles/storage.admin
```

Create a private key for the service account:

```
gcloud iam service-accounts keys create ${HOME}/.ssh/minikube_key.json \
  --iam-account=${SERVICE_ACCOUNT}
```

## 4. Create a minikube cluster

To start a minikube instance:
```
minikube start --cpus 4 --memory 8096 --disk-size=40g
```

RBAC permissions allow your user to install kubeflow components on the cluster.

```
kubectl create clusterrolebinding cluster-admin-binding-${USER} \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
./create_context.sh minikube ${NAMESPACE}
```

Create a namespace:
```
kubectl create namespace ${NAMESPACE}
```

### Create k8s secrets

Since our project is private, we need to provide access to resources via the use
of service accounts. We need two different types of secrets for storing these
credentials. One of type `docker-registry` for pulling images from GCR and one
one of type `generic` for accessing private assets.

```
kubectl -n $NAMESPACE create secret docker-registry gcp-registry-credentials \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat $HOME/.ssh/minikube_key.json)" \
  --docker-email=minikube@${DEMO_PROJECT}.iam.gserviceaccount.com

kubectl -n $NAMESPACE create secret generic gcp-credentials \
  --from-file=key.json="${HOME}/.ssh/minikube_key.json"
```

### Prepare the ksonnet app

Create the minikube environment:

```
cd ../demo
ks env add minikube --namespace=${NAMESPACE}
```

Set parameter values for training:

```
ks param set --env minikube t2tcpu \
  dataDir ${GCS_TRAINING_DATA_DIR}
ks param set --env minikube t2tcpu \
  outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_LOCAL}
ks param set --env minikube t2tcpu \
  cpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-cpu:latest
ks param set --env minikube t2tcpu \
  gpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-gpu:latest
```

Set parameter values for serving component:
```
ks param set --env minikube serving modelPath ${GCS_TRAINING_OUTPUT_DIR_LOCAL}/export/Servo
```

## 5. Create a GKE cluster

### Setup a CPU/GPU cluster

#### Create the cluster

Create a config file:

```
cp gke/cluster-kubeflow-demo-base.yaml gke/cluster-${DEMO_PROJECT}.yaml
```

Modify any desired values, such as cluster name. Issue the following command, which uses Deployment Manager to create a GKE
cluster:

```
gcloud deployment-manager deployments create gke-${CLUSTER} \
  --project=${DEMO_PROJECT} \
  --config=gke/cluster-${DEMO_PROJECT}.yaml
```

#### Setup kubectl access

The script [create_context.sh](./create_context.sh) creates a kubectl context
referencing the correct namespace, enabling the use of `kubectl` in all future
commands.

```
gcloud container clusters get-credentials ${CLUSTER} \
  --project=${DEMO_PROJECT} \
  --zone=${ZONE}
./create_context.sh gke ${NAMESPACE}
```

#### Add RBAC permissions

This allows your user to install kubeflow components on the cluster.

```
kubectl create clusterrolebinding cluster-admin-binding-${USER} \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

#### Install GPU device drivers

```
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

#### Create the kubeflow namespace

```
kubectl create namespace ${NAMESPACE}
```

#### Create the ksonnet environment

The ksonnet app can be found in the [demo](../demo) directory of this repo. Add an
environment referencing the newly created GKE cluster for deployment of each
component.

```
cd ../demo
ks env add ${ENV} --namespace=${NAMESPACE}
```

## 6. Prepare the ksonnet app

The ksonnet app can be found in the [demo](../demo) directory of this repo. Add an
environment referencing the newly created GKE cluster for deployment of each
component.

### Set parameter values for training components

```
ks param set --env ${ENV} t2tcpu \
  dataDir ${GCS_TRAINING_DATA_DIR}
ks param set --env ${ENV} t2tcpu \
  outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_CPU}
ks param set --env ${ENV} t2tcpu \
  cpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-cpu:latest
ks param set --env ${ENV} t2tcpu \
  gpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-gpu:latest

ks param set --env ${ENV} t2tgpu \
  dataDir ${GCS_TRAINING_DATA_DIR}
ks param set --env ${ENV} t2tgpu \
  outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_GPU}
ks param set --env ${ENV} t2tgpu \
  cpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-cpu:latest
ks param set --env ${ENV} t2tgpu \
  gpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-gpu:latest

ks param set --env ${ENV_TPU} t2ttpu \
  dataDir ${GCS_TRAINING_DATA_DIR}
ks param set --env ${ENV_TPU} t2ttpu \
  outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_TPU}
ks param set --env ${ENV_TPU} t2ttpu \
  cpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-cpu:latest
ks param set --env ${ENV_TPU} t2ttpu \
  gpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-gpu:latest
```

### Set parameter values for serving component

Choose the directory depending on whether you want to serve from the CPU, GPU, or TPU model.

```
ks param set --env ${ENV} serving modelPath ${GCS_TRAINING_OUTPUT_DIR_GPU}/export/Servo
ks param set --env ${ENV_TPU} serving modelPath ${GCS_TRAINING_OUTPUT_DIR_TPU}/export/Servo
```

## 7. Generate and store artifacts

To safeguard against potential failures while running a live demo, pre-generate
artifacts and squirrel them away for use in a break-glass scenario.

The following artifacts are useful to have ready:
1. [Training data](#generate-training-data)
1. [Training & UI images](#generate-training-and-ui-images)
1. [Trained model files](#generate-traiend-model-files)

### Generate training data

Ensure that you are using the right conda environment:

```
source activate kfdemo
```

Set the following environment variable temporarily:

```
export MAX_CASES=0
```

In the
[./yelp/yelp_sentiment/yelp_problem.py](../yelp/yelp_sentiment/yelp_problem.py#L13)
file, set the constant YELP_DATASET_URL to the full dataset (i.e. yelp-dataset.zip).

Generate a dataset for training and store in GCS.
`${GOOGLE_APPLICATION_CREDENTIALS}` must be set properly in order for this to
work.

Warning: this command takes around 45-60 mins to complete on the full Yelp
dataset. Smaller versions are available for faster processing
(yelp_review_10000.zip).

```
cd ../yelp/

t2t-datagen \
  --t2t_usr_dir=${USR_DIR} \
  --problem=${PROBLEM} \
  --data_dir=${GCS_TRAINING_DATA_DIR} \
  --tmp_dir=${TMP_DIR}-${MAX_CASES} \
  --max_cases=${MAX_CASES}
```

Cleanup data files:

```
rm -rf ${TMP_DIR}-${MAX_CASES}
```


### Generate training and UI images

Generate all necessary docker images and store them in GCR. This generates a
CPU, GPU, and UI image.

```
cd ..
make PROJECT=${DEMO_PROJECT} set-image

```

### Generate trained model files

Warning: this command takes 8+ hours to complete.

```
cd demo
ks param set --env ${ENV} t2tcpu trainSteps 20000
ks param set --env ${ENV} t2tcpu dataDir ${GCS_TRAINING_DATA_DIR}
ks param set --env ${ENV} t2tcpu outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_CPU}
ks param set --env ${ENV} t2tcpu cpuImage gcr.io/${DEMO_PROJECT}/kubeflow-yelp-demo-cpu:latest
ks apply ${ENV} -c t2tcpu
```

#### Export the trained model

This will export the model to an `export/` directory in output_dir.

```
cd ../yelp
t2t-exporter \
  --t2t_usr_dir=${USR_DIR} \
  --model=${MODEL} \
  --hparams_set=${HPARAMS_SET} \
  --problem=${PROBLEM} \
  --data_dir=${GCS_TRAINING_DATA_DIR} \
  --output_dir=${GCS_TRAINING_OUTPUT_DIR_CPU}
```

## 8. Troubleshooting

### Updating node pools in CPU/GPU clusters

The update method for node pools does not allow arbitrary fields to be
changed. To make a change to node pools, do the following:

* Make any changes to the node pool config
* Bump the property pool-version
  * This causes the existing pool to be deleted and new ones to be created with a different name.
* Issue an update command:

```
gcloud deployment-manager deployments update gke-${CLUSTER} \
  --project=${DEMO_PROJECT} \
  --config=gke/cluster-${DEMO_PROJECT}.yaml
```

