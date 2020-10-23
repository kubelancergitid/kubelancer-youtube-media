# Openshift 4.5 Replacing an unhealthy etcd member
# 

#### Pre-Req:

* Access to the cluster as a user with the cluster-admin role.
* Ensure have taken an etcd backup.


## Identifying an unhealthy etcd member

1. Check the status of the EtcdMembersAvailable status

```bash
        $ oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}'
```
2. Review the output:
        ```
            2 of 3 members are available, ip-10-0-131-183.ec2.internal is unhealthy
        ```
## Determining the state of the unhealthy etcd member

1. Determine if the machine is not running:

```bash
        $ oc get machines -A -ojsonpath='{range .items[*]}{@.status.nodeRef.name}{"\t"}{@.status.providerStatus.instanceState}{"\n"}' | grep -v running
```
        Example output
        ```
            ip-10-0-131-183.ec2.internal  stopped
        ```

2. Determine if the node is not ready

```bash
        $ oc get nodes -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"\t"}{range .spec.taints[*]}{.key}{" "}' | grep unreachable
```
        Example output
        ```
            ip-10-0-131-183.ec2.internal	node-role.kubernetes.io/master node.kubernetes.io/unreachable node.kubernetes.io/unreachable
        ```

#### If the node is still reachable, then check whether the node is listed as NotReady:

```bash
        $ oc get nodes -l node-role.kubernetes.io/master | grep "NotReady"
```
        Example output
    ```
        ip-10-0-131-183.ec2.internal   NotReady   master   122m   v1.18.3
    ```

3. Determine if the etcd Pod is crashlooping.

If the machine is running and the node is ready, then check whether the etcd Pod is crashlooping.

Verify that all master nodes are listed as Ready:

```bash
      $ oc get nodes -l node-role.kubernetes.io/master
```
        Example output
    ```
        NAME                           STATUS   ROLES    AGE     VERSION
        ip-10-0-131-183.ec2.internal   Ready    master   6h13m   v1.18.3
        ip-10-0-164-97.ec2.internal    Ready    master   6h13m   v1.18.3
        ip-10-0-154-204.ec2.internal   Ready    master   6h13m   v1.18.3
    ```
Check whether the status of an etcd Pod is either Error or CrashloopBackoff:
```bash
        $ oc get pods -n openshift-etcd | grep etcd
```
    Example output
```
    etcd-ip-10-0-131-183.ec2.internal                2/3     Error       7          6h9m
    etcd-ip-10-0-164-97.ec2.internal                 3/3     Running     0          6h6m
    etcd-ip-10-0-154-204.ec2.internal                3/3     Running     0          6h6m
```

# Replacing the unhealthy etcd member

## Replacing an unhealthy etcd member whose machine is not running or whose node is not ready

#### Procedure

#### Remove the unhealthy member.

Choose a Pod that is not on the affected node:
In a terminal that has access to the cluster as a cluster-admin user, run the following command:

```bash
        $ oc get pods -n openshift-etcd | grep etcd
```
        Example output
```
        etcd-ip-10-0-131-183.ec2.internal                3/3     Running     0          123m
        etcd-ip-10-0-164-97.ec2.internal                 3/3     Running     0          123m
        etcd-ip-10-0-154-204.ec2.internal                3/3     Running     0          124m
```

Connect to the running etcd container, passing in the name of a Pod that is not on the affected node:
In a terminal that has access to the cluster as a cluster-admin user, run the following command:

```bash
        $ oc rsh -n openshift-etcd etcd-ip-10-0-154-204.ec2.internal
```
        View the member list:
```bash
        sh-4.2# etcdctl member list -w table
```
        Example output
```
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        |        ID        | STATUS  |             NAME             |        PEER ADDRS         |       CLIENT ADDRS        |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        | 6fc1e7c9db35841d | started | ip-10-0-131-183.ec2.internal | https://10.0.131.183:2380 | https://10.0.131.183:2379 |
        | 757b6793e2408b6c | started |  ip-10-0-164-97.ec2.internal |  https://10.0.164.97:2380 |  https://10.0.164.97:2379 |
        | ca8c2990a0aa29d1 | started | ip-10-0-154-204.ec2.internal | https://10.0.154.204:2380 | https://10.0.154.204:2379 |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
````
        Take note of the ID and the name of the unhealthy etcd member, because these values are needed later in the procedure.

        Remove the unhealthy etcd member by providing the ID to the etcdctl member remove command:
```bash

        sh-4.2# etcdctl member remove 6fc1e7c9db35841d
```
        Example output
```
        Member 6fc1e7c9db35841d removed from cluster baa565c8919b060e
```
        View the member list again and verify that the member was removed:

```bash

        sh-4.2# etcdctl member list -w table
```
        Example output
```
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        |        ID        | STATUS  |             NAME             |        PEER ADDRS         |       CLIENT ADDRS        |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        | 757b6793e2408b6c | started |  ip-10-0-164-97.ec2.internal |  https://10.0.164.97:2380 |  https://10.0.164.97:2379 |
        | ca8c2990a0aa29d1 | started | ip-10-0-154-204.ec2.internal | https://10.0.154.204:2380 | https://10.0.154.204:2379 |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
```
        You can now exit the node shell.

        Remove the old secrets for the unhealthy etcd member that was removed.

        List the secrets for the unhealthy etcd member that was removed.
```bash

        $ oc get secrets -n openshift-etcd | grep ip-10-0-131-183.ec2.internal
```
        Pass in the name of the unhealthy etcd member that you took note of earlier in this procedure.

        Example output
```
        etcd-peer-ip-10-0-131-183.ec2.internal              kubernetes.io/tls                     2      47m
        etcd-serving-ip-10-0-131-183.ec2.internal           kubernetes.io/tls                     2      47m
        etcd-serving-metrics-ip-10-0-131-183.ec2.internal   kubernetes.io/tls                     2      47m
```

        Delete the secrets for the unhealthy etcd member that was removed.

        Delete the peer secret:
        ```bash

        $ oc delete secret -n openshift-etcd etcd-peer-ip-10-0-131-183.ec2.internal
        ```
        Delete the serving secret:
        ```bash

        $ oc delete secret -n openshift-etcd etcd-serving-ip-10-0-131-183.ec2.internal
        ```
        Delete the metrics secret:
        ```bash

        $ oc delete secret -n etcd-serving-metrics-ip-10-0-131-183.ec2.internal
        ```
        Delete and recreate the master machine. After this machine is recreated, a new revision is forced and etcd scales up automatically.



## Replacing an unhealthy etcd member whose etcd Pod is crashlooping

#### Procedure

    Stop the crashlooping etcd Pod.

        Debug the node that is crashlooping.

        In a terminal that has access to the cluster as a cluster-admin user, run the following command:
        ```bash

        $ oc debug node/ip-10-0-131-183.ec2.internal
        ```
        	Replace this with the name of the unhealthy node.

        Change your root directory to the host:
        ```bash

        sh-4.2# chroot /host
        ```
        Move the existing etcd Pod file out of the kubelet manifest directory:
        ```bash

        sh-4.2# mkdir /var/lib/etcd-backup

        sh-4.2# mv /etc/kubernetes/manifests/etcd-pod.yaml /var/lib/etcd-backup/
        ```
        Move the etcd data directory to a different location:
        ```bash

        sh-4.2# mv /var/lib/etcd/ /tmp
        ```
        You can now exit the node shell.

    Remove the unhealthy member.

        Choose a Pod that is not on the affected node.

        In a terminal that has access to the cluster as a cluster-admin user, run the following command:
        ```bash

        $ oc get pods -n openshift-etcd | grep etcd
        ```
        Example output
````
        etcd-ip-10-0-131-183.ec2.internal                2/3     Error       7          6h9m
        etcd-ip-10-0-164-97.ec2.internal                 3/3     Running     0          6h6m
        etcd-ip-10-0-154-204.ec2.internal                3/3     Running     0          6h6m
```
        Connect to the running etcd container, passing in the name of a Pod that is not on the affected node.

        In a terminal that has access to the cluster as a cluster-admin user, run the following command:
        ```bash

        $ oc rsh -n openshift-etcd etcd-ip-10-0-154-204.ec2.internal
        ```
        View the member list:
        ```bash

        sh-4.2# etcdctl member list -w table
        ```
        Example output
```
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        |        ID        | STATUS  |             NAME             |        PEER ADDRS         |       CLIENT ADDRS        |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        | 62bcf33650a7170a | started | ip-10-0-131-183.ec2.internal | https://10.0.131.183:2380 | https://10.0.131.183:2379 |
        | b78e2856655bc2eb | started |  ip-10-0-164-97.ec2.internal |  https://10.0.164.97:2380 |  https://10.0.164.97:2379 |
        | d022e10b498760d5 | started | ip-10-0-154-204.ec2.internal | https://10.0.154.204:2380 | https://10.0.154.204:2379 |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
```
        Take note of the ID and the name of the unhealthy etcd member, because these values are needed later in the procedure.

        Remove the unhealthy etcd member by providing the ID to the etcdctl member remove command:
        ```bash

        sh-4.2# etcdctl member remove 62bcf33650a7170a
```
        Example output
```
        Member 62bcf33650a7170a removed from cluster ead669ce1fbfb346
```
        View the member list again and verify that the member was removed:
        ```bash

        sh-4.2# etcdctl member list -w table
```
        Example output
```
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        |        ID        | STATUS  |             NAME             |        PEER ADDRS         |       CLIENT ADDRS        |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
        | b78e2856655bc2eb | started |  ip-10-0-164-97.ec2.internal |  https://10.0.164.97:2380 |  https://10.0.164.97:2379 |
        | d022e10b498760d5 | started | ip-10-0-154-204.ec2.internal | https://10.0.154.204:2380 | https://10.0.154.204:2379 |
        +------------------+---------+------------------------------+---------------------------+---------------------------+
```
        You can now exit the node shell.

    Remove the old secrets for the unhealthy etcd member that was removed.

        List the secrets for the unhealthy etcd member that was removed.
        ```bash

        $ oc get secrets -n openshift-etcd | grep ip-10-0-131-183.ec2.internal
```
        	Pass in the name of the unhealthy etcd member that you took note of earlier in this procedure.


        Example output
```
        etcd-peer-ip-10-0-131-183.ec2.internal              kubernetes.io/tls                     2      47m
        etcd-serving-ip-10-0-131-183.ec2.internal           kubernetes.io/tls                     2      47m
        etcd-serving-metrics-ip-10-0-131-183.ec2.internal   kubernetes.io/tls                     2      47m
```
        Delete the secrets for the unhealthy etcd member that was removed.

            Delete the peer secret:
            ```bash

            $ oc delete secret -n openshift-etcd etcd-peer-ip-10-0-131-183.ec2.internal
```
            Delete the serving secret:
            ```bash

            $ oc delete secret -n openshift-etcd etcd-serving-ip-10-0-131-183.ec2.internal
```
            Delete the metrics secret:
            ```bash

            $ oc delete secret -n etcd-serving-metrics-ip-10-0-131-183.ec2.internal
```
    Force etcd redeployment.

    In a terminal that has access to the cluster as a cluster-admin user, run the following command:
    ```bash

    $ oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "single-master-recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
```
    	The forceRedeploymentReason value must be unique, which is why a timestamp is appended.

    When the etcd cluster Operator performs a redeployment, it ensures that all master nodes have a functioning etcd Pod.

    Verify that the new member is available and healthy.

        Connect to the running etcd container again.

        In a terminal that has access to the cluster as a cluster-admin user, run the following command:
        ```bash

        $ oc rsh -n openshift-etcd etcd-ip-10-0-154-204.ec2.internal
```
        Verify that all members are healthy:
        ```bash

        sh-4.2# etcdctl endpoint health --cluster
```
        Example output
```
        https://10.0.131.183:2379 is healthy: successfully committed proposal: took = 16.671434ms
        https://10.0.154.204:2379 is healthy: successfully committed proposal: took = 16.698331ms
        https://10.0.164.97:2379 is healthy: successfully committed proposal: took = 16.621645ms
```
