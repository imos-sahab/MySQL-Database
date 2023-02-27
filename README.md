# MySQL-Database

Deploy MySQL
The example MySQL deployment consists of a ConfigMap, two Services, and a StatefulSet.

Create a ConfigMap
Create the ConfigMap from the following YAML configuration file:

apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
data:
  primary.cnf: |
    # Apply this config only on the primary.
    [mysqld]
    log-bin    
  replica.cnf: |
    # Apply this config only on replicas.
    [mysqld]
    super-read-only    


kubectl apply -f https://k8s.io/examples/application/mysql/mysql-configmap.yaml
This ConfigMap provides my.cnf overrides that let you independently control configuration on the primary MySQL server and its replicas. In this case, you want the primary server to be able to serve replication logs to replicas and you want replicas to reject any writes that don't come via replication.

There's nothing special about the ConfigMap itself that causes different portions to apply to different Pods. Each Pod decides which portion to look at as it's initializing, based on information provided by the StatefulSet controller.

Create Services
Create the Services from the following YAML configuration file:

application/mysql/mysql-services.yaml Copy application/mysql/mysql-services.yaml to clipboard
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the primary: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
    readonly: "true"
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-services.yaml
The headless Service provides a home for the DNS entries that the StatefulSet controllers creates for each Pod that's part of the set. Because the headless Service is named mysql, the Pods are accessible by resolving <pod-name>.mysql from within any other Pod in the same Kubernetes cluster and namespace.

The client Service, called mysql-read, is a normal Service with its own cluster IP that distributes connections across all MySQL Pods that report being Ready. The set of potential endpoints includes the primary MySQL server and all replicas.

Note that only read queries can use the load-balanced client Service. Because there is only one primary MySQL server, clients should connect directly to the primary MySQL Pod (through its DNS entry within the headless Service) to execute writes.

Create the StatefulSet
Finally, create the StatefulSet from the following YAML configuration file:

application/mysql/mysql-statefulset.yaml Copy application/mysql/mysql-statefulset.yaml to clipboard
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
      app.kubernetes.io/name: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
        app.kubernetes.io/name: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi          
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql          
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing replica. (Need to remove the tailing semicolon!)
            cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_slave_info xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from primary. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm -f xtrabackup_binlog_info xtrabackup_slave_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            mysql -h 127.0.0.1 \
                  -e "$(<change_master_to.sql.in), \
                          MASTER_HOST='mysql-0.mysql', \
                          MASTER_USER='root', \
                          MASTER_PASSWORD='', \
                          MASTER_CONNECT_RETRY=10; \
                        START SLAVE;" || exit 1
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"          
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-statefulset.yaml
You can watch the startup progress by running:

kubectl get pods -l app=mysql --watch
After a while, you should see all 3 Pods become Running:

NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          2m
mysql-1   2/2       Running   0          1m
mysql-2   2/2       Running   0          1m
Press Ctrl+C to cancel the watch.

Note: If you don't see any progress, make sure you have a dynamic PersistentVolume provisioner enabled, as mentioned in the prerequisites.
This manifest uses a variety of techniques for managing stateful Pods as part of a StatefulSet. The next section highlights some of these techniques to explain what happens as the StatefulSet creates Pods.

Understanding stateful Pod initialization
The StatefulSet controller starts Pods one at a time, in order by their ordinal index. It waits until each Pod reports being Ready before starting the next one.

In addition, the controller assigns each Pod a unique, stable name of the form <statefulset-name>-<ordinal-index>, which results in Pods named mysql-0, mysql-1, and mysql-2.

The Pod template in the above StatefulSet manifest takes advantage of these properties to perform orderly startup of MySQL replication.

Generating configuration
Before starting any of the containers in the Pod spec, the Pod first runs any init containers in the order defined.

The first init container, named init-mysql, generates special MySQL config files based on the ordinal index.

The script determines its own ordinal index by extracting it from the end of the Pod name, which is returned by the hostname command. Then it saves the ordinal (with a numeric offset to avoid reserved values) into a file called server-id.cnf in the MySQL conf.d directory. This translates the unique, stable identity provided by the StatefulSet into the domain of MySQL server IDs, which require the same properties.

The script in the init-mysql container also applies either primary.cnf or replica.cnf from the ConfigMap by copying the contents into conf.d. Because the example topology consists of a single primary MySQL server and any number of replicas, the script assigns ordinal 0 to be the primary server, and everyone else to be replicas. Combined with the StatefulSet controller's deployment order guarantee, this ensures the primary MySQL server is Ready before creating replicas, so they can begin replicating.

Cloning existing data
In general, when a new Pod joins the set as a replica, it must assume the primary MySQL server might already have data on it. It also must assume that the replication logs might not go all the way back to the beginning of time. These conservative assumptions are the key to allow a running StatefulSet to scale up and down over time, rather than being fixed at its initial size.

The second init container, named clone-mysql, performs a clone operation on a replica Pod the first time it starts up on an empty PersistentVolume. That means it copies all existing data from another running Pod, so its local state is consistent enough to begin replicating from the primary server.

MySQL itself does not provide a mechanism to do this, so the example uses a popular open-source tool called Percona XtraBackup. During the clone, the source MySQL server might suffer reduced performance. To minimize impact on the primary MySQL server, the script instructs each Pod to clone from the Pod whose ordinal index is one lower. This works because the StatefulSet controller always ensures Pod N is Ready before starting Pod N+1.

Starting replication
After the init containers complete successfully, the regular containers run. The MySQL Pods consist of a mysql container that runs the actual mysqld server, and an xtrabackup container that acts as a sidecar.

The xtrabackup sidecar looks at the cloned data files and determines if it's necessary to initialize MySQL replication on the replica. If so, it waits for mysqld to be ready and then executes the CHANGE MASTER TO and START SLAVE commands with replication parameters extracted from the XtraBackup clone files.

Once a replica begins replication, it remembers its primary MySQL server and reconnects automatically if the server restarts or the connection dies. Also, because replicas look for the primary server at its stable DNS name (mysql-0.mysql), they automatically find the primary server even if it gets a new Pod IP due to being rescheduled.

Lastly, after starting replication, the xtrabackup container listens for connections from other Pods requesting a data clone. This server remains up indefinitely in case the StatefulSet scales up, or in case the next Pod loses its PersistentVolumeClaim and needs to redo the clone.

Sending client traffic
You can send test queries to the primary MySQL server (hostname mysql-0.mysql) by running a temporary container with the mysql:5.7 image and running the mysql client binary.

kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
Use the hostname mysql-read to send test queries to any server that reports being Ready:

kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
You should get output like this:

Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
To demonstrate that the mysql-read Service distributes connections across servers, you can run SELECT @@server_id in a loop:

kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
You should see the reported @@server_id change randomly, because a different endpoint might be selected upon each connection attempt:

+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2006-01-02 15:04:05 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         102 | 2006-01-02 15:04:06 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2006-01-02 15:04:07 |
+-------------+---------------------+
You can press Ctrl+C when you want to stop the loop, but it's useful to keep it running in another window so you can see the effects of the following steps.

Simulate Pod and Node failure
To demonstrate the increased availability of reading from the pool of replicas instead of a single server, keep the SELECT @@server_id loop from above running while you force a Pod out of the Ready state.

Break the Readiness probe
The readiness probe for the mysql container runs the command mysql -h 127.0.0.1 -e 'SELECT 1' to make sure the server is up and able to execute queries.

One way to force this readiness probe to fail is to break that command:

kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off
This reaches into the actual container's filesystem for Pod mysql-2 and renames the mysql command so the readiness probe can't find it. After a few seconds, the Pod should report one of its containers as not Ready, which you can check by running:

kubectl get pod mysql-2
Look for 1/2 in the READY column:

NAME      READY     STATUS    RESTARTS   AGE
mysql-2   1/2       Running   0          3m
At this point, you should see your SELECT @@server_id loop continue to run, although it never reports 102 anymore. Recall that the init-mysql script defined server-id as 100 + $ordinal, so server ID 102 corresponds to Pod mysql-2.

Now repair the Pod and it should reappear in the loop output after a few seconds:

kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql.off /usr/bin/mysql
Delete Pods
The StatefulSet also recreates Pods if they're deleted, similar to what a ReplicaSet does for stateless Pods.

kubectl delete pod mysql-2
The StatefulSet controller notices that no mysql-2 Pod exists anymore, and creates a new one with the same name and linked to the same PersistentVolumeClaim. You should see server ID 102 disappear from the loop output for a while and then return on its own.

Drain a Node
If your Kubernetes cluster has multiple Nodes, you can simulate Node downtime (such as when Nodes are upgraded) by issuing a drain.

First determine which Node one of the MySQL Pods is on:

kubectl get pod mysql-2 -o wide
The Node name should show up in the last column:

NAME      READY     STATUS    RESTARTS   AGE       IP            NODE
mysql-2   2/2       Running   0          15m       10.244.5.27   kubernetes-node-9l2t
Then, drain the Node by running the following command, which cordons it so no new Pods may schedule there, and then evicts any existing Pods. Replace <node-name> with the name of the Node you found in the last step.

Caution: Draining a Node can impact other workloads and applications running on the same node. Only perform the following step in a test cluster.
# See above advice about impact on other workloads
kubectl drain <node-name> --force --delete-emptydir-data --ignore-daemonsets
Now you can watch as the Pod reschedules on a different Node:

kubectl get pod mysql-2 -o wide --watch
It should look something like this:

NAME      READY   STATUS          RESTARTS   AGE       IP            NODE
mysql-2   2/2     Terminating     0          15m       10.244.1.56   kubernetes-node-9l2t
[...]
mysql-2   0/2     Pending         0          0s        <none>        kubernetes-node-fjlm
mysql-2   0/2     Init:0/2        0          0s        <none>        kubernetes-node-fjlm
mysql-2   0/2     Init:1/2        0          20s       10.244.5.32   kubernetes-node-fjlm
mysql-2   0/2     PodInitializing 0          21s       10.244.5.32   kubernetes-node-fjlm
mysql-2   1/2     Running         0          22s       10.244.5.32   kubernetes-node-fjlm
mysql-2   2/2     Running         0          30s       10.244.5.32   kubernetes-node-fjlm
And again, you should see server ID 102 disappear from the SELECT @@server_id loop output for a while and then return.

Now uncordon the Node to return it to a normal state:

kubectl uncordon <node-name>
Scaling the number of replicas
When you use MySQL replication, you can scale your read query capacity by adding replicas. For a StatefulSet, you can achieve this with a single command:

kubectl scale statefulset mysql  --replicas=5
Watch the new Pods come up by running:

kubectl get pods -l app=mysql --watch
Once they're up, you should see server IDs 103 and 104 start appearing in the SELECT @@server_id loop output.

You can also verify that these new servers have the data you added before they existed:

kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
Scaling back down is also seamless:

kubectl scale statefulset mysql --replicas=3
Note:
Although scaling up creates new PersistentVolumeClaims automatically, scaling down does not automatically delete these PVCs.

This gives you the choice to keep those initialized PVCs around to make scaling back up quicker, or to extract data before deleting them.

You can see this by running:

kubectl get pvc -l app=mysql
Which shows that all 5 PVCs still exist, despite having scaled the StatefulSet down to 3:

NAME           STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
data-mysql-0   Bound     pvc-8acbf5dc-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-1   Bound     pvc-8ad39820-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-2   Bound     pvc-8ad69a6d-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-3   Bound     pvc-50043c45-b1c5-11e6-93fa-42010a800002   10Gi       RWO           2m
data-mysql-4   Bound     pvc-500a9957-b1c5-11e6-93fa-42010a800002   10Gi       RWO           2m
If you don't intend to reuse the extra PVCs, you can delete them:

kubectl delete pvc data-mysql-3
kubectl delete pvc data-mysql-4
Cleaning up
Cancel the SELECT @@server_id loop by pressing Ctrl+C in its terminal, or running the following from another terminal:

kubectl delete pod mysql-client-loop --now
Delete the StatefulSet. This also begins terminating the Pods.

kubectl delete statefulset mysql
Verify that the Pods disappear. They might take some time to finish terminating.

kubectl get pods -l app=mysql
You'll know the Pods have terminated when the above returns:

No resources found.
Delete the ConfigMap, Services, and PersistentVolumeClaims.

kubectl delete configmap,service,pvc -l app=mysql
If you manually provisioned PersistentVolumes, you also need to manually delete them, as well as release the underlying resources. If you used a dynamic provisioner, it automatically deletes the PersistentVolumes when it sees that you deleted the PersistentVolumeClaims. Some dynamic provisioners (such as those for EBS and PD) also release the underlying resources upon deleting the PersistentVolumes.

What's next
Learn more about scaling a StatefulSet.
Learn more about debugging a StatefulSet.
Learn more about deleting a StatefulSet.
Learn more about force deleting StatefulSet Pods.
Look in the Helm Charts repository for other stateful application examples.
  
  
  Scale a StatefulSet
This task shows how to scale a StatefulSet. Scaling a StatefulSet refers to increasing or decreasing the number of replicas.

Before you begin
StatefulSets are only available in Kubernetes version 1.5 or later. To check your version of Kubernetes, run kubectl version.

Not all stateful applications scale nicely. If you are unsure about whether to scale your StatefulSets, see StatefulSet concepts or StatefulSet tutorial for further information.

You should perform scaling only when you are confident that your stateful application cluster is completely healthy.

Scaling StatefulSets
Use kubectl to scale StatefulSets
First, find the StatefulSet you want to scale.

kubectl get statefulsets <stateful-set-name>
Change the number of replicas of your StatefulSet:

kubectl scale statefulsets <stateful-set-name> --replicas=<new-replicas>
Make in-place updates on your StatefulSets
Alternatively, you can do in-place updates on your StatefulSets.

If your StatefulSet was initially created with kubectl apply, update .spec.replicas of the StatefulSet manifests, and then do a kubectl apply:

kubectl apply -f <stateful-set-file-updated>
Otherwise, edit that field with kubectl edit:

kubectl edit statefulsets <stateful-set-name>
Or use kubectl patch:

kubectl patch statefulsets <stateful-set-name> -p '{"spec":{"replicas":<new-replicas>}}'
Troubleshooting
Scaling down does not work right
You cannot scale down a StatefulSet when any of the stateful Pods it manages is unhealthy. Scaling down only takes place after those stateful Pods become running and ready.

If spec.replicas > 1, Kubernetes cannot determine the reason for an unhealthy Pod. It might be the result of a permanent fault or of a transient fault. A transient fault can be caused by a restart required by upgrading or maintenance.

If the Pod is unhealthy due to a permanent fault, scaling without correcting the fault may lead to a state where the StatefulSet membership drops below a certain minimum number of replicas that are needed to function correctly. This may cause your StatefulSet to become unavailable.

If the Pod is unhealthy due to a transient fault and the Pod might become available again, the transient error may interfere with your scale-up or scale-down operation. Some distributed databases have issues when nodes join and leave at the same time. It is better to reason about scaling operations at the application level in these cases, and perform scaling only when you are sure that your stateful application cluster is completely healthy.
  
  
  Delete a StatefulSet
This task shows you how to delete a StatefulSet.

Before you begin
This task assumes you have an application running on your cluster represented by a StatefulSet.
Deleting a StatefulSet
You can delete a StatefulSet in the same way you delete other resources in Kubernetes: use the kubectl delete command, and specify the StatefulSet either by file or by name.

kubectl delete -f <file.yaml>
kubectl delete statefulsets <statefulset-name>
You may need to delete the associated headless service separately after the StatefulSet itself is deleted.

kubectl delete service <service-name>
When deleting a StatefulSet through kubectl, the StatefulSet scales down to 0. All Pods that are part of this workload are also deleted. If you want to delete only the StatefulSet and not the Pods, use --cascade=orphan. For example:

kubectl delete -f <file.yaml> --cascade=orphan
By passing --cascade=orphan to kubectl delete, the Pods managed by the StatefulSet are left behind even after the StatefulSet object itself is deleted. If the pods have a label app.kubernetes.io/name=MyApp, you can then delete them as follows:

kubectl delete pods -l app.kubernetes.io/name=MyApp
Persistent Volumes
Deleting the Pods in a StatefulSet will not delete the associated volumes. This is to ensure that you have the chance to copy data off the volume before deleting it. Deleting the PVC after the pods have terminated might trigger deletion of the backing Persistent Volumes depending on the storage class and reclaim policy. You should never assume ability to access a volume after claim deletion.

Note: Use caution when deleting a PVC, as it may lead to data loss.
Complete deletion of a StatefulSet
To delete everything in a StatefulSet, including the associated pods, you can run a series of commands similar to the following:

grace=$(kubectl get pods <stateful-set-pod> --template '{{.spec.terminationGracePeriodSeconds}}')
kubectl delete statefulset -l app.kubernetes.io/name=MyApp
sleep $grace
kubectl delete pvc -l app.kubernetes.io/name=MyApp
In the example above, the Pods have the label app.kubernetes.io/name=MyApp; substitute your own label as appropriate.

Force deletion of StatefulSet pods
If you find that some pods in your StatefulSet are stuck in the 'Terminating' or 'Unknown' states for an extended period of time, you may need to manually intervene to forcefully delete the pods from the apiserver. This is a potentially dangerous task. Refer to Force Delete StatefulSet Pods for details.
