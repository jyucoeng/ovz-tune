# ovz-tune

给 Debian / Ubuntu 小鸡（OpenVZ / LXC / KVM 都行）做一次系统调优：内核参数、BBR + fq、路径 MTU、journald 限额、文件句柄、时区/主机名/swap。

改完之后写在哪：

| 文件 | 内容 |
|---|---|
| `/etc/sysctl.d/99-vps-tuning.conf` | 内核参数（带 `# BEGIN/END vps-tune` 标记） |
| `/etc/systemd/journald.conf.d/99-vps-limit.conf` | journald 磁盘上限 |
| `/etc/ssh/sshd_config.d/99-vps-keepalive.conf` | sshd 保活与登录策略 |
| `/etc/security/limits.conf` | `nofile 65535` 段 |
| `/etc/systemd/system/vps-tune-mtu.service` | 只在用过 `--mtu` 时才有 |

备份在 `/var/backups/vps-tune/`。

## 安装

```sh
# 装成 ovz-tune 命令（之后可反复用、可离线）
curl -fsSL https://raw.githubusercontent.com/jyucoeng/ovz-tune/main/install.sh | bash
```

装到 `/usr/local/bin/ovz-tune`。不想装就直接跑：

```sh
curl -fsSL https://raw.githubusercontent.com/jyucoeng/ovz-tune/main/ovz-tune.sh | bash -s -- --apply
```

## 用法

```sh
ovz-tune                    # 无参数：有终端进交互菜单，否则只演练
ovz-tune --apply            # 执行推荐调优
ovz-tune -h                 # 全部参数
```

| 参数 | 作用 |
|---|---|
| `--apply` | 真执行。不加就只演练 |
| `--rollback` | 撤销本脚本的所有改动 |
| `--upgrade` | 顺带 `apt upgrade`（可能重启服务，默认不做） |
| `--tz Asia/Shanghai` | 同时设时区 |
| `--hostname xxx` | 同时改主机名 |
| `--journald-max 200M` | 指定 journald 上限（默认按内存自动） |
| `--swap 1024` | 尝试加 1G swap（容器里会自动放弃） |
| `--mtu auto\|1400` | 实测路径 MTU 并持久化，或直接指定 |

**默认会改 sshd**（`PermitRootLogin yes`、`PasswordAuthentication yes`）。公网机器想跳过，用 [vps-tune](https://github.com/jyucoeng/vps-tune) 跑，它默认带 `--no-ssh`。

## 回滚

```sh
ovz-tune --rollback
```

删掉上表所有文件、清 `limits.conf` 里的段、qdisc 回 `fq_codel`、移除 MTU unit；老版 `ovz-tune` 留下的同名文件也一并清。

不自动还原：主机名、时区、swap、已经进内核的参数（重启才回默认）、当前 MTU。

## 卸载

```sh
rm -f /usr/local/bin/ovz-tune
```

只删命令，**不还原系统改动**；要撤调优先 `--rollback`。

## 被外部调用

[vps-tune](https://github.com/jyucoeng/vps-tune) 的 Debian/Ubuntu 后端 `tune-debian.sh` 就是本脚本（v1.2.0，加了 `--no-ssh` / `--no-journald` / `--no-clean`）。链路：

```sh
sb opt          → vps-tune --apply --no-ssh → tune-debian.sh
```

## 免责

按「现状」提供，无担保。会改 sysctl / journald / limits.conf / sshd 和网络队列，先跑演练自己备份。License: MIT
