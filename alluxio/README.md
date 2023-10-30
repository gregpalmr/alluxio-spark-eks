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

Use the helm repo commands to remove any older Alluxio Helm chart repos:

     $ helm repo list
     NAME              URL
     alluxio-charts    https://alluxio-charts.storage.googleapis.com/openSource/2.6.2

     $ helm repo rm alluxio-charts
     "alluxio-charts" has been removed from your repositories

Use the helm repo command to add the Alluxio Helm chart to the current repo list:

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

- (Optional) If you are using the Enterprise Edition of Alluxio, replace PUT_YOUR_LICENSE_BASE64_VALUE_HERE with your BASE64 version of the license key and uncomment the line that begins with "#license:". Use the following command to get the BASE64 version of your license key:
     - $ cat /path/to/license.json | base64 |  tr -d "\n"
- Alluxio requires a root under file system (UFS) but you can add other UFSs later. Replace PUT_YOUR_S3_BUCKET_NAME_HERE with the name of an S3 bucket that Alluxio can use as the root UFS. If you do not have instance IAM roles configured, you can specify the accessKeyId and secretKey by changing PUT_YOUR_AWS_ACCESS_KEY_ID_HERE and PUT_YOUR_AWS_SECRET_KEY_HERE. If you do have instance roles configured, keep those commented out.
- Change the jvmOptions values for the master, worker, job_master, and job_worker pods, as needed. Alluxio provides guidence on tuning the Alluxio master node and worker node JVMs here: 
     - https://docs.alluxio.io/os/user/stable/en/administration/Performance-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/administration/Scalability-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/kubernetes/Running-Alluxio-On-Kubernetes.html?q=JAVA_OPTS
- The default values in the template file assume that the EKS nodes can provide:
     - Alluxio master node pods:
          -  4 CPU cores
          - 24 GB of Java Heap and 10 GB of direct memory 
          -  2 300 GB (unformatted) NVMe persistent volumes for cache storage (see: "quota: 270GB,270GB")
     - Alluxio worker node pods:
          -  8 CPU cores
          - 32 GB of Java Heap and 10 GB of direct memory 
          -  2 600 GB (unformatted) NVMe persistent volumes for cache storage (see: "quota: 547GB,547GB")
- If you don't want to install Alluxio on all the EKS nodes, you can define a toleration that will cause Alluxio pods not to get scheduled on specific nodes. Change PUT_YOUR_TOLERATION_KEY_HERE and PUT_YOUR_TOLERATION_VALUE_HERE, and uncomment that section.

Use your favorite editor to modify the Alluxio-helm-values.yaml file:

     $ vi alluxio/alluxio-helm-values.yaml

### c. Deploy Alluxio pods with the Helm chart

With the helm values yaml file configured for Alluxio master nodes and worker nodes (and persistent storage for each), deploy the Alluxio pods using the Helm chart command. Use the command:

     $ helm install alluxio -f alluxio/alluxio-helm-values.yaml alluxio-charts/alluxio

### d. Verify the Alluxio cluster deployed successfully

If you have an issue deploying the helm chart, you can delete it and try again. Delete it with the command:

     $ helm delete alluxio

Check to see if the Alluxio master and worker pods are running with the command:

     $ kubectl get pods
     NAME                   READY   STATUS    RESTARTS   AGE
     alluxio-master-0       2/2     Running   0          70s
     alluxio-master-1       2/2     Running   0          70s
     alluxio-master-2       2/2     Running   0          70s
     alluxio-worker-ktrx7   0/2     Pending   0          70s
     alluxio-worker-wwp7z   0/2     Pending   0          70s
     alluxio-worker-x2rkp   0/2     Pending   0          70s

If you see some pods stuck in the Pending status, you can view the log files for the pod to try to understand what might be keeping the pod from successfully running. Use the command:

     $ kubectl describe pod alluxio-worker-x2rkp

The command will display several screens worth of information about the pod. The master pods are made up of two containers, master, and job_master. The worker pods are made up of two containers, worker and job_worker. The last few lines will usually show why a pod is stuck in Pending mode. Here is an example message.

     Events:
       Type     Reason            Age    From               Message
       ----     ------            ----   ----               -------
       Warning  FailedScheduling  3m28s  default-scheduler  0/6 nodes are available: 6 persistentvolumeclaim "alluxio-nvme0" not found. preemption: 0/6 nodes are available: 6 Preemption is not helpful for scheduling.

Based on the message, you may have to tune the configuration of your EKS cluster and the resources available, and retry.

You can also get the log entries for a pod, using the command:

     $ kubectl logs  alluxio-master-0
     
Once all the Alluxio master and worker pods are running, you can verify that they have successfully attached the persistent volumes using the commands:

     $ kubectl get pv

     $ kubectl get pvc

### e. Format the Alluxio master node journal

Before using Alluxio, the master nodes must format their journal storage. The master Pods in the StatefulSet use an initContainer to format the journal on startup. The initContainer is switched on by property: journal.format.runFormat=true. By default, the journal is not formatted when the master starts.

Use the following helm upgrade command to format the journals:

     $ helm upgrade alluxio -f alluxio/alluxio-helm-values.yaml --set journal.format.runFormat=true alluxio-charts/alluxio

### f. Run Alluxio CLI commands

You can run Alluxio CLI commands from within the Alluxio master pods. Use the following kubectl command to open a shell session in on e of the Alluxio master pods:

     $ kubectl exec -ti alluxio-master-0 -- /bin/bash

To view the Alluxio properties that were configured for the Alluxio master process, use the command:

     $ ps -ef | grep alluxio

You will see all of the properties that were defined in the Helm chart values.yaml file as -D options to the Java JVM used to run the master process. Like this:

     /usr/lib/jvm/java-1.8.0-openjdk/bin/java -cp /opt/alluxio-2.9.3/conf/::/opt/alluxio-2.9.3/assembly/alluxio-server-2.9.3.jar -Dalluxio.logger.type=Console,MASTER_LOGGER -Dalluxio.master.audit.logger.type=MASTER_AUDIT_LOGGER -Dalluxio.master.journal.type=UFS -Dalluxio.master.journal.folder=/journal -Dalluxio.job.master.job.capacity=200000 -Dalluxio.job.master.network.max.inbound.message.size=100MB -Dalluxio.job.master.worker.timeout=300sec ... -Dalluxio.master.hostname= -Xms24g -Xmx24g -XX:MaxDirectMemorySize=10g -XX:MetaspaceSize=256M alluxio.master.AlluxioMaster

You can run the Alluxio CLI command "alluxio fsadmin report" to see an overview of the Alluxio cluster. Like this:

     $ alluxio fsadmin report

You can test the root under file system (UFS) with the built in test:

     $ alluxio runTests

---

Please direct questions or comments to greg.palme@alluxio.com