# ☸️ Kubernetes Cheat Sheet

A practical, copy-paste-ready reference for `kubectl` and the surrounding Kubernetes tooling (`kubeadm`, `crictl`, `etcdctl`, `helm`, `kustomize`).

> **Conventions used below**
> `<pod>` = resource name · `-n <ns>` = namespace · `-A` = all namespaces · `-l` = label selector
> Most `kubectl` commands accept `-n`, `-A`, `-o`, `-l`, `--context` — they are only repeated where it matters.

---

## 📑 Table of Contents

1. [Setup, Aliases & Autocompletion](#1-setup-aliases--autocompletion)
2. [Kubeconfig, Contexts & Clusters](#2-kubeconfig-contexts--clusters)
3. [Cluster Information](#3-cluster-information)
4. [Resource Types & Short Names](#4-resource-types--short-names)
5. [Viewing Resources (get / describe)](#5-viewing-resources-get--describe)
6. [Output Formats, JSONPath & Custom Columns](#6-output-formats-jsonpath--custom-columns)
7. [Creating Resources (Imperative)](#7-creating-resources-imperative)
8. [Declarative Management (apply / diff)](#8-declarative-management-apply--diff)
9. [Generating YAML (dry-run)](#9-generating-yaml-dry-run)
10. [Updating Resources (edit / patch / set)](#10-updating-resources-edit--patch--set)
11. [Deleting Resources](#11-deleting-resources)
12. [Pods: Logs, Exec, Copy, Attach](#12-pods-logs-exec-copy-attach)
13. [Deployments & Rollouts](#13-deployments--rollouts)
14. [Scaling & Autoscaling](#14-scaling--autoscaling)
15. [Services, Networking & Port Forwarding](#15-services-networking--port-forwarding)
16. [ConfigMaps & Secrets](#16-configmaps--secrets)
17. [Storage (PV / PVC / StorageClass)](#17-storage-pv--pvc--storageclass)
18. [Jobs & CronJobs](#18-jobs--cronjobs)
19. [Labels, Annotations & Selectors](#19-labels-annotations--selectors)
20. [Namespaces](#20-namespaces)
21. [Nodes: Cordon, Drain, Taints](#21-nodes-cordon-drain-taints)
22. [RBAC, ServiceAccounts & Auth](#22-rbac-serviceaccounts--auth)
23. [Resource Usage & Events](#23-resource-usage--events)
24. [Debugging & Troubleshooting](#24-debugging--troubleshooting)
25. [Waiting & Scripting](#25-waiting--scripting)
26. [Kustomize](#26-kustomize)
27. [Plugins & Krew](#27-plugins--krew)
28. [kubeadm (Cluster Lifecycle)](#28-kubeadm-cluster-lifecycle)
29. [etcd Backup & Restore](#29-etcd-backup--restore)
30. [Container Runtime (crictl)](#30-container-runtime-crictl)
31. [Node-level Troubleshooting](#31-node-level-troubleshooting)
32. [Helm Essentials](#32-helm-essentials)
33. [Global Flags Reference](#33-global-flags-reference)
34. [Pod Phases & Common Errors](#34-pod-phases--common-errors)
35. [YAML Skeletons](#35-yaml-skeletons)

---

## 1. Setup, Aliases & Autocompletion

```bash
# Version
kubectl version                          # client + server
kubectl version --client
kubectl version -o yaml

# Bash completion
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Zsh completion
source <(kubectl completion zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc

# PowerShell completion
kubectl completion powershell | Out-String | Invoke-Expression

# Alias + completion for the alias
alias k=kubectl
complete -o default -F __start_kubectl k

# Handy exports
export do="--dry-run=client -o yaml"     # k create deploy x --image=nginx $do
export now="--force --grace-period=0"    # k delete pod x $now
```

**Popular aliases**

```bash
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes -o wide'
alias kns='kubectl config set-context --current --namespace'
```

---

## 2. Kubeconfig, Contexts & Clusters

```bash
kubectl config view                                  # merged kubeconfig
kubectl config view --minify                         # current context only
kubectl config view --raw                            # include certs/secrets
kubectl config view -o jsonpath='{.users[*].name}'

kubectl config get-contexts                          # list contexts
kubectl config current-context
kubectl config use-context <context>                 # switch context
kubectl config rename-context <old> <new>
kubectl config delete-context <context>

kubectl config get-clusters
kubectl config get-users
kubectl config delete-cluster <cluster>
kubectl config delete-user <user>

# Set default namespace for current context
kubectl config set-context --current --namespace=<ns>

# Build a context manually
kubectl config set-cluster <cluster> --server=https://<ip>:6443 --certificate-authority=ca.crt --embed-certs
kubectl config set-credentials <user> --client-certificate=user.crt --client-key=user.key --embed-certs
kubectl config set-credentials <user> --token=<token>
kubectl config set-context <ctx> --cluster=<cluster> --user=<user> --namespace=<ns>

# Unset values
kubectl config unset users.<user>

# Use multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/cluster2.yaml
kubectl config view --flatten > ~/.kube/merged.yaml

# Per-command overrides
kubectl get pods --kubeconfig=./other.yaml
kubectl get pods --context=<ctx>

# Cloud provider kubeconfig
az aks get-credentials -g <rg> -n <cluster>          # AKS
az aks get-credentials -g <rg> -n <cluster> --admin
aws eks update-kubeconfig --region <region> --name <cluster>   # EKS
gcloud container clusters get-credentials <cluster> --region <region>  # GKE
```

---

## 3. Cluster Information

```bash
kubectl cluster-info
kubectl cluster-info dump                            # large debug dump
kubectl cluster-info dump --output-directory=./dump

kubectl get nodes
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'                  # API server health
kubectl get --raw='/livez?verbose'
kubectl get --raw /metrics | head
kubectl get componentstatuses                        # deprecated, may still work

kubectl api-versions                                 # all API groups/versions
kubectl api-resources                                # all resource kinds
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
kubectl api-resources --api-group=apps
kubectl api-resources -o wide                        # shows verbs
kubectl api-resources --verbs=list,get

kubectl explain pod                                  # schema docs
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy --recursive
kubectl explain pods --api-version=v1
```

---

## 4. Resource Types & Short Names

| Resource | Short | Resource | Short |
|---|---|---|---|
| pods | `po` | services | `svc` |
| deployments | `deploy` | replicasets | `rs` |
| statefulsets | `sts` | daemonsets | `ds` |
| namespaces | `ns` | nodes | `no` |
| configmaps | `cm` | secrets | — |
| persistentvolumes | `pv` | persistentvolumeclaims | `pvc` |
| storageclasses | `sc` | serviceaccounts | `sa` |
| ingresses | `ing` | ingressclasses | — |
| networkpolicies | `netpol` | endpoints | `ep` |
| endpointslices | — | events | `ev` |
| horizontalpodautoscalers | `hpa` | poddisruptionbudgets | `pdb` |
| cronjobs | `cj` | jobs | — |
| replicationcontrollers | `rc` | resourcequotas | `quota` |
| limitranges | `limits` | customresourcedefinitions | `crd`, `crds` |
| priorityclasses | `pc` | certificatesigningrequests | `csr` |
| roles / rolebindings | — | clusterroles / clusterrolebindings | — |

```bash
kubectl get all                # pods, svc, deploy, rs, sts, ds, jobs, cj, hpa (NOT cm/secrets/pvc/ing)
kubectl get all -A
kubectl get cm,secret,pvc,ing -n <ns>
```

---

## 5. Viewing Resources (get / describe)

```bash
kubectl get pods
kubectl get pods -A                                  # all namespaces
kubectl get pods -n <ns>
kubectl get pods -o wide                             # node, IP, nominated node
kubectl get pods --show-labels
kubectl get pods -L app,tier                         # labels as columns
kubectl get pods -l app=web                          # label selector
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l 'app,!canary'
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=<node>
kubectl get pods --field-selector metadata.namespace!=default
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
kubectl get pods -w                                  # watch
kubectl get pods --watch-only
kubectl get pods --no-headers
kubectl get pods <pod1> <pod2>
kubectl get deploy,svc,pods                          # multiple kinds
kubectl get pod <pod> -o yaml                        # full manifest
kubectl get pod <pod> -o yaml --show-managed-fields=false
kubectl get pods --chunk-size=500                    # page large lists
kubectl get pods --ignore-not-found
kubectl get -f manifest.yaml                         # resources defined in a file
kubectl get -k ./overlay                             # resources from kustomization

kubectl describe pod <pod>
kubectl describe pods -l app=web
kubectl describe node <node>
kubectl describe deploy <deploy>
kubectl describe -f manifest.yaml
kubectl describe pod <pod> --show-events=false
```

---

## 6. Output Formats, JSONPath & Custom Columns

| Flag | Description |
|---|---|
| `-o wide` | Extra columns |
| `-o yaml` / `-o json` | Full object |
| `-o name` | `kind/name` only (great for piping) |
| `-o jsonpath='{...}'` | Extract fields |
| `-o jsonpath-file=file` | JSONPath from file |
| `-o custom-columns=...` | Build your own table |
| `-o custom-columns-file=file` | Columns from file |
| `-o go-template='...'` | Go template |

```bash
# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pod <pod> -o jsonpath='{.status.podIP}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u
kubectl get secret <s> -o jsonpath='{.data.password}' | base64 -d
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGES:.spec.containers[*].image'
kubectl get nodes -o custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'

# With jq
kubectl get pods -o json | jq -r '.items[] | select(.status.phase!="Running") | .metadata.name'
```

---

## 7. Creating Resources (Imperative)

```bash
# Pods
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --port=80 --labels=app=web,tier=fe
kubectl run nginx --image=nginx --env=KEY=value --env=FOO=bar
kubectl run nginx --image=nginx --restart=Never              # plain pod
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh   # throwaway shell
kubectl run nginx --image=nginx --expose --port=80           # pod + ClusterIP svc
kubectl run nginx --image=nginx --command -- sleep 3600      # override command
kubectl run nginx --image=nginx -- arg1 arg2                 # override args
kubectl run nginx --image=nginx --requests='cpu=100m,memory=128Mi'   # removed in newer versions; use YAML
kubectl run nginx --image=nginx --overrides='{"spec":{"nodeName":"node1"}}'

# Deployments
kubectl create deployment web --image=nginx
kubectl create deployment web --image=nginx --replicas=3 --port=80
kubectl create deployment web --image=nginx --image=redis      # multi-container

# Services
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose deployment web --type=NodePort --port=80
kubectl expose deployment web --type=LoadBalancer --port=80 --name=web-lb
kubectl expose pod nginx --port=80 --name=nginx-svc
kubectl create service clusterip my-svc --tcp=80:8080
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30080
kubectl create service loadbalancer my-svc --tcp=80:8080
kubectl create service externalname my-svc --external-name=example.com

# Other kinds
kubectl create namespace dev
kubectl create configmap app-cfg --from-literal=KEY=value
kubectl create secret generic db --from-literal=password=s3cr3t
kubectl create job pi --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'
kubectl create job manual-run --from=cronjob/<cj>
kubectl create cronjob backup --image=busybox --schedule="0 2 * * *" -- /bin/sh -c 'date'
kubectl create serviceaccount app-sa
kubectl create quota team-q --hard=cpu=4,memory=8Gi,pods=20
kubectl create priorityclass high --value=1000 --description="high prio"
kubectl create poddisruptionbudget web-pdb --selector=app=web --min-available=2
kubectl create ingress web --class=nginx --rule="example.com/*=web:80"
kubectl create ingress web --rule="example.com/api*=api:8080,tls=my-tls-secret"
kubectl create token <sa> --duration=1h                      # short-lived SA token

# From files
kubectl create -f manifest.yaml
kubectl create -f ./dir/
kubectl create -f https://example.com/manifest.yaml
kubectl create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: demo
EOF
```

---

## 8. Declarative Management (apply / diff)

```bash
kubectl apply -f manifest.yaml
kubectl apply -f ./dir/                          # all files in dir
kubectl apply -f ./dir/ -R                       # recursive
kubectl apply -f https://url/manifest.yaml
kubectl apply -k ./overlay                       # kustomize
cat x.yaml | kubectl apply -f -

kubectl apply -f x.yaml --dry-run=client         # local validation
kubectl apply -f x.yaml --dry-run=server         # validated by API server/admission
kubectl apply -f x.yaml --server-side            # server-side apply (SSA)
kubectl apply -f x.yaml --server-side --force-conflicts
kubectl apply -f x.yaml --field-manager=my-tool
kubectl apply -f ./dir --prune -l app=web        # delete objects no longer in files (use with care)
kubectl apply -f x.yaml --validate=strict

kubectl diff -f manifest.yaml                    # show what would change
kubectl diff -k ./overlay

kubectl replace -f manifest.yaml                 # full replace
kubectl replace --force -f manifest.yaml         # delete + recreate

# Last-applied configuration
kubectl apply view-last-applied deploy/<name>
kubectl apply set-last-applied -f manifest.yaml --create-annotation
kubectl apply edit-last-applied deploy/<name>
```

---

## 9. Generating YAML (dry-run)

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml
kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml
kubectl create configmap cfg --from-file=app.conf --dry-run=client -o yaml > cm.yaml
kubectl create secret generic s --from-literal=k=v --dry-run=client -o yaml > secret.yaml
kubectl create job j --image=busybox --dry-run=client -o yaml -- echo hi > job.yaml
kubectl create cronjob cj --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- date
kubectl create role r --verb=get,list --resource=pods --dry-run=client -o yaml
kubectl create ingress ing --rule="a.com/*=svc:80" --dry-run=client -o yaml

# Export a live object as clean-ish YAML
kubectl get deploy web -o yaml --show-managed-fields=false > web.yaml
```

---

## 10. Updating Resources (edit / patch / set)

```bash
# Edit in $EDITOR
kubectl edit deploy/<name>
KUBE_EDITOR="nano" kubectl edit svc/<name>
kubectl edit deploy/<name> -o json

# set — targeted updates
kubectl set image deploy/web nginx=nginx:1.27            # container=image
kubectl set image deploy/web *=nginx:1.27                # all containers
kubectl set env deploy/web KEY=value
kubectl set env deploy/web KEY-                          # remove var
kubectl set env deploy/web --from=configmap/app-cfg
kubectl set env deploy/web --from=secret/db --prefix=DB_
kubectl set env deploy/web --list
kubectl set resources deploy/web -c=nginx --limits=cpu=500m,memory=512Mi --requests=cpu=100m,memory=128Mi
kubectl set serviceaccount deploy/web app-sa
kubectl set selector svc/web app=web-v2
kubectl set subject rolebinding/rb --user=alice --serviceaccount=ns:sa

# patch — strategic merge (default)
kubectl patch deploy web -p '{"spec":{"replicas":5}}'
kubectl patch node <node> -p '{"spec":{"unschedulable":true}}'
# JSON merge patch
kubectl patch svc web --type=merge -p '{"spec":{"type":"NodePort"}}'
# JSON patch (RFC 6902)
kubectl patch pod <pod> --type=json -p='[{"op":"replace","path":"/spec/containers/0/image","value":"nginx:1.27"}]'
kubectl patch deploy web --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/env/0"}]'
# Patch from file / subresource
kubectl patch deploy web --patch-file=patch.yaml
kubectl patch deploy web --subresource=scale -p '{"spec":{"replicas":2}}'
# Remove finalizers from a stuck object
kubectl patch <kind>/<name> -p '{"metadata":{"finalizers":null}}' --type=merge
```

---

## 11. Deleting Resources

```bash
kubectl delete pod <pod>
kubectl delete pod <pod1> <pod2>
kubectl delete -f manifest.yaml
kubectl delete -k ./overlay
kubectl delete pods -l app=web
kubectl delete pods --all -n <ns>
kubectl delete all --all -n <ns>                         # does not delete cm/secrets/pvc
kubectl delete ns <ns>                                   # deletes everything inside
kubectl delete pod <pod> --grace-period=0 --force        # force kill
kubectl delete pod <pod> --now                           # grace period = 1
kubectl delete deploy web --cascade=orphan               # keep ReplicaSets/pods
kubectl delete deploy web --cascade=foreground
kubectl delete pod <pod> --wait=false
kubectl delete pods --field-selector=status.phase=Failed -A
kubectl delete pods --field-selector=status.phase=Succeeded -A
kubectl delete pod <pod> --dry-run=server
kubectl delete pods -l app=web -i                         # interactive confirm
```

---

## 12. Pods: Logs, Exec, Copy, Attach

```bash
# Logs
kubectl logs <pod>
kubectl logs <pod> -c <container>                        # specific container
kubectl logs <pod> --all-containers
kubectl logs <pod> -f                                    # follow
kubectl logs <pod> --previous                            # crashed instance (-p)
kubectl logs <pod> --tail=100
kubectl logs <pod> --since=1h
kubectl logs <pod> --since-time=2026-01-01T10:00:00Z
kubectl logs <pod> --timestamps
kubectl logs -l app=web --all-containers --prefix --max-log-requests=10
kubectl logs deploy/web                                  # picks one pod
kubectl logs job/<job>
kubectl logs <pod> --limit-bytes=10000

# Exec
kubectl exec <pod> -- ls /
kubectl exec -it <pod> -- sh
kubectl exec -it <pod> -c <container> -- bash
kubectl exec deploy/web -- env
kubectl exec <pod> -- cat /etc/resolv.conf

# Attach
kubectl attach <pod> -it
kubectl attach <pod> -c <container>

# Copy (requires tar in the container)
kubectl cp <pod>:/path/file ./file
kubectl cp ./file <pod>:/path/file
kubectl cp <ns>/<pod>:/path ./local -c <container>
kubectl cp ./dir <pod>:/tmp/dir --retries=3
```

---

## 13. Deployments & Rollouts

```bash
kubectl rollout status deploy/web
kubectl rollout status deploy/web --timeout=120s
kubectl rollout history deploy/web
kubectl rollout history deploy/web --revision=3
kubectl rollout undo deploy/web                          # previous revision
kubectl rollout undo deploy/web --to-revision=2
kubectl rollout restart deploy/web                       # rolling restart
kubectl rollout restart deploy -n <ns>                   # all deployments in ns
kubectl rollout pause deploy/web
kubectl rollout resume deploy/web

# Works for daemonsets and statefulsets too
kubectl rollout restart ds/<ds>
kubectl rollout status sts/<sts>

# Record a change cause (shows in history)
kubectl annotate deploy/web kubernetes.io/change-cause="upgrade to 1.27" --overwrite

# Inspect ReplicaSets behind a deployment
kubectl get rs -l app=web
```

---

## 14. Scaling & Autoscaling

```bash
kubectl scale deploy/web --replicas=5
kubectl scale deploy/web --current-replicas=5 --replicas=3   # conditional
kubectl scale sts/db --replicas=3
kubectl scale --replicas=0 deploy --all -n <ns>              # scale everything down
kubectl scale -f deploy.yaml --replicas=2

# HPA (requires metrics-server)
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=70
kubectl get hpa
kubectl get hpa -w
kubectl describe hpa web
kubectl delete hpa web
```

---

## 15. Services, Networking & Port Forwarding

```bash
kubectl get svc
kubectl get svc -o wide
kubectl get endpoints <svc>
kubectl get endpointslices -l kubernetes.io/service-name=<svc>
kubectl get ingress
kubectl describe ingress <ing>
kubectl get ingressclass
kubectl get networkpolicy
kubectl describe networkpolicy <np>

# Port forwarding
kubectl port-forward pod/<pod> 8080:80
kubectl port-forward svc/<svc> 8080:80
kubectl port-forward deploy/<deploy> 8080:80
kubectl port-forward pod/<pod> 8080:80 9090:9090          # multiple ports
kubectl port-forward pod/<pod> :80                         # random local port
kubectl port-forward --address 0.0.0.0 svc/<svc> 8080:80   # listen on all interfaces

# API proxy
kubectl proxy                                              # http://127.0.0.1:8001
kubectl proxy --port=8080
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods
curl http://127.0.0.1:8001/api/v1/namespaces/<ns>/services/<svc>:<port>/proxy/

# DNS & connectivity tests
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup <svc>.<ns>.svc.cluster.local
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl -sv http://<svc>.<ns>:80
kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash
```

**DNS format:** `<service>.<namespace>.svc.cluster.local` · Pod: `<pod-ip-dashed>.<namespace>.pod.cluster.local`

---

## 16. ConfigMaps & Secrets

```bash
# ConfigMaps
kubectl create configmap cfg --from-literal=A=1 --from-literal=B=2
kubectl create configmap cfg --from-file=app.properties
kubectl create configmap cfg --from-file=custom-key=app.properties
kubectl create configmap cfg --from-file=./config-dir/
kubectl create configmap cfg --from-env-file=.env
kubectl get cm cfg -o yaml
kubectl describe cm cfg

# Secrets
kubectl create secret generic db --from-literal=user=admin --from-literal=pass='p@ss'
kubectl create secret generic db --from-file=./username.txt --from-file=./password.txt
kubectl create secret generic db --from-env-file=.env
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
kubectl create secret docker-registry regcred \
  --docker-server=<registry> --docker-username=<u> --docker-password=<p> --docker-email=<e>
kubectl create secret generic regcred --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson

# Read / decode
kubectl get secret db -o jsonpath='{.data.pass}' | base64 -d
kubectl get secret db -o go-template='{{range $k,$v := .data}}{{$k}}={{$v|base64decode}}{{"\n"}}{{end}}'

# Update a ConfigMap/Secret in place from files
kubectl create configmap cfg --from-file=app.conf --dry-run=client -o yaml | kubectl apply -f -

# Encode / decode helpers
echo -n 'value' | base64
echo 'dmFsdWU=' | base64 -d
```

> ⚠️ Secrets are only **base64-encoded**, not encrypted. Enable encryption at rest and use RBAC / an external secret store.

---

## 17. Storage (PV / PVC / StorageClass)

```bash
kubectl get pv
kubectl get pvc -A
kubectl get sc
kubectl get volumeattachments
kubectl get csidrivers
kubectl describe pvc <pvc>
kubectl describe pv <pv>

# Set default StorageClass
kubectl patch sc <sc> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Expand a PVC (if allowVolumeExpansion: true)
kubectl patch pvc <pvc> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Change reclaim policy
kubectl patch pv <pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Release a "Released" PV so it can be re-bound
kubectl patch pv <pv> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
```

**Access modes:** `RWO` ReadWriteOnce · `ROX` ReadOnlyMany · `RWX` ReadWriteMany · `RWOP` ReadWriteOncePod
**Reclaim policies:** `Retain` · `Delete` · (`Recycle` deprecated)

---

## 18. Jobs & CronJobs

```bash
kubectl create job hello --image=busybox -- echo "hello"
kubectl get jobs
kubectl describe job hello
kubectl logs job/hello
kubectl wait --for=condition=complete job/hello --timeout=60s
kubectl delete job hello

kubectl create cronjob nightly --image=busybox --schedule="0 0 * * *" -- echo run
kubectl get cronjobs
kubectl create job nightly-manual --from=cronjob/nightly      # trigger now
kubectl patch cronjob nightly -p '{"spec":{"suspend":true}}'   # pause
kubectl patch cronjob nightly -p '{"spec":{"suspend":false}}'  # resume
kubectl get jobs --selector=batch.kubernetes.io/job-name=hello # (label varies by version)
```

**Cron format:** `minute hour day-of-month month day-of-week` → `*/15 * * * *` every 15 min

---

## 19. Labels, Annotations & Selectors

```bash
kubectl label pod <pod> env=prod
kubectl label pod <pod> env=staging --overwrite
kubectl label pod <pod> env-                             # remove label
kubectl label pods --all team=platform
kubectl label pods -l app=web tier=frontend
kubectl label node <node> disktype=ssd

kubectl annotate pod <pod> owner="omar"
kubectl annotate pod <pod> owner-                        # remove
kubectl annotate deploy web description="frontend" --overwrite

# Selectors
kubectl get pods -l env=prod                             # equality
kubectl get pods -l env!=prod
kubectl get pods -l 'env in (prod,qa)'                   # set-based
kubectl get pods -l 'env notin (dev)'
kubectl get pods -l 'app'                                # label exists
kubectl get pods -l '!app'                               # label absent
kubectl get pods -l app=web,tier=frontend                # AND
```

---

## 20. Namespaces

```bash
kubectl get ns
kubectl create ns dev
kubectl delete ns dev
kubectl config set-context --current --namespace=dev
kubectl config view --minify -o jsonpath='{..namespace}'   # current ns
kubectl get all -n dev
kubectl get resourcequota -n dev
kubectl get limitrange -n dev
kubectl describe quota -n dev

# Find what's blocking a namespace stuck in "Terminating"
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <ns>
kubectl get ns <ns> -o json | jq '.spec.finalizers'
```

---

## 21. Nodes: Cordon, Drain, Taints

```bash
kubectl get nodes -o wide
kubectl describe node <node>
kubectl get node <node> -o jsonpath='{.status.allocatable}'
kubectl get nodes --show-labels
kubectl get nodes -l node-role.kubernetes.io/control-plane

kubectl cordon <node>                                     # mark unschedulable
kubectl uncordon <node>
kubectl drain <node>                                      # evict pods
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl drain <node> --ignore-daemonsets --force --grace-period=30
kubectl drain <node> --pod-selector='app!=critical'
kubectl drain <node> --timeout=5m --disable-eviction       # bypass PDBs (careful)

# Taints
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key=value:PreferNoSchedule
kubectl taint nodes <node> key=value:NoExecute
kubectl taint nodes <node> key=value:NoSchedule-          # remove
kubectl taint nodes <node> key-                           # remove all effects for key
kubectl taint nodes <node> node-role.kubernetes.io/control-plane:NoSchedule-
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

kubectl delete node <node>
```

---

## 22. RBAC, ServiceAccounts & Auth

```bash
# Roles (namespaced)
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create role cm-editor --verb=* --resource=configmaps --resource-name=app-cfg
kubectl create rolebinding read-pods --role=pod-reader --user=alice -n dev
kubectl create rolebinding sa-read --role=pod-reader --serviceaccount=dev:app-sa -n dev
kubectl create rolebinding grp-read --role=pod-reader --group=devs -n dev

# ClusterRoles (cluster-wide)
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes
kubectl create clusterrole aggregated --aggregation-rule="rbac.example.com/aggregate=true"
kubectl create clusterrolebinding node-read --clusterrole=node-reader --user=alice
kubectl create rolebinding view-dev --clusterrole=view --user=bob -n dev   # reuse built-in

kubectl get roles,rolebindings -n dev
kubectl get clusterroles,clusterrolebindings
kubectl describe clusterrole admin

# Permission checks
kubectl auth can-i create deployments
kubectl auth can-i delete pods -n prod
kubectl auth can-i '*' '*'                                # am I cluster-admin?
kubectl auth can-i --list -n dev
kubectl auth can-i get pods --as=alice
kubectl auth can-i get pods --as=system:serviceaccount:dev:app-sa
kubectl auth can-i get pods --as=bob --as-group=devs
kubectl auth whoami                                        # who am I authenticated as
kubectl auth reconcile -f rbac.yaml

# Impersonation on any command
kubectl get pods --as=alice -n dev

# ServiceAccounts
kubectl create sa app-sa
kubectl get sa
kubectl create token app-sa
kubectl create token app-sa --duration=24h --audience=api

# Certificate signing requests (user certs)
kubectl get csr
kubectl certificate approve <csr>
kubectl certificate deny <csr>
kubectl get csr <csr> -o jsonpath='{.status.certificate}' | base64 -d > user.crt
```

**Built-in ClusterRoles:** `cluster-admin` · `admin` · `edit` · `view`

---

## 23. Resource Usage & Events

```bash
# Requires metrics-server
kubectl top nodes
kubectl top nodes --sort-by=cpu
kubectl top pods
kubectl top pods -A --sort-by=memory
kubectl top pod <pod> --containers
kubectl top pods -l app=web

# Events
kubectl get events
kubectl get events -A --sort-by=.lastTimestamp
kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp
kubectl get events --field-selector type=Warning
kubectl get events --field-selector involvedObject.name=<pod>
kubectl get events -w
kubectl events                                             # newer command
kubectl events --for pod/<pod>
kubectl events --types=Warning -A
kubectl events --watch
```

---

## 24. Debugging & Troubleshooting

```bash
# Ephemeral debug container in a running pod
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl debug -it <pod> --image=nicolaka/netshoot --profile=netadmin

# Copy a pod and debug the copy
kubectl debug <pod> -it --copy-to=<pod>-dbg --container=<container> -- sh
kubectl debug <pod> --copy-to=<pod>-dbg --set-image=*=busybox
kubectl debug <pod> -it --copy-to=<pod>-dbg --share-processes --image=busybox

# Debug a node (pod with host namespaces, host fs at /host)
kubectl debug node/<node> -it --image=ubuntu
#   then: chroot /host

# Quick health triage
kubectl get pods -A | grep -vE 'Running|Completed'
kubectl get pods -A --field-selector=status.phase!=Running
kubectl describe pod <pod> | sed -n '/Events/,$p'
kubectl logs <pod> --previous
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'

# Verbose API calls (great for learning / debugging)
kubectl get pods -v=6          # show HTTP requests
kubectl get pods -v=8          # include request/response bodies
kubectl get pods -v=9          # maximum verbosity, curl equivalents

# Raw API access
kubectl get --raw /api/v1/namespaces/default/pods
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl create --raw /api/v1/namespaces -f ns.json
kubectl delete --raw /api/v1/namespaces/<ns>/pods/<pod>

# Control-plane pods (kubeadm clusters)
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## 25. Waiting & Scripting

```bash
kubectl wait --for=condition=Ready pod/<pod> --timeout=60s
kubectl wait --for=condition=Ready pods -l app=web --timeout=120s
kubectl wait --for=condition=Available deploy/web
kubectl wait --for=condition=complete job/<job> --timeout=300s
kubectl wait --for=delete pod/<pod> --timeout=60s
kubectl wait --for=jsonpath='{.status.phase}'=Running pod/<pod>
kubectl wait --for=jsonpath='{.status.loadBalancer.ingress}' svc/<svc>   # create-time checks

# Loop over resources
for p in $(kubectl get pods -l app=web -o name); do kubectl logs "$p" --tail=5; done
kubectl get pods -o name | xargs -I{} kubectl describe {}

# Restart every deployment in a namespace
kubectl get deploy -n <ns> -o name | xargs -n1 kubectl rollout restart -n <ns>

# List all images running in the cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u

# Pods per node
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
```

---

## 26. Kustomize

```bash
kubectl kustomize ./overlay                    # render
kubectl apply -k ./overlay
kubectl diff -k ./overlay
kubectl delete -k ./overlay

# Standalone kustomize CLI
kustomize build ./overlay
kustomize create --autodetect
kustomize edit set image nginx=nginx:1.27
kustomize edit add resource deploy.yaml
kustomize edit set namespace dev
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
resources:
  - ../../base
images:
  - name: nginx
    newTag: "1.27"
patches:
  - path: replica-patch.yaml
configMapGenerator:
  - name: app-cfg
    literals: [ENV=dev]
```

---

## 27. Plugins & Krew

```bash
kubectl plugin list
# Any executable named kubectl-<name> on $PATH becomes `kubectl <name>`

# Krew (plugin manager)
kubectl krew update
kubectl krew search
kubectl krew install ctx ns neat tree stern view-secret
kubectl krew list
kubectl krew upgrade
kubectl krew uninstall <plugin>

# Popular plugins
kubectl ctx <context>          # switch context
kubectl ns <namespace>         # switch namespace
kubectl neat get pod <pod> -o yaml   # strip clutter from YAML
kubectl tree deploy <name>     # ownership tree
kubectl stern <pattern>        # multi-pod log tail
kubectl view-secret <secret>   # decode secrets
```

**Related tools:** `k9s` (TUI) · `kubectx`/`kubens` · `stern` · `kubeconform` (manifest validation) · `kube-linter`

---

## 28. kubeadm (Cluster Lifecycle)

```bash
# Initialize control plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
sudo kubeadm init --control-plane-endpoint=<lb>:6443 --upload-certs
sudo kubeadm init --config=kubeadm-config.yaml
sudo kubeadm config print init-defaults > kubeadm-config.yaml
sudo kubeadm config images list
sudo kubeadm config images pull

# Configure kubectl after init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Join nodes
kubeadm token create --print-join-command
kubeadm token list
kubeadm token delete <token>
sudo kubeadm join <ip>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<hash>
sudo kubeadm join <ip>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
sudo kubeadm init phase upload-certs --upload-certs        # new certificate key

# Upgrades (control plane)
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.X.Y
# Upgrades (other nodes)
sudo kubeadm upgrade node

# Certificates
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
sudo kubeadm certs renew apiserver

# Tear down
sudo kubeadm reset
sudo kubeadm reset --force
```

**Typical node upgrade flow:**
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# upgrade kubeadm package → kubeadm upgrade (apply|node) → upgrade kubelet & kubectl packages
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon <node>
```

**Important paths:**
| Path | Purpose |
|---|---|
| `/etc/kubernetes/manifests/` | Static pod manifests (apiserver, etcd, scheduler, controller-manager) |
| `/etc/kubernetes/pki/` | Cluster certificates |
| `/etc/kubernetes/admin.conf` | Admin kubeconfig |
| `/var/lib/kubelet/config.yaml` | Kubelet config |
| `/var/lib/etcd/` | etcd data dir |

---

## 29. etcd Backup & Restore

```bash
export ETCDCTL_API=3
CERTS="--cacert=/etc/kubernetes/pki/etcd/ca.crt \
       --cert=/etc/kubernetes/pki/etcd/server.crt \
       --key=/etc/kubernetes/pki/etcd/server.key \
       --endpoints=https://127.0.0.1:2379"

# Health / members
etcdctl $CERTS endpoint health
etcdctl $CERTS endpoint status -w table
etcdctl $CERTS member list -w table

# Backup
etcdctl $CERTS snapshot save /backup/etcd-$(date +%F).db

# Verify snapshot (newer releases use etcdutl)
etcdutl snapshot status /backup/etcd.db -w table

# Restore to a new data dir, then point the etcd static pod at it
etcdutl snapshot restore /backup/etcd.db --data-dir=/var/lib/etcd-restored
#   edit /etc/kubernetes/manifests/etcd.yaml → hostPath for etcd-data = /var/lib/etcd-restored

# Inspect keys
etcdctl $CERTS get /registry/pods --prefix --keys-only | head
```

---

## 30. Container Runtime (crictl)

```bash
sudo crictl info
sudo crictl ps                     # running containers
sudo crictl ps -a                  # including exited
sudo crictl pods
sudo crictl pods --name <pod>
sudo crictl images
sudo crictl pull nginx:1.27
sudo crictl rmi <image>
sudo crictl rmi --prune            # remove unused images
sudo crictl logs <container-id>
sudo crictl logs -f --tail=50 <container-id>
sudo crictl exec -it <container-id> sh
sudo crictl inspect <container-id>
sudo crictl inspectp <pod-id>
sudo crictl stats
sudo crictl stop <container-id>
sudo crictl rm <container-id>
sudo crictl stopp <pod-id> && sudo crictl rmp <pod-id>

# containerd native CLI
sudo ctr -n k8s.io containers list
sudo ctr -n k8s.io images list
sudo nerdctl -n k8s.io ps           # if nerdctl is installed
```

---

## 31. Node-level Troubleshooting

```bash
# kubelet
systemctl status kubelet
sudo systemctl restart kubelet
journalctl -u kubelet -f
journalctl -u kubelet --since "10 min ago" --no-pager

# Container runtime
systemctl status containerd
journalctl -u containerd -f

# Static pods & control-plane logs
ls /etc/kubernetes/manifests/
ls /var/log/pods/
ls /var/log/containers/

# Networking basics
ip a; ip route
sudo iptables -t nat -L KUBE-SERVICES | head
sudo ipvsadm -Ln                    # if kube-proxy uses IPVS
cat /etc/cni/net.d/*
ls /opt/cni/bin/

# Resources
df -h; free -m; top
```

---

## 32. Helm Essentials

```bash
helm version
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo nginx
helm search hub prometheus
helm show values bitnami/nginx > values.yaml
helm show chart bitnami/nginx

helm install web bitnami/nginx -n web --create-namespace
helm install web bitnami/nginx -f values.yaml --set replicaCount=3
helm install web ./mychart --dry-run --debug
helm upgrade web bitnami/nginx -f values.yaml
helm upgrade --install web bitnami/nginx -n web --create-namespace --atomic --wait
helm list -A
helm status web -n web
helm history web -n web
helm rollback web 1 -n web
helm get values web -n web
helm get manifest web -n web
helm uninstall web -n web

helm template web ./mychart -f values.yaml > rendered.yaml
helm lint ./mychart
helm create mychart
helm package ./mychart
helm dependency update ./mychart
helm pull bitnami/nginx --untar
helm install web oci://registry.example.com/charts/web --version 1.2.3
```

---

## 33. Global Flags Reference

| Flag | Purpose |
|---|---|
| `-n, --namespace` | Target namespace |
| `-A, --all-namespaces` | All namespaces |
| `-o, --output` | Output format |
| `-l, --selector` | Label selector |
| `--field-selector` | Field selector |
| `-f, --filename` | File, dir or URL |
| `-k, --kustomize` | Kustomize dir |
| `-R, --recursive` | Recurse into dirs with `-f` |
| `-w, --watch` | Watch for changes |
| `--context` | Use a kubeconfig context |
| `--kubeconfig` | Use a kubeconfig file |
| `--as`, `--as-group` | Impersonate user/group |
| `--dry-run=client\|server` | Simulate |
| `--server-side` | Server-side apply |
| `--request-timeout` | e.g. `30s` |
| `-v=<0-9>` | Log verbosity |
| `--sort-by` | Sort by JSONPath |
| `--show-labels` | Show labels column |
| `--no-headers` | Hide header row |
| `--grace-period`, `--force` | Deletion behaviour |
| `--insecure-skip-tls-verify` | Skip TLS checks (avoid in prod) |

```bash
kubectl options            # list all global flags
kubectl <command> --help   # per-command help & examples
```

---

## 34. Pod Phases & Common Errors

| Status | Usual cause | First command |
|---|---|---|
| `Pending` | No node fits (resources, taints, affinity), unbound PVC | `kubectl describe pod` → Events |
| `ContainerCreating` | Image pull, volume mount, CNI issue | `kubectl describe pod` |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image/tag, private registry without `imagePullSecrets` | `kubectl describe pod` |
| `CrashLoopBackOff` | App exits on start, bad command/config, failing probes | `kubectl logs --previous` |
| `OOMKilled` | Memory limit exceeded | `kubectl describe pod` → Last State |
| `CreateContainerConfigError` | Missing ConfigMap/Secret/key | `kubectl describe pod` |
| `Evicted` | Node pressure (disk/memory) | `kubectl describe node` |
| `Terminating` (stuck) | Finalizers, unreachable node | `kubectl delete --force --grace-period=0` |
| `Init:Error` / `Init:CrashLoopBackOff` | Init container failing | `kubectl logs <pod> -c <init>` |
| Node `NotReady` | kubelet / runtime / CNI down | `journalctl -u kubelet` |

---

## 35. YAML Skeletons

<details>
<summary><b>Pod</b></summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels: { app: web }
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports: [{ containerPort: 80 }]
      resources:
        requests: { cpu: 100m, memory: 128Mi }
        limits:   { cpu: 500m, memory: 256Mi }
      readinessProbe:
        httpGet: { path: /, port: 80 }
        initialDelaySeconds: 5
      livenessProbe:
        httpGet: { path: /, port: 80 }
        periodSeconds: 10
```
</details>

<details>
<summary><b>Deployment + Service</b></summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          envFrom:
            - configMapRef: { name: app-cfg }
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
```
</details>

<details>
<summary><b>Ingress</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  tls:
    - hosts: [example.com]
      secretName: web-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```
</details>

<details>
<summary><b>PersistentVolumeClaim</b></summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standard
  resources:
    requests: { storage: 10Gi }
```
</details>

<details>
<summary><b>NetworkPolicy (default deny + allow from app)</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
spec:
  podSelector:
    matchLabels: { app: api }
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: frontend }
      ports:
        - { protocol: TCP, port: 8080 }
```
</details>

<details>
<summary><b>CronJob</b></summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox
              command: ["sh", "-c", "date; echo backing up"]
```
</details>

<details>
<summary><b>Role + RoleBinding</b></summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
</details>

---

## 📚 References

- [kubectl reference](https://kubernetes.io/docs/reference/kubectl/)
- [Official kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [JSONPath support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
- [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Helm docs](https://helm.sh/docs/)

---

⭐ If this helped you, star the repo! Contributions and PRs are welcome.
