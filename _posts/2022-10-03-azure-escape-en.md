---
title: "Azure Container Instances Escape"
author: "GRoomPang"

catalog: true
tags:
  - [BoB, Cloud, Azure, serverless, escape]

toc: true
toc_sticky: true
 
date: 2022-09-18
last_modified_at: 2022-10-03
---

## Executive summary

---

Azure Container Instances (ACI) is a Container-as-a-Service (CaaS) service that enables developer to deploy containers on microsoft azure cloud without having to manage any underlying infrastructure. In search for serverless container vulnerabilities, our team GRoomPang discovered container escape vulnerability within ACI, which was reported to Microsoft Security Response Center (MSRC) and swiftly patched. 

With this vulnerability, attacker is able to gain root access of node, enumerate other pods in kubernetes cluster, and manipulate container logs for azure website.

## Exploration on Azure Container Instances

---

Serverless services like ACI provisions standardized nodes and containers to users. Exploring the internals of ACI**,** we need to identify and exploit its container runtime. Serverless container runtime can be customed or open sourced. By setting container image’s entrypoint to `/proc/self/exe`, runtime points itself as entrypoint and executes itself. 

![Figure 1.  container log of entrypoint /proc/self/exe at Azure Portal](https://user-images.githubusercontent.com/54650556/193565747-0567d419-b8bf-466b-ac4c-4bce82c2b108.png)
*Figure 1.  container log of entrypoint /proc/self/exe at Azure Portal*

ACI was running in runc 1.0.0-rc10, which is vulnerable for CVE-2021-30465!

## Applying CVE-2021-30465

---

CVE-2021-30465 is a vulnerability that allows container filesystem breakout by exploiting TOCTOU in mounting symlink tmpfs volume logic at runc before 1.0.0-rc95. Details are explained in this document ([http://blog.champtar.fr/runc-symlink-CVE-2021-30465/](http://blog.champtar.fr/runc-symlink-CVE-2021-30465/)). 

We tried mounting emptyDir volume to multiple containers in ACI pod to trigger this vulnerability. However there were some constraints to do so :

- **cannot mount emptyDir as tmpfs filesystem**
    
    In mounting logic at runC libcontainer/rootfs_linux.go, there is a time difference between function for checking whether given directory is a valid directory and function for mount. When we mount as tmpfs, this difference is greatest and thus becomes easier to exploit, but in ACI user can only bind mount emptyDir volume as ext4 filesystem.
    
- **pod creation fail for invalid image pulls**
    
    To trigger CVE-2021-30465, the first container executes symlink exchange attack, and then other containers should initialize to trigger TOCTOU. Exploit code for kubernetes did this by setting invalid images for other containers which delayed container creation as pending, and updating the images so that the other containers can start after symlink exchange. This method could not be used in ACI pod. 
    

- **ACI mount option**
    
    ACI mount option was set to MS_SHARED which mounts volume as `Bidirectional` mode and [propagates mount](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation). Therefore once one container mounts a volume, the mount propagates to all containers, including the first container executing symlink exchange. Once mounted, symlink exchange does not happen, which makes triggering with multiple containers useless.
    
    ![Figure2. exponential mount occuring because of bidirectional mode](https://user-images.githubusercontent.com/54650556/193566091-e1232839-e77a-4204-b684-24fec91ed942.png)*Figure2. exponential mount occuring because of bidirectional mode*
    
    Also the mount happens exponentially because `bidirectional` mode is equal to rshared(`MS_REC` + `MS_SHARED`). This made mounting very slow so we had to limit the number of our container.
    

With some test at local, we found out by starting and stopping containers it is possible to trigger CVE (but with very low possibility). This is because start/stop method has shorter time gap between containers than using pending method, and because we had to use very few containers for above reasons.

## Changed Environment by Container Numbers

---

To increase triggering possibility within ACI limitations, the number of container and volume must be greater than CPU number. We increased volumes and decreased CPU number and memory for each container and fiddled with resource numbers([reference](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

![Untitled](https://user-images.githubusercontent.com/54650556/193566623-377f1c9c-80de-4bea-8693-76c9fc89dfb8.png){: width="80%" height="80%"}

![Figure 3, 4. differing container environments](https://user-images.githubusercontent.com/54650556/193566637-8cf06374-1785-4601-8918-d3fe204b8a0b.png){: width="80%" height="80%"}
*Figure 3, 4. differing container environments*

Interestingly by changing container numbers  the hostname changed(SandboxHost-* → wk-caas-*)!

![Figure 5. the change of the container’s environment](https://user-images.githubusercontent.com/54650556/193566770-8d08d2b8-1c47-412e-8e01-f665ff9e83a2.png)
*Figure 5. the change of the container’s environment*

After some tests, we found out that the environment of the container changed like [Figure 5] (1.0.0-rc10 → 1.0.0-rc2) if there are more than 5 containers in ACI pod or we mount a gitRepo volume which works for old version of kubernetes. We extracted container runtime by using whoC, a container image created by unit42 for extracting container runtime.

![Figure 6-1. runC that we extracted](https://user-images.githubusercontent.com/54650556/193566664-c7680140-1249-4ca6-af36-7c16f8dcfdf9.png){: width="80%" height="80%"}
*Figure 6-1. runC that we extracted*

![Figure 6-2. [the vulnerable runC of Azurescape](https://www.paloaltonetworks.com/blog/2021/09/azurescape/)](https://user-images.githubusercontent.com/54650556/193566789-be69af01-e148-4f67-aac0-d40c80322603.png){: width="80%" height="80%"}
*Figure 6-2. [the vulnerable runC of Azurescape](https://www.paloaltonetworks.com/blog/2021/09/azurescape/)*

We also extracted host’s runC binary and found out they were using older version of runC, the same one as Azurescape vulnerability released by Paloalto at September ****2021.

Since runC v1.0.0-rc2 had more stable container escape vulnerability CVE-2019-5736, we used that instead of CVE-2021-30465.

[Reference] the vulnerable runC of Azurescape: [https://www.paloaltonetworks.com/blog/2021/09/azurescape/](https://www.paloaltonetworks.com/blog/2021/09/azurescape/)

## Container Escape (CVE-2019-5736)

---

CVE-2019-5736 is a vulnerability that controls runC binary by setting the ENTRYPOINT to `/proc/self/exe` to escape to host within container. 

![Figure 7. Connecting reverse shell by applying CVE-2019-5736 trigger image](https://user-images.githubusercontent.com/54650556/193566113-2e7aa838-ed02-4aa9-ad0e-fe2815adcddc.png)
*Figure 7. Connecting reverse shell by applying CVE-2019-5736 trigger image*

By assigning more than 5 containers in ACI pod or mounting a gitRepo volume to lower runC version and planting CVE-2019-5736 trigger image, we can escape to ACI node!

## Attempts to gain Cluster control

---

![Figure 8. the architecture of the exploited(escaped) ACI](https://user-images.githubusercontent.com/54650556/193566123-ef5b9a36-5596-4df5-b6ac-b53afe0245d2.png){: width="70%" height="70%"}
*Figure 8. the architecture of the exploited(escaped) ACI*

By reaching ACI container’s host node, we could enumerate other pods within kubernetes cluster, and manipulate container logs for azure website. But because azure manages one node per one account, we could not get access to resources of other account. We continued searching for multi-tenant vulnerabilities like information disclosure or cluster takeover.

We could find lot of details about inner workings of ACI service and manipulated our kubelet process for testing cluster takeover, but older runC usage was patched and we ended our research. This recording below is the full ACI container escape process.

[![Video Label](https://user-images.githubusercontent.com/54650556/193575221-4c104a56-a815-4093-b6f6-1a74e0f78779.png)](https://www.youtube.com/watch?v=xitHuQj8k24)

## Azure’s Fix

---

runC version was updated in conditions mentioned above to prevent host escape with CVE-2019-5736.

### Timeline

11.26.2021: Container Escape discovered

12.09.2021: reported to MSRC

1.15.2022: vulnerability accepted

1.18.2022: vulnerability officially patched

## Conclusion

---

Even though azurescape multi-tenent vulnerability were patched at September 2021, loopholes for container escape were still there to let attackers investigate which could lead to another multi-tenent vulnerabilities like azurescape. Apart from patching its root cause, closing all opened doors for that root cause is also important.