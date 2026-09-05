# XopProtector 加固仓库（GPG 对称加密）

对 **大 APK（如 0.5G）** 用 **GPG 对称加密**（`gpg -c` + 自定义口令，比非对称快得多）导入导出。

```
加密的 APK(.gpg) ──▶ Actions 用 Secret 里的口令解密
                    ──▶ XopProtector 加固（DEX加密+壳+profile）
                    ──▶ zipalign + apksigner 重签（keystore 来自 Secrets）
                    ──▶ 模拟器安装+启动（崩溃则失败）
                    ──▶ 成品 APK 用同一口令加密导出(.gpg) 到 Artifacts
```

## 首次部署步骤

### 1. 推送到 GitHub

```bash
cd xop-harden
git init
git add .
git commit -m "init"
git remote add origin <你的仓库地址>
git push -u origin main
```

### 2. 配置 GitHub Secrets（6 个）

**Settings → Secrets and variables → Actions → New repository secret**：

| Name | 值 |
|------|-----|
| `KEYSTORE_BASE64` | 正式 keystore 的 base64（见 `keystore/release.keystore.b64`） |
| `KEYSTORE_PASSWORD` | keystore 密码 |
| `KEY_ALIAS` | 别名 |
| `KEY_PASSWORD` | 别名密码 |
| `GPG_PASSPHRASE` | **你的自定义 GPG 加密口令**（导入导出都用它） |
| `GPG_ARMORED_KEY` | 可选，导入非对称加密时用（本方案用对称，可不填） |

> ⚠️ keystore 与 KEYINFO.txt 已被 .gitignore 忽略，**不要提交**。

### 3. 加密并提交待加固 APK

```bash
# 用你的自定义口令对称加密 0.5G 的 APK（很快）
gpg --symmetric --cipher-algo AES256 -o apk/app.apk.gpg 你的app.apk
# 会提示输入口令——这个口令必须填到 Secrets 的 GPG_PASSPHRASE

git add apk/app.apk.gpg
git commit -m "add encrypted apk"
git push
```

### 4. 触发加固

**Actions → XopProtector 加固 → Run workflow**，选项：

- **encrypted_input**：输入是否加密 —— `true`（默认，解密 `.gpg`）或 `false`（明文 `.apk`）
- **encrypt_output**：成品是否加密导出 —— `true`（默认）或 `false`（直接下载明文 APK）
- **apk_url**：可选，加密文件或明文 APK 的下载直链（留空用 `apk/` 目录）
- **profile**：加固档位。`max` 最强（默认），`balanced` 最稳不易打不开

跑完到 **Actions → 本次运行 → Artifacts** 下载。

- 若 **encrypt_output=true**：下载 `hardened-apk-encrypted`，里面有 `final.apk.gpg`
- 若 **encrypt_output=false**：下载 `hardened-apk`，里面有 `final.apk`（明文成品）

### 5. 解密成品（如果导出时加密了）

```bash
# 用你自己的口令解密
gpg --decrypt final.apk.gpg > final.apk
```

## 常见问题

- **应用打不开/闪退**：`profile` 从 `max` 降到 `balanced`；看 emulator 步骤的 logcat。
- **解密失败（口令错误）**：确认 Secrets 的 `GPG_PASSPHRASE` 与加密时用的口令一致。
- **覆盖安装报签名冲突**：换了新 keystore，先卸载旧版再装。
- **0.5G 文件太大**：上传/下载 artifact 会较慢，属正常；模拟器冒烟测试后如需提速可设 `encrypt_output=false`。