1. Create filesystem (https://us-west-1.console.aws.amazon.com/efs/home?region=us-west-1#/file-systems)
2. give it a name "awsefs"
3. select the proper vpc for the volume to be available in
4. click create

5. Get the security group that is assigned to the network endpoionts
6. Review the security group assigned to the network endpoint.
7. If the security group does not allow NFS (2049) from your worker and masters, you will need to update the security group
 
https://docs.openshift.com/container-platform/4.6/storage/persistent_storage/persistent-storage-efs.html


1. oc new-project efsprovisioner
2. 
efsconfigmap.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: efs-provisioner
  namespace: efsprovisioner
data:
  file.system.id: fs-a752b4bf 
  aws.region:  us-west-1
  provisioner.name: openshift.org/aws-efs 
  dns.name: "fs-a752b4bf.efs.us-west-1.amazonaws.com" 
```
3. oc create -f efsconfigmap.yaml
4. oc create serviceaccount efs-provisioner
5. vi clusterrole.yaml
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: efs-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    verbs: ["use"]
    resourceNames: ["hostmount-anyuid"]
```
6. vi clusterrolebinding.yaml
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-efs-provisioner
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    namespace: efsprovisioner 
roleRef:
  kind: ClusterRole
  name: efs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```
7. vi role.yaml
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```
8. vi rolebinding.yaml
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    namespace:  efsprovisioner
roleRef:
  kind: Role
  name: leader-locking-efs-provisioner
  apiGroup: rbac.authorization.k8s.io
```
9. oc create -f clusterrole.yaml,clusterrolebinding.yaml,role.yaml,rolebinding.yaml
10. vi storageclass.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-efs
provisioner: openshift.org/aws-efs
parameters:
  gidMin: "2048" 
  gidMax: "2147483647" 
  gidAllocate: "true" 
```
11. oc create -f storageclass.yaml
12. vi provisioner-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-provisioner
  labels:
    app: efs-provisioner
  namespace: efsprovisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: efs-provisioner
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
        serviceAccount: efs-provisioner
        containers:
            - name: efs-provisioner
              image: quay.io/external_storage/efs-provisioner:latest
              env:
                - name: PROVISIONER_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: efs-provisioner
                      key: provisioner.name
                - name: FILE_SYSTEM_ID
                  valueFrom:
                    configMapKeyRef:
                      name: efs-provisioner
                      key: file.system.id
                - name: AWS_REGION
                  valueFrom:
                    configMapKeyRef:
                      name: efs-provisioner
                      key: aws.region
                - name: DNS_NAME
                  valueFrom:
                    configMapKeyRef:
                      name: efs-provisioner
                      key: dns.name
                      optional: true
              volumeMounts:
                - name: pv-volume
                  mountPath: /persistentvolumes
        volumes:
            - name: pv-volume
              nfs:
                server: <file-system-id>.efs.<region>.amazonaws.com 
                path: /
```
13.  oc create -f provisioner-deployment.yaml
14.  oc get pods
15.  oc logs \/<nfs provisioner pod\/>
16.  set storageclass as default `oc patch storageclass aws-efs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
17.  oc new-project test-efs
18.  vi testpvc.yaml
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: efs-claim 
  namespace: test-efs
  annotations:
    volume.beta.kubernetes.io/storage-provisioner: openshift.org/aws-efs
  finalizers:
    - kubernetes.io/pvc-protection
spec:
  accessModes:
    - ReadWriteOnce 
  resources:
    requests:
      storage: 5Gi 
  storageClassName: aws-efs 
  volumeMode: Filesystem
```
1.  oc create -f testpvc.yaml
2.  

