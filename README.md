BIRD K8s example
================

This is an example on how to deploy [BIRD](https://bird.network.cz/) PodS on K8s and have them talk BGP to each other. 

This example should be adapted to suit your needs and potentially include a service if you are only using a single Pod. 

Example overview
----------------

The example itself just deploys two ConfigMaps which contain a basic BIRD configuration example with some static routes and a BGP configuration. 

It will also deploy two deployments with each a Pod with a single [pdellaert/bird:latest](https://hub.docker.com/r/pdellaert/bird/) container. Each container is pointed towards its own ConfigMap for the BIRD configuration file.

Usage overview
--------------

Steps to deploy this example, this assumes you checked out this repository and have `kubectl` installed:

1. *Editing and creating the BIRD configs*
   Edit the config maps in the `config.yaml` file as you see fit and change the BIRD config. 

   After editing the file, you can create the ConfigMaps:
   ```
   $ kubectl apply -f config.yaml
   configmap/bird-config-1 created
   configmap/bird-config-2 created
   ```

   If you already have a `bird.conf` file and want to use that directly, you can just create the ConfigMap from that file:
   ```
   $ kubectl create configmap bird-config-1 --from-file=path/to/first/bird.conf
   configmap/bird-config created
   $ kubectl create configmap bird-config-2 --from-file=path/to/second/bird.conf
   configmap/bird-config created
   ```

   If you want to update the `config.yaml` file with the content of your `bird.conf`, you can use the `--dry-run` feature of `kubectl` to return a YAML representation and copy the data section.

   ```
   $ kubectl create configmap bird-config --from-file=path/to/first/bird.conf --dry-run -o yaml
   apiVersion: v1
   data:
     bird.conf: |+
       debug protocols all;
   
       protocol device {
           scan time 10;
       }
   
       protocol static {
           ipv4 {};
           route 172.16.10.10/32 via "eth0";
           route 172.17.10.10/32 via "eth0";
           route 172.18.10.10/32 via "eth0";
           route 172.19.10.10/32 via "eth0";
           route 172.20.10.10/32 via "eth0";
           route 172.21.10.10/32 via "eth0";
           route 172.22.10.10/32 via "eth0";
           route 172.23.10.10/32 via "eth0";
           route 172.24.10.10/32 via "eth0";
           route 172.25.10.10/32 via "eth0";
           route 172.26.10.10/32 via "eth0";
           route 172.27.10.10/32 via "eth0";
           route 172.28.10.10/32 via "eth0";
           route 172.29.10.10/32 via "eth0";
           route 172.30.10.10/32 via "eth0";
           route 172.31.10.10/32 via "eth0";
       }
   
       protocol bgp {
           local as 65010;
           neighbor 192.168.99.11 as 65011;
           ipv4 {
               next hop self;
               import all;
               export all;
           };
       }
   
   kind: ConfigMap
   metadata:
     creationTimestamp: null
     name: bird-config
   ```

   Make sure you use the right names for your ConfigMaps. In the `deployment.yaml` the ConfigMaps are referred to as `bird-config-1` and `bird-config-2`.


2. *Create the BIRD Pods*
   Once the ConfigMaps exist, you can deploy the BIRD Pods:
   ```
   $ kubectl apply -f deployment.yaml
   deployment.extensions/bird-1 created
   deployment.extensions/bird-2 created
   ```

Connectivity between Pods
-------------------------

In the BIRD configuration example included and in the `config.yaml` file, the BIRD Pods are configured to setup a BGP connection with each other to exchange routes.

The BGP neighbor IP might not be known the moment the ConfigMap is created in K8s, and that means you will need to update BIRD configuration after you have deployed both Pods. 

* To do so, retreive the IPs of the BIRD Pods:
  ```
  $ kubectl get pods -o jsonpath="{.items[*]['metadata.name','status.podIP']}"
  bird-1-545f859f9-6xjrn bird-2-7bc4c6c795-mb7t6 172.16.4.99 172.16.4.150
  ```

  In the above output, it first prints the name of the Pods, and then the IPs, in the same order.

* Update the ConfigMaps with the right IP in the BGP neighbor configuration

* Re-apply the ConfigMaps
   ```
   $ kubectl apply -f config.yaml
   configmap/bird-config-1 updated
   configmap/bird-config-2 updated
   ```

* Reload the configuration of both BIRD pods using the `birdcl` command. The below command expects that the namespace only has the BIRD Pods

  ```
  $ for i in `kubectl  get pods -o jsonpath="{.items[*].metadata.name}"`; do kubectl  exec -it ${i} birdcl configure; done
  BIRD 2.0.2 ready.
  Reading configuration from /etc/bird/bird.conf
  Reconfigured
  BIRD 2.0.2 ready.
  Reading configuration from /etc/bird/bird.conf
  Reconfigured
  ```

Checking BIRD status using `birdcl`
-----------------------------------

You can always access the `birdcl` BIRD CLI by running it within the Pod:

```
$ kubectl -n playground exec -it bird-1-545f859f9-6xjrn birdcl
BIRD 2.0.2 ready.
bird> show status
BIRD 2.0.2
Router ID is 172.100.4.99
Current server time is 2018-09-02 00:15:33.541
Last reboot on 2018-09-01 22:38:50.512
Last reconfiguration on 2018-09-02 00:10:57.869
Daemon is up and running
bird> show protocols
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     22:38:50.512
static1    Static     master4    up     22:38:50.512
bgp1       BGP        ---        up     22:42:30.591  Established
bird>
```

