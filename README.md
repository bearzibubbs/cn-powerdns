# PowerDNS on OpenShift

## Overview

This README will guide you through the process of deploying PowerDNS on OpenShift.

### What this repository provides:

This repository will provide a few critical things to help you get started with PowerDNS on OpenShift:

- Manifests for configuring the [MariaDB Operator](https://github.com/mariadb-operator/mariadb-operator) in preparation for the PowerDNS environment. *(Optional: Please see the [Requirements](#requirements) section for more information)*
- Manifests for [PowerDNS Recursor](https://doc.powerdns.com/recursor/)
- Manifests for [PowerDNS Authoritive Nameserver](https://doc.powerdns.com/authoritative/)

### Optional dependencies that make this project more useful:

There are two optional requirements, which these instrucntions cover:

- [MetalLB](https://metallb.universe.tf/)
   
   MetalLB is a hard requirement due to the fact that PowerDNS is a network service that requires a LoadBalancer service type for DNS queries (UDP/53).
- [MariaDB Operator](https://github.com/mariadb-operator/mariadb-operator)
   
   Although the MariaDB Operator is not required specifically to run PowerDNS on OpenShift, it is recommended as it follows OpenShift's operator model for software running on OpenShift.
- [Power DNS Operator](https://github.com/Orange-OpenSource/PowerDNS-Operator-helm-chart) via [Helm Chart](https://helm.sh/). 
   
   The PowerDNS Operator is more of a Day 2 operational tool for managing PowerDNS records on OpenShift. It is not required and can be replaced with other tools such as [PowerDNS Admin](https://github.com/PowerDNS-Admin/PowerDNS-Admin), however recently the PowerDNS Admin project has been nearly absent of support from the project owner so it is my personal belief that the PowerDNS Operator is the best tool for managing PowerDNS records on OpenShift at this time.

### Hard-Coded Considerations (at this time):

- `namespace: jinkit-ops` is hard-coded in the manifests at this time. This will be updated in the future, once this project is converted into a Helm chart.
- The MetalLB IPPool is currently hard-coded to `ippool-roderika-01` for the [`03-service-pdns-auth-udp.yaml`](./02-powerdns-auth/02-service-pdns-auth-udp.yaml) manifest. This will also be updated in the future, once this project is converted into a Helm chart.

## Prerequisite Configuration: MetalLB Operator

Source: [OpenShift MetalLB Documentation](https://docs.openshift.com/container-platform/4.17/networking/networking_operators/metallb-operator/metallb-operator-install.html)

Since we are deploying a DNS service in Kubernetes (which uses UDP/53), we need to have a LoadBalancer service type for the PowerDNS Authoritative Nameserver. This is where MetalLB comes in, as it can provide a reasonable solution for UDP (non-HTTP) services.

### Install via OperatorHub:

Deploy the MetalLB Operator by using the left-navigation panel and clicking through the following:

   1. Click on **Operators** > **OperatorHub**
   2. Make sure that the Project dropdwn is set to **All Projects**.
   3. Search for the term `MetalLB` in the "Filter by key word" search box, and select the **MetalLB Operator**. (This is a Red Hat maintained operator)
   4. On the next page, click on the blue box that says **Install**.
   5. Finally, you can leave all options on "default" settings (channel: stable, version: 4.16.x or higher, etc), and click on the blue box that says **Install**.

### Install via CLI:

You can also install MetalLB via the CLI, if you prefer.

1. You can copy/paste the following YAML to install MetalLB:

   ```yaml
   cat << EOF | oc apply -f -
   ---
   apiVersion: v1
   kind: Namespace
   metadata:
     name: metallb-system

   ---
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: metallb-operator
     namespace: metallb-system

   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: metallb-operator-sub
     namespace: metallb-system
   spec:
     channel: stable
     name: metallb-operator
     source: redhat-operators 
     sourceNamespace: openshift-marketplace
   EOF
   ```

2. Once installed, you can enable monitoring with the following label:

   ```bash
   oc label ns metallb-system "openshift.io/cluster-monitoring=true"
   ```

3. You can check on the installation by running the following command:

   ```bash
   oc get installplan -n metallb-system
   oc get csv -n metallb-system
   ```

4. With the operator installed, now you can deploy a MetalLB system (the part that makes it all work).

   ```yaml
   cat << EOF | oc apply -f -
   apiVersion: metallb.io/v1beta1
   kind: MetalLB
   metadata:
     name: metallb
     namespace: metallb-system
   EOF
   ```

### For BGP (L3) Deployments:

I'm going to show you what I personally run for my BGP deployments. I can document how I configure this on my router/firewall later (I have BGP/FRR configured for OPNSense), but for now, I really want to explore the OpenShift/MetalLB side.

1. First, create an `IPAddressPool` resource for MetalLB:

   ```yaml
   cat << EOF | oc apply -f -
   ---
   apiVersion: metallb.io/v1beta1
   kind: IPAddressPool
   metadata:
     name: ippool-roderika-01
     namespace: metallb-system
     labels:
       ippool: vlan50
   spec:
     addresses:
       - 10.50.1.0/22
     autoAssign: true
     avoidBuggyIPs: true
     protocol: bgp
   EOF
   ```

2. Next, we need to create a `BGPAdvertisement` using the following YAML:

   ```yaml
   cat << EOF | oc apply -f -
   ---
   apiVersion: metallb.io/v1beta1
   kind: BGPAdvertisement
   metadata:
     name: bgpadvertisement-1
     namespace: metallb-system
   spec:
     ipAddressPools:
       - ippool-roderika-01
     peers:
       - opnsense
     aggregationLength: 32
     aggregationLengthV6: 128
     localPref: 100
   EOF
   ```

3. Finally, we need to create a `BGPPeer` resource:

   ```yaml
   cat << EOF | oc apply -f -
   ---
   apiVersion: metallb.io/v1beta2
   kind: BGPPeer
   metadata:
     namespace: metallb-system
     name: opnsense
   spec:
     peerAddress: 192.168.3.1
     peerASN: 64510
     myASN: 64512
     routerID: 192.168.3.99
     sourceAddress: 192.168.3.99
   EOF
   ```

   **FULL DISCLOSURE:** *This is really the bare minimum for getting a BGP-based MetalLB deployment configure. There are many more settings that you should explore in either [MetalLB's documentation](metallb.universe.tf) or preferably [OpenShift's official MetalLB's documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/networking/load-balancing-with-metallb).*

### MariaDB Operator Installation (Updated)

Everyone will have an opionion on how which database to use for PowerDNS. I have gone through using a Postgres operator, but I just didn't really like it very much. The MariaDB operator is easy to use, attempts to be cloud-native with separate users, grants, sqljobs, and other artifacts that made sense to me. If you don't like that the MariaDB operator was used and have other thoughts/opinions on this approach, make a case for other options in the issues section of this repository.

1. We're going to deploy this entire solution inside of a specific namespace called `jinkit-ops` ("JinkIT" is just my version of "Acme", a fictional company name). In the future, this will be parameterized in a Helm chart.

   ```bash
   oc new-project jinkit-ops
   ```

2. Next, deploy the MariaDB Operator to the `jinkit-ops` namespace by using the following commands. 

   ```bash
   oc project jinkit-ops

   helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
   
   helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds
   helm install mariadb-operator mariadb-operator/mariadb-operator
   ```

   IMPORTANT: If you use the OpenShift provided version of MariaDB operator, the deployment will fail due to the operator not being able to create the necessary `ServiceAccount` and `RoleBinding` resources. This is due to the fact that the OpenShift version of the MariaDB operator is not configured to run in an OpenShift environment. The Helm chart version of the MariaDB operator is configured to run in an OpenShift environment. (bbj - TODO).

3. With the MariaDB operator installed, you can now install the manifests found in the [`00-mariadb`](./00-mariadb) directory of this repository. This will create a MariaDB instance in the `jinkit-ops` namespace.

   ```bash
   oc apply -k 00-mariadb/
   ```

   NOTE: We're using [Kustomized](https://kustomize.io/) deployments for this project to make instructions and deployment/removal easier.

## PowerDNS Installation

With the prerequisites out of the way, we can now deploy PowerDNS. If you haven't used the MariaDB Operator, you can use a DB of your choosing. Just be sure to make any required PowerDNS configuration changes as well. You can find these in the following `ConfigMap` resources:

- [`00-configmap-pdns-rec-config.yaml`](./01-powerdns-rec/00-configmap-pdns-rec-config.yaml)
- [`00-configmap-pdns-auth-config.yaml`](./02-powerdns-auth/00-configmap-pdns-auth-config.yaml)

### PowerDNS-Recursor Installation

In order to install PowerDNS to OpenShift, you will need to create a `pdns` [`ServiceAccount`](https://docs.openshift.com/container-platform/4.17/authentication/understanding-and-creating-service-accounts.html) with the appropriate [SCC Policy](https://docs.openshift.com/container-platform/4.17/authentication/managing-security-context-constraints.html) applied. This needs to be done within the same namespace you deploy PowerDNS (both the Recursor and Authoritative Nameserver).


### PowerDNS Operator Installation

Source: [PowerDNS Operator from Orange](https://github.com/Orange-OpenSource/PowerDNS-Operator-helm-chart)

1. First, add the Orange Open Source Helm repository to your Helm configuration:

   ```bash
   helm repo add orange-opensource https://orange-opensource.github.io/PowerDNS-Operator-helm-chart
   ```

2. Next, install the PowerDNS Operator using the following command:

   ```bash
   helm install -f ./03-powerdns-operator/values.yaml powerdns-operator orange-opensource/powerdns-operator
   ```

### Testing

1. This repository provides a test pod, along with a sample zone/rrset that can be used. These can be found in the [testing](./testing/) folder. You can deploy the test pod by running the following command:

   ```bash
   oc apply -f ./testing/01-zone-rrset-testing.yaml
   ```

2. You can validate that the `RRSet` was created by running the following command:

   ```bash
   ❯ oc get rrset
   NAME                 ZONE             NAME                  TYPE   TTL   STATUS      RECORDS
   ns1.lab.jinkit.com   lab.jinkit.com   ns1.lab.jinkit.com.   A      300   Succeeded   ["10.50.1.5"]
   ```

3. And you can validate that the `Zone` was created correctly with the following command:

   ```bash
   ❯ oc get zone
   NAME             SERIAL       ID
   lab.jinkit.com   1735868384   lab.jinkit.com.
   ```

4. You can perform a `dig` querey from an external host to validate that UDP/53 is working correctly to the BPG IP address that MetalLB is providing:

   ```bash
   ❯ dig ns1.lab.jinkit.com @10.50.1.5

   ; <<>> DiG 9.10.6 <<>> ns1.lab.jinkit.com @10.50.1.5
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52509
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 1232
   ;; QUESTION SECTION:
   ;ns1.lab.jinkit.com.            IN      A

   ;; ANSWER SECTION:
   ns1.lab.jinkit.com.     300     IN      A       10.50.1.5

   ;; Query time: 58 msec
   ;; SERVER: 10.50.1.5#53(10.50.1.5)
   ;; WHEN: Thu Jan 02 20:40:48 EST 2025
   ;; MSG SIZE  rcvd: 63
   ```

5. If you want to test within the cluster, you can run sometime similar to the following:

   ```bash
   oc create -f ./testing/02-pod-dns-testing.yaml     # <-- Create the dns-testing pod
   oc rsh -n jinkit-ops dns-testing     # <-- Remote ssh into the dns-testing pod

   ~# ping pdns-auth-udp     # <-- Attempt to ping the pdns-auth-udp short name service endpoint (we know this will not work, but we will get the IP address)
   PING pdns-auth-udp.jinkit-ops.svc.cluster.local (172.30.67.219) 56(84) bytes of data.
   ^C
   --- pdns-auth-udp.jinkit-ops.svc.cluster.local ping statistics ---
   2 packets transmitted, 0 received, 100% packet loss, time 1060ms

   ~# dig ns1.lab.jinkit.com @172.30.67.219     # <-- Use the dig command to query the rrset that was deployed.

   ; <<>> DiG 9.18.25 <<>> ns1.lab.jinkit.com @172.30.67.219
   ;; global options: +cmd
   ;; Got answer:
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32822
   ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
   ;; WARNING: recursion requested but not available

   ;; OPT PSEUDOSECTION:
   ; EDNS: version: 0, flags:; udp: 1232
   ;; QUESTION SECTION:
   ;ns1.lab.jinkit.com.            IN      A

   ;; ANSWER SECTION:
   ns1.lab.jinkit.com.     300     IN      A       10.50.1.5

   ;; Query time: 12 msec
   ;; SERVER: 172.30.67.219#53(172.30.67.219) (UDP)
   ;; WHEN: Fri Jan 03 01:58:51 UTC 2025
   ;; MSG SIZE  rcvd: 63
   ```
