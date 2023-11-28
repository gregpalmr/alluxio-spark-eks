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

## Step 2. Deploy an EKS cluster

### a. Modify the EKS cluster yaml file.

The EKS yaml file is used to deploy the EKS cluster and the Kubernetes nodes that will host the Spark pods and the Alluxio pods. 

Use an EC2 instance type that has enough NVMe storage to allow Alluxio to cache the needed data from the S3 buckets. For example, the m5d.4xlarge instance type has two 300 GB NVMe volumes, the m5d.8xlarge has 2 600 GB NVMe volumes.

Make a working copy of the eks-cluster.yaml file that will be used to launch the EKS cluster. If you are deploying an EKS cluster for a PRODUCTION environment, then it is recommended to use the following command to create a PROD oriented EKS cluster with large enough EC2 instances to support an average Alluxio cluster configuration for a production environment, including EC2 types with NVMe storage for Alluxio master node metadata storage and for Alluxio worker node cache storage:

     $ cp eks/eks-cluster-prod.yaml.template eks/eks-cluster.yaml

If you are deploying a simple DEV environment, use the command to deploy a small EKS cluster using just 2 (by default) m5.xlarge instance types:

     $ cp eks/eks-cluster-dev.yaml.template eks/eks-cluster.yaml

Modify the yaml file for your deployment. Use your favorite editor to modify the yaml file:

     $ vi eks/eks-cluster.yaml

- To restrict access to your EKS cluster, replace PUT_YOUR_YOUR_PUBLIC_IP_HERE with your computer's public IP address. On Linux or MacOS, you can run the following command to get your server's public IP address:
     - $ curl ifconfig.me
- Change the references to the AWS region and availability zones. Change us-west-1, us-west-1a and us-west-1b as needed. 
- Add a reference to your private SSH key, 
- If you want to be able to SSH into the EC2 instances, replace both occurrences PUT_YOUR_PATH_TO_PUB_SSH_KEY_HERE with the path to your public ssh key. If you don't have an SSH key pair, you can generate one with the command:
     - $ ssh-keygen -t rsa -N '' -f ./eks_ssh_key <<< y
- Change the managedNodeGroups section to specify the EC2 instanceType configuration. Use m5d.8xlarge to support higher client side loads and larger cache storage requirements and use m5d.4xlarge to support lower client side loads and smaller cache storage requirements. It defaults to using the smaller m5d.4xlarge instance type.
- Change the number of worker nodes in your EKS cluster to support the workloads you are running. For an Alluxio PROD cluster, you will require a minimum of 3 master nodes and 3 worker nodes. Change PUT_YOUR_MAX_WORKER_COUNT_HERE to the maximum number of worker nodes and change PUT_YOUR_DESIRED_WORKER_COUNT_HERE to your desired number of worker nodes.
- Change the EC2 instance types for the master nodes and worker nodes. Make sure you choose instance types that have enough cpu vcores, memory and NVMe storage to support running both Spark pods and Alluxio pods and that allow Alluxio to cache enough data on NVMe storage to improve performance. Alluxio requirements and tuning best practives can be found here:
     - https://docs.alluxio.io/os/user/stable/en/deploy/Requirements.html
     - https://docs.alluxio.io/os/user/stable/en/administration/Performance-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/administration/Scalability-Tuning.html

### b. Deploy the EKS cluster

Use the eksctl command line tool to launch the EKS cluster:

     $ eksctl create cluster -f eks/eks-cluster.yaml
        2023-10-27 12:16:56 [ℹ]  creating addon
        2023-10-27 12:17:50 [ℹ]  addon "kube-proxy" active
        2023-10-27 12:17:52 [ℹ]  kubectl command should work with "/Users/greg/.kube/config", try 'kubectl get nodes'
        2023-10-27 12:17:52 [✔]  EKS cluster "eks-spark-alluxio" in "us-west-1" region is ready

After the eksctl tool reports that the cluster was created, you can display the EKS nodes and other cluster information using the commands:

     $ eksctl get clusters --region=us-west-1
        NAME			REGION		EKSCTL CREATED
        eks-spark-alluxio	us-west-1	True

     $ eksctl get nodegroups --region=us-west-1 --cluster=eks-spark-alluxio
        CLUSTER			NODEGROUP	STATUS	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID	ASG NAME					TYPE
        eks-spark-alluxio	master		ACTIVE	2023-10-27T16:13:42Z	3		3		3			m5d.4xlarge	AL2_x86_64	eks-master-1ec5b8f4-d323-5829-a3ed-a47424e3ad70	managed
        eks-spark-alluxio	worker		ACTIVE	2023-10-27T16:13:38Z	3		3		3			m5d.4xlarge	AL2_x86_64	eks-worker-08c5b8f4-cba5-5b99-ae13-0d53ac57839c	managed

     $ kubectl get nodes -o wide
        NAME                                           STATUS   ROLES    AGE   VERSION                INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
        ip-192-168-0-54.us-west-1.compute.internal     Ready    <none>   13m   v1.25.13-eks-43840fb   192.168.0.54     13.52.182.115    Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
        ip-192-168-1-177.us-west-1.compute.internal    Ready    <none>   14m   v1.25.13-eks-43840fb   192.168.1.177    54.183.71.208    Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
        ip-192-168-19-221.us-west-1.compute.internal   Ready    <none>   14m   v1.25.13-eks-43840fb   192.168.19.221   54.241.86.29     Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
        ip-192-168-24-236.us-west-1.compute.internal   Ready    <none>   13m   v1.25.13-eks-43840fb   192.168.24.236   13.52.102.101    Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
        ip-192-168-27-14.us-west-1.compute.internal    Ready    <none>   14m   v1.25.13-eks-43840fb   192.168.27.14    3.101.144.207    Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
        ip-192-168-4-241.us-west-1.compute.internal    Ready    <none>   13m   v1.25.13-eks-43840fb   192.168.4.241    54.183.161.177   Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19

     $ kubectl describe node ip-192-168-0-54.us-west-1.compute.internal

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

The Local Volume Static Provisioner CSI driver uses a Kubernetes ConfigMap to specify where to look for mounted EC2 NVMe instance store volumes and how to expose them as PVs. It will search for mounted NVMe instance store volumes in the /mnt/fast-disk directory.

Kubernetes StorageClass specifies a type of storage available in the cluster. The config map manifest file includes a StorageClass of fast-disks to identify that the PVs relate to NVMe instance store volumes.

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

     $ vi eks/csi-driver-daemon-set.yaml

Then run the following command to create the StorageClass and ConfigMap.

     $ kubectl apply -f eks/csi-driver-daemon-set.yaml

To see the daemon set running on each eks node, use the following command:

     $ kubectl get daemonset --namespace=kube-system
        NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
        aws-node                   6         6         6       6            6           <none>          30m
        kube-proxy                 6         6         6       6            6           <none>          30m
        local-volume-provisioner   6         6         6       6            6           <none>          23s

To see the persistent volumes that were created by the daemon set, use the following command:

    $ kubectl get pv
        NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
        local-pv-40f2bb5    274Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-32acd0e5   274Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-36e2c318   274Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-68d1ddd7   274Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-91349fa8   274Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-cc50917e   274Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-45f8cecc   549Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-553062c6   549Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-9f839586   549Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-ae0bb037   549Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-c205f533   549Gi      RWO            Retain           Available           fast-disks              41s
        local-pv-c4d9cb37   549Gi      RWO            Retain           Available           fast-disks              41s

### f. Create a "standard" storage class

Alluxio can provide "short circuit" reads by sensing when a client workload (such as a Spark executor pod) is running on the same EKS node as the Alluxio worker pod. It senses this using an EKS domain socket PVC which needs to be part of a storage class. Create the standard storage class for the alluxio-worker-domain-socket PVC to use:

     kubectl apply -f eks/standard-storage-class.yaml

### g. Deploy the Kubernetes metric server pod

To use the "kubectl top node" or "kubectl top pod" command, the Kubernetes metrics-server service must be deployed. Deploy it with the command:

     kubectl apply -f eks/metrics-server.yaml

Wait a few minutes for the service to complete the deployment steps and then run the command to test it:

     kubectl top node

     kubectl top pod --all-namespaces

### h. (Optional) Install the Kubernetes Autoscaler

Cluster Autoscaler is used for automatically adjusting the size of your Kubernetes cluster based on the current resource demands, optimizing resource utilization and cost.

Make a working copy of the autoscaler-helm-values.yaml file:

     $ cp eks/autoscaler-helm-values.yaml.template eks/autoscaler-helm-values.yaml

Modify the yaml file for your deployment, by doing the following:

- Change the AWS region to match the region you specified in Step 2 above.
- Added a service account if you are using one.

Use your favorite editor to modify the autoscaler-helm-values.yaml file:

     $ vi eks/autoscaler-helm-values.yaml

Then, enable the autoscaler help chart to be used to deploy the autoscaler:

     $ helm repo add autoscaler https://kubernetes.github.io/autoscaler
        "autoscaler" has been added to your repositories

Finally, deploy the autoscaler using the helm chart:

     $ helm install nodescaler autoscaler/cluster-autoscaler \
          --namespace kube-system \
          --values eks/autoscaler-helm-values.yaml --debug

Verify that cluster-autoscaler has started, run the command:

     $ kubectl --namespace=kube-system get pods -l "app.kubernetes.io/name=aws-cluster-autoscaler,app.kubernetes.io/instance=nodescaler"
        NAME                                                 READY   STATUS    RESTARTS   AGE
        nodescaler-aws-cluster-autoscaler-7f85d89688-x9lm2   1/1     Running   0          29s

### i. Destroy the EKS cluster

To destroy the EKS cluster (and all the Alluxio and Spark pods running on it), use the following command:

     $ helm delete --namespace kube-system nodescaler

     $ eksctl delete cluster --region us-west-1 --name=eks-spark-alluxio

CAUTION: All persistent volumes will be release and any data on them will be lost.

### Continue with the next step:

[Deploy Alluxio on the EKS Cluster](../alluxio/README.md)

---

Please direct questions or comments to greg.palmer@alluxio.com



