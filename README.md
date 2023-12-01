
# Tests with S3 Nooba using ODF

## This repo is to test application using S3 storage

Procedures that will be performed in this tests are

1. Create a Object bucket class (OBC)  using Nooba - ODF in openshift 
2. Create a sample container which will upload a test file on the S3 bucket
3. Create another container which will write the stdout to the same file in s3 bucket


## 1. Creating an OBC using Nooba - ODF in Openshift

In the current setup , ODF has already created a storage class that will utilize Nooba as an S3 Multi cloud gateway in the existing environment

```
[root@ocp-bastion ocp-ipi-vmware]# oc get sc
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lso-odf                       kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  18d
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   18d
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  18d
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   18d
odf                           kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  24d
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  18d <------Nooba storage class
thin (default)                kubernetes.io/vsphere-volume            Delete          Immediate              false                  31d
thin-csi                      csi.vsphere.vmware.com                  Delete          WaitForFirstConsumer   true                   31d

```

We will create an Object Bucket claim (OBC) that will bind an Object Bucket (OB).This OBC will create a configmap and a secret as well.

Creating an OBC 

```
apiVersion: objectbucket.io/v1alpha1
  kind: ObjectBucketClaim
  metadata:
    labels:
      app: noobaa
      bucket-provisioner: openshift-storage.noobaa.io-obc
      noobaa-domain: openshift-storage.noobaa.io
    name: s3-test
    namespace: test-s3
  spec:
    additionalConfig:
      bucketclass: noobaa-default-bucket-class
    generateBucketName: s3-test
    storageClassName: openshift-storage.noobaa.io

```

The above creates a config map , a secret and an Object bucket as well

```
[root@ocp-bastion ocp-ipi-vmware]# oc get cm,secret,ob | grep -i s3-test <---- s3-test is the prefix that we selected in OBC 
configmap/s3-test                    5      102m
secret/s3-test                    Opaque                                2      102m
objectbucket.objectbucket.io/obc-test-s3-s3-test   openshift-storage.noobaa.io                                  Delete           Bound   102m

```



## 2. Creating a sample container which uploads a file 

In this step we shall create a sample container which will create a file and upload it to the S3 bucket 

Creating a file with below contents 

```
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod 
  namespace: test-s3
spec:
  containers:
  - name: s3-container
    image: public.ecr.aws/aws-cli/aws-cli
    imagePullPolicy: IfNotPresent
    command: ["/bin/bash", "-c","echo this is a file created by $HOSTNAME created on `date` | tee -a  test.txt && aws s3 --endpoint http://$BUCKET_HOST sync . s3://$BUCKET_NAME/ && rm -f test.txt && sleep 10000000"]
    env: 
      - name: AWS_CA_BUNDLE
        value: /run/secrets/kubernetes.io/serviceaccount/service-ca.crt
    envFrom:
      - configMapRef: 
          name: s3-test
      - secretRef:
          name: s3-test 

```

The command in the above example creates a file called test.txt with the pod hostname and creation date.It then uploads it to the S3 bucket using aws cli.
Note that we are referncing the environment variables in the container using "envFrom" from the configmap and the secret that the OBC created.

Once the above file is applied it creates a pod as below

```
[root@ocp-bastion ocp-ipi-vmware]# oc get pods
NAME          READY   STATUS    RESTARTS   AGE
s3-test-pod   1/1     Running   0          44m

```


## 3. Creating a sample container which changes the S3 file 

```
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod-stream
  namespace: test-s3
spec:
  containers:
  - name: s3-container-stream
    image: public.ecr.aws/aws-cli/aws-cli
    imagePullPolicy: IfNotPresent
    command: ["/bin/bash", "-c","echo this line created by $HOSTNAME created on `date` |  aws s3 --endpoint http://$BUCKET_HOST cp - s3://$BUCKET_NAME/test.txt && sleep 10000000"]
    env: 
      - name: AWS_CA_BUNDLE
        value: /run/secrets/kubernetes.io/serviceaccount/service-ca.crt
    envFrom:
      - configMapRef: 
          name: s3-test
      - secretRef:
          name: s3-test 

```

To validate if the second pod did write to the s3 file , go to any of the two pods and run the below commands

```
[root@ocp-bastion ocp-ipi-vmware]# oc rsh s3-test-pod

sh-4.2# aws s3 --endpoint http://$BUCKET_HOST cp s3://$BUCKET_NAME/test.txt -

```


You will see that the line created by our first test pod has been replaced by our second test pod , something like below

```
sh-4.2# aws s3 --endpoint http://$BUCKET_HOST cp s3://$BUCKET_NAME/test.txt -
this line created by s3-test-pod-stream created on Fri Dec 1 16:09:57 UTC 2023


```

## 4. Creating a sample container which pulls the S3 file

Creating the yaml file as below



Applying the file 

```

[root@ocp-bastion ocp-ipi-vmware]# oc create -f s3-test-pod-download.yaml 
Warning: would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "s3-container-download" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "s3-container-download" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "s3-container-download" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "s3-container-download" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/s3-test-pod-download created
[root@ocp-bastion ocp-ipi-vmware]# 
[root@ocp-bastion ocp-ipi-vmware]# 
[root@ocp-bastion ocp-ipi-vmware]# 
[root@ocp-bastion ocp-ipi-vmware]# 
[root@ocp-bastion ocp-ipi-vmware]# oc get pods
NAME                   READY   STATUS    RESTARTS   AGE
s3-test-pod            1/1     Running   0          59m
s3-test-pod-download   1/1     Running   0          3s
s3-test-pod-stream     1/1     Running   0          10m
```


After rsh'ing into the container we can see that the container has downloaded the s3 file with the contents of our second pod

```
[root@ocp-bastion ocp-ipi-vmware]# oc rsh s3-test-pod-download 
sh-4.2# ls
test.txt
sh-4.2# cat test.txt 
this line created by s3-test-pod-stream created on Fri Dec 1 16:09:57 UTC 2023
sh-4.2# 

```

