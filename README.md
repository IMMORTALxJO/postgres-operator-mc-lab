# Postgres Operator Multi-Cluster Lab

This lab creates two local `kind` clusters and checks whether a single Patroni cluster can be built across two Kubernetes clusters.

What it creates:

- `pgmesh-c1` and `pgmesh-c2` on `kind`
- Cilium CNI in each cluster
- Cilium Cluster Mesh between the clusters
- one shared `etcd` in `pgmesh-c1/test`, reachable from `pgmesh-c2` through a Cilium global service
- Postgres Operator `v1.15.1` in each cluster
- Spilo/Postgres `ghcr.io/zalando/spilo-17:4.0-p3`
- test Patroni scope `test`

Important: the current upstream Zalando Postgres Operator hard-codes `SCOPE=<metadata.name>` for the Spilo pod. Because of that, fully identical CRs with `metadata.name: test` in two Kubernetes clusters conflict on the Patroni member name `test-0`. The working lab workaround is described below: Kubernetes resource names are different (`test-c1`, `test-c2`), while `SCOPE` is manually patched in the StatefulSet to the shared value `test`.

## Prerequisites

You need these tools locally:

```bash
docker version
kind version
kubectl version --client
helm version
cilium version --client
```


## 1. Create kind clusters

```bash
kind create cluster --config kind-c1.yaml
kind create cluster --config kind-c2.yaml

kubectl --context kind-pgmesh-c1 get nodes
kubectl --context kind-pgmesh-c2 get nodes
```

## 2. Install Cilium

```bash
cilium install \
  --context kind-pgmesh-c1 \
  --version 1.19.5 \
  --set cluster.name=pgmesh-c1 \
  --set cluster.id=1 \
  --set ipam.mode=kubernetes \
  --set operator.replicas=1

cilium install \
  --context kind-pgmesh-c2 \
  --version 1.19.5 \
  --set cluster.name=pgmesh-c2 \
  --set cluster.id=2 \
  --set ipam.mode=kubernetes \
  --set operator.replicas=1

cilium status --context kind-pgmesh-c1 --wait
cilium status --context kind-pgmesh-c2 --wait
```

## 3. Enable Cilium Cluster Mesh

```bash
cilium clustermesh enable --context kind-pgmesh-c1 --service-type NodePort
cilium clustermesh enable --context kind-pgmesh-c2 --service-type NodePort
```

Connect the clusters:

```bash
cilium clustermesh connect \
  --context kind-pgmesh-c1 \
  --destination-context kind-pgmesh-c2 \
  --connection-mode bidirectional

cilium clustermesh status --context kind-pgmesh-c1 --wait
cilium clustermesh status --context kind-pgmesh-c2 --wait
```

## 4. Start the shared etcd

Patroni in Spilo `4.0-p3`, when configured through `ETCD_HOST`, uses the etcd v2 API. Therefore the lab uses `registry.k8s.io/etcd:3.5.21-0` with `--enable-v2=true`.

```bash
kubectl --context kind-pgmesh-c1 apply -f shared-etcd-c1.yaml
kubectl --context kind-pgmesh-c2 apply -f shared-etcd-c2-service.yaml

kubectl --context kind-pgmesh-c1 -n test rollout status deploy/patroni-etcd --timeout=180s
```

Check the v2 API from both clusters:

```bash
kubectl --context kind-pgmesh-c1 -n test exec deploy/patroni-etcd -- \
  sh -c 'ETCDCTL_API=2 /usr/local/bin/etcdctl --endpoints=http://127.0.0.1:2379 ls /'

kubectl --context kind-pgmesh-c1 -n test run etcd-client --rm -i --restart=Never \
  --image=registry.k8s.io/etcd:3.5.21-0 \
  --command -- /usr/local/bin/etcdctl \
  --endpoints=http://patroni-etcd.test.svc.cluster.local:2379 endpoint health

kubectl --context kind-pgmesh-c2 -n test run etcd-client --rm -i --restart=Never \
  --image=registry.k8s.io/etcd:3.5.21-0 \
  --command -- /usr/local/bin/etcdctl \
  --endpoints=http://patroni-etcd.test.svc.cluster.local:2379 endpoint health
```

## 5. Install Postgres Operator

```bash
helm repo add postgres-operator https://opensource.zalando.com/postgres-operator/charts/postgres-operator


helm --kube-context kind-pgmesh-c1 upgrade --install postgres-operator --wait \
  postgres-operator/postgres-operator --version 1.15.1 \
  --namespace postgres-operator \
  --create-namespace \
  -f postgres-operator-shared-etcd-values.yaml

helm --kube-context kind-pgmesh-c2 upgrade --install postgres-operator --wait \
  postgres-operator/postgres-operator --version 1.15.1 \
  --namespace postgres-operator \
  --create-namespace \
  -f postgres-operator-shared-etcd-values.yaml

kubectl --context kind-pgmesh-c1 -n postgres-operator rollout status deploy/postgres-operator
--timeout=180s
kubectl --context kind-pgmesh-c2 -n postgres-operator rollout status deploy/postgres-operator --timeout=180s
```

Check that both operators point to the shared etcd and the `test` namespace:

```bash
kubectl --context kind-pgmesh-c1 -n postgres-operator get operatorconfiguration postgres-operator \
  -o jsonpath='{.configuration.etcd_host}{"\n"}{.configuration.kubernetes.watched_namespace}{"\n"}'

kubectl --context kind-pgmesh-c2 -n postgres-operator get operatorconfiguration postgres-operator \
  -o jsonpath='{.configuration.etcd_host}{"\n"}{.configuration.kubernetes.watched_namespace}{"\n"}'
```

Expected:

```text
patroni-etcd.test.svc.cluster.local:2379
test
```

## 7. Create test Postgres CRs

```bash
kubectl --context kind-pgmesh-c1 apply -f shared-postgres-secrets.yaml
kubectl --context kind-pgmesh-c2 apply -f shared-postgres-secrets.yaml

kubectl --context kind-pgmesh-c1 apply -f postgres-test-site-c1.yaml
kubectl --context kind-pgmesh-c2 apply -f postgres-test-site-c2.yaml
```

At this stage the operator creates two independent Patroni scopes:

- `test-c1`
- `test-c2`

This is expected: `spec.env: SCOPE=test` in the CR does not override the hard-coded `SCOPE` that the operator sets from `metadata.name`.

## 8. Manually build one Patroni scope

This step is needed only to verify the hypothesis of "one Patroni cluster across two Kubernetes clusters". It manually works around the limitation of the current operator.

Stop the operators so they do not roll back the manual patch:

```bash
kubectl --context kind-pgmesh-c1 -n postgres-operator scale deploy/postgres-operator --replicas=0
kubectl --context kind-pgmesh-c2 -n postgres-operator scale deploy/postgres-operator --replicas=0
```

Stop the StatefulSets, clean up the PVCs from the second/previous bootstrap, and clean up DCS:

```bash
kubectl --context kind-pgmesh-c1 -n test scale sts/test-c1 --replicas=0
kubectl --context kind-pgmesh-c2 -n test scale sts/test-c2 --replicas=0

kubectl --context kind-pgmesh-c1 -n test wait --for=delete pod/test-c1-0 --timeout=120s || true
kubectl --context kind-pgmesh-c2 -n test wait --for=delete pod/test-c2-0 --timeout=120s || true

kubectl --context kind-pgmesh-c1 -n test delete pvc pgdata-test-c1-0 --ignore-not-found
kubectl --context kind-pgmesh-c2 -n test delete pvc pgdata-test-c2-0 --ignore-not-found

kubectl --context kind-pgmesh-c1 -n test exec deploy/patroni-etcd -- sh -c '
for key in /service/test /service/test-c1 /service/test-c2; do
  ETCDCTL_API=2 /usr/local/bin/etcdctl --endpoints=http://127.0.0.1:2379 rm --recursive "$key" 2>/dev/null || true
done
'
```

Patch the shared `SCOPE=test` into both StatefulSets:

```bash
kubectl --context kind-pgmesh-c1 -n test patch sts test-c1 --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/env/0/value","value":"test"}]'

kubectl --context kind-pgmesh-c2 -n test patch sts test-c2 --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/env/0/value","value":"test"}]'
```

Start them again:

```bash
kubectl --context kind-pgmesh-c1 -n test scale sts/test-c1 --replicas=1
kubectl --context kind-pgmesh-c2 -n test scale sts/test-c2 --replicas=1

kubectl --context kind-pgmesh-c1 -n test wait --for=condition=Ready pod/test-c1-0 --timeout=240s
kubectl --context kind-pgmesh-c2 -n test wait --for=condition=Ready pod/test-c2-0 --timeout=240s
```

Check Patroni:

```bash
kubectl --context kind-pgmesh-c1 -n test exec test-c1-0 -- \
  sh -c 'curl -fsS localhost:8008/cluster; echo'

kubectl --context kind-pgmesh-c2 -n test exec test-c2-0 -- \
  sh -c 'curl -fsS localhost:8008/cluster; echo'
```

Expected result:

```text
scope: test
test-c1-0: leader or replica
test-c2-0: leader or replica
replica state: streaming
lag: 0
```

Check etcd DCS:

```bash
kubectl --context kind-pgmesh-c1 -n test exec deploy/patroni-etcd -- sh -c '
echo leader=$(ETCDCTL_API=2 /usr/local/bin/etcdctl --endpoints=http://127.0.0.1:2379 get /service/test/leader)
ETCDCTL_API=2 /usr/local/bin/etcdctl --endpoints=http://127.0.0.1:2379 ls --recursive /service/test/members
'
```

## 9. Check switchover between sites

First, check the current leader/candidate:

```bash
kubectl --context kind-pgmesh-c1 -n test exec test-c1-0 -- \
  sh -c 'curl -fsS localhost:8008/cluster; echo'
```

Example switchover from `test-c1-0` to `test-c2-0`:

```bash
kubectl --context kind-pgmesh-c1 -n test exec test-c1-0 -- sh -c '
curl -sS -i -X POST localhost:8008/switchover \
  -H "Content-Type: application/json" \
  -d "{\"leader\":\"test-c1-0\",\"candidate\":\"test-c2-0\"}"
'
```

Reverse switchover:

```bash
kubectl --context kind-pgmesh-c2 -n test exec test-c2-0 -- sh -c '
curl -sS -i -X POST localhost:8008/switchover \
  -H "Content-Type: application/json" \
  -d "{\"leader\":\"test-c2-0\",\"candidate\":\"test-c1-0\"}"
'
```

After switchover:

```bash
kubectl --context kind-pgmesh-c1 -n test exec test-c1-0 -- \
  sh -c 'curl -fsS localhost:8008/cluster; echo'

kubectl --context kind-pgmesh-c2 -n test exec test-c2-0 -- \
  sh -c 'curl -fsS localhost:8008/cluster; echo'
```

## What was verified

1. Cilium Cluster Mesh connects the pod networks of two kind clusters.
2. `pgmesh-c2` reaches the shared `etcd` through a Cilium global service.
3. Patroni in two Kubernetes clusters can use one DCS.
4. The replica performs `basebackup` and streaming replication through Cluster Mesh.
5. The leader can switch between sites through Patroni switchover.

## Limitations of the current operator

Fully identical `postgresql` CRs with `metadata.name: test` in both Kubernetes clusters do not work as a single Patroni cluster:

- both StatefulSets create pod `test-0`
- Spilo generates Patroni `postgresql.name` from the hostname/pod name
- both members try to claim `/service/test/members/test-0`
- Patroni writes errors like `system ID mismatch` or `there is already a node named 'test-0' running`

`PATRONI_NAME` did not help: the env var exists, but Spilo still writes `postgresql.name: test-0` to `/run/postgres.yml`.

`PATRONI_SCOPE` did not help either: the env var exists, but Spilo has already generated `scope:` in `/run/postgres.yml` from `SCOPE`.

Where this happens:

- the operator sets `SCOPE` in `pkg/cluster/k8sres.go`, `generateSpiloPodEnvVars()`: `Name: "SCOPE", Value: c.Name`
- `c.Name` comes from the CR `metadata.name`
- Spilo writes `scope: &scope '{{SCOPE}}'` in `/scripts/configure_spilo.py`
- the final Patroni config is written to `/run/postgres.yml`

A production-ready solution needs an operator/Spilo patch that separates:

- Kubernetes resource name
- Patroni scope
- Patroni member name

## Cleanup

Delete the lab:

```bash
kind delete cluster --name pgmesh-c1
kind delete cluster --name pgmesh-c2
```
