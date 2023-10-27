# alluxio-spark-eks

### Launch the Alluxio data platform and Spark on Amazon EKS.

---

## INTRODUCTION

The Alluxio high performance data platform allows compute workloads to run faster by caching remote data locally and by providing a unified namespace to mix disparate storage providers in the same data path. Alluxio helps reduce cloud storage data egress costs (from multiple regions) and helps reduce cloud storage API costs by caching metadata as well as real data.

This git repo provides a complete environment for demonstrating how to deploy Alluxio and Spark on an Amazon EKS Kubernetes cluster using S3 as the persistent object store.

For more information on Alluxio, see: https://www.alluxio.io/

For more information on running Spark on EKS, see: https://aws.amazon.com/blogs/big-data/introducing-amazon-emr-on-eks-job-submission-with-spark-operator-and-spark-submit/

## PREREQUISITES

TBD

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

TBD

---

Please direct questions or comments to greg.palme@alluxio.com
