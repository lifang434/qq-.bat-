# qq-.bat-
可用来获取密钥和自动解密，并将聊天记录转化成可用来训练的.json文件
> **English abstract** — A local, offline toolkit for reading your own NTQQ (QQ NT / QQ 9.x) chat databases on Windows: extract the SQLCipher key, decrypt `nt_msg.db`, export conversations to TXT, and turn them into fine-tuning datasets. Pure Python standard library — no third-party dependencies.

---

# ntqq-chat-export

把自己的 QQ（NTQQ / QQ 9.x）聊天记录从加密数据库里导出来，转成可读文本或微调数据集。

**全程本地，不联网，不上传任何数据。**

```
加密的 nt_msg.db
      │  ① 从内存抓 SQLCipher 密钥（登录瞬间）
      ▼
  明文 SQLite
      │  ② 按会话提取 + 解 Protobuf
      ▼
 聊天记录.txt / conversations.jsonl
      │  ③ 切段、合并回合
      ▼
 训练集（QLoRA / 其他）
```

---

## ⚠️ 先读这一段

- **只能用于你自己账号的数据。** 用它读别人的聊天记录，在绝大多数地方都是违法的。
- **密钥和数据库同等敏感。** 拿到密钥等于拿到整个本地数据库的钥匙，不要提交到仓库、不要发群里、不要贴 issue。
- **导出结果是明文**。整个流程结束后，`nt_msg_plain.db` 就是一份没有加密的私聊库，请自己决定什么时候删。
- 本项目**不修改**你的任何原始数据：只读、只复制。

---

## 它能做什么 / 不能做什么

| 能力 | 状态 |
|---|---|
| 解密 `nt_msg.db` / `profile_info.db` / `group_info.db` 等 NTQQ 数据库 | ✅ |
| 从已解密的库里按会话（好友 / 群）导出文字 | ✅ |
| 解析消息体里的 Protobuf，拿到纯文本 | ✅ |
| 生成 Alpaca / chat 格式的微调数据集 | ✅ |
| 直接打开 QQ 的 `.bak` 备份文件 | ❌ 那是另一种容器格式，得先导回 QQ |
| 恢复已删除的消息 | ❌ 不在库里就是不在 |
| Linux / macOS | ❌ 目前只有 Windows 的取密钥脚本 |

---

## 原理（够看懂就行）

NTQQ 把聊天记录存在 SQLCipher 4 加密的 SQLite 里，文件开头还有一段 **1024 字节的自定义头**（伪装的 `SQLite header 3` + `QQ_NT DB` 标记）。

实测参数（QQ 9.9.36）：

| 参数 | 值 |
|---|---|
| 文件头 | 1024 字节，解密前必须剥掉 |
| `cipher_page_size` | 4096 |
| 加密 | AES-256-CBC |
| KDF | PBKDF2-HMAC-SHA512，**4000 次** |
| 每页保留区 | 48 字节 = IV(16) + HMAC-SHA1(20) + 补齐(12) |
| 明文页的 `reserved bytes` | **80**（`usable = 4016`） |

> 最后一行的 80 很容易踩坑：如果你自己写解密器把它填成 48，SQLite 会报 `database disk image is malformed`。

**密钥是 16 个字符的 ASCII 字符串，登录时由服务器下发，不落盘、不常驻内存。** 所以只能在 QQ 登录、调用 `nt_sqlite3_key_v2` 打开数据库的那一瞬间去拿——扫描内存是没用的（这个项目早期试过 290 万个候选，全灭）。

---

## 快速开始（Windows）

需要：Python 3.9+（只用标准库）、能登录的 QQ。

### ① 取密钥

先**完全退出 QQ**（托盘退出，任务管理器里确认没有 `QQ.exe`），然后在 PowerShell 里：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
irm https://raw.githubusercontent.com/QQBackup/qq-win-db-key/master/scripts/windows/ntqq/windows_ntqq_get_key.ps1 | iex
```

脚本会**自己启动 QQ 并挂调试器**，你正常登录，它会在 `nt_sqlite3_key_v2` 断点处读出密钥：

```
========================================
找到密钥: xxxxxxxxxxxxxxxx
========================================
```

> 拿到 20 个字符的 key 是 `login.db` 的（固定值），16 个字符的才是 `nt_msg.db` 的。
> 本仓库 `scripts/get_key.ps1` 放了一份离线副本，网络不通时可以用：
> `powershell -ExecutionPolicy Bypass -File scripts\get_key.ps1`

### ② 解密

先**退出 QQ**（避免它一边写你一边读）：

```powershell
python decrypt_ntdb.py --db "D:\...\nt_qq\nt_db\nt_msg.db" --key <你的16位密钥> --out nt_msg_plain.db
```

产出：

- `nt_msg_plain.db` —— 明文 SQLite，任意工具都能打开
- 顺带解密 `profile_info.db` 就能查到好友昵称

### ③ 看看有哪些会话

```powershell
python extract_chat.py --db nt_msg_plain.db --list
```

输出每个会话的对象号和消息条数。也可以看表结构：

```powershell
python extract_chat.py --db nt_msg_plain.db --schema
```

### ④ 导出 / 生成训练集

单会话导成纯文本：

```powershell
python extract_chat.py --db nt_msg_plain.db --peer 123456789 --self-uin 987654321 --out chat.txt
```

直接生成三件套（可读文本 + 多轮对话 jsonl + 单轮问答 jsonl）：

```powershell
python build_dataset.py --db nt_msg_plain.db --peer 123456789 --self-uin 987654321 ^
  --peer-name 对方 --self-name 我 --outdir out
```

`out/` 里会有：

| 文件 | 内容 |
|---|---|
| `聊天记录.txt` | `[时间] 我/对方: 内容` |
| `conversations.jsonl` | `{"messages":[{"role":"user",...}]}`，多轮 |
| `dataset_alpaca.jsonl` | `{"instruction","input","output"}`，单轮 |
| `stats.txt` | 统计 |

角色约定：**对方说的话 = `user`，你自己说的话 = `assistant`**。想学谁的语气就换 `--self-uin`。

---

## 自检：不碰自己数据也能验证解密器

`login.db` 的密钥是**硬编码在客户端里的公开固定值**（[QQDecrypt 文档](https://qqbackup.github.io/QQDecrypt/database/db_file_analysis/login/index.html)）：

```
BD156D6710D54D8782F4
```

拿它跑一遍，能解出 `login_table` 就说明你的环境和算法都对：

```powershell
python decrypt_ntdb.py --db "C:\Users\<你>\Documents\Tencent Files\nt_qq\global\nt_db\login.db" ^
  --key BD156D6710D54D8782F4 --out login_plain.db
python -c "import sqlite3;print(sqlite3.connect('login_plain.db').execute(\"select name from sqlite_master where type='table'\").fetchall())"
```

---

## 参数说明

### `decrypt_ntdb.py`

| 参数 | 说明 |
|---|---|
| `--db` | 加密的 NTQQ 数据库 |
| `--key` | 16 字符密钥（`login.db` 用 20 字符的固定值） |
| `--out` | 输出明文库 |
| `--pages N` | 只解前 N 页，用来快速验证密钥 |

### `extract_chat.py`

| 参数 | 说明 |
|---|---|
| `--list` | 列出所有会话和消息数 |
| `--schema` | 打印表名和列名 |
| `--peer` | 会话对象：好友 QQ 号 / 群号 |
| `--self-uin` | 你自己的 QQ 号（用来区分"我"和"对方"） |
| `--limit N` | 只取最近 N 条 |

### `build_dataset.py`

| 参数 | 默认 | 说明 |
|---|---|---|
| `--gap` | 30 | 两条消息间隔超过多少分钟就切成新对话 |
| `--min-chars` | 2 | 短于这个长度的消息丢掉（滤掉"嗯""哦"） |
| `--max-turns` | 40 | 单段对话最多保留多少回合 |

---

## 数据库里都是些什么

NTQQ 把表名和列名都换成了数字，所以直接看库会一脸懵：

| 表 | 含义 |
|---|---|
| `c2c_msg_table` | 私聊 |
| `group_msg_table` | 群聊 |
| `profile_info_v6` / `buddy_list` | 好友资料、昵称 |

常用列（`c2c_msg_table`）：

| 列 | 含义 |
|---|---|
| `40001` | 消息 ID |
| `40013` | 方向（0=收到 / 2=发出） |
| `40030` | 会话对象 QQ 号 |
| `40033` | 发送者 QQ 号 |
| `40050` | Unix 时间戳 |
| `40800` | 消息体（Protobuf，文本在 `45101`） |

更完整的字段分析见 [QQDecrypt 的文档](https://qqbackup.github.io/QQDecrypt/database/db_file_analysis/nt_msg/c2c_msg_table.html)。

---

## 常见问题

**Q：`ollama create` 或者别的工具说 `MLX runtime is not available`？**
A：那是 Ollama 0.40+ 在 Windows 上导入 safetensors 时只认 Apple 的 MLX 后端。解法是先转 GGUF，再导入。本项目不涉及推理，但如果你要把导出的数据拿去微调，会撞上这个。

**Q：pip 装东西卡住、CPU 一直 100%？**
A：多半是 pip 缓存目录坏了。加 `--no-cache-dir`。

**Q：`FileNotFoundError: No usable temporary directory`？**
A：TEMP 目录权限有问题。把 `$env:TEMP` 指到别的盘再跑。

**Q：`--quantize q4_K_M` 报不支持？**
A：新版 Ollama 只认 `int4/int8/nvfp4/mxfp4/mxfp8`。GGUF 要先自己用 `llama-quantize` 压好。

**Q：扫内存能找到密钥吗？**
A：不能。密钥只在登录那一瞬间存在，用完就没了。`extras/find_key_by_scan.py` 留着当反例，别指望它。

**Q：解出来的库 SQLite 打不开 / 报 malformed？**
A：十有八九是明文页的 `reserved bytes` 填错了，必须是 80。

---

## 目录结构

```
ntqq-chat-export/
├── decrypt_ntdb.py          # SQLCipher 解密器（纯 Python AES）
├── extract_chat.py          # 按会话提取 + Protobuf 解文本
├── build_dataset.py         # 生成训练集
├── pureaes.py               # AES-256-CBC 实现（已用 FIPS-197 向量验证）
├── scripts/
│   └── get_key.ps1          # 取密钥脚本（第三方，见 THIRD_PARTY.md）
└── extras/
    └── find_key_by_scan.py  # 内存扫描版（通常无效，留作参考）
```

只依赖 Python 标准库——没有 `pycryptodome`，也没有 `sqlcipher3`。

---

## 致谢

- [QQBackup/qq-win-db-key](https://github.com/QQBackup/qq-win-db-key) —— 取密钥脚本，MIT
- [QQBackup/nt_msg_db_util](https://github.com/QQBackup/nt_msg_db_util) —— 数据库结构与字段分析
- [QQDecrypt 文档](https://qqbackup.github.io/QQDecrypt/) —— 加密参数与表结构参考
- [NapNeko/NapCatQQ](https://github.com/NapNeko/NapCatQQ) —— SQLCipher 参数常量

## License

MIT，见 [LICENSE](LICENSE)。第三方代码见 [THIRD_PARTY.md](THIRD_PARTY.md)。
