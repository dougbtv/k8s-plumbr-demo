# Plumbr on K8s between two AWS Clusters 

Let's run Steinwurf's plumbr as a sidecar and send the tun traffic over a macvlan interface using Kubernetes!

Steinwurf's RAMP technology delivers robust connectivity for 

### General gist is...

* Create a secondary pod interface for macvlan ("net1" interface)
* Plumr runs in an init container and creates a tun with IP addressing related to the macvlan interface
    * This is a kind of "side car" type approach, and the plumr binary runs as a background process

### Today and tomorrow

Today: Mostly everything is defined statically, as a demo we opted for a one-to-one connectivity between two endpoints.

In the future: We'd envision a richer use of the CNI stack in concert with the Steinwurf technology which would provide one-to-many, many-to-one and many-to-many connectivity with some degree of service discovery in order to dynamically achieve connectivity between endpoints.

### Requirements

* Two OpenShift clusters on AWS with Multus CNI installed (and operational).
* Must have connectivity on bridges over a VPC (which I didn't set up in my case)

### Limitations

* Uses statically defined IP addressing
    * And pods pinned to specific nodes.
    * This is for a single 1:1 point-to-point connection.
        * As opposed to a many:1 or 1:many.
    * Further enhancement will require a form of service discovery.
* Plumr is run as a side car "manually"
    * That is, with a yaml specification for it.
    * This could be improved to be automated (addressing, too).
* Has rather loose security restrictions for the pod itself
    * This can be addressed with CNI + a controller.
* This uses bridge for connectivity
    * Unsure about performance impact.
* If the plumr binary fails, the pod will keep running
    * This could be improved with a controller as well.

## Setup

First, you'll need to get the binaries from Steinwurf, so, please do get in touch with contact@steinwurf.com so you can try it out.

Copy the `plumr` and `shm_printer` binary to each host.

These examples use a path @ `/tmp/plumr/` where the binary is copied.

This is a manual step required as this binary is not publicly available.

### Connectivity

In our demo cluster using AWS, there's bridging across a VPC, which works by subnet, and it's setup like this...

```
10.10.0.0/16 - US
10.10.1.0/24 - US (worker 1)
10.10.2.0/24 - US (worker 2)

10.20.0.0/16 - APAC
10.20.1.0/24 - apac (worker 1)
10.20.2.0/24 - apac (worker 2)
```

In the examples I have the pods with the IP addresses of...

```
10.10.1.200 == Ohio
10.20.1.200 == Mumbai
```

In order to determine which node to use for which, we used a bastion host and had directories with our install-config yaml for OpenShift, I would look at the deployment yaml, for example...

I had an install-config that I'd look at to figure out the node and subnet associations...

```
[rosa@bastion Mumbai]$ cat *w1* | grep -P "range|nodeSelector|kubernetes.io/hostname"
  nodeSelector: 
    kubernetes.io/hostname: ip-10-1-241-95.ap-south-1.compute.internal
      "range": "10.20.1.0/24",
  nodeSelector:
    kubernetes.io/hostname: ip-10-1-241-95.ap-south-1.compute.internal
      nodeSelector:
        kubernetes.io/hostname: ip-10-1-241-95.ap-south-1.compute.internal
```

Here we can see that the range `10.20.1.0/24` is associated with the host `ip-10-1-241-95.ap-south-1.compute.internal`

*IMPORTANT*: You must change the `kubernetes.io/hostname` and the IP addressing to match.

## Inspecting the plumr enabled pods

You'll create the resources using the examples that follow in the following sections.

Then, you'll want to inspect the pods... Here's an example where we exec into one of the plumr pods.

```
$?oc exec -it plumbr-example-b -c workload -- /bin/bash
```

You can look at the configuration used for plumr with:

```
[root@plumbr-example-b /]# cat /shared-data/config.json
```

And you can inspect the logs from plumr (typically: "basically nothing" is good in my experience)

```
[root@plumbr-example-b /]# cat /shared-data/entrypoint.log
templating json...
running plumr...
```

You can run a test ping to the other side over the bridge...

```
[root@plumbr-example-b /]# ping -c 2 10.10.1.200
PING 10.10.1.200 (10.10.1.200) 56(84) bytes of data.
64 bytes from 10.10.1.200: icmp_seq=1 ttl=59 time=199 ms
64 bytes from 10.10.1.200: icmp_seq=2 ttl=59 time=197 ms
```

And inspect the available interfaces...

```
[root@plumbr-example-b /]# ip a | grep -P "^\d"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
2: eth0@if61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8901 qdisc noqueue state UP group default 
3: net1@if62: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
4: tun: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1452 qdisc fq_codel state UNKNOWN group default qlen 500
```


Note: The iperf3 binary is available.

```
[root@plumbr-example-b /]# iperf3 --version
iperf 3.14 (cJSON 1.7.15)
```

Also to run the `shm_printer`, execute...

```
[root@plumbr-example-a /]# /plumrbin/shm_printer --name plumr_metrics
Start reader
Configure
Waiting for producer with name plumr_metrics
```

## Resources for Ohio-side cluster

*NOTE* The JSON configuration has been removed as the plumr binary is under construction and may not represent a working, please contact@steinwurf.com for a JSON configuration.

```yaml=
apiVersion: v1
kind: ConfigMap
metadata:
  name: plumbr-entrypoint-config
data:
  plumbr-entrypoint.sh: |
    #!/bin/sh

    TUN_IP="${TUN_IP:-10.4.0.1}"
    UDP_REMOTE_IP="${UDP_REMOTE_IP:-192.168.122.111}"

    JSON_TEMPLATE='{ [removed!] }'
    echo "templating json..."

    JSON_CONTENT=$(printf "$JSON_TEMPLATE" "$TUN_IP" "$UDP_REMOTE_IP")
    echo "$JSON_CONTENT" > /shared-data/config.json

    echo "running plumr..."

    /plumrbin/plumr -f /shared-data/config.json

    sleep infinity
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: br-plumr-config
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "bridge",
      "isGateway": true,
      "ipam": {
        "type": "static",
        "capabilites": { "ips": true },
        "routes": [
             { "dst": "10.10.2.0/24" },
             { "dst": "10.20.0.0/16" }
          ]
      }
    }'
---
apiVersion: v1
kind: Pod
metadata:
  name: plumbr-example-a
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      {
        "name": "br-plumr-config",
        "ips": [ "10.10.1.200/24" ]
      }
    ]'
spec:
  shareProcessNamespace: true
  nodeSelector:
    kubernetes.io/hostname: ip-10-0-154-10.us-east-2.compute.internal
  initContainers:
  - name: run-plumr
    image: quay.io/dosmith/fedora-vlc-iptools:gen2
    command: ["/bin/sh", "-c", "/entrypoint/plumbr-entrypoint.sh > /shared-data/entrypoint.log 2>&1 &"]
    env:
    - name: TUN_IP
      value: "10.4.0.1"
    - name: UDP_REMOTE_IP
      value: "10.20.1.200"
    volumeMounts:
    - name: host-bin
      mountPath: /plumrbin/
    - name: plumbr-entrypoint-volume
      mountPath: /entrypoint
    - name: shared-volume
      mountPath: /shared-data
    securityContext:
      privileged: true
      capabilities:
        add: ["NET_ADMIN","NET_RAW"]
  containers:
  - name: workload
    image: quay.io/dosmith/fedora-vlc-iptools:gen2
    command: ["/bin/sh", "-c", "sleep 10000000000000000"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared-data
    - name: host-bin
      mountPath: /plumrbin/
    securityContext:
      privileged: true
  volumes:
  - name: host-bin
    hostPath:
      path: /tmp/plumr
      type: Directory
  - name: plumbr-entrypoint-volume
    configMap:
      name: plumbr-entrypoint-config
      defaultMode: 0744
  - name: shared-volume
    emptyDir: {}
```

# Resources for Mumbai-side cluster

   
```yaml=
apiVersion: v1
kind: ConfigMap
metadata:
  name: plumbr-entrypoint-config
data:
  plumbr-entrypoint.sh: |
    #!/bin/sh

    TUN_IP="${TUN_IP:-10.4.0.1}"
    UDP_REMOTE_IP="${UDP_REMOTE_IP:-192.168.122.111}"

    JSON_TEMPLATE='{ [removed!] }'

    echo "templating json..."

    JSON_CONTENT=$(printf "$JSON_TEMPLATE" "$TUN_IP" "$UDP_REMOTE_IP")
    echo "$JSON_CONTENT" > /shared-data/config.json

    echo "running plumr..."

    /plumrbin/plumr -f /shared-data/config.json

    sleep infinity
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: br-plumr-config
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "bridge",
      "isGateway": true,
      "ipam": {
        "type": "static",
        "capabilites": { "ips": true },
        "routes": [
             { "dst": "10.20.2.0/24" },
             { "dst": "10.20.3.0/24" },
             { "dst": "10.10.0.0/16" }
          ]
      }
    }'
---
apiVersion: v1
kind: Pod
metadata:
  name: plumbr-example-b
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      {
        "name": "br-plumr-config",
        "ips": [ "10.20.1.200/24" ]
      }
    ]'
spec:
  shareProcessNamespace: true
  nodeSelector:
    kubernetes.io/hostname: ip-10-1-236-160.ap-south-1.compute.internal
  initContainers:
  - name: run-plumr
    image: quay.io/dosmith/fedora-vlc-iptools:gen2
    command: ["/bin/sh", "-c", "/entrypoint/plumbr-entrypoint.sh > /shared-data/entrypoint.log 2>&1 &"]
    env:
    - name: TUN_IP
      value: "10.4.0.2"
    - name: UDP_REMOTE_IP
      value: "10.10.1.200"
    volumeMounts:
    - name: host-bin
      mountPath: /plumrbin/
    - name: plumbr-entrypoint-volume
      mountPath: /entrypoint
    - name: shared-volume
      mountPath: /shared-data
    securityContext:
      privileged: true
      capabilities:
        add: ["NET_ADMIN","NET_RAW"]
  containers:
  - name: workload
    image: quay.io/dosmith/fedora-vlc-iptools:gen2
    command: ["/bin/sh", "-c", "sleep 10000000000000000"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared-data
    - name: host-bin
      mountPath: /plumrbin/
    securityContext:
      privileged: true
  volumes:
  - name: host-bin
    hostPath:
      path: /tmp/plumr
      type: Directory
  - name: plumbr-entrypoint-volume
    configMap:
      name: plumbr-entrypoint-config
      defaultMode: 0744
  - name: shared-volume
    emptyDir: {}
```


### The `Dockerfile` for the `dougbtv/fedora-iptools` image

```
FROM registry.fedoraproject.org/fedora:38
RUN dnf install -y iproute iputils tcpdump iperf3
```

### The `Dockerfile` for the `dougbtv/fedora-vlc-iptools` image

```
FROM registry.fedoraproject.org/fedora:38
RUN dnf install -y iproute iputils tcpdump iperf3 procps-ng iproute-tc
RUN yes | dnf install \
  https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
RUN dnf install -y vlc
```

### The `Dockerfile` for the `quay.io/dosmith/fedora-vlc-iptools-speedtest:gen1` image

```
FROM registry.fedoraproject.org/fedora:38

# Install PHP, Apache and the required extensions, plus the ip utils.
RUN dnf -y install httpd php php-pdo php-pgsql php-mysqlnd \
    php-gd php-pear php-cli php-pdo php-json \
    git net-tools nano iproute iputils tcpdump iperf3 procps-ng iproute-tc htop

# Clone Fatih's repo
RUN git clone https://github.com/fenar/speedtest /var/www/html/speedtest
RUN chown -R apache:apache /var/www/html/*

# Set permissions
RUN chown -R apache:apache /var/www/html/*

# VLC install
RUN yes | dnf install \
    https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
RUN dnf install -y vlc

# Expose port 80
EXPOSE 80

# Start Apache
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
```
