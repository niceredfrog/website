---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This page provides a real-world example of how to configure Redis using a ConfigMap.


## Why Configure Redis Using a ConfigMap?

<!-- Here is where I would add the use case / business reason. Would need SME or PM input. However, here's a guess: -->

Redis is a popular open-source database. When deployed on Kubernetes, it gains the benefits of container orchestration,
including increased reliability and performance. To deploy Redis on Kubernetes, you must set up a Redis Pod using a
ConfigMap.

## Who Should Do This Tutorial?

<!-- Again, this is guessing, and would need SME input. But it's useful to let people know whether they should be in this page. -->

This Tutorial is for Redis administrators who want to use Kubernetes to manage their Redis servers.

## Terms


* **ConfigMap**: A Kubernetes configuration file used to set values in a Pod. See [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/).
* **Redis**: (REmote DIctionary Server) An open-source, in-memory database. See [Introduction to Redis](https://redis.io/about/) (external link).
* **Pod**: The smallest unit of computing that you can deploy in Kubernetes. See [Pods](https://kubernetes.io/docs/concepts/workloads/pods/).


## {{% heading "objectives" %}}

By the end of this Tutorial, you will:

* Create a ConfigMap with Redis configuration values
* Create a Redis Pod that mounts and uses the created ConfigMap
* Verify that the configuration was correctly applied.



## {{% heading "prerequisites" %}}


{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

The example shown on this page works with `kubectl` 1.14 and above.
Understand the concepts and procedures in [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).



<!-- lessoncontent -->


## Tutorial Steps: Configuring Redis using a ConfigMap

Follow these steps  to configure a Redis cache using data stored in a ConfigMap:

* [Step 1: Create a ConfigMap](#step-1-create-a-configmap)
* [Step 2: Apply the ConfigMap and Pod Manifest](#step-2-apply-the-configmap-and-pod-manifest)
* [Step 3: Examine the Created Objects](#step-3-examine-the-created-objects)
* [Step 4: Check the Current Pod Configuration](#step-4-check-the-current-pod-configuration)
* [Step 5: Add Configuration Values](#step-5-add-configuration-values)
* [Step 6: Restart the Redis Pod](#step-6-restart-the-redis-pod)
* [Step 7: Clean Up](#step-7-clean-up)

### Step 1: Create a ConfigMap

Create a ConfigMap named `example-redis-config` with an empty configuration block (`redis-config: ""`):

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

### Step 2: Apply the ConfigMap and Pod Manifest

Apply the ConfigMap you created, along with the example Redis Pod manifest `redis-pod.yaml` that has been provided to you as part of this Tutorial:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

In this step, you used the `kubectl apply` command. This command applies a configuration to a resource - in this case, your Pod.
Because the Pod did not already exist, the command creates it.

For more information about this command, see [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/).

Before moving on, you can learn more about what you did in this step.
Examine the contents of the Redis Pod manifest by opening the file `kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml`.

{{% code_sample file="pods/config/redis-pod.yaml" %}}

In the manifest, you can see the following:

* A volume named `config` is created by `spec.volumes[1]`.
* The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
  `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
* The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config`
ConfigMap above as `/redis-master/redis.conf` inside the new Pod.

### Step 3: Examine the Created Objects

To verify that the Pod was created correctly, examine the created objects.

First, get the information about the Pod and ConfigMap:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see the following output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

Next, take a closer look at the ConfigMap. Recall that you left the `redis-config` key blank in the `example-redis-config` ConfigMap.
Run the following command to confirm:

```shell
kubectl describe configmap/example-redis-config
```

You should see an empty `redis-config` key:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

In this step, you used two commands. If you want to learn more about them, see:

* [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
* [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)

### Step 4: Check the Current Pod Configuration


To check the current configuration of your new Redis Pod, you need to run commands on Redis.
You can use the `kubectl exec` command to execute commands inside a container, like your Redis Pod.
Access the Pod and start the Redis command-line tool `redis-cli`:

```shell
kubectl exec -it redis -- redis-cli
```

Now you can use Redis commands to check the current configuration.

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should show the default value, 0:

```shell
1) "maxmemory"
2) "0"
```

Similarly, check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It should show the default value, `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

For more information about the commands you used in this step, see:
* [kubectl exec](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/)
* [Redis CLI](https://redis.io/docs/latest/develop/tools/cli/) (external link)
* [CONFIG GET](https://redis.io/docs/latest/commands/config-get/) (external link)

### Step 5: Add Configuration Values

Now that the Redis Pod has been created, add some configuration values to the `example-redis-config` ConfigMap
and apply them to the Pod.
Open the file `pods/config/example-redis-config.yaml` and edit the `redis-config` section to look like the following:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

Apply the updated ConfigMap to the Redis Pod:

```shell
kubectl apply -f example-redis-config.yaml
```

Confirm that the ConfigMap was updated:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values you just added:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

### Step 6: Restart the Redis Pod

The ConfigMap is updated with new configuration values, but you have to restart the Pod for the new configuration to take effect.
If you want to confirm that the new configuration has not yet been applied, check the Redis Pod again using `kubectl exec` and  `redis-cli`:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It remains at the default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It remains at the `noeviction` default setting:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

When you restart a Pod, it gets updated values from all of its associated ConfigMaps. 

To restart the Redis Pod, delete and recreate it:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

To verify, check the configuration values again:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should now show the updated value of 2097152:

```shell
1) "maxmemory"
2) "2097152"
```

Similarly, `maxmemory-policy` has also been updated:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It now shows the desired value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

### Step 7: Clean Up

Clean up your work by deleting the resources you created:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## Troubleshooting

<!-- Would need help from SME, maybe Customer Support, to write this section -->

If you need help at any point while doing this Tutorial, contact our [Tutorial Hotline](https://fictionalsite.com).

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
