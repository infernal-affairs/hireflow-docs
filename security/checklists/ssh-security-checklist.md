# SSH 安全检查清单

> 适用范围：本机 macOS 开发环境。  
> 原则：只检查文件权限、指纹、配置和连接结果，不读取、不上传、不打印私钥内容。

## 一、检查文件权限

```bash
stat -f '%Sp %OLp %N' "$HOME/.ssh"
find "$HOME/.ssh" -maxdepth 1 -type f -exec stat -f '%Sp %OLp %N' {} \;
```

推荐权限：

| 文件 | 推荐权限 | 原因 |
| --- | --- | --- |
| `~/.ssh` | `700` | 只有本人可以进入目录 |
| 私钥 | `600` | 只有本人可以读写 |
| `authorized_keys` | `600` | 只有本人可以修改授权关系 |
| 公钥、`config` | `644` | 不包含秘密，可读 |
| `known_hosts` | `600` | 防止被非本人修改 |

## 二、确认私钥是否设置口令

```bash
ssh-keygen -y -P '' -f "$HOME/.ssh/id_ed25519" >/dev/null 2>&1
printf 'exit=%s\n' "$?"
```

- 返回 `0`：私钥没有口令，离开本机后更容易被直接使用，应评估是否加口令。
- 返回非 `0`：私钥有口令或文件不可用。

不要输出私钥内容。

## 三、检查公钥指纹

```bash
ssh-keygen -lf "$HOME/.ssh/id_ed25519.pub" -E sha256
```

指纹可用于与 GitHub 账号中登记的密钥比对。

## 四、检查 SSH 代理

```bash
ssh-add -l
```

`The agent has no identities.` 表示当前代理没有加载密钥。

## 五、检查实际生效配置

```bash
ssh -G github.com | rg '^(hostname|user|port|forwardagent|permitlocalcommand|proxycommand|strictHostKeyChecking|identityfile) '
```

重点确认：

- `forwardagent no`
- `permitlocalcommand no`
- 主机名和端口符合预期
- 没有意外的 `ProxyCommand`

## 六、确认 GitHub 主机指纹

```bash
known=$(ssh-keygen -F github.com -f "$HOME/.ssh/known_hosts" 2>/dev/null | awk '$1=="github.com" && $2=="ssh-ed25519" {print $3; exit}')
live=$(ssh-keyscan -T 10 -t ed25519 github.com 2>/dev/null | awk '$1=="github.com" && $2=="ssh-ed25519" {print $3; exit}')
[ -n "$known" ] && [ "$known" = "$live" ] && echo MATCH || echo MISMATCH
```

正式检查时应与 GitHub 官方公布的 SSH 指纹比对。

## 七、测试身份认证

```bash
ssh -o BatchMode=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=15 -T git@github.com
```

成功时 GitHub 通常返回：

```text
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

GitHub 不提供 shell 时命令退出码可能非零，这是预期行为。

## 八、检查 Git 远程协议

```bash
for repo in backend frontend docs; do
  printf '%s: ' "$repo"
  git -C "$repo" remote get-url origin
done
```

确认远程地址是预期的 HTTPS 或 SSH，不包含凭据、用户密码或未知域名。

## 九、检查危险 Git 配置

```bash
git config --global --get-regexp '^(url\..*\.insteadof|credential\.|http\.|https\.)'
```

关注未知的 `insteadOf`、凭据明文、不安全 HTTP 协议和异常 credential helper。

## 十、检查 GitHub CLI 登录范围

```bash
gh auth status
```

确认登录账号正确，token 范围符合实际需要。

## 十一、检查禁用项

- 私钥被提交到任何仓库
- `ForwardAgent yes`
- 未知 `ProxyCommand`
- `authorized_keys` 中有不明公钥
- `.ssh` 目录权限为 `777` 或 `755`

## 十二、发现问题后的处理顺序

1. 立即修正本机文件权限。
2. 停止使用可疑密钥。
3. 在 GitHub 账号中撤销可疑公钥。
4. 重新生成密钥并设置口令。
5. 重新登记公钥并验证指纹。
6. 检查近期仓库提交和远程地址。

## 十三、本次检查结论

检查日期：2026-10-29

- `~/.ssh` 原权限：`755`，已修正为 `700`。
- `id_ed25519` 私钥权限：`600`。
- `known_hosts` 权限：`600`。
- `id_ed25519.pub` 权限：`644`。
- 未发现 `authorized_keys`。
- SSH 代理当前没有加载身份。
- GitHub ED25519 主机指纹与 `known_hosts` 匹配。
- GitHub 身份认证成功，当前账号为 `infernal-affairs`。
- 私钥当前没有口令。建议后续评估加口令；本次未擅自修改密钥。
- Git 远程当前使用 HTTPS；本次推送通过 SSH 临时通道完成，未修改远程 URL。
- 三个仓库当前均无未提交改动，本地 `main` 与远程一致。

> 本次没有输出私钥内容，也没有修改 GitHub 账号中的密钥。
