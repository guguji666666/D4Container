# D4Container

## 1. Test network connection

* 节点带有 `CriticalAddonsOnly=true` taint（需要 toleration）
* 部署使用 `default` namespace（不建议），而 Defender 多部署在 `kube-system`
* Pod 镜像拉取失败（需要访问公网）
* 缺少 CPU/内存资源限制（调度器可能不给资源）

---

## ✅ 完善后的 DaemonSet YAML（100%兼容 AKS）

以下是优化后的 **可直接在 AKS 中使用的 DaemonSet**，具备：

* ✅ 容忍所有 taint（保证能部署到 system node）
* ✅ 资源限制（防止被驱逐）
* ✅ 设置 namespace 为 `kube-system`
* ✅ 使用稳定镜像 `nicolaka/netshoot`
* ✅ 自动打印测试日志
* ✅ 支持节点调度器过滤兼容

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

