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

According to the current implementation of Direct Volume Migration, we create the 3 resources on source cluster side - Stunnel Client Pod, Rsync Client Pod and a Stunnel Service ( to facilitate communication between the Rsync and Stunnel Client Pods). The core problem being that the inter pod communication over the Stunnel Service is not encrypted as Rsync does not support encrytption, thus the data transferred over the Service is not secure. Therefore, in order to overcome this security vulnerability, we would like to combine the Stunnel Pod and Rsync Pod in to one single Pod and spawn the Stunnel and Rsync clients as individual containers inside this single Pod. Thus eliminating the insecure data transfer over the Stunnel Service by using localhost instead, inside the Pod.

## Problems to be addressed

If we go ahead with the proposal of combining the two pods into one, there are some problems which need to be addressed. 
- Wait for Stunnel Client Pods to be Running: The current Direct Volume Migration itinerary has phase called `WaitForStunnelClientPodsRunning` . This phase checks whether all the Stunnel Client Pods are in running state or not before creation of Rsync Client Pods. This is essential because if the Stunnel Pods are not in a `Running` state then the data transfer from the Rsync Client Pod over the Stunnel Service would not be possbile as the Stunnel server would not respond. But now with our new proposal of combining the two pods into one and spawning each client pod as a container inside the Source Transfer Pod, we need a new logic/mechanism to check up on whether the Stunnel process is in running state before spawning the Rsync client process. The Stunnel process needs to be in running state so that the Rsync once started can communicate with the Stunnel server over localhost.
- Wait for Rsync Client Pods to be Completed: The Direct Volume Migration itinerary has another phase called `WaitForRsyncClientPodsCompleted` . This phase checks whether the spawned Rsync Client Pods are in Completed State or Failed State. The whole mechanism for this is based on the Status present on Direct Volume Migration Progress (DVMP) CR. The DVMP CR updates it status based on the Rsync Client Pods status. Since we are removing the Rsync Client Pod and spawning it as a container, we need to change the logic/mechanism to update DVMP CR Status based on the Rsync container status of the new Source Transfer Pod. 

## Proposed solution
### Separate container approach:
In this approach we will be spawning 2 containers - Stunnel and Rsync under a single Pod (Source Transfer Pod). We will be introducing a shared volume called `rsync-stunnel-ipc` and this volume will be mounted
for both the containers. The utility of this shared volume is to facilitate inter-process communication amongst the 2 containers. Now Let's take a look at the workflow and lifecyle of each
container.

For Stunnel container:
- Start the Stunnel process with the Stunnel config in background mode (as we are embedding the stunnel command in a bash script).
- Continuously check for the existence of a file named `rsync-client-container-done` on the shared volume.
- Once the file exists (the existence of file named `rsync-client-container-done` indicates the successful completion of rsync command) then we `break` from the loop.
- Finally, we will `exit` the container and thus the stunnel process will be killed, consequently marking the container as `completed` (Solves problem 2).
- The above logic will be embedded in a bash script in the container command as follows:
```
  - command:
    - /bin/bash
    - -c
    - |
      /bin/stunnel /etc/stunnel/stunnel.conf
      while true
      do test -f /rsync-data/rsync-client-container-done
      if [ $? -eq 0 ]
      then
      break
      fi
      done
      exit 0
```

For Rsync Container:
- Continuously checkup (for 10 minutes) on the `localhost` interface (port 2222) whether the stunnel server container is up and ready to accept connections using the `nc` command (Solves problem 1).
- Once the stunnel container is ready to accept connections, execute the rsync command.
- Post completion of the rsync command (irrespective of whether successful or not), create a file named `rsync-client-container-done` on the shared volume using `trap` to indicate successful completion of rsync command.
- Break the monitoring loop on `localhost` interface and thus container goes to completion.
- The above logic will be embedded in a bash script in the container command as follows:
```
  - command:
    - /bin/bash
    - -c
    - |
      trap "touch /usr/share/rsync-data/rsync-client-container-done" exit $rc
      timeout=600
      SECONDS=0
      while [ $SECONDS -lt $timeout ]
      do nc -z localhost 2222
      if [ $? -eq 0 ]
      then
      rsync --archive --delete --recursive --hard-links --partial --info=COPY2,DEL2,REMOVE2,SKIP2,FLIST2,PROGRESS2,STATS2 --human-readable --port "2222" --log-file /dev/stdout /mnt/app-3/postgresql/ rsync://root@localhost/postgresql
      rc=$?
      break
      fi
      done
```
### Single container approach
In this approach we will be spawning a single container to perform the operations and functionalities of Stunnel as well as Rsync utilities.
For the single container:
- We will start off with starting the Stunnel process with the Stunnel config in background mode.
- Then continuously checkup on the `localhost` interface (port 2222) whether the stunnel process is up and ready to accept connections using the `nc` command (Solves problem 1).
- Once the Stunnel process server is ready to accept connection requests (again based on the `nc` commands exit code), we will execute the Rsync command
- On completion of the Rsync command execution we will break the loop and `exit` the container, thus killing the stunnel process, consequently marking the container (and Pod) as complete. (Solves Problem 2)
- The above logic will be embedded in a bash script in the container command as follows:
```
  - command:
    - /bin/bash
    - -c
    - |
      /bin/stunnel /etc/stunnel/stunnel.conf
      while true
      do nc -z localhost 2222
      if [ $? -eq 0 ]
      then
      rsync --archive --delete --recursive --hard-links --partial --info=COPY2,DEL2,REMOVE2,SKIP2,FLIST2,PROGRESS2,STATS2 --human-readable --port "2222" --log-file /dev/stdout /mnt/app-3/postgresql/ rsync://root@localhost/postgresql 
      break
      fi
      done
      exit 0
```

## Comparing Rsync + stunnel as 2 containers vs Rsync + Stunnel in a single container

| Parameters                                  | Single Containers                                                                                                                                               | Separate Containers                                                                                                                                                                                    | Preferred approach  |
|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------|
| Logging and Event timeline                  | The rsync and stunnel logs will be  merged into single view and combined  logging provides a merged timeline of events  that occurred during the data transfer. | Each rsync and stunnel container will have their own separate logs and as logs are decoupled, both the containers will have a separate and exclusive timeline of events only pertaining to themselves. | Tie                 |
| Competition for Resources (CPU, Memory etc) | There is only one container for both the  processes, competition for resources might take place in this approach.                                               | Separate resource constraints and allocation can be done for both the processes as they have separate containers, resource competition scenario is unlikely to take place.                             | Separate containers |
| Updating DVMP                               | DVMP will update based on the container status of the combined container of rsync + stunnel.                                                                    | DVMP will update itself based on the container status of the rsync container only.                                                                                                                     | Separate containers |
| Event timeline                              | Combined logging provides a merged timeline of events  that occurred during the data transfer.                                                                  | As logs are decoupled, both the containers will have a  separate and exclusive timeline of events only pertaining to themselves.                                                                       | Single containers   |
## Outcome
We have decided that we will be going ahead with the separate container approach because of the following reasons:
- Separate container approach had 2 main concerns (which handled by logic embedded in the bash scripts for each container):
    - Dependency of Stunnel process on Rsync execution: Consider a case when Rsync did not execute successfully, it does not matter because the inter-process communication
    would not falter as we are creating the `rsync-client-container-done` file on the shared volume irrespective of Rsync's successful execution. This Rsync and Stunnel both will go to completion, and the 
    dependency would not hinder the migration workflow.
    - Dependency of Rsync process on Stunnel: Consider a case when Stunnel process does not start, and the stunnel container goes in error state, consequently the rsync container
    would not get executed as the stunnel process would not be available for connection acceptance, thus both containers would not be successfully executed, and the Pod status as a whole
    would be updated accordingly, being evident of what is happening with Stunnel and Rsync.
- Single container does not offer anything different from the separate container approach, except that the merged logs of both the processes provide a lucid timeline of events that took place.
At the same time, we are not settling for any lower debug experience from what we have currently, that is separate logs in different Pods, this shall remain the same in separate container approach.
- The main concern with single container approach is the competition of resources (CPU, Memory), this might result in varying problems, and these problems might need different solutions in different production
environments, as a whole not a nice problem to solve if occurred. On the other hand as we have seen, separate container concerns can be solved with the help of embedding logic in the bash scripts once in for all. 
- All in all, decoupling the stunnel process from rsync enables us to keep everything a bit modular.

## Implementation

- Stunnel Configuration changes:
    - Removal of `foreground = yes` so that the stunnel process runs as daemon.
    - Addition of `output = /dev/stdout` for streaming of stunnel process logs as we are running the process in daemon mode.
    - Updating `accept = localhost:2222` in order to facilitate data transfer between rsync and stunnel over the lcoalhost interface.
- Transfer Image changes: These involve the addition of new packages/utilities used for embedding new workflow log in the container commands of rsync and stunnel.
- Accommodate Stunnel alongside Rsync in the `RunRsyncOperations` phase:
    - Modifying `getRsyncClientPodRequirements` to have destination IP for rsync process as `localhost`.
    - Modifying  `getRsyncClientPodTemplate` :
        - To add Stunnel Volumes consisting of stunnel configuration file, stunnel certs as well as the volume called `rsync-stunnel-ipc` (used for facilitating inter-process communication between rsync and stunnel).  
        - To add Stunnel VolumeMounts of stunnel certs, configs, `rsync-stunnel-ipc`. 
        - To add `rsync-stunnel-ipc` alongside the PV involved in the migration of the application.
        - To add bash scripts for rsync container command as well as the stunnel contaienr command.
        - To add stunnel container manifest consisting of all the above changes stunnel changes.
- DVMP seemed to work fine with the above changes, hence no additional changes needed in its controller.