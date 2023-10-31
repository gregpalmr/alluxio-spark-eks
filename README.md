# alluxio-spark-eks

### Launch the Alluxio data platform and Spark on Amazon EKS.

---

## INTRODUCTION

The Alluxio high performance data platform allows compute workloads to run faster by caching remote data locally and by providing a unified namespace to mix disparate storage providers in the same data path. Alluxio helps:

- Improve performance of data analytics and AI/ML workloads by caching S3 bucket objects on fast, local NVMe storage.
- Reduces cloud storage data egress costs (from multiple regions) by reducing the number of repedative reads agsint S3 buckets. 
- Helps reduce cloud storage API costs by caching metadata locally and reducing the number of API calls to the REST API.

This git repo provides a complete environment for demonstrating how to deploy Alluxio and Spark on an Amazon EKS Kubernetes cluster using S3 as the persistent object store.

For more information on Alluxio, see: https://www.alluxio.io/

For more information on running Spark on EKS, see: https://aws.amazon.com/blogs/big-data/introducing-amazon-emr-on-eks-job-submission-with-spark-operator-and-spark-submit/

## PREREQUISITES

To use the commands outlined in the repo, you will need the following:

- The git CLI installed - See: https://github.com/git-guides/install-git
- The AWS CLI installed - See: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- The AWS EKS CLI installed - See: https://eksctl.io/installation/
- The Kubernetes CLI - See: https://kubernetes.io/docs/tasks/tools/#kubectl
- The helm chart CLI - See: https://helm.sh/docs/intro/install/
- Your AWS credentials defined defined in the `~/.aws/credentials`, like this:

     - [default]
     - aws_access_key_id=[AWS ACCESS KEY]
     - aws_secret_access_key=[AWS SECRET KEY]

- You also need IAM role membership and permissions to create the following objects:
     - AWS S3 Buckets (or already have one available)
     - EKS clusters (and the various resources that get created)
     - CloudFormation stacks
     - EC2 instance types as specfied in the eks/eks-cluster.yaml file

## USAGE

### Step 1. Clone this repo

Use the git command to clone this repo (or download the zip file from the github.com site).

     git clone https://github.com/gregpalmr/alluxio-spark-eks

     cd alluxio-spark-eks

### Step 2. Deploy an EKS cluster

See: [Deploy an EKS Cluster](eks/README.md)

### Step 3. Deploy Alluxio on the EKS cluster

See: [Deploy Alluxio on the EKS Cluster](alluxio/README.md)

### Step 4. Deploy Spark on the EKS cluster

See: [Deploy Spark on the EKS cluster](spark/README.md)

### Step 5. Run test Spark jobs against Alluxio

See: [Deploy Spark on the EKS Cluster](spark/README.md)

### Step 6. Destroy the EKS cluster

To destroy the EKS cluster (and all the Alluxio and Spark pods running on it), use the following command:

     $ eksctl delete cluster --region us-west-1 --name=emr-spark-alluxio

CAUTION: All persistent volumes will be release and any data on them will be lost.

---

Please direct questions or comments to greg.palmer@alluxio.com
