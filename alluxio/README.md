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

Create the Alluxio Helm chart values file that will be used by the Helm chart to deploy Alluxio on the EKS cluster. 

Make a working copy of the alluxio-helm-values.yaml file:

#### DEV

If you are just experimenting with Alluxio and will not be doing performance testing or at-scale testing, you may want to use the "dev" version of the Helm chart values. This version of the Helm values only deploy 1 master node (no master failover) and do not use persistent volumes to store metadata and cache real data. Instead, it uses emptyDir storage type to store metadata and a RAM disk to cache files. Copy the template like this:

     $ cp alluxio/alluxio-helm-values-dev.yaml.template alluxio/alluxio-helm-values-dev.yaml

#### PROD

If you are planning on supporting production workloads, then you should use the "prod" version of the Helm values because it deploys 3 master pods with failover, stores master node metadata on persistent volumes and stores cached data on persistent volumes. Copy the template like this:

     $ cp alluxio/alluxio-helm-values-prod.yaml.template alluxio/alluxio-helm-values-prod.yaml

Modify the yaml file for your Alluxio deployment, by doing the following:

- (Optional) If you are using the Enterprise Edition of Alluxio, replace PUT_YOUR_LICENSE_BASE64_VALUE_HERE with your BASE64 version of the license key and uncomment the line that begins with "#license:". Use the following command to get the BASE64 version of your license key:
     - $ cat /path/to/license.json | base64 |  tr -d "\n"
- Alluxio requires a root under file system (UFS) but you can add other UFSs later. Replace PUT_YOUR_S3_BUCKET_NAME_HERE with the name of an S3 bucket that Alluxio can use as the root UFS. If you do not have instance IAM roles configured, you can specify the accessKeyId and secretKey by changing PUT_YOUR_AWS_ACCESS_KEY_ID_HERE and PUT_YOUR_AWS_SECRET_KEY_HERE. If you do have instance roles configured, keep those commented out.
- Change the jvmOptions values for the master, worker, job_master, and job_worker pods, as needed. Alluxio provides guidence on tuning the Alluxio master node and worker node JVMs here: 
     - https://docs.alluxio.io/os/user/stable/en/administration/Performance-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/administration/Scalability-Tuning.html
     - https://docs.alluxio.io/os/user/stable/en/kubernetes/Running-Alluxio-On-Kubernetes.html?q=JAVA_OPTS
- For the PROD version of the Helm chart, the default values in the template file assume that the EKS nodes can provide:
     - Alluxio master node pods:
          -  4 CPU cores
          - 24 GB of Java Heap and 10 GB of direct memory 
          -  2 300 GB (unformatted) NVMe volumes for metadata storage 
     - Alluxio worker node pods:
          -  8 CPU cores
          - 32 GB of Java Heap and 10 GB of direct memory 
          -  2 600 GB (unformatted) NVMe volumes for cache storage 
- If you don't want to install Alluxio on all the EKS nodes, you can define a toleration that will cause Alluxio pods not to get scheduled on specific nodes. Change PUT_YOUR_TOLERATION_KEY_HERE and PUT_YOUR_TOLERATION_VALUE_HERE, and uncomment that section.

Use your favorite editor to modify the Alluxio-helm-values.yaml file:

     $ vi alluxio/alluxio-helm-values-dev.yaml

or

     $ vi alluxio/alluxio-helm-values-prod.yaml

### c. Create the alluxio namespace 

To help organize the Kubernetes cluster, create a namespace for your specific environment. Usethis namespace name on the helm and kubectl commands that you use later. Create the namespace with the command:

     $ kubectl create namespace alluxio

### d. Deploy Alluxio pods with the Helm chart

With the helm values yaml file configured for Alluxio master nodes and worker nodes (and persistent storage for each), deploy the Alluxio pods using the Helm chart command. The first time the Alluxio cluster is deployed, you must format the master node journals, so add the --set journal.format.runFormat=true argument to the command. Use the command:

     $ helm install alluxio --namespace alluxio --set journal.format.runFormat=true \
          -f alluxio/alluxio-helm-values-dev.yaml alluxio-charts/alluxio

or

     $ helm install alluxio --namespace alluxio --set journal.format.runFormat=true \
          -f alluxio/alluxio-helm-values-prod.yaml alluxio-charts/alluxio

### e. Verify the Alluxio cluster deployed successfully

Check to see if the Alluxio master and worker pods are running with the command:

     $ kubectl get pods --namespace alluxio
     NAME                   READY   STATUS    RESTARTS   AGE
     alluxio-master-0       2/2     Running   0          70s
     alluxio-master-1       2/2     Running   0          70s
     alluxio-master-2       2/2     Running   0          70s
     alluxio-worker-ktrx7   0/2     Running   0          70s
     alluxio-worker-wwp7z   0/2     Running   0          70s
     alluxio-worker-x2rkp   0/2     Running   0          70s

If you see some pods stuck in the Pending status, you can view the log files for the pod to try to understand what might be keeping the pod from successfully running. Use the command:

     $ kubectl describe pod --namespace alluxio alluxio-worker-x2rkp

The command will display several screens worth of information about the pod. The master pods are made up of two containers, master, and job_master. The worker pods are made up of two containers, worker and job_worker. The last few lines will usually show why a pod is stuck in Pending mode. Here is an example message.

     Events:
       Type     Reason            Age    From               Message
       ----     ------            ----   ----               -------
       Warning  FailedScheduling  3m28s  default-scheduler  0/6 nodes are available: 6 persistentvolumeclaim "alluxio-nvme0" not found. preemption: 0/6 nodes are available: 6 Preemption is not helpful for scheduling.

Based on the message, you may have to tune the configuration of your EKS cluster and the resources available, and retry.

You can also get the log entries for a pod, using the command:

     $ kubectl logs --namespace alluxio alluxio-master-0
     
Once all the Alluxio master and worker pods are running, you can verify that they have successfully attached the persistent volumes using the commands:

     $ kubectl get pv --namespace alluxio

     $ kubectl get pvc --namespace alluxio

### f. Deploy a service in front of the Alluxio REST API daemonset

Alluxio deploys a daemonset that runs an Alluxio REST API and S3 API proxy on every node. This API is designed for Python programs, Go programs and AWS S3 client applications to interact with Alluxio without having to have any client side jar files present. 

To support access to the Alluxio REST API from applications running within the EKS cluster, deploy a Kubernetes service in front of the Alluxio REST/S3 API using the hostname alias "alluxio-proxy". Note that if you would like to access the Alluxio REST API from outside of the EKS cluster, then you will need to use a load balancer such as Nginx to expose the Alluxio proxy daemonset pods to the enterprise DNS environment.

Copy the service template file like this:

     $ cp alluxio/alluxio/alluxio-rest-api-service.yaml.template alluxio/alluxio-rest-api-service.yaml

Modify the yaml file for your Alluxio deployment, by doing the following:

- Replace PUT_YOUR_ALLUXIO_HELM_CLUSTER_NAME_HERE with the name you used when you deployed the Alluxio pods.

The name is the one you specified with the helm command. If you used the helm command "helm install alluxio-dev", then the CLUSTER_NAME would be changed to "alluxio-dev".

Deploy the service using the command:

     $ kubectl apply --namespace=alluxio -f alluxio/alluxio-rest-api-service.yaml

Very that the service has been deployed and has a cluster wide IP address (the host name is "alluxio-proxy"):

     $ kubectl get services --namespace=alluxio
     NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                    
     alluxio-master-0   ClusterIP   None           <none>        19998/TCP,19999/TCP,20001/TCP,20002/TCP,19200/TCP,20003/TCP  
     alluxio-proxy      ClusterIP   10.100.54.53   <none>        39999/TCP       

You can test that the alluxio-proxy service is working, by issuing a curl command against the API end point. 

First, open a shell session into the alluxio-master-0 pod:

     $ kubectl exec -ti --namespace alluxio --container alluxio-master alluxio-master-0 -- /bin/bash

Then, try to list the Alluxio S3 bucket contents using the curl command (the hostname "alluxio-proxy" is provided by the service):

     $ curl -i \
          -H "Authorization: AWS4-HMAC-SHA256 Credential=alluxio/" \
          -X GET http://alluxio-proxy:39999/api/v1/s3/<alluxio_s3_mount>/

Download an Alluxio S3 object to a local file:

     $ curl -i --output ./myfile.parquet \
          -H "Authorization: AWS4-HMAC-SHA256 Credential=alluxio/" \
          -X GET http://alluxio-proxy:39999/api/v1/s3/<alluxio_s3_mount>/<path to a parquet file>.parquet

     $ ls -al myfile.parquet

Later, we will run Spark jobs that referense the Alluxio S3 REST API using a method similar to this:

     %pyspark
     from pyspark.sql import SparkSession
     
     conf = SparkConf()
     
     conf.set("spark.hadoop.fs.s3a.access.key", "alluxio_user")
     conf.set("spark.hadoop.fs.s3a.secret.key", "NOT NEEDED")
     conf.set("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
     conf.set("spark.hadoop.fs.s3a.endpoint", "http://alluxio-proxy:39999/api/v1/s3")
     conf.set("spark.hadoop.fs.s3a.path.style.access", "true")
     conf.set("spark.hadoop.fs.s3a.connection.ssl.enabled","false")
     
     spark = SparkSession.builder \
         .config(conf=conf) \
         .appName("Spark") \
         .getOrCreate()

     # Loads parquet file from Alluxio into RDD Data Frame
     df = spark.read.parquet("s3a://<alluxio_s3_mount>/<my_data>/")
     
     df.printSchema()  

### g. Disable Alluxio master node journal formatting

Since the argument "--set journal.format.runFormat=true" was used to initially deploy the Alluxio cluster, we must upgrade the deployment using the "helm upgrade" command, and specify the "runFormat=false" argument. This way, if a master node gets restarted by the Kubernetes scheduler, it will not format the existing (and still usable) journal on the persistent storage.

Use the following helm upgrade command to not format the journals:

     $ helm upgrade alluxio --namespace alluxio --set journal.format.runFormat=false \
          -f alluxio/alluxio-helm-values-dev.yaml alluxio-charts/alluxio
or

     $ helm upgrade alluxio --namespace alluxio --set journal.format.runFormat=false \
          -f alluxio/alluxio-helm-values-prod.yaml alluxio-charts/alluxio

### h. Run Alluxio CLI commands

You can start a shell session in a worker node with the command:

     $ kubectl exec -ti --namespace alluxio --container alluxio-worker alluxio-worker-6bwkw -- /bin/bash

And you can view the worker node log files using the commands:

     $ cd /opt/alluxio/logs/
     $ vi worker.log

You can start a shell session in a master node with the command:

     $ kubectl exec -ti --namespace alluxio --container alluxio-master alluxio-master-0 -- /bin/bash

And you can view the master node log files using the commands:

     $ cd /opt/alluxio/logs/
     $ vi master.log

In the master node shell, you can view the Alluxio properties that were configured for the Alluxio master process, use the command:

     $ ps -ef | grep alluxio

You will see all of the properties that were defined in the Helm chart values.yaml file as -D options to the Java JVM used to run the master process. Like this:

     /usr/lib/jvm/java-1.8.0-openjdk/bin/java -cp /opt/alluxio-2.9.3/conf/:/opt/alluxio-2.9.3/assembly/alluxio-server-2.9.3.jar -Dalluxio.logger.type=Console,MASTER_LOGGER -Dalluxio.master.audit.logger.type=MASTER_AUDIT_LOGGER -Dalluxio.master.journal.type=UFS -Dalluxio.master.journal.folder=/journal -Dalluxio.job.master.job.capacity=200000 -Dalluxio.job.master.network.max.inbound.message.size=100MB -Dalluxio.job.master.worker.timeout=300sec ... -Dalluxio.master.hostname= -Xms24g -Xmx24g -XX:MaxDirectMemorySize=10g -XX:MetaspaceSize=256M alluxio.master.AlluxioMaster

You can run the Alluxio CLI command "alluxio fsadmin report" to see an overview of the Alluxio cluster. Like this:

     $ alluxio fsadmin report

It should show an overview of the cluster, like this:

     Alluxio cluster summary:
         Master Address: alluxio-master-0:19998
         Web Port: 19999
         Rpc Port: 19998
         Started: 10-31-2023 18:49:18:932
         Uptime: 0 day(s), 0 hour(s), 1 minute(s), and 19 second(s)
         Version: 2.9.3
         Safe Mode: false
         Zookeeper Enabled: false
         Raft-based Journal: true
         Raft Journal Addresses:
             alluxio-master-0:19200
             alluxio-master-1:19200
             alluxio-master-2:19200
         Live Workers: 3
         Lost Workers: 0
         Total Capacity: 3000.00GB
             Tier: SSD  Size: 3000.00GB
         Used Capacity: 0B
             Tier: SSD  Size: 0B
         Free Capacity: 3000.00GB

You can also view the Alluxio cache storage on each worker node pod by running the command:

     $ alluxio fsadmin report capacity
     Capacity information for all workers:
         Total Capacity: 3000.00GB
             Tier: SSD  Size: 3000.00GB
         Used Capacity: 0B
             Tier: SSD  Size: 0B
         Used Percentage: 0%
         Free Percentage: 100%
     Worker Name      Last Heartbeat   Storage       SSD              Version          Revision
     192.168.16.61    0                capacity      1000.00GB        2.9.3            44b59ac84b0bcef9b268a81481d08da96dc27d58
                                       used          0B (0%)
     192.168.21.17    0                capacity      1000.00GB        2.9.3            44b59ac84b0bcef9b268a81481d08da96dc27d58
                                       used          0B (0%)
     192.168.23.190   0                capacity      1000.00GB        2.9.3            44b59ac84b0bcef9b268a81481d08da96dc27d58
                                       used          0B (0%)

You can test the integration with the root under file system (UFS) using a built in test utility, like this:

     $ alluxio runTests
     ...
     runTest --operation BASIC_NON_BYTE_BUFFER --readType NO_CACHE --writeType ASYNC_THROUGH
     2023-10-31 23:10:37,802 INFO  [main](BasicNonByteBufferOperations.java:93) - writeFile to file /default_tests_files/BASIC_NON_BYTE_BUFFER_NO_CACHE_ASYNC_THROUGH took 17 ms.
     2023-10-31 23:10:37,807 INFO  [main](BasicNonByteBufferOperations.java:126) - readFile file /default_tests_files/BASIC_NON_BYTE_BUFFER_NO_CACHE_ASYNC_THROUGH took 5 ms.
     Passed the test!

NOTE: If you see "permission denied" errors when running the tests, you may not have the correct instance IAM roles to allow the Alluxio pods to access your "root" under file system or UFS. You many need to specify the AWS access key id and secret key as specified in step b. above.

View the created test files with the command:

     $ alluxio fs ls /default_tests_files

Remove the test files with the command:

     $ alluxio fs rm -R /default_tests_files

### i. Destroy the Alluxio cluster

You can destroy the Alluxio master and worker pods and remove the namespace with the commands:

     $ helm delete --namespace alluxio alluxio

     $ kubectl delete --namespace alluxio -f alluxio/alluxio-worker-pvc.yaml

     $ kubectl delete namespace alluxio

### Continue with the next step:

[Deploy Spark on the EKS Cluster](../spark/README.md)

---

Please direct questions or comments to greg.palmer@alluxio.com
