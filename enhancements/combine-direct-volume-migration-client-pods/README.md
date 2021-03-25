---
title: combine-direct-volume-migration-client-pods
authors:
  - "@shubham-pampattiwar"
reviewers:
  - "@alaypatel07"
  - "@pranavgaikwad"
approvers:
  - "@alaypatel07"
  - "@pranavgaikwad"
creation-date: 2021-03-05
last-updated: 2021-03-05
status: implementable
see-also:
  - "N/A"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Combine Direct Volume Migration Client Pods

The goal of this document is to focus on combining the Direct Volume Migration client pods (Rsync client pod + Stunnel client pod) on the source cluster into one single pod per namespace involved in the migration. Let's call this single combined pod as Source Transfer Pod.

## Summary

According to the current implementation of Direct Volume Migration, we create the 3 reosurces on source cluster side - Stunnel Client Pod, Rsync Client Pod and a Stunnel Service ( to facilitate communication between the Rsync and Stunnel Client Pods). The core problem being that the inter pod communication over the Stunnel Service is not encrypted as Rsync does not support encrytption, thus the data transferred over the Service is not secure. Therefore, in order to overcome this security vulnerability, we would like to combine the Stunnel Pod and Rsync Pod in to one single Pod and spawn the Stunnel and Rsync clients as individual containers inside this single Pod. Thus eliminating the insecure data transfer over the Stunnel Service by using localhost instead, inside the Pod.

## Problems to be addressed

If we go ahead with the proposal of combining the two pods into one, there are some problems which need to be addressed. 
- Wait for Stunnel Client Pods to be Running: The current Direct Volume Migration itinerary has phase called `WaitForStunnelClientPodsRunning` . This phase checks whether all the Stunnel Client Pods are in running state or not before creation of Rsync Client Pods. This is essential because if the Stunnel Pods are not in a `Running` state then the data transfer from the Rsync Client Pod over the Stunnel Service would not be possbile as the Stunnel server would not respond. But now with our new proposal of combining the two pods into one and spawning each client pod as a container inside the Source Transfer Pod, we need a new logic/mechanism to check up on whether the Stunnel container is in running state before spawning the Rsync container. The Stunnel container needs to be in running state so that the Rsync container once created can communicate with the Stunnel server over localhost.
- Wait for Rsync Client Pods to be Completed: The Direct Volume Migration itinerary has another phase called `WaitForRsyncClientPodsCompleted` . This phase checks whether the spawned Rsync Client Pods are in Completed State or Failed State. The whole mechanism for this is based on the Status present on Direct Volume Migration Progress (DVMP) CR. The DVMP CR updates it status based on the Rsync Client Pods status. Since we are removing the Rsync Client Pod and spawning it as a container, we need to change the logic/mechanism to update DVMP CR Status based on the Rsync container status of the new Source Transfer Pod. 

## Proposed solution
For problem 1, We will be doing the following things:
- Remove the Service between Stunnel and Rsync Pods
- We will be using the localhost for data transfer between the rsync and stunnel containers
- A shell script or custom command to check up on the availabiliy of the stunnel port over the `localhost` interface, and if the port reponds with a connection acceptance status code then the rsync container can go ahead and execute the `rsync` command
- This shell script/custom command needs to be embedded in the rsync container manifest, the shell script can be like (we are using `nc` command here) :
```
#!/bin/sh
while true # Some amount of time can be set here
do
cmd_output=`nc -z localhost <STUNNEL_PORT>`
if [ "$cmd_output" == "0" ]
then
    # Execute Rsync Command
fi
done
```

For problem 2, We will do the following things:
- The check whether the Rsync Pods are in completed state or not is based on the status in DVMP. Now in order to change the DVMP CR status update logic we need to update its status explicitly based on the Rsync container rather than the Pod. DVMP will mark the rsync to be completed from the status obtained from rsync container inside the Src transfer pod. The `dvmp.Status.PodPhase` needs to be updated based on the rsync container status and not Pod status. We will be checking/looking for the rsync container status to be `terminated` and the reason to be `Completed` in order to update the DVMP status to `succeeded`/`Completed`, thus helping us correctly identify whether the rsync container finished its execution successfully or not. 

## Alternative solutions:
- One Alternative for problem 2 is marking the Stunnel Pod completed based on the existence of the file created by Rsync upon its completion, this would subsequently mark the Src transfer Pod as completed because both its container will be in completed state.

## Comparing Rsync + stunnel as 2 containers vs Rsync + Stunnel in a single container

### Seperate containers
Pros:
- Rsync and Stunnel logs will be separate from each other.
- As the containers are decoupled, debugging will a bit easier.
- Stunnel and Rsync container statuses are different, they succeed differently, Rsync container is terminated and goes to completion but Stunnel keeps in running, DVMP depends on the Rsync container status, it might be a bit easier to explicitly track Rsync container status and then update DVMP accordingly.
- Volumes to be migrated can be seperately mounted for Rsync container and similarly stunnel certs and configs will be separately mounted by stunnel container.
  
Cons: 
- The proposed Src transfer pod (Rsync + Stunnel) will be created per volume to be migrated, and this pod has 2 containers, this may lead to competition for resources amongst containers.


### Single container
Pros:
- Both Rsync and Stunnel containers use the same container image, resource limits and resource requests, use same security context, thus easier to combine them into one container.
- As the Rsync and Stunnel utilities will be present in the same container, there will be no competition of resources.

Cons:
- Rsync and Stunnel logs will not seperate from each other.
- No decoupling in containers, debugging might get tricky.
- Updating DVMP is difficult here as we do not have explicity Rsync container status to be mapped with.
- Rsync needs PVCs to be migrated as volumes but Stunnel does not, it just needs stunnel certs and configs, in single container approach we will be mounting things onto a single container which are not necessary for each utility.
- Combining the container commands of Stunnel as well as Rsync will be a requirement in this apporach.