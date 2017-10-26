# KOPS README AWS

# Creating a cluster:

## Notes:
* The following demonstrates how to create a config and apply it in AWS.
* After creating the cluster with `kops update cluster...` the kube config is generated automagically for you
  under ~/.kube/config.  Do not lose this file as it is how kubectl will interact with the cluster.
* If you want to use the existing config it is at `s3://srflaxu40-files-internal/kube-dev-config`.  Other files
  exist for other kube environments.
* See example.kops.sh for a live-example.

## Delete Config:
* Cleanup previous ownership (if necessary):
  * `kops delete cluster --name kubernetes-develop.k8s.local --yes`

## Create Config:
```
./kops.sh
```
* The above command will create a config in the specified bucket outlined by your KOPS_STATE_STORE environment variable, which automatically points at s3://kubernetes-(develop|staging|production).

## Get your config:
* Note - you can get your IGs by listing them:
`kops get ig --name ${CLUSTER_NAME}`

```
kops get cluster ${CLUSTER_NAME} -o yaml --full > cluster.yaml 
kops get ig nodes --name ${CLUSTER_NAME} -o yaml > nodes.yaml 
kops get ig <MASTER IG NAME> --name ${CLUSTER_NAME} -o yaml > master.yaml
```
* Merge the `cluster.yaml`, `nodes.yaml` and `master.yaml` into cluster.yaml separated by `---`.

## Edit Config:
  * You may need to edit things in the config like subnet CIDRs so they do not conflict with existing ones.

## Remove old config:
  * Remove the old config that was created in the bucket; in this example our cluster is in develop.
```
kops cluster --name=kubernetes-develop.k8s.local --yes
```

## Now re-create config in stateful bucket:
```
kops create -f cluster.yaml
```

## Create Secret:
* The `id_rsa.pub` file is a root key for each respective environment (srflaxu40-cali.. in dev / staging and srflaxu40-east... in prod).
```
kops create secret --name kubernetes-develop.k8s.local sshpublickey admin -i /home/ubuntu/.ssh/id_rsa.pub
```

## Create the cluster and its resources in AWS:
```
kops update cluster --create-kube-config --yes --name kubernetes-develop.k8s.local
```

## You should get the following output (after grabbing a coffee and waiting about 5-10 minutes for the infrastructure to get created):
```
syntax: NAME -i <PublicKeyPath>
root@ip-10-9-2-86:~# kops update cluster --create-kube-config --yes --name kubernetes-^Cvelop.k8s.local
root@ip-10-9-2-86:~# kops create secret --name kubernetes-develop.k8s.local sshpublickey admin -i /home/ubuntu/.ssh/id_rsa.pub 
root@ip-10-9-2-86:~# kops update cluster --create-kube-config --yes --name kubernetes-develop.k8s.local
I0912 03:14:53.784342   17483 apply_cluster.go:420] Gossip DNS: skipping DNS validation
W0912 03:14:53.807242   17483 firewall.go:202] Opening etcd port on masters for access from the nodes, for calico.  This is unsafe in untrusted environments.
I0912 03:14:53.912335   17483 executor.go:91] Tasks: 0 done / 75 total; 34 can run
I0912 03:14:54.433916   17483 vfs_castore.go:422] Issuing new certificate: "kube-scheduler"
I0912 03:14:54.653238   17483 vfs_castore.go:422] Issuing new certificate: "kube-controller-manager"
I0912 03:14:54.675100   17483 vfs_castore.go:422] Issuing new certificate: "kubelet"
I0912 03:14:54.922132   17483 vfs_castore.go:422] Issuing new certificate: "kubecfg"
I0912 03:14:55.107849   17483 vfs_castore.go:422] Issuing new certificate: "kube-proxy"
I0912 03:14:55.329383   17483 vfs_castore.go:422] Issuing new certificate: "kops"
I0912 03:14:55.701362   17483 executor.go:91] Tasks: 34 done / 75 total; 14 can run
I0912 03:14:56.066381   17483 executor.go:91] Tasks: 48 done / 75 total; 21 can run
I0912 03:14:57.596387   17483 launchconfiguration.go:327] waiting for IAM instance profile "nodes.kubernetes-develop.k8s.local" to be ready
I0912 03:14:58.102142   17483 launchconfiguration.go:327] waiting for IAM instance profile "masters.kubernetes-develop.k8s.local" to be ready
I0912 03:15:08.302765   17483 executor.go:91] Tasks: 69 done / 75 total; 4 can run
I0912 03:15:09.894392   17483 vfs_castore.go:422] Issuing new certificate: "master"
I0912 03:15:09.964679   17483 executor.go:91] Tasks: 73 done / 75 total; 2 can run
I0912 03:15:09.981992   17483 natgateway.go:268] Waiting for NAT Gateway "nat-0ebd1bc9b740073c9" to be available (this often takes about 5 minutes)
I0912 03:17:11.140117   17483 executor.go:91] Tasks: 75 done / 75 total; 0 can run
I0912 03:17:11.140307   17483 kubectl.go:133] error running kubectl config view --output json
I0912 03:17:11.140319   17483 kubectl.go:134] 
I0912 03:17:11.140332   17483 kubectl.go:135] 
W0912 03:17:11.140354   17483 update_cluster.go:235] error reading kubecfg: error getting config from kubectl: error running kubectl: exec: "kubectl": executable file not found in $PATH
I0912 03:17:11.351310   17483 update_cluster.go:247] Exporting kubecfg for cluster
Kops has set your kubectl context to kubernetes-develop.k8s.local

Cluster changes have been applied to the cloud.


Changes may require instances to restart: kops rolling-update cluster
```

## Bastion Server Access (only do this once one bastion is needed alone):
* Create the bastion:
```
kops create instancegroup --role Bastion --subnet utility-us-west-1a bastion-ig
```

* Apply Changes to the active cluster:
```
kops update cluster --create-kube-config --yes --name kubernetes-develop.k8s.local
```

* SSH to the bastion so you can interact with any cluster hosts:
```
ssh -i ~/credentials/ssh/srflaxu40-california-20140821.pem admin@bastion-kubernetes-develo-61ii5m-1245457584.us-west-1.elb.amazonaws.com
```

## KOPSome-Sauce:
* KOPS just created a config for you under `~/.kube/config`
* Copy this to s3://srflaxu40-files-internal as it specifies the configuration to interact with your cluster in AWS.

---

# Handy Tricks:

## Deleting your cluster:
```
kops cluster --name=kubernetes-develop.k8s.local --yes
```

## Validating your cluster is working properly:
```
kops validate cluster
```

## Generating access-tokens:
* See the "Accesing-kube" documentation below.

---

## References:
* [Bastion](https://github.com/kubernetes/kops/blob/master/docs/bastion.md)
* [REACTIVE-KOPS](http://blog.reactiveops.com/kops-an-inside-look-at-deploying-kubernetes-on-aws-the-reactiveops-way)
* [HA-KOPS](https://github.com/kubernetes/kops/blob/master/docs/high_availability.md)
* [Install the kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [KOPS RELEASE NOTES](https://github.com/kubernetes/kops/releases/tag/1.7.0)
* [KOPS Installation](https://github.com/kubernetes/kops)
* [GOSSIPING KUBIES](http://blog.arungupta.me/gossip-kubernetes-aws-kops/)
* [accessing-kube](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/)
* [calico](https://docs.projectcalico.org/v2.0/getting-started/kubernetes/)
* You can see how kops is installed in the ./kops.sh script.
