# Issue in Scaling of MCM during Hibernation of Shoot


## Problem Statement
Recently, some clusters were seen on canary where shoot hibernation was stuck due to machines not being deleted. 
This was happening because the MCM was not up. The links for those clusters are as follows:
1. https://dashboard.garden.canary.k8s.ondemand.com/namespace/garden-prj-hxm/shoots/cilium-noenc/
2. https://dashboard.garden.canary.k8s.ondemand.com/namespace/garden-prj-hxm/shoots/noenc/
3. https://dashboard.garden.canary.k8s.ondemand.com/namespace/garden-prj-hxm/shoots/noolk-ebfdp/

In all the above mentioned clusters, the shoot was first in the creation phase, the `MCM` gets deployed and brings up the machines.
After this for some reason, the shoot creation fails and at the same time shoot hibernation is enabled. So in the next reconciliation, the
shoot hibernation logic is executed. According to [this](https://github.com/gardener/gardener/blob/219d828fcdea81fb3edf13de2736daf81e137923/extensions/pkg/controller/worker/genericactuator/actuator_reconcile.go#L59-L74),
logic, the replica count of MCM deployment will always be `0`. So even though the machines are up and running, there will be no `MCM` to bring them down and so the 
hibernation will be stuck.

There is already an issue opened by `Plamen Kokanov` regarding the same. It can be checked out [here](https://github.com/gardener/gardener/issues/7151). 

## Possible shoot operations:-
The following are the possible shoot `lastOperationTypes`:-
```go
const (
	// LastOperationTypeCreate indicates a 'create' operation.
	LastOperationTypeCreate LastOperationType = "Create"
	// LastOperationTypeReconcile indicates a 'reconcile' operation.
	LastOperationTypeReconcile LastOperationType = "Reconcile"
	// LastOperationTypeDelete indicates a 'delete' operation.
	LastOperationTypeDelete LastOperationType = "Delete"
	// LastOperationTypeRestore indicates a 'restore' operation.
	LastOperationTypeRestore LastOperationType = "Restore"
	// LastOperationTypeMigrate indicates a 'migrate' operation.
	LastOperationTypeMigrate LastOperationType = "Migrate"
)
```

## MCM Replica func

The function that decided the mcm replica count is as follows:-
```go
var mcmReplicaFunc = func() int32 {
		switch {
		// If the cluster is hibernated then there is no further need of MCM and therefore its desired replicas is 0
		case extensionscontroller.IsHibernated(cluster):
			return 0
		// If the cluster is created with hibernation enabled, then desired replicas for MCM is 0
		case extensionscontroller.IsHibernationEnabled(cluster) && extensionscontroller.IsCreationInProcess(cluster):
			return 0
		// If shoot is either waking up or in the process of hibernation then, MCM is required and therefore its desired replicas is 1
		case extensionscontroller.IsHibernatingOrWakingUp(cluster):
			return 1
		// If the shoot is awake then MCM should be available and therefore its desired replicas is 1
		default:
			return 1
		}
	}
```

## Possible Scenarios for Shoot Hibernation.

### 1. Shoot is already hibernated (status.hibernated: false)
In this case, we do not need MCM as the cluster is already hibernated. The replica function will also return `0`, so this case is handled.

### 2. Shoot is reconciling and Hibernation is enabled (lastOperationType: Reconcile) 
In this case we need MCM, because there might be machines in the cluster that need to be deleted or the cluster is waking up from hibernation and MCM is needed to bring up machines.
In both cases the MCM replica function will return `1`. So this case is handled correctly.

### 3. Shoot is created with Hibernation enabled (lastOperationType: Create, spec.hibernation.enabled: true)
In this case the shoot reconciliation flow will run and all the control plane components should come up with `0` replicas.
The MCM replica func will return `0` and so this case is handled correctly.

### 4. Hibernation is enabled and shoot creation is in Progress (lastOperationType: Create)
Here we can have the following cases:-
#### a. Hibernation is enabled before machine related resources are deployed:-
There are 2 possible cases in this:-
##### i. reconciliation flow has no errors:-
First creation of cluster will happen and then reconcile Operation will happen and hibernation will take place.
MCM replica func will return `1`. This case is handled correctly.
##### ii. reconciliation flow fails due to some error:-
Unable to reproduce this for some reason. But should not be an issue as MCM won't be needed anyways.

#### b. Hibernation is enabled after machine related resources are deployed.
There are 2 possible cases in this:-
1. **reconciliation flow has no errors**:-
First creation of cluster will happen and then reconcile Operation will happen and hibernation will take place.
MCM replica func will return `1`. This case is handled correctly.
2. **reconciliation flow fails due to some error**:-
This case was reproduced by scaling down the api-server after machines are up and running. This causes failure in the creation of cluster.
In the next reconciliation, the MCM replica func will return `0` but the machines are still there. So, this case is not handled correctly.
Check out the dev cluster on which the case was reproduced [here](https://dashboard.garden.dev.k8s.ondemand.com/namespace/garden-i544024/shoots/hib-test/).

### 5. spec.hibernation.enabled: true and deletionTimestamp!=nil
The deletion logic of the worker controller deploys MCM, cleans up everything and then deletes MCM, so no errors in this case.
The following cases were checked:-
1. Create a shoot, enable hibernation and in between delete the shoot.
2. Create a shoot with hibernation enabled and delete the shoot in between.
### 6. Control Plane Migration
No test has been done for this but after taking a look at the code, it seems that there should be no issues here.
See [actuator_migrate](https://github.com/gardener/gardener/blob/0d37c8a33483d8e497a0f79dc70f38a0aeecfd57/extensions/pkg/controller/worker/genericactuator/actuator_migrate.go#L32) and [actuator_restore](https://github.com/gardener/gardener/blob/0d37c8a33483d8e497a0f79dc70f38a0aeecfd57/extensions/pkg/controller/worker/genericactuator/actuator_restore.go#L37).
