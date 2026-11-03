---
name: nas-ssh-ops
description: 通过 SSH 操作 NAS 的标准规范（以绿联 DH4300 Plus/UGOS 为例，等价适用于群晖、飞牛、普通 Linux）。当需要 SSH 登录 NAS、查看或修改配置文件（如 frpc.toml）、装与维护 Docker 容器、重启服务、排查 SSH 连接失败/端口/日志/容器故障时触发。核心规范：BatchMode=yes 强制密钥、'bash -s'+引号 heredoc 批量执行、敏感配置脱敏展示、备份→sed→diff 的配置修改流程、借容器 root 写宿主机目录、docker 组免 sudo、systemd kill 重启法、安全与纪律（不代持密码）。
---

# 通过 SSH 操作 NAS 的标准规范

> 以绿联 DH4300 Plus（UGOS）为例，等价适用于群晖、飞牛、普通 Linux。
> 本规范为用户钦定标准，命令可直接复制。

## 0. 前置条件

### 0.1 开启 NAS 的 SSH

UGOS：**控制面板 → 终端机与 SNMP → 勾选「启用 SSH 服务」**（默认端口 22）。

### 0.2 本机准备 SSH 密钥（免密）

```bash
# 没有密钥就先生成
ls ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -N ""

# 把公钥装到 NAS 上（只需输一次密码）
ssh-copy-id user@nas
```

### 0.3 验证免密

```bash
ssh -o BatchMode=yes -o ConnectTimeout=4 user@nas 'echo OK'
```

`BatchMode=yes` 表示禁止密码交互：**只有密钥可用时才会成功**，适合脚本化调用。

## 1. 三种连接方式

### 1.1 交互式（手动排查）

```bash
ssh user@nas
```

### 1.2 单条命令（非交互）

```bash
ssh user@nas 'hostname; id; df -h'
```

### 1.3 多条命令 / 脚本（heredoc，推荐）

```bash
ssh -o BatchMode=yes user@nas 'bash -s' <<'EOF'
hostname
id
ss -ltnp | grep ':22 '
ls -ld /volume1/docker
EOF
```

**为什么用 `'bash -s'` + 引号 heredoc：**

- `<<'EOF'` 带引号 → 内容在本地**不做变量展开**，原样送到远端；
- 脚本里的单引号、双引号、`$` 都不用层层转义；
- 一次连接执行一整套逻辑，避免反复握手。

## 2. 读文件与脱敏

```bash
ssh host 'cat /path/file'                 # 全文
ssh host 'grep -n "serverAddr" /path/f'   # 定位行
ssh host 'ls -la /path'                   # 详情
```

**敏感配置先脱敏再看**（token、密码类字段不要原样输出）：

```bash
ssh host "sed -E 's|^(auth\.token[[:space:]]*=).*|\1 \"<HIDDEN>\"|' /path/frpc.toml"
```

## 3. 写文件

### 3.1 普通可写目录

```bash
ssh host 'cat > /path/file' <<'EOF'
内容
EOF
```

### 3.2 需要 root 权限的目录（借用 docker 的 root）

宿主机目录被 UGOS 收成 `700 root` 时，普通用户写不进去，但**容器内 root 可以**：

```bash
ssh host "docker exec -i 容器名 sh -c 'cat > /path/in/container'" <<'EOF'
内容
EOF
```

因为该路径通常是挂载点，写进去就等于写到宿主机对应目录。

## 4. Docker 权限怎么拿

- `docker.sock` 默认是 `root:docker 660`，普通用户直接 `docker` 会 `permission denied`。
- 加入 docker 组（需 sudo）：

```bash
sudo usermod -aG docker $USER
```

- **新 SSH 会话自动生效**（每次 ssh 都是新登录，PAM 重新加载组），不用重启 NAS。
- 验证：

```bash
ssh host 'id | grep docker; docker ps'
```

### 没有免密 sudo 怎么办

`systemctl restart 服务` 需要 root，普通用户做不了。两个办法：

1. 让用户手动执行一次（推荐，别代持密码）；
2. 若服务是 `Restart=always` 且进程属主是你，`kill <PID>` 后 systemd 会自动拉起并读取新配置——

```bash
kill 11080   # systemd 5 秒后自动重启
```

## 5. 改配置的标准流程（以 frpc.toml 为例）

```bash
f=/volume1/docker/frpc/frpc.toml
bak="$f.bak.$(date +%Y%m%d%H%M%S)"

cp -p "$f" "$bak"                      # 1) 先备份
sed -i 's|旧值|新值|' "$f"              # 2) 精确替换，只改目标行
diff "$bak" "$f"                       # 3) 校验：只应出现预期的那几行
# 4) 重启服务   5) 验证端口/日志
```

**要点：**

- 永远先备份，备份名带时间戳；
- `sed` 只匹配目标键，别用宽泛正则（避免误伤 `remotePort` 这类相似字段）；
- `diff` 是最后一道保险，确认「只改了该改的」。

## 6. 常用诊断命令

```bash
ss -ltnp | grep ':端口 '        # 谁在监听这个端口
ps aux | grep '[进]程名'        # 进程（方括号技巧避免匹配自己）
systemctl status 服务           # systemd 状态
journalctl -u 服务 -n 50        # 最近日志
docker ps -a                    # 所有容器
docker logs --tail 50 容器      # 容器日志
ip -4 addr show 接口            # 接口地址
```

连不通时按这个顺序查：**DNS/IP → 端口监听 → 防火墙 → 服务进程 → 日志**。

## 7. 安全与纪律

1. **先备份，再动手**；改完必须 `diff` 校验。
2. **敏感信息不外泄**：token、serverAddr、auth key 输出时脱敏，不贴进聊天记录。
3. **不代持密码**：需要交互 sudo 的操作交给用户执行，避免密码经手。
4. **改动前后各验证一次**：改前记录基线（端口/进程/PID），改后对比。
5. **只用密钥、不用密码**：`BatchMode=yes` 强制密钥，杜绝脚本卡在密码提示。

## 8. 场景对照表（来自实际排障记录）

| 场景                        | 用到的技巧                              |
| --------------------------- | --------------------------------------- |
| 查 NAS 有没有 frpc.toml     | 1.3 heredoc 执行 `find`                 |
| 读 frpc.toml 但不泄露 token | 2 脱敏 `sed`                            |
| 改 serverAddr / serverPort  | 5 备份 + `sed` + `diff`                 |
| 重启 frpc                   | 4 `kill` 让 systemd 拉起                |
| 装 Tailscale 容器           | 3.2 容器 root 写 compose                |
| 排 SSH 失败                 | 6 诊断 + `docker logs` 查 `getent` 报错 |