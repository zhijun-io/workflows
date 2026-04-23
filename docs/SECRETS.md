# GitHub Secrets 配置指南

本文件说明如何配置项目 CI/CD 所需的 GitHub Secrets。

## 必需 Secrets

### 1. Maven Central 发布凭证

这些 Secrets 用于发布到 Maven Central (Sonatype)：

| Secret 名称 | 说明 | 获取方式 |
|-------------|------|----------|
| `MAVEN_USERNAME` | Sonatype Central 用户名 | [Sonatype Central 账户](https://central.sonatype.com/) |
| `MAVEN_PASSWORD` | Sonatype Central 密码 | Sonatype Central 账户设置 |

### 2. GPG 签名密钥（仅 Release）

用于签署发布到 Maven Central 的工件：

| Secret 名称 | 说明 | 获取方式 |
|-------------|------|----------|
| `GPG_SECRET_KEY` | GPG 私钥（armored 格式） | `gpg --armor --export-secret-keys your-key-id` |
| `GPG_PASSPHRASE` | GPG 私钥密码 | 创建 GPG 密钥时设置的密码 |

## 配置步骤

### 步骤 1：获取 Sonatype 账户

1. 访问 https://central.sonatype.com/
2. 注册账户或使用 JIRA 账户登录
3. 验证邮箱

### 步骤 2：生成 GPG 密钥

```bash
# 生成 GPG 密钥
gpg --full-generate-key

# 查看密钥 ID
gpg --list-secret-keys

# 导出私钥（替换 YOUR_KEY_ID）
gpg --armor --export-secret-keys YOUR_KEY_ID

# 复制输出内容（包括 -----BEGIN PGP PRIVATE KEY----- 行）
```

### 步骤 3：在 GitHub 配置 Secrets

1. 访问仓库：https://github.com/zhijunio/skillsjars-example-spring-ai/settings/secrets/actions
2. 点击 "New repository secret"
3. 添加以下 Secrets：

```
Name: MAVEN_USERNAME
Value: your-sonatype-username

Name: MAVEN_PASSWORD
Value: your-sonatype-password

Name: GPG_SECRET_KEY
Value: -----BEGIN PGP PRIVATE KEY-----
... (完整的私钥内容)
-----END PGP PRIVATE KEY-----

Name: GPG_PASSPHRASE
Value: your-gpg-passphrase
```

## 验证配置

### 测试 Snapshot 发布

```bash
# 推送到 main 分支会自动触发
git push origin main

# 或手动触发
# 访问 Actions → Publish Snapshot → Run workflow
```

### 测试 Release 发布

```bash
# 手动触发
# 访问 Actions → Release → Run workflow
# 输入版本号，例如 0.0.1
```

## 常见问题

### Q: 发布失败 "401 Unauthorized"

**原因**: MAVEN_USERNAME 或 MAVEN_PASSWORD 错误

**解决**:
1. 登录 https://central.sonatype.com/ 验证账户
2. 在 GitHub Secrets 中更新凭证

### Q: 发布失败 "gpg: signing failed: No secret key"

**原因**: GPG_SECRET_KEY 格式不正确

**解决**:
1. 确保私钥是 armored 格式（包含 BEGIN/END PGP PRIVATE KEY）
2. 确保完整复制私钥内容（包括换行）

### Q: 发布失败 "403 Forbidden"

**原因**: 账户没有发布权限

**解决**:
- 确认 Sonatype 账户已验证
- 确认项目 groupId 已注册到你的账户

### Q: 如何查看发布状态？

1. **GitHub Actions**: https://github.com/zhijunio/skillsjars-example-spring-ai/actions
2. **Maven Central**: https://central.sonatype.com/artifact/io.zhijun/skillsjars-example-spring-ai
3. **Snapshots**: https://central.sonatype.com/repository/maven-snapshots/io/zhijun/skillsjars-example-spring-ai/

## 安全提示

- ⚠️ **永远不要**将 Secrets 提交到代码库
- ⚠️ **永远不要**在日志中打印 Secrets
- ✅ 定期轮换 GPG 密钥和密码
- ✅ 使用具有最小权限的账户

## 相关链接

- [GitHub Secrets 文档](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Sonatype Central 发布指南](https://central.sonatype.org/publish/publish-guide/)
- [GPG 密钥生成指南](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key)

---

最后更新：2026-04-22
