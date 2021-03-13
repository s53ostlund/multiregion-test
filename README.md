## Overview
The goal here is to create a regional cluster with nodes and two
zones with read-write priviledges on the same persistent disk. It
seems to work, but seems it should not since accessModes in pv.yaml
is ReadWriteOnce. The only way I can understand this is that GKE
uses disk replication to actually allow this via regional disk
replication.

It seems that the correct way to do this should be with the
--multi-writer flag on disk creation, available with the beta version
of the gcloud command. But that introduces a host of other problems
such as no snapshotting and no creation of image from snapshots,
and no e2-standard-2 possibility of VM and unavailability in some
regions. The no-snapshotting is a dealbreaker so I abandoned the
--multi-writer option.

 

The description page below from google makes no mention of the disk
being writable only from one pod but the example assumes it since
the wordpress example is not replicated. Same story with the
nfs-server example of timberry although there it is explicitly
stated that the disk should be accessible from a single pod.

However by experimenting, it seems that the regional disk is in
fact Read/Writable  from all pods in the two chosen zones in the region. 
I verified that the nginx containers were in two nodes and
did read read and write the nginx containers and saw that the files did
appear in the other pods (immediately? latency?)
I don't know if there will be problems with locks on the disks and
can find no guidance if write works and is replicated across the
zone-shared disk.

## Source
	https://cloud.google.com/solutions/using-kubernetes-engine-to-deploy-apps-with-regional-persistent-disks
	https://timberry.dev/posts/shared-storage-gke-regional-pds-nfs/
## Instructions
	gcloud init
	# set default zone to 17
	kubectl config set-context --current --namespace=default
	# For blank disk creation 
	gcloud compute disks create gce-disk-1 \
   		--size 200Gi \
   		--region europe-west1 \
   		--replica-zones europe-west1-b,europe-west1-c

	# EXAMPLE WITH DISK CREATION FROM SNAPSHOT
	# note that the snapshot disk actually comes from another region; nice!
	gcloud beta compute disks create gce-disk-1 
		--project=production-XXXX
		--type=pd-ssd --size=200GB 
		--region=europe-west1 
		--replica-zones=projects/production-XXXXX/zones/europe-west1-b,projects/production-XXXXX/zones/europe-west1-c 
		--source-snapshot=opentaproject-mnt-europe-west3-a-nnnnnnn-el0he5zo

    	# Note that the CLUSTER_VERSION flag should not be necessary since default should be OK
	# CLUSTER_VERSION=$(gcloud container get-server-config \
 	#	 --region europe-west1 --format='value(validMasterVersions[0])')

	gcloud container clusters create ${CLUSTER_NAME} \
		# --cluster-version=${CLUSTER_VERSION} \
		--machine-type=e2-standard-2 \
		--region=europe-west1 \
		--num-nodes=1 \
		--node-locations=europe-west1-b,europe-west1-c
	gcloud container clusters get-credentials $CLUSTER_NAME  --region europe-west1
	k apply -f storageclass.yaml
	k apply -f pv.yaml
	k apply -f pvc.yaml
	k apply -f deployment.yaml
	k apply -f service.yaml
	
	k get pods 
	
	# k exec -it  nfs-serverXXXXX -- /bin/bash
	# in the pod
		cd /nfsshare
		echo "<h1> Success </h1>" > index.html
		exit

	k apply -f nginx-deployment.yaml
	k apply -f nginx-service.yaml
	
## Test it
	kubectl get pod -o=custom-columns=NODE:.spec.nodeName,NAME:.metadata.name 
	curl -k http://34.76.237.104

