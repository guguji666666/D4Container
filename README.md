# D4Container

## 1. Test network connection

---

````markdown
# 🔍 Microsoft Defender 连接性测试（AKS 环境）

本操作指南用于在 Azure Kubernetes Service (AKS) 中测试 Microsoft Defender 相关 API 域名的网络连通性，适用于目标容器中无法直接 `exec` 进入的场景（如 `distroless` 镜像）。

---

## 🚀 步骤 1：在目标节点上运行调试 Pod

### 📄 创建调试 Pod 配置文件（`debug-net.yaml`）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-net
  namespace: kube-system
spec:
  nodeSelector:
    kubernetes.io/hostname: aks-agentpool-11763858-vmss000007
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
  restartPolicy: Never
````

> ✅ 替换 `nodeSelector` 的值为 Defender Pod 所在节点名

---

### 📥 应用 Pod

```bash
kubectl apply -f debug-net.yaml
```

检查 Pod 状态：

```bash
kubectl get pod debug-net -n kube-system
```

---

## 🧪 步骤 2：进入调试 Pod 并运行连通性测试

```bash
kubectl exec -it debug-net -n kube-system -- bash
```

进入后粘贴以下脚本：

```bash
for url in \
  "https://ami.cloud-dev.defender.microsoft.com" \
  "https://api.cloud-dev.defender.microsoft.com" \
  "https://api.cloud-stg.defender.microsoft.com" \
  "https://api.cloud.defender.microsoft.com" \
  "https://api.defender.microsoft.com"; do

  echo -e "\n🔍 Testing: $url"
  domain=$(echo "$url" | awk -F[/:] '{print $4}')

  echo "1️⃣ DNS Lookup:"
  nslookup $domain || echo "❌ DNS failed"

  echo "2️⃣ HTTPS Connectivity:"
  curl -s -o /dev/null -w " ↪ HTTP Code: %{http_code}, Time: %{time_total}s\n" --connect-timeout 5 "$url" || echo "❌ Curl failed"

  echo "3️⃣ TLS Certificate Info:"
  echo | openssl s_client -connect "$domain:443" -servername "$domain" 2>/dev/null | openssl x509 -noout -dates -subject -issuer || echo "❌ TLS Cert fetch failed"

done
```

---

## 🧹 步骤 3：测试完成后清理调试 Pod

```bash
kubectl delete pod debug-net -n kube-system
```

---

## ✅ 输出示例（成功）

```
🔍 Testing: https://api.defender.microsoft.com
1️⃣ DNS Lookup:
Name:	api.defender.microsoft.com
Address: 20.190.130.10

2️⃣ HTTPS Connectivity:
 ↪ HTTP Code: 403, Time: 0.134s

3️⃣ TLS Certificate Info:
notBefore=Mar 15 00:00:00 2025 GMT
notAfter=Jun 15 23:59:59 2025 GMT
issuer=Microsoft Azure TLS Issuing CA 06
```

---

> 💡 可用于 Defender for IoT / MDI / MDE 等组件的域名连接性排查。
