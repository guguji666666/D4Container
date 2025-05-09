# D4Container

## 1. View Pod Logs in Kubernetes

This guide helps you inspect logs from various Microsoft Defender components running in your AKS or Kubernetes cluster.

> ✅ This is useful for debugging collector, publisher, and misc pods.

---

## 📌 Prerequisites

Ensure you have:

- `kubectl` installed and configured
- Access to the `kube-system` namespace
- Sufficient permission to get pods and logs

---

## 🔍 Step 1: List Defender Pods

Run the following command to find Defender-related pods:

```bash
kubectl get pods -n kube-system --no-headers | grep microsoft-defender
````

Typical output:

```
microsoft-defender-collector-ds-dbjpz                  2/2   Running   0     45h
microsoft-defender-collector-ds-rrb7n                  2/2   Running   0     45h
microsoft-defender-collector-ds-wq5x5                  2/2   Running   0     45h
microsoft-defender-collector-ds-xrjw8                  2/2   Running   0     45h
microsoft-defender-collector-misc-67f54549c-xcprr      1/1   Running   0     45h
microsoft-defender-publisher-ds-9wvn2                  1/1   Running   0     26h
microsoft-defender-publisher-ds-cjt6t                  1/1   Running   0     26h
microsoft-defender-publisher-ds-dz8gn                  1/1   Running   0     26h
microsoft-defender-publisher-ds-vv7dg                  1/1   Running   0     26h
```

---

## 🧩 Step 2: Get Container Names (for Multi-Container Pods)

Some pods (like `collector-ds-*`) have **multiple containers**, so you need to fetch their names:

```bash
kubectl get pod <pod name> -n kube-system -o jsonpath='{.spec.containers[*].name}'
```

Sample
```bash
kubectl get pod microsoft-defender-collector-ds-dbjpz -n kube-system -o jsonpath='{.spec.containers[*].name}'
```

Sample output:

```
microsoft-defender-pod-collector microsoft-defender-low-level-collector
```

---

## 📄 Step 3: View Logs (No Looping, Manual Inspection)

### 🧪 Collector Pods (multi-container)

```bash
kubectl logs microsoft-defender-collector-ds-dbjpz -n kube-system -c microsoft-defender-pod-collector
kubectl logs microsoft-defender-collector-ds-dbjpz -n kube-system -c microsoft-defender-low-level-collector

kubectl logs microsoft-defender-collector-ds-rrb7n -n kube-system -c microsoft-defender-pod-collector
kubectl logs microsoft-defender-collector-ds-rrb7n -n kube-system -c microsoft-defender-low-level-collector

# Repeat for each pod as needed
```

### 🧬 Misc and Publisher Pods (single-container)

```bash
kubectl logs microsoft-defender-collector-misc-67f54549c-xcprr -n kube-system
kubectl logs microsoft-defender-publisher-ds-9wvn2 -n kube-system
kubectl logs microsoft-defender-publisher-ds-cjt6t -n kube-system
kubectl logs microsoft-defender-publisher-ds-dz8gn -n kube-system
kubectl logs microsoft-defender-publisher-ds-vv7dg -n kube-system
```

---

## 🧠 Tips

* You can replace pod names dynamically using shell scripts or aliases if needed.
* If pods are in `CrashLoopBackOff`, logs are still accessible unless evicted.

---


## 📚 References

* [Microsoft Defender for Containers Docs](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-containers-introduction)
* [kubectl logs](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)

---

💬 **Feel free to submit an issue or PR if you’d like to improve this guide.**

```

---

是否需要我将其直接转换为 GitHub 仓库的 `README.md` 文件并给出仓库结构建议？
```

---


## 2. Test network connection

* 节点带有 `CriticalAddonsOnly=true` taint（需要 toleration）
* 部署使用 `default` namespace（不建议），而 Defender 多部署在 `kube-system`
* Pod 镜像拉取失败（需要访问公网）
* 缺少 CPU/内存资源限制（调度器可能不给资源）

---

## ✅ 完善后的 DaemonSet YAML（100%兼容 AKS）

### ✅ 特性一览（Features）

* ✅ 支持部署到带有 taint 的 system 节点（容忍所有 taint）
* ✅ 设置合适的资源限制，避免被 kubelet 驱逐（eviction）
* ✅ 默认部署在 `kube-system` 命名空间中
* ✅ 使用稳定可靠的基础镜像：`nicolaka/netshoot`
* ✅ 启动后自动执行网络测试并输出日志
* ✅ 支持使用节点调度器（nodeSelector / affinity）进行灵活调度

---

### 📄 文件名：`defender-connectivity-daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: defender-netcheck
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: defender-netcheck
  template:
    metadata:
      labels:
        app: defender-netcheck
    spec:
      tolerations:
      - operator: Exists
      containers:
      - name: checker
        image: nicolaka/netshoot
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "[🟢 Start connectivity test on $(hostname)]"
            for url in \
              "https://ami.cloud-dev.defender.microsoft.com" \
              "https://api.cloud-dev.defender.microsoft.com" \
              "https://api.cloud-stg.defender.microsoft.com" \
              "https://api.cloud.defender.microsoft.com" \
              "https://api.defender.microsoft.com"; do
              echo "==== $url ====";
              domain=$(echo $url | awk -F[/:] '{print $4}');
              nslookup $domain || echo "DNS failed";
              curl -s -o /dev/null -w "↪ HTTP Code: %{http_code}, Time: %{time_total}s\n" --connect-timeout 5 $url || echo "Curl failed";
              echo "";
            done
            sleep 3600
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
      restartPolicy: Always
```

---

## ✅ 正确的部署命令：

```bash
kubectl apply -f defender-connectivity-daemonset.yaml
```

---

## ✅ 查看日志输出（可多节点对比）：

```bash
kubectl get pods -n kube-system -l app=defender-netcheck -o wide
```
![image](https://github.com/user-attachments/assets/815ba46a-37c9-45f5-9d1c-e8fb0c279e58)

```bash
kubectl logs -n kube-system <pod-name>
```

---

## ✅ 完成后清理：

```bash
kubectl delete -f defender-connectivity-daemonset.yaml
```

---

