## provision 2-nodes cluster in aws
```bash
cd cloudformation/
aws cloudformation deploy --template-file k8s_cluster.yaml --stack-name K8SCluster
```
get manager and worker node IPs:
```bash
aws cloudformation describe-stacks --stack-name K8SCluster | jq -r '.Stacks[0].Outputs[] | {OutputKey, OutputValue} | join(": ")'
```
## install & configure kubernetes software
```bash
cd ../ansible
export ANSIBLE_HOST_KEY_CHECKING=False
```
update ips in `inventory.yaml`

```bash
ansible-playbook install_prerequisites.yaml -i inventory.yaml
ansible-playbook install_container_runtime.yaml -i inventory.yaml
ansible-playbook install_k8s.yaml -i inventory.yaml
ansible-playbook init_k8s_cluster.yaml -i inventory.yaml
ansible-playbook install_cni.yaml -i inventory.yaml
```

to restart failed task use `--start-at-task`:
```bash
ansible-playbook init_k8s_cluster.yaml -i inventory.yaml --start-at-task="mkdir -p ~/.kube"
```

## verify
ssh to manager
```bash
ssh -i ~/.ssh/ec2key.pem ubuntu@13.37.245.112
kubectl get nodes
kubectl get po -n kube-system
```
pods `weave-net-*` and `coredns-*` should be up and running

verify resource creation & pod networking
```bash
ubuntu@ip-172-31-35-173:~$ alias k=kubectl
ubuntu@ip-172-31-35-173:~$ k run nginx --image=nginx
pod/nginx created
ubuntu@ip-172-31-35-173:~$ k get po -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          58s   10.44.0.1   node01   <none>           <none>
ubuntu@ip-172-31-35-173:~$
ubuntu@ip-172-31-35-173:~$ k run test --image=busybox --rm -it -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -qO- 10.44.0.1 | grep "<h1>Welcome"
<h1>Welcome to nginx!</h1>
/ # 
```
verify service networking & dns:
```bash
ubuntu@ip-172-31-35-173:~$ k expose pod/nginx --name=nginx-svc --port=80
service/nginx-svc exposed
ubuntu@ip-172-31-35-173:~$ k get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   11m
nginx-svc    ClusterIP   10.104.21.86   <none>        80/TCP    8s
ubuntu@ip-172-31-35-173:~$ curl -s http://10.104.21.86/ | grep "<h1>Welcome"
<h1>Welcome to nginx!</h1>
ubuntu@ip-172-31-35-173:~$ k run test --image=busybox --rm -it -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -qO- nginx-svc | grep "<h1>Welcome"
<h1>Welcome to nginx!</h1>
/ # wget -qO- 10.104.21.86 | grep "<h1>Welcome"
<h1>Welcome to nginx!</h1>
/ # 
```
(from node01 service works by IP, from pod - by dns name)

