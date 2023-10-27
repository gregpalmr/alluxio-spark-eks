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

TBD

## USAGE (con't)

## Step 2. Deploy an EKS cluster

### a. Modify the EKS cluster yaml file.

The EKS yaml file is used to deploy the EKS cluster and the Kubernetes nodes that will host the Spark pods and the Alluxio pods. 

Use an EC2 instance type that has enough NVMe storage to allow Alluxio to cache the needed data from the S3 buckets. The m5d.4xlarge instance type has two 300 GB NVMe volumes, the m5d.8xlarge has 2 600 GB NVMe volumes.

Make a working copy of the eks-cluster.yaml file:

     $ cp eks/eks-cluster.yaml.template eks/eks-cluster.yaml

Modify the yaml file for your deployment, by doing the following:

- To restrict access to your EKS cluster, replace PUT_YOUR_YOUR_PUBLIC_IP_HERE/32 with your computer's public IP address.
- Change the references to the AWS region and availability zones. Change us-west-1, us-west-1a and us-west-1b as needed. 
- Add a reference to your private SSH key, 
- If you want to be able to SSH into the EC2 instances, replace PUT_YOUR_PATH_TO_PUB_SSH_KEY_HERE with the path to your public ssh key. 
- Change the number of nodes in your EKS cluster to support the workloads you are running. For Alluxio, you will require a minimum of 3 master nodes and 3 worker nodes.
- Change the EC2 instance types for the master nodes and worker nodes. Make sure you choose instance types that have enough cpu vcores, memory and NVMe storage to support running both Spark pods and Alluxio pods and that allow Alluxio to cache enough data on NVMe storage to improve performance. Alluxio requirements and tuning best practives can be found here:
https://docs.alluxio.io/os/user/stable/en/deploy/Requirements.html
https://docs.alluxio.io/os/user/stable/en/administration/Performance-Tuning.html
https://docs.alluxio.io/os/user/stable/en/administration/Scalability-Tuning.html

Use your favorite editor to modify the yaml file:

     $ vi eks/eks-cluster.yaml

### b. Deploy the EKS cluster

Use the eksctl command line tool to launch the EKS cluster:

     $ eksctl create cluster -f eks/eks-cluster.yaml

After the eksctl tool reports that the cluster was created, you can display the EKS nodes and other cluster information using the commands:

     $ eksctl get clusters --region=us-west-1

     $ eksctl get nodegroups --region=us-west-1 --cluster=emr-spark-alluxio

     $ kubectl get nodes -o wide

     $ kubectl describe node ip-192-168-12-142.us-west-1.compute.internal

### c. Setup a Service Account

The Kerbernetes CSI driver needs permission to issue API calls to the Kubernetes control plane to manage the lifecycle of the persistent volumes (PVs). Use a manifest file that defines a Kubernetes service account and attaches a Kubernetes cluster role that grants the necessary Kubernetes API permissions.

Make a working copy of the eks/service-account.yaml file:

     $ cp eks/service-account.yaml.template eks/service-account.yaml

Modify the yaml file for your deployment, by doing the following:

- Change any rules and resources that are required for your environment. Most users can leave the yaml file as is.

Use your favorite editor to modify the yaml file:

     $ vi eks/service-account.yaml

Create the ServiceAccount, ClusterRole, and ClusterRoleBinding

Run the following command to create the ServiceAccount, ClusterRole, and ClusterRoleBinding:

     $ kubectl apply -f eks/service-account.yaml

### d. Configure the CSI Driver ConfigMap

The Local Volume Static Provisioner CSI driver uses a Kubernetes ConfigMap to know where to look for mounted EC2 NVMe instance store volumes and how to expose them as PVs. Use a ConfigMap yaml file that specifies where the Local Volume Static Provisioner should look for mounted NVMe instance store volumes in the /mnt/fast-disk directory.

Kubernetes StorageClass specifies a type of storage available in the cluster. The manifest includes a new StorageClass of fast-disks to identify that the PVs relate to NVMe instance store volumes.

Make a working copy of the eks/config-map.yaml file:

     $ cp eks/config-map.yaml.template eks/config-map.yaml

Modify the yaml file for your deployment, by doing the following:

- Most users can leave the yaml file as is.

Use your favorite editor to modify the yaml file:

     $ vi eks/config-map.yaml

Then run the following command to create the StorageClass and ConfigMap.

     $ kubectl apply -f eks/config-map.yaml

### e. Deploy the CSI Driver as a DaemonSet

The Local Volume Static Provisioner CSI Driver runs on each EKS node needing its NVMe instance store volumes exposed as Kubernetes PVs. Often Kubernetes clusters have multiple instance types in the cluster, where some nodes might not have NVMe instance store volumes. The DaemonSet in the following manifest specifies a nodeAffinity selector to only schedule the DaemonSet on an Amazon EKS node with a label of fast-disk-node and corresponding value of either pv-raid or pv-nvme.

Make a working copy of the eks/csi-driver-daemon-set.yaml file:

     $ cp eks/csi-driver-daemon-set.yaml.template eks/csi-driver-daemon-set.yaml

Modify the yaml file for your deployment, by doing the following:

- Most users can leave the yaml file as is.

Use your favorite editor to modify the yaml file:

     $ vi eks/ccsi-driver-daemon-set.yaml

Then run the following command to create the StorageClass and ConfigMap.

     $ kubectl apply -f eks/csi-driver-daemon-set.yaml

To see the daemon set running on each eks node, use the following command:

     $ kubectl get daemonset --namespace=kube-system
       NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
       aws-node                   3         3         3       3            3           <none>          102m
       kube-proxy                 3         3         3       3            3           <none>          102m
       local-volume-provisioner   3         3         3       3            3           <none>          30s

To see the persistent volumes that were created by the daemon set, use the following command:

    $ kubectl get pv
      NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
      local-pv-44b6b86e   274Gi      RWO            Retain           Available           fast-disks              2m16s
      local-pv-4c0706b3   274Gi      RWO            Retain           Available           fast-disks              2m16s
      local-pv-505a86f    274Gi      RWO            Retain           Available           fast-disks              2m16s
      local-pv-620113e1   274Gi      RWO            Retain           Available           fast-disks              2m16s
      local-pv-7eda114c   274Gi      RWO            Retain           Available           fast-disks              2m16s
      local-pv-bba1b1da   274Gi      RWO            Retain           Available           fast-disks              2m16s

### f. (Optional) Install the Kubernetes Autoscaler

Cluster Autoscaler is used for automatically adjusting the size of your Kubernetes cluster based on the current resource demands, optimizing resource utilization and cost.

Make a working copy of the autoscaler-helm-values.yaml file:

     $ cp eks/eks-cluster.yaml.template eks/eks-cluster.yaml

Modify the yaml file for your deployment, by doing the following:

- Change the AWS region to match the region you specified in Step 2 above.
- Added a service account if you are using one.

Use your favorite editor to modify the autoscaler-helm-values.yaml file:

     $ vi eks/autoscaler-helm-values.yaml

Then, enable the autoscaler help chart to be used to deploy the autoscaler:

     $ helm repo add autoscaler https://kubernetes.github.io/autoscaler

Finally, deploy the autoscaler using the helm chart:

     $ helm install nodescaler autoscaler/cluster-autoscaler \
          --namespace kube-system \
          --values autoscaler-helm-values.yaml --debug

---

Please direct questions or comments to greg.palme@alluxio.com



