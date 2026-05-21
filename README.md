# Windows 远程服务器挂载为本地驱动器教程

将远程服务器（GPU 云、实验室服务器、VPS 等）的文件系统通过 SFTP 挂载为 Windows 本地驱动器，方便在本地编辑器/IDE 中直接操作远程文件。

## 前置条件

- Windows 10/11
- 远程服务器已开启 SSH 服务（默认端口 22，云平台可能自定义端口）
- 已获取服务器的连接信息：地址、端口、用户名、密码（或密钥）

### 常见服务器的 SSH 连接信息来源

| 平台 | 获取方式 |
|------|---------|
| AutoDL | 控制台 → 容器实例 → SSH 指令 |
| 阿里云 / 腾讯云 / 华为云 | ECS 控制台 → 实例详情 → 公网 IP + 端口（默认22） |
| 实验室服务器 | 咨询管理员 |
| 自建 VPS | 创建时配置的 IP 和端口 |

连接信息格式：`ssh -p <端口> <用户名>@<地址>`

---

## 方案一：RaiDrive（GUI，推荐）

[RaiDrive](https://www.raidrive.com/) 免费版即可，图形界面操作，无需命令行。

### 安装

```bash
winget install --id OpenBoxLab.RaiDrive --accept-package-agreements
```

### 配置

1. 打开 RaiDrive，点击 **添加（+）**
2. 选择 **NAS → SFTP**
3. 填写连接信息：

| 字段 | 值 |
|------|-----|
| 驱动器 | `Z:`（或任意盘符） |
| 地址 | 服务器地址，如 `connect.westd.seetacloud.com` |
| 端口 | SSH 端口，如 `22`、`44776` |
| 用户名 | 如 `root`、`ubuntu` |
| 密码 | 你的密码 |
| 路径 | 需要挂载的目录，如 `/root`、`/home/user` |

4. 点击 **连接**

### 使用

- 连接成功后，指定盘符即为服务器对应目录
- 在文件资源管理器、VS Code、任意软件中均可直接读写
- 可在 RaiDrive 设置中开启 **开机自动连接**

---

## 方案二：rclone（命令行）

适合脚本自动化、Agent 配置、或不希望安装 GUI 的场景。

### 安装

```bash
winget install --id Rclone.Rclone --accept-package-agreements
```

### 配置

创建配置文件 `%APPDATA%\rclone\rclone.conf`（Windows）或 `~/.config/rclone/rclone.conf`（Linux/macOS）：

```ini
[myserver]
type = sftp
host = <服务器地址>
port = <SSH端口>
user = <用户名>
pass = <运行 rclone obscure <密码> 获取>
```

获取混淆密码（不可明文写入配置）：

```bash
rclone obscure <你的密码>
```

### 挂载

```bash
rclone mount myserver: Z: --vfs-cache-mode full --volname "Remote-Server"
```

参数说明：
- `--vfs-cache-mode full`：启用读写缓存，提升性能
- `--volname`：驱动器显示名称
- 添加 `--daemon` 可后台运行（关闭终端不会断开）

### 断开

```bash
rclone mount Z: --unmount
```

---

## 常见问题

### SSHFS-Win 方案不可用？

SSHFS-Win 与 Git for Windows 存在 `cygwin1.dll` 冲突，会导致二进制崩溃。推荐使用上述两种方案替代。

### 挂载后读写速度慢？

- rclone 方案：增大缓存 `--vfs-cache-mode full`，或设置 `--buffer-size 64M`
- RaiDrive：在设置中调整缓存大小
- 受限于网络带宽，大文件操作建议直接在服务器上进行

### 连接断开后如何恢复？

- RaiDrive：点击断开再重新连接，或重启 RaiDrive
- rclone：重新运行挂载命令

---

## 注意事项

- 挂载盘符仅在当前用户会话有效，重启后需重新挂载（RaiDrive 可设置自动连接）
- 不建议通过挂载盘运行服务器上的可执行文件，仅用于文件编辑和传输
- 服务器关机/重启后挂载会断开，需重新连接

---

## Agent 自动配置指令

如果你使用 Claude Code、Cursor、Windsurf 等 AI 编程助手，可以直接将以下内容发给 Agent，让它自动完成配置。

### 发给 Agent 的 Prompt

```
帮我将远程服务器挂载为本地驱动器，连接信息如下：

- SSH: ssh -p <端口> <用户名>@<地址>
- 密码: <密码>
- 挂载路径: <远程目录，如 /root>
- 盘符: <盘符，如 Z:>

请按以下步骤操作：

1. 安装 rclone（如果未安装）：
   winget install --id Rclone.Rclone --accept-package-agreements --accept-source-agreements

2. 找到 rclone.exe 的实际路径（winget 安装后路径不固定）

3. 用 rclone obscure 命令加密密码

4. 写入配置文件 %APPDATA%\rclone\rclone.conf：
   [myserver]
   type = sftp
   host = <地址>
   port = <端口>
   user = <用户名>
   pass = <加密后的密码>

5. 验证连接：rclone lsd myserver:

6. 挂载为本地驱动器：
   rclone mount myserver: <盘符>: --vfs-cache-mode full --volname "Remote"

7. 验证读写：ls <盘符>:/，创建并删除测试文件
```

### Agent 执行的核心命令

Agent 实际执行的命令序列如下（供参考/排查）：

```bash
# 1. 安装 rclone
winget install --id Rclone.Rclone --accept-package-agreements --accept-source-agreements

# 2. 找到 rclone 路径
find "$LOCALAPPDATA/Microsoft/WinGet/Packages" -name "rclone.exe" 2>/dev/null

# 3. 加密密码
RCLONE="<rclone路径>"
OBS=$("$RCLONE" obscure '<密码>')

# 4. 写入配置
cat > "$APPDATA/rclone/rclone.conf" << EOF
[myserver]
type = sftp
host = <地址>
port = <端口>
user = <用户名>
pass = $OBS
EOF

# 5. 验证连接
"$RCLONE" lsd myserver:

# 6. 挂载
"$RCLONE" mount myserver: Z: --vfs-cache-mode full --volname "Remote"

# 7. 验证
ls Z:/
```

### 给 Agent 的补充说明

- 如果用户同时安装了 RaiDrive（GUI 方案），需确保盘符不冲突
- rclone mount 是前台进程，关闭终端会断开；可加 `--daemon` 参数后台运行
- 配置文件路径：`%APPDATA%\rclone\rclone.conf`（Windows）或 `~/.config/rclone/rclone.conf`（Linux/macOS）
- 密码必须用 `rclone obscure` 加密后写入配置，不能明文存储
- 如果服务器使用密钥认证，将 `pass` 行替换为 `key_file = <密钥路径>`
- 可配置多个 remote（如 `[server1]`、`[server2]`），分别挂载到不同盘符
