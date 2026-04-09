# Cloudflared Tunnel Token 安全部署手册

**目标：** 客户将 CF tunnel token 安全存储在自己的 AWS Secrets Manager，CMI 使用 Docker 容器运行 cloudflared，token 通过 `--token-file` 注入容器，不以明文出现在任何配置文件或进程参数中。

**验证环境：** Amazon Linux 2 + Docker 25.0.14 + Docker Compose v5.1.1

**预计完成时间：** 90 分钟

---

## 执行位置说明

| 标记 | 含义 |
|------|------|
| `[MAC]` | 在你自己的 Mac / 电脑上执行 |
| `[宿主机]` | SSH 登录到 CMI 提供的 EC2 上执行 |
| `[浏览器]` | 在浏览器中操作 |

---

## 整体架构与安全边界

### 两层隔离

1. **存储层隔离：** token 存储在客户自己的 AWS Secrets Manager。CMI 没有客户的 AWS 账号权限，`GetSecretValue` 返回 403。
2. **运行层隔离：** cloudflared 运行在 Docker 容器内，使用 `--token-file` 读取密钥文件，token 值不出现在 `docker inspect`、`ps aux` 或任何配置文件中。

```
┌─────────────────────────────────────────────────────────────┐
│           客户自己的 AWS 账号（CMI 无任何访问权限）             │
│                                                             │
│  AWS Secrets Manager                                        │
│  └── prod/cloudflare/warp-reverse-tunnel-token-aws         │
│        AES-256 加密存储                                     │
│        访问控制：只允许绑定到宿主机的 IAM Role 读取           │
│        CMI IAM → GetSecretValue → 403 拒绝                 │
└─────────────────────────┬───────────────────────────────────┘
                          │ Instance Profile 自动认证
┌─────────────────────────▼───────────────────────────────────┐
│                   CMI 宿主机（EC2）                           │
│                                                             │
│  AWS CLI 拉取 token → /opt/cloudflared/secrets/tunnel.token │
│  目录权限：700（仅 root 可进入）                             │
│  文件权限：604（root 读写，容器内 UID 65532 只读）           │
│                                                             │
│  docker-compose.yml（CMI 运维可见，但无 token 值）           │
│                                                             │
│  cloudflared 容器（以 UID 65532 运行）                       │
│  └── --token-file /run/secrets/tunnel_token                 │
└─────────────────────────┬───────────────────────────────────┘
                          │ QUIC / TLS 1.3（4条连接）
                          ▼
                 Cloudflare 全球网络
```

### 透明说明：剩余暴露窗口

任何运行中的进程，其持有的凭证在技术上都可以被宿主机 root 用户通过内存读取获得。cloudflared 也不例外：

```bash
sudo docker exec cloudflared-prod cat /run/secrets/tunnel_token
# → token 明文可见
```

这是 Linux 操作系统的基本属性，不是本方案的缺陷。**本方案解决的是静态存储暴露（配置文件、docker inspect、ps aux）。** 运行时内存风险通过 CMI 的合规审计机制约束——见文末"CMI 合规审计承诺"章节。

---

## 前置条件

| 条件 | 说明 |
|------|------|
| 独立的 AWS 账号 | 必须是 CMI 完全无访问权限的账号。如果 CMI 也管理你的 AWS 账号，本方案无效。 |
| AWS IAM 管理权限 | 能创建 IAM Policy、Role、Instance Profile |
| CMI 提供的 EC2 实例 | Amazon Linux 2 或 Ubuntu 20.04+ |
| 能 SSH 登录 EC2 | |
| Cloudflare One 账号 | 已开通 Zero Trust，能创建 Tunnel |
| Mac 上有 AWS CLI | 如未安装见 Step 1 |

---

## Step 1 `[MAC]` — 配置 Mac 上的 AWS CLI 认证

### 1-A：确认 AWS CLI 已安装

```bash
aws --version
# 预期：aws-cli/2.x.x Python/3.x.x Darwin/...
```

如未安装，访问 https://aws.amazon.com/cli/ 下载 macOS 安装包。

### 1-B：创建 AWS Access Key

1. 登录 https://console.aws.amazon.com
2. 右上角点击用户名 → **Security credentials**
3. 向下滚动到 **Access keys** → **Create access key**

> **如果按钮是灰色：** AWS 每个账号最多 2 个 Access Key。先删除一个最久未使用的旧 Key，按钮即可点击。

4. Use case 选 **Command Line Interface (CLI)** → 勾选确认 → Next → Create
5. **立即复制两个值，只显示一次。**

| 字段 | 格式 | 示例（替换为你的真实值） |
|------|------|----------------------|
| Access key ID | AKIA 开头，20位 | `AKIAIOSFODNN7EXAMPLE` |
| Secret access key | 40位字符串 | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |

### 1-C：配置 AWS CLI

```bash
aws configure
```

```
AWS Access Key ID [None]:      AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]:  wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]:    ap-southeast-1     # ← 你的区域
Default output format [None]:  json
```

**常用区域代码：**

| 区域 | 代码 |
|------|------|
| 新加坡 | `ap-southeast-1` |
| 香港 | `ap-east-1` |
| 东京 | `ap-northeast-1` |
| 美国东部 | `us-east-1` |

### 1-D：验证配置成功

```bash
aws sts get-caller-identity
```

预期输出：
```json
{
    "UserId": "AIDAIOSFODNN7EXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

记录 `Account` 字段——这是你的 **AWS Account ID**（12位数字）。后续所有步骤中 `123456789012` 均替换为你的真实值。

> **报错 "Could not connect to the endpoint URL"：** 尝试加 `--region ap-southeast-1`。检查网络是否能访问 AWS。

---

## Step 2 `[MAC]` — 记录关键变量

本手册所有占位符：

| 占位符 | 替换为 | 来源 |
|--------|--------|------|
| `123456789012` | 你的 12 位 AWS Account ID | Step 1-D 输出 |
| `ap-southeast-1` | 你的 AWS 区域代码 | Step 1-C |

---

## Step 3 `[浏览器]` — 在 CF Dashboard 创建 Tunnel 并获取 Token

1. 打开 https://one.dash.cloudflare.com，选择账号
2. **Networks → Connectors → Cloudflare Tunnels → Create a tunnel**
3. Connector type 选 **Cloudflared** → Next
4. Tunnel name 填，例：`warp-reverse-tunnel` → **Save tunnel**
5. 下一页 OS 选 **Docker**
6. 页面显示命令，**只复制 eyJ 开头的 token 字符串**，不执行整条命令：

```bash
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIjoiMDI5N2Rm...
#                                                                           ↑ 只复制这里
```

> 选哪个操作系统不影响 token 值，选 Docker 只是让格式与我们的部署方式一致。

> **此时不要点 Next，也不要执行任何命令。** 页面显示 "No connectors installed" 是正常的，Step 10 部署完成后会自动出现 connector。

记录：

| 变量 | 值 |
|------|----|
| CF_TUNNEL_TOKEN | 完整 eyJ... 字符串 |
| CF_TUNNEL_NAME | tunnel 名字 |

---

## Step 4 `[MAC]` — 把 Token 存入 AWS Secrets Manager

> **如果同名 Secret 处于"待删除"状态：** AWS 默认保留已删除的 Secret 7 天，这期间不能创建同名 Secret。遇此情况，改用其他名称，并在后续所有命令中使用新名称。

### 方式 A：AWS CLI

```bash
aws secretsmanager create-secret \
  --name "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --description "Cloudflare tunnel token for Warp Reverse solution" \
  --secret-string "eyJhIjoiMDI5N2Rm...(替换为 Step 3 复制的完整 token)" \
  --region ap-southeast-1
```

预期输出：
```json
{
    "ARN": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/cloudflare/warp-reverse-tunnel-token-aws-AbCdEf",
    "Name": "prod/cloudflare/warp-reverse-tunnel-token-aws"
}
```

**记录完整 ARN（含末尾随机后缀如 `-AbCdEf`），Step 5 要用。**

### 方式 B：AWS Console

1. AWS Console → 搜索 Secrets Manager → **Store a new secret**
2. Secret type 选 **Other type of secret** → 切换到 **Plaintext** → 粘贴 token → Next
3. 名称填 `prod/cloudflare/warp-reverse-tunnel-token-aws` → Next → Next → Store
4. 进入该 Secret，复制 **Secret ARN**

### 验证

```bash
# describe-secret 只返回元数据，不返回 token 明文
aws secretsmanager describe-secret \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1
```

> 如输出含 `"DeletedDate"`，说明 Secret 处于待删除状态，先执行：
> ```bash
> aws secretsmanager restore-secret \
>   --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
>   --region ap-southeast-1
> ```

---

## Step 5 `[MAC]` — 创建 IAM Policy（最小权限）

```bash
# 把 ap-southeast-1 替换为你的区域，123456789012 替换为你的 Account ID
cat > /tmp/cf-tunnel-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadTunnelToken",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/cloudflare/warp-reverse-tunnel-token-aws-*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name "CloudflareTunnelTokenReadOnly" \
  --description "Read-only access to Cloudflare tunnel token in Secrets Manager" \
  --policy-document file:///tmp/cf-tunnel-policy.json
```

> Resource 末尾的 `-*` 通配符匹配 AWS 自动添加的随机后缀。

预期输出含：
```json
{
    "Policy": {
        "PolicyName": "CloudflareTunnelTokenReadOnly",
        "Arn": "arn:aws:iam::123456789012:policy/CloudflareTunnelTokenReadOnly"
    }
}
```

---

## Step 6 `[MAC]` — 创建 IAM Role 并绑定 Policy

```bash
# 创建 Trust Policy
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name "cf-tunnel-ec2-role" \
  --description "EC2 role for cloudflared — read tunnel token from Secrets Manager" \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# 绑定 Policy（123456789012 替换为你的 Account ID）
aws iam attach-role-policy \
  --role-name "cf-tunnel-ec2-role" \
  --policy-arn "arn:aws:iam::123456789012:policy/CloudflareTunnelTokenReadOnly"

aws iam create-instance-profile \
  --instance-profile-name "cf-tunnel-ec2-profile"

aws iam add-role-to-instance-profile \
  --instance-profile-name "cf-tunnel-ec2-profile" \
  --role-name "cf-tunnel-ec2-role"
```

验证：
```bash
aws iam list-attached-role-policies --role-name "cf-tunnel-ec2-role"
# 应看到 CloudflareTunnelTokenReadOnly
```

---

## Step 7 `[浏览器 / MAC]` — 把 IAM Role 绑定到 EC2 实例

### 方式 A：AWS Console

1. EC2 → Instances → 勾选目标实例
2. **Actions → Security → Modify IAM role**
3. 选择 `cf-tunnel-ec2-role` → **Update IAM role**

### 方式 B：AWS CLI

```bash
# i-xxxxxxxxxxxxxxxxx 替换为你的 EC2 Instance ID
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --iam-instance-profile Name="cf-tunnel-ec2-profile" \
  --region ap-southeast-1
```

### 在宿主机验证

```bash
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
# 预期输出：cf-tunnel-ec2-role
```

如无输出，等 1-2 分钟再试。

---

## Step 8 `[宿主机]` — 安装 Docker 和 AWS CLI

### Docker — Amazon Linux 2

> **不要使用 `curl -fsSL https://get.docker.com | sudo sh`**，Amazon Linux 2 不支持该脚本，会报 `ERROR: Unsupported distribution 'amzn'`。
>
> 如机器上有损坏的第三方 repo（如 ookla speedtest），yum 会整体失败。使用 `--disablerepo` 绕过。

```bash
sudo yum install -y docker \
  --disablerepo="*" \
  --enablerepo="amzn2-core,amzn2extra-docker"

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker  # 刷新用户组（或重新登录 SSH）
```

### Docker — Ubuntu / Debian

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

### Docker Compose Plugin（Amazon Linux 2 需单独安装）

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

验证：
```bash
docker --version && docker compose version
# Docker version 25.0.14, build 0bab007
# Docker Compose version v5.1.1
```

### AWS CLI v2

```bash
curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
sudo yum install -y unzip 2>/dev/null || sudo apt-get install -y unzip 2>/dev/null
unzip -q /tmp/awscliv2.zip -d /tmp/
sudo /tmp/aws/install
aws --version
```

### 验证 Instance Profile 权限（无需 aws configure）

```bash
aws secretsmanager describe-secret \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1
# 能看到 ARN 和 Name 说明权限正常，无需配置任何 Access Key
```

> **AccessDeniedException：** IAM Role 未绑定，回到 Step 7。  
> **UnrecognizedClientException：** 区域填错或 Instance Profile 未生效，等 2 分钟再试。

---

## Step 9 `[宿主机]` — 从 AWS 拉取 Token 并写入本地密钥文件

```bash
# 创建密钥目录（仅 root 可进入）
sudo mkdir -p /opt/cloudflared/secrets
sudo chmod 700 /opt/cloudflared/secrets

# 从 AWS 拉取 token 写入文件
aws secretsmanager get-secret-value \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1 \
  --query "SecretString" \
  --output text \
  | sudo tee /opt/cloudflared/secrets/tunnel.token > /dev/null

# 设置文件权限
# 604 = root 读写，group 无权，other 只读
# 重要：cloudflared 容器内以 UID 65532 运行（非 root）
# 容器需要 other 读权限（r--），否则报 permission denied
# chmod 600 会导致容器启动失败
sudo chmod 604 /opt/cloudflared/secrets/tunnel.token
sudo chown root:root /opt/cloudflared/secrets/tunnel.token

# 验证（普通用户无法进入目录，用 sudo 查看）
sudo stat /opt/cloudflared/secrets/tunnel.token | grep Access
```

预期输出：
```
Access: (0604/-rw----r--)  Uid: (0/root)   Gid: (0/root)
```

权限说明：
- `rw-`（owner=root）：root 可读写
- `---`（group）：组无权限
- `r--`（other）：容器内 UID 65532 通过 other 权限读取
- 目录 `chmod 700`：普通运维账号连 `ls` 都报 Permission denied

---

## Step 10 `[宿主机]` — 创建 docker-compose.yml 并启动容器

```bash
sudo mkdir -p /opt/cloudflared

sudo tee /opt/cloudflared/docker-compose.yml > /dev/null << 'EOF'
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-prod
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token-file /run/secrets/tunnel_token
    secrets:
      - tunnel_token
    tmpfs:
      - /tmp
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "5"

secrets:
  tunnel_token:
    file: /opt/cloudflared/secrets/tunnel.token
EOF
```

**关键配置说明：**

| 配置项 | 原因 |
|--------|------|
| `--token-file` 而非 `--token` | token 值不出现在 `docker inspect` 或进程参数中 |
| `secrets.tunnel_token.file` | Docker secrets 机制将文件挂载进容器 |
| `tmpfs: /tmp` | 防止容器将临时文件写入宿主机磁盘 |
| `restart: unless-stopped` | 宿主机重启后容器自动恢复 |

> **为什么没有 `read_only: true`：** Docker secrets 挂载到 `/run/secrets/`，read_only 模式下容器内 UID 65532 无法读取该路径，会报 `permission denied`。改用 `tmpfs: /tmp` 作为等效保护。

```bash
cd /opt/cloudflared
sudo docker compose up -d
sudo docker compose ps
```

预期输出：
```
NAME               IMAGE                           STATUS
cloudflared-prod   cloudflare/cloudflared:latest   Up X seconds
```

---

## Step 11 `[宿主机 / 浏览器]` — 验证 Tunnel 正常运行

### 查看日志确认 4 条连接建立

```bash
sudo docker logs cloudflared-prod --tail 30
```

预期关键行：
```
INF Registered tunnel connection connIndex=0 ... location=sin21 protocol=quic
INF Registered tunnel connection connIndex=1 ... location=sin12 protocol=quic
INF Registered tunnel connection connIndex=2 ... location=sin22 protocol=quic
INF Registered tunnel connection connIndex=3 ... location=sin17 protocol=quic
```

### 验证 docker inspect 不暴露 token

```bash
sudo docker inspect cloudflared-prod | grep -iE "token|eyJ"
# 预期：只含文件路径，无 eyJ... token 值
```

### 验证进程参数不含 token

```bash
ps aux | grep cloudflared
# 预期：...run --token-file /run/secrets/tunnel_token（文件路径，不是 token 值）
```

### CF Dashboard 确认 Healthy

1. https://one.dash.cloudflare.com
2. Networks → Connectors → Cloudflare Tunnels
3. 对应 tunnel 显示 **Healthy**（绿色）

---

## Step 12 `[宿主机]` — 安全防护效果验证（含 root 视角透明说明）

### 测试 1：docker inspect

```bash
sudo docker inspect cloudflared-prod | grep -iE "token|eyJ"
```
✅ **防住。** 只含文件路径，无 token 值。

### 测试 2：读 docker-compose.yml

```bash
cat /opt/cloudflared/docker-compose.yml | grep -iE "eyJ"
```
✅ **防住。** 配置文件无 token 值。

### 测试 3：普通运维账号读密钥文件

```bash
ls /opt/cloudflared/secrets/tunnel.token
cat /opt/cloudflared/secrets/tunnel.token
```
✅ **防住。**
```
ls: cannot access /opt/cloudflared/secrets/tunnel.token: Permission denied
cat: /opt/cloudflared/secrets/tunnel.token: Permission denied
```

### 测试 4：CMI 尝试访问客户 AWS Secrets Manager

```bash
AWS_ACCESS_KEY_ID="" AWS_SECRET_ACCESS_KEY="" \
aws secretsmanager get-secret-value \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1 \
  --no-sign-request 2>&1 || true
```
✅ **防住。** `An error occurred (AuthFailure): Request must be signed`

### 测试 5：root 通过 docker exec 进容器读取 ⚠️ 剩余暴露

```bash
sudo docker exec cloudflared-prod cat /run/secrets/tunnel_token
```
⚠️ **token 明文可见。**

这是已知且公开说明的剩余暴露面。等效路径：
```bash
sudo cat /proc/$(docker inspect --format '{{.State.Pid}}' cloudflared-prod)/root/run/secrets/tunnel_token
```

**这是 Linux 操作系统的基本属性。** 本方案的价值在于将 token 暴露从"任何人随时可见"提升为"需要 root 主动操作且留下审计痕迹"。

---

## 安全防护总结

| 读取手段 | 结果 | 说明 |
|---------|------|------|
| `docker inspect` | ✅ 防住 | `--token-file` — 只含文件路径 |
| 读 docker-compose.yml | ✅ 防住 | 配置无 token 值 |
| `ps aux` | ✅ 防住 | 进程参数只有文件路径 |
| 普通运维账号读密钥文件 | ✅ 防住 | 目录 700，Permission denied |
| CMI 访问客户 AWS | ✅ 防住 | 无 IAM 权限，403 |
| root docker exec 进容器 | ⚠️ 可见 | 需主动操作，全程审计记录 |
| root 读进程内存 | ⚠️ 可见 | 操作系统级别，无纯软件方案 |

### 与传统方式对比

| 对比维度 | 传统方式（`--token eyJ...` 直传）| 本方案 |
|---------|--------------------------------|--------|
| `docker inspect` 暴露 | ❌ token 在 Args | ✅ 只含文件路径 |
| 配置文件落盘 | ❌ token 在 compose/service 文件 | ✅ 密钥文件 root-only |
| CMI 访问 AWS | N/A | ✅ 无 IAM 权限，403 |
| Token Rotate | 改配置文件 + 重启 | ✅ 重拉 + 重启容器 |
| 主动攻击门槛 | 低（cat 文件即可）| 高（需 root + 主动操作 + 留审计痕迹）|

---

## CMI 合规审计承诺

鉴于技术透明性——宿主机 root 用户在技术上仍可通过主动操作读取运行中容器内的 token——CMI 承诺以下合规机制：

### 最小权限原则
- 日常运维账号（如 ec2-user）不具备 root 或 sudo 权限，无法进入密钥目录或执行 `docker exec`
- root / sudo 权限仅限于特定系统管理员，实行双人审批机制
- CMI 运维账号在 OpenBao（CMI 内部 Secret 管理系统）中对客户 token 路径**无读取权限**

### 全程审计日志
- 宿主机启用 `auditd`，记录所有 `docker exec`、`ptrace`、`/proc` 访问，日志不可篡改
- OpenBao 记录所有 Secret 读写事件（操作者、时间戳、来源 IP）
- AWS CloudTrail 记录 Secrets Manager 所有 API 调用（客户可在自己 AWS 账号独立查看）
- 日志保留期：不少于 90 天

### 客户可审查权
- 客户可随时申请审计报告，CMI 提供该客户 token 的访问记录（仅限该客户，不含其他客户信息）
- 客户可通过自己的 AWS CloudTrail 独立验证，无需依赖 CMI

### 合同层面约束
- 服务协议明确禁止 CMI 员工访问客户 tunnel token 及衍生凭证
- 违反此条款视为重大安全事故，适用相应赔偿和法律责任条款

### Token Rotate 建议
- 建议最少每季度 rotate 一次
- 相关人员变动时立即 rotate
- Rotate 后旧 token 在 Cloudflare 侧立即失效，即使之前被读取也无法继续使用

---

## 常见问题排查

### `yum install docker` 报错 "One of the configured repositories failed"

第三方 repo（如 ookla speedtest）损坏导致 yum 整体失败：

```bash
sudo yum install -y docker \
  --disablerepo="*" \
  --enablerepo="amzn2-core,amzn2extra-docker"
```

### 容器反复重启，日志显示 "Failed to read token file: permission denied"

```bash
# 确认容器内用户
sudo docker inspect cloudflared-prod | grep '"User"'
# 应显示 "User": "65532:65532"

# 修正权限
sudo chmod 604 /opt/cloudflared/secrets/tunnel.token

# 重启
cd /opt/cloudflared && sudo docker compose down && sudo docker compose up -d
```

### Tunnel 显示 Unhealthy 或无连接

```bash
sudo docker logs cloudflared-prod --tail 50 | grep -E "ERR|WARN|failed"
```

常见原因：出站端口被阻断。确保以下出站规则开放：
- `443/tcp` → `api.cloudflare.com`
- `7844/tcp + 7844/udp` → `region1.v2.argotunnel.com` 和 `region2.v2.argotunnel.com`

### 如何 Rotate Token

**Step 1** `[浏览器]` CF Dashboard → Networks → Tunnels → tunnel → Edit → **Refresh token** → 复制新 token

**Step 2** `[MAC]` 更新 AWS Secrets Manager：
```bash
aws secretsmanager update-secret \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --secret-string "eyJ...(新的完整 token)" \
  --region ap-southeast-1
```

**Step 3** `[宿主机]` 重拉并重启：
```bash
aws secretsmanager get-secret-value \
  --secret-id "prod/cloudflare/warp-reverse-tunnel-token-aws" \
  --region ap-southeast-1 \
  --query "SecretString" \
  --output text \
  | sudo tee /opt/cloudflared/secrets/tunnel.token > /dev/null

cd /opt/cloudflared && sudo docker compose restart cloudflared
```

如怀疑有未授权 connector，强制断开所有连接：
```bash
# ACCOUNT_ID、TUNNEL_ID、CLOUDFLARE_API_TOKEN 替换为真实值
curl "https://api.cloudflare.com/client/v4/accounts/ACCOUNT_ID/cfd_tunnel/TUNNEL_ID/connections" \
  --request DELETE \
  --header "Authorization: Bearer CLOUDFLARE_API_TOKEN"
```

---

*版本：2026-04 | 验证环境：Amazon Linux 2 + Docker 25.0.14 | 作者：Dong Zhang*
