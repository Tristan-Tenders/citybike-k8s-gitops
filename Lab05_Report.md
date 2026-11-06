# Lab 05 report

Student: Tristan  
Repo: https://github.com/Tristan-Tenders/citybike-k8s-gitops  
Cluster: kind `citybike-gitops` (Kubernetes v1.37.0) with Argo CD from the stable manifests. Solo lab, so I made each change on its own branch and checked the diff myself before merging into `main`.

## Mission 0: Safe setup

```
$ kubectl config current-context
kind-citybike-gitops

$ kubectl -n argocd rollout status deployment/argocd-repo-server --timeout=300s
deployment "argocd-repo-server" successfully rolled out

$ kubectl -n argocd get pods
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          43s
argocd-applicationset-controller-794bb95c9d-rt42j   1/1     Running   0          43s
argocd-dex-server-66cf64bfcf-qqdzz                  1/1     Running   0          43s
argocd-notifications-controller-5878c8d644-9vfq7    1/1     Running   0          43s
argocd-redis-5869b7547-nln4f                        1/1     Running   0          43s
argocd-repo-server-7d55cc466b-7xsz9                 1/1     Running   0          43s
argocd-server-85b849c688-rsqc8                      0/1     Running   0          43s
```

## Mission 1: Desired state

```
$ kubectl kustomize gitops | grep -n citybike-script
39:  name: citybike-script-6dgthf428d
122:          name: citybike-script-6dgthf428d

$ kubectl -n argocd get applications.argoproj.io citybike
NAME       SYNC STATUS   HEALTH STATUS
citybike   Synced        Healthy

$ kubectl -n citybike get deploy,pods,svc
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/citybike   2/2     2            2           18s

NAME                            READY   STATUS    RESTARTS   AGE
pod/citybike-7f9949cc86-7jb2l   1/1     Running   0          18s
pod/citybike-7f9949cc86-wsm85   1/1     Running   0          18s

NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/citybike   ClusterIP   10.96.184.23   <none>        80/TCP    18s

$ bash scripts/smoke-service.sh
{"stations":[{"id":"central","name":"Central Square","bikes":9},{"id":"riverside","name":"Riverside Park","bikes":0},{"id":"campus","name":"University Campus","bikes":6}],"release":"v1"}
PASS: Service is reachable on port 18080
```

`bootstrap/application.yaml` points at my repo, path `gitops`, with `prune` and `selfHeal` on. It's gitignored (`git check-ignore` confirms it).

**Kustomize and the ConfigMap hash:** `configMapGenerator` builds a ConfigMap from `app/server.js` and adds a hash of its content to the name (`citybike-script-6dgthf428d`). It also rewrites the Deployment's volume to use that name. If `server.js` changes, the name changes, so the pod template changes and Kubernetes rolls out new pods. A ConfigMap edited in place wouldn't restart anything.

**A) Why are the manifests the source of truth?** Argo keeps comparing the cluster with what `gitops/` renders on `main` and changes the cluster to match. Git is the reviewed and versioned copy. Anything changed only in the cluster gets undone.

**B) What does Synced mean?** The live resources match the manifests from the Git revision Argo is tracking. It says nothing about whether requests actually work.

**C) Can Synced coexist with a broken user experience?** Yes. If Git declares something wrong, Argo applies it perfectly and still says Synced. Mission 3 shows exactly that.

## Mission 2: Drift

```
$ kubectl -n citybike scale deployment/citybike --replicas=0
deployment.apps/citybike scaled
```

| Time | Generation | Desired | Ready | Argo |
|---|---|---|---|---|
| 15:12:25 | 1 | 2 | 2 | Synced / Healthy |
| 15:12:27 | 3 | 2 | 0 | Synced / Progressing |
| 15:12:33 | 3 | 2 | 2 | Synced / Healthy |

```
$ kubectl -n argocd get applications.argoproj.io citybike -o jsonpath=...
last operation initiatedBy: {"automated":true}
phase: Succeeded
message: successfully synced (all tasks run)
finished: 2026-11-06T14:12:27Z

$ kubectl -n citybike get events --sort-by=.lastTimestamp
Normal   ScalingReplicaSet   deployment/citybike   Scaled down replica set citybike-7f9949cc86 from 2 to 0
Normal   ScalingReplicaSet   deployment/citybike   Scaled up replica set citybike-7f9949cc86 from 0 to 2
```

Argo put `replicas: 2` back in the same second I scaled it. That's why the generation jumped from 1 to 3 and I never caught `spec.replicas=0` in my 1-second samples. The pods were gone for about 6 seconds until the new ones passed readiness. I didn't touch anything, and Git didn't change.

**A) Why doesn't `kubectl scale` persist?** It only changes the live object, while Git still says 2. Argo watches the live resources, sees they no longer match, and with `selfHeal` on it syncs straight back to Git.

**B) What if selfHeal were disabled?** Automated sync only reacts to new commits. The app would show OutOfSync and stay at 0 replicas (a full outage) until someone clicked Sync or pushed something.

## Mission 3: Bad targetPort

```
$ git diff main...feature/broken-service
-      targetPort: 3000
+      targetPort: 3999

$ kubectl -n argocd get applications.argoproj.io citybike
NAME       SYNC STATUS   HEALTH STATUS
citybike   Synced        Healthy

$ kubectl -n citybike get pods,svc,endpointslice
NAME                            READY   STATUS    RESTARTS   AGE
pod/citybike-7f9949cc86-c9m9m   1/1     Running   0          17s
pod/citybike-7f9949cc86-p5k5d   1/1     Running   0          17s

NAME                                            ADDRESSTYPE   PORTS   ENDPOINTS
endpointslice.discovery.k8s.io/citybike-bk6qw   IPv4          3999    10.244.0.14,10.244.0.15

$ kubectl -n citybike describe svc citybike
TargetPort:               3999/TCP
Endpoints:                10.244.0.14:3999,10.244.0.15:3999

$ bash scripts/smoke-service.sh
FAIL: Service is unreachable. Port-forward details:
Forwarding from 127.0.0.1:18080 -> 3999
... failed to connect to localhost:3999 ... dial tcp4 127.0.0.1:3999: connect: connection refused
```

From inside the cluster it was the same story: through the Service I got a connection refused, but straight to a pod on 3000 it worked.

```
$ wget http://citybike.citybike.svc/api/stations
wget: can't connect to remote host (10.96.184.23): Connection refused
$ wget http://10.244.0.15:3000/api/stations
{"stations":[...],"release":"v1"}
```

**Root cause:** the Service sends traffic to port 3999, but the container listens on 3000 (`containerPort: 3000`). Nothing is listening on 3999.

**A) Why do the pods stay Ready?** Kubelet runs the probes straight against the pod on port `http` (3000) and never goes through the Service, so they keep passing. Because the pods are Ready, they stay in the EndpointSlice (`ready=true`) even though the port is wrong.

**B) Does Synced guarantee clients are served?** No. Synced only means the cluster matches Git, and Healthy only looks at the pods. Neither one sends a real request through the Service. Only the smoke test caught this.

Fix: a revert commit on `main`, no kubectl.

```
$ kubectl -n citybike get endpointslice
NAME             ADDRESSTYPE   PORTS   ENDPOINTS
citybike-bk6qw   IPv4          3000    10.244.0.14,10.244.0.15

$ bash scripts/smoke-service.sh
PASS: Service is reachable on port 18080
```

## Mission 4: Restricted security

Turned on `warn` and `audit` first. A server-side dry run of `enforce` showed what would have been blocked:

```
$ kubectl label namespace citybike pod-security.kubernetes.io/enforce=restricted --overwrite --dry-run=server
Warning: existing pods in namespace "citybike" violate the new PodSecurity enforce level "restricted:latest"
Warning: citybike-7f9949cc86-c9m9m (and 1 other pod): allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
```

Hardening diff (`feature/restricted-hardening`):

```diff
+      automountServiceAccountToken: false
+      securityContext:
+        runAsNonRoot: true
+        runAsUser: 1000
+        runAsGroup: 1000
+        seccompProfile:
+          type: RuntimeDefault
       containers:
         - name: citybike
...
+          securityContext:
+            allowPrivilegeEscalation: false
+            readOnlyRootFilesystem: true
+            runAsNonRoot: true
+            capabilities:
+              drop: ["ALL"]
```

Before merging, I ran a server dry run of the rendered manifests and got no PodSecurity warning. After the rollout finished I turned on enforce. The warning that still showed up was about the two old v1 pods that were still terminating. Then I bumped `APP_RELEASE` to v2 in Git so brand-new pods had to pass admission:

```
$ kubectl get namespace citybike --show-labels
... pod-security.kubernetes.io/audit=restricted,pod-security.kubernetes.io/enforce=restricted,pod-security.kubernetes.io/warn=restricted

$ kubectl -n citybike rollout status deployment/citybike --timeout=180s
deployment "citybike" successfully rolled out

$ kubectl -n citybike get pods
NAME                        READY   STATUS    RESTARTS   AGE
citybike-7df5d979fb-gn6c9   1/1     Running   0          6s
citybike-7df5d979fb-ztdpg   1/1     Running   0          12s

$ kubectl -n citybike get events --field-selector reason=FailedCreate
No resources found in citybike namespace.

$ bash scripts/smoke-service.sh
{"stations":[...],"release":"v2"}
PASS: Service is reachable on port 18080
```

To make sure enforcement was actually on and not bypassed, I tried an unhardened pod:

```
$ kubectl -n citybike run psa-negative-test --image=node:22-alpine --restart=Never --dry-run=server -- node -e 1
Error from server (Forbidden): pods "psa-negative-test" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false ...
```

| Control | Risk it covers |
|---|---|
| `runAsNonRoot` + `runAsUser: 1000` | If someone takes over the app, they aren't root. `node:22-alpine` runs as root by default, so `runAsNonRoot` alone would make kubelet refuse to start it. 1000 is the image's `node` user. |
| `seccompProfile: RuntimeDefault` | Blocks dangerous syscalls, so there's less kernel to attack |
| `allowPrivilegeEscalation: false` | setuid binaries can't be used to get root |
| `capabilities.drop: ["ALL"]` | No extra kernel powers. Port 3000 is above 1024, so the app doesn't need any. |
| `readOnlyRootFilesystem: true` | Nobody can drop tools or change files in the container. The app only reads its ConfigMap, so no writable volume is needed. |
| `automountServiceAccountToken: false` | Not needed for Restricted, but the app never calls the API, so there's no token to steal |

**Why this order:** warn and audit don't block anything, so I could see the problems first. Enforce only checks new pods, so turning it on after the hardened rollout meant I never blocked my own pods. `maxUnavailable: 0` kept 2 pods serving the whole time.

## Mission 5: Readiness vs liveness

```
$ kubectl -n citybike rollout status deployment/citybike --timeout=75s
Waiting for deployment "citybike" rollout to finish: 1 out of 2 new replicas have been updated...
error: timed out waiting for the condition

$ kubectl -n citybike get pods
NAME                        READY   STATUS    RESTARTS   AGE
citybike-6985bc7958-6zxbb   0/1     Running   0          77s
citybike-7df5d979fb-gn6c9   1/1     Running   0          89s
citybike-7df5d979fb-ztdpg   1/1     Running   0          95s

$ kubectl -n citybike describe deployment citybike
Replicas:               2 desired | 1 updated | 3 total | 2 available | 1 unavailable
RollingUpdateStrategy:  0 max unavailable, 1 max surge
      READY_MODE:   fail

$ kubectl -n citybike describe pod citybike-6985bc7958-6zxbb
Warning  Unhealthy  1s (x16 over 76s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503
```

Same unready pod, both endpoints:

```
/health/live   {"status":"live","release":"v2"}  HTTP 200
/health/ready  {"status":"unready","reason":"simulated_dependency_failure","release":"v2"}  HTTP 503
restartCount=0
```

What actually happened: the new pod started but never became Ready. Because of `maxUnavailable: 0`, Kubernetes didn't remove either old pod, so the rollout just got stuck. The smoke test still passed (the Service only sends traffic to Ready pods), and Argo showed Synced / Progressing. Liveness kept returning 200, so the pod was never restarted.

**Why keep them separate?** Readiness answers "can I take traffic right now?" and liveness answers "is the process stuck?". A dependency failure should take the pod out of the Service and stop the rollout, which is what happened here. If that check were on liveness, kubelet would restart every pod in a loop and a dependency problem would turn into a full outage.

Fix: a commit putting `READY_MODE` back to `ok`. The template now matched the last good one exactly, so Kubernetes reused ReplicaSet `7df5d979fb` and just removed the bad pod:

```
$ kubectl -n citybike rollout status deployment/citybike --timeout=180s
deployment "citybike" successfully rolled out

$ kubectl -n argocd get applications.argoproj.io citybike
NAME       SYNC STATUS   HEALTH STATUS
citybike   Synced        Healthy

$ bash scripts/smoke-service.sh
PASS: Service is reachable on port 18080
```

## Mission 6: Wrap-up

Final state on `main`: `targetPort: 3000`, hardened Deployment, `APP_RELEASE: v2`, `READY_MODE: ok`.

```
$ kubectl -n argocd get applications.argoproj.io citybike
NAME       SYNC STATUS   HEALTH STATUS
citybike   Synced        Healthy

$ kubectl get namespace citybike --show-labels
... pod-security.kubernetes.io/audit=restricted,pod-security.kubernetes.io/enforce=restricted,pod-security.kubernetes.io/warn=restricted

$ git grep -nIE 'ghp_|github_pat_|PRIVATE KEY|password\s*[:=]'
no secrets found
```

`bootstrap/application.yaml` is gitignored, so my repo URL setup stays local.

**Why isn't `kubectl apply` in production an acceptable fix here?** Argo would undo it within seconds, because Git still has the old version. Even with self-heal off, Git and the cluster would disagree, and the next sync or rebuild would bring the bug back. It also skips the review and leaves no record of who changed what or why.

**Why can an app be Synced while the service is failing?** Synced compares config, not behaviour. In Mission 3 Git had a wrong but perfectly valid port, Argo applied it exactly, and the pods were healthy, so everything showed green while every request failed. You only find out with a real request, like the smoke test.
