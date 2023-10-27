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

## USAGE (con't)

## Step 3. Deploy Alluxio on the EKS cluster

### a. Enable the Alluxio Helm chart

Remove any older Alluxio Helm chart repos:

     $ helm repo list
     NAME              URL
     alluxio-charts    https://alluxio-charts.storage.googleapis.com/openSource/2.6.2

     $ helm repo rm alluxio-charts
     "alluxio-charts" has been removed from your repositories
Use the helm command to add the Alluxio Helm chart to the current repo list:

     $ helm repo add alluxio-charts https://alluxio-charts.storage.googleapis.com/openSource/2.9.3
     "alluxio-charts" has been added to your repositories

### b. Configure the Alluxio Helm chart

Create the alluxio-helm-values.yaml file that will be used by the Helm chart to deploy Alluxio on the EKS cluster. There are several things that must be configured including:
- Persistent storage volumes for the Alluxio master nodes' metadata repository and journal storage.
- Persistent storage volumes for the Alluxio worker nodes' cache storage.
- TBD

Make a working copy of the alluxio-helm-values.yaml file:

     $ cp alluxio/alluxio-helm-values.yaml.template alluxio/alluxio-helm-values.yaml

Modify the yaml file for your Alluxio deployment, by doing the following:

- (Optional) If you are using the Enterprise Edition of Alluxio, replace "PUT_YOUR_LICENSE_BASE64_VALUE_HERE" with your BASE64 version of the license key and uncomment the line that begins with "#license:". Use the following command to get the BASE64 version of your license key:
     $ cat /path/to/license.json | base64 |  tr -d "\n"
- Alluxio requires a root under file system (UFS) and you can add other UFSs later. Replace "PUT_YOUR_S3_BUCKET_NAME_HERE" with the name of an S3 bucket that Alluxio can use as the root UFS. If you do not have instance IAM roles configured, you can specify the accessKeyId and secretKey by changing PUT_YOUR_AWS_ACCESS_KEY_ID_HERE and PUT_YOUR_AWS_SECRET_KEY_HERE. If you do have instance roles configured, keep those commented out.
- Change the jvmOptions values for the master, worker, job_master, and job_worker pods, as needed. Alluxio provides guidence on tuning the Alluxio master node and worker node JVMs here: 
     - https://docs.alluxio.io/os/user/stable/en/administration/Performance-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/administration/Scalability-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/kubernetes/Running-Alluxio-On-Kubernetes.html?q=JAVA_OPTS
- The default values in the template file assume that the EKS nodes can provide:
     - Alluxio master node pods:
          -  4 CPU cores
          - 24 GB of Java Heap and 10 GB of direct memory 
          -  2 300 GB NVMe (unformatted) persistent volumes for cache storage (see: "quota: 300GB,300GB")
     - Alluxio worker node pods:
          -  8 CPU cores
          - 32 GB of Java Heap and 10 GB of direct memory 
          -  2 600 GB (unformatted) NVMe persistent volumes for cache storage (see: "quota: 550GB,550GB")
- If you don't want to install Alluxio on all the EKS nodes, you can define a toleration that will cause Alluxio pods not to get scheduled on specific nodes. Change PUT_YOUR_TOLERATION_KEY_HERE and PUT_YOUR_TOLERATION_VALUE_HERE, and uncomment that section.

Use your favorite editor to modify the Alluxio-helm-values.yaml file:

     $ vi alluxio/alluxio-helm-values.yaml

---

Please direct questions or comments to greg.palme@alluxio.com
