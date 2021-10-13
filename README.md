# k3s-upgrade

k3s-upgrade is an image that is responsible of upgrading k3s version via the [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller), it does that by doing the following:

- Replace the k3s binary with the new version
- Kill the old k3s process allowing the supervisor to restart k3s with the new version

## Example

1- To use the image with the system-upgrade-controller, you have first to run the controller either directly or deploy it on the k3s cluster:
```bash
kubectl apply -f system-upgrade-controller.yaml
```

You should see the upgrade controller starting in `system-upgrade` namespace.

2- Label the nodes you want to upgrade with the right label:
```
kubectl label node <node-name> k3s-upgrade=true
```

3- Run the upgrade plan in the k3s cluster
```bash
kubectl apply -f upgrade-plan.yml
```
```bash
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server
  namespace: system-upgrade
  labels:
    k3s-upgrade: server
spec:
  concurrency: 1 # Batch size (roughly maps to maximum number of unschedulable nodes)
  version: v1.21.5+k3s2
  nodeSelector:
    matchExpressions:
      - {key: k3s-upgrade, operator: Exists}
      - {key: k3s-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: k3os.io/mode, operator: DoesNotExist}
      - {key: node-role.kubernetes.io/control-plane, operator: Exists}
  serviceAccountName: system-upgrade
  tolerations:
  - key: master
    operator: Equal
    value: master
    effect: NoSchedule
  cordon: true
  upgrade:
    image: rancher/k3s-upgrade
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-agent
  namespace: system-upgrade
  labels:
    k3s-upgrade: agent
spec:
  concurrency: 2 # Batch size (roughly maps to maximum number of unschedulable nodes)
  version: v1.21.5+k3s2
  nodeSelector:
    matchExpressions:
      - {key: k3s-upgrade, operator: Exists}
      - {key: k3s-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: k3os.io/mode, operator: DoesNotExist}
      - {key: node-role.kubernetes.io/control-plane, operator: DoesNotExist}
  serviceAccountName: system-upgrade
  tolerations:
  - key: master
    operator: Equal
    value: master
    effect: NoSchedule
  prepare:
    # Defaults to the same "resolved" tag that is used for the `upgrade` container, NOT `latest`
    image: rancher/k3s-upgrade
    args: ["prepare", "k3s-server"]
  cordon: true  
#  drain:
#    force: true
#    skipWaitForDeleteTimeout: 60 # 1.18+ (honor pod disruption budgets up to 60 seconds per pod then moves on)
  upgrade:
    image: rancher/k3s-upgrade
``` 

The upgrade controller should watch for this plan and execute the upgrade on the labeled nodes. For more information about system-upgrade-controller and plan options please visit [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) official repo.
  
  
[If you have any questions/suggestions you can contact me directly in Telegram Here](https://t.me/vainkop)