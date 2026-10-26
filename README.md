# clash-updater

Ansible role + playbook for deploying a daily Clash/Mihomo subscription updater on Linux.

安装一个 systemd timer，每天定时从订阅地址拉取配置并更新 Clash/Mihomo 配置目录。

## 目录结构

```
.
├── main.yml                              # playbook 入口
├── group_vars/
│   └── mihomo/
│       ├── main.yml                      # 非敏感变量（公开）
│       └── vault.yml                     # 订阅 URL（加密，不进 git）
└── roles/
    └── clash-updater/
        ├── defaults/main.yml             # 变量兜底
        ├── files/
        │   ├── clash-updater             # 更新脚本
        │   ├── clash-updater.service
        │   └── clash-updater.timer
        ├── handlers/main.yml             # daemon-reload
        ├── tasks/main.yml                # 安装 + 配置 + 启用
        └── templates/
            └── environment.conf.j2       # systemd drop-in，注入 URL/CONFIG
```

## 部署效果

- 脚本安装到 `/usr/local/bin/clash-updater`（`0755`）
- systemd 单元安装到 `/usr/local/lib/systemd/system/`（`0644`）
- drop-in 安装到 `/usr/local/lib/systemd/system/clash-updater.service.d/environment.conf`（`0600`）
- timer 每天 `03:30` 触发，`RandomizedDelaySec=5min`，`Persistent=true`
- 启用 `clash-updater.timer`

## 前置要求

- 被控主机：已安装 `systemd-timer`（systemd ≥ 240）与 `openssh-server`
- 本机：`Ansible`

## 使用

### 1. 准备 vault

`group_vars/mihomo/vault.yml` 不在仓库里，需自行创建并加密。

```bash
cat > group_vars/mihomo/vault.yml <<'EOF'
clash_updater_url: "https://example.com/api/v1/client/subscribe?token=REPLACE_ME"
EOF

ansible-vault encrypt group_vars/mihomo/vault.yml
```

之后 `.gitignore` 会忽略它，不会被误提交。

### 2. 配置 inventory

在 `main.yml` 同目录或 `ansible.cfg` 指定的位置准备 `hosts.yml`，例如：

```yaml
all:
  hosts:
    cachyos:
      ansible_host: 192.0.2.10
```

或直接用 `~/.ssh/config` 里的 Host 名。

### 3. 运行

```bash
# 语法检查
ansible-playbook main.yml --syntax-check

# 预演
ansible-playbook main.yml --limit=cachyos --check --diff

# 实际部署
ansible-playbook main.yml --limit=cachyos --ask-vault-pass
```

如已配置 `vault_password_file`（`ansible.cfg` 的 `[defaults]`），可省略 `--ask-vault-pass`。

### 4. 验证

```bash
# drop-in 是否被 systemd 识别
systemctl cat clash-updater.service

# 环境变量是否注入
systemctl show clash-updater.service -p Environment

# 手动触发一次
systemctl start clash-updater.service
journalctl -u clash-updater.service -n 50 --no-pager
```

## 变量

| 变量 | 所在文件 | 说明 |
|---|---|---|
| `clash_updater_url` | `group_vars/mihomo/vault.yml`（加密） | 订阅地址，含 token |
| `clash_updater_config` | `group_vars/mihomo/main.yml`（公开） | 配置文件输出路径，例如 `/etc/mihomo/config.yaml` |

`roles/clash-updater/defaults/main.yml` 里两者默认空字符串。

## 修改调度时间

编辑 `roles/clash-updater/files/clash-updater.timer` 里的 `OnCalendar=`，重跑 playbook 即可。

当前默认：

```ini
OnCalendar=*-*-* 03:30:00
RandomizedDelaySec=5min
Persistent=true
```

## 卸载

```bash
systemctl disable --now clash-updater.timer
rm -f /usr/local/lib/systemd/system/clash-updater.timer \
      /usr/local/lib/systemd/system/clash-updater.service \
      /usr/local/bin/clash-updater
rm -rf /usr/local/lib/systemd/system/clash-updater.service.d
systemctl daemon-reload
```

## 许可

见 [LICENSE](LICENSE)。
