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


