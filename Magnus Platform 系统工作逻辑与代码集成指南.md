# Magnus Platform 系统工作逻辑与代码集成指南

## 第一部分：系统是怎么工作的？

### 1. 整体架构（用生活例子解释）

把 Magnus Platform 想象成一个 **“AI 代工厂”**：

┌─────────────────────────────────────────────────┐
│  Magnus Platform                                │
│                                                 │
│  你 (用户)                                      │
│  │                                              │
│  ├── 写好"订单" ──→ 蓝图 (模板/食谱)            │
│  │                    │                         │
│  │                    ▼                         │
│  ├── 提交"订单" ──→ 任务 (一次性的活)           │
│  │                    │                         │
│  │                    ▼                         │
│  │                 排队等待                     │
│  │                    │                         │
│  │                    ▼                         │
│  │                 集群 (工厂车间)              │
│  │                 ┌──────────────┐             │
│  │                 │ CPU GPU 内存 │             │
│  │                 │ 工人 机器 桌子│            │
│  │                 └──────────────┘             │
│  │                    │                         │
│  │                    ▼                         │
│  │                 任务完成/失败                │
│  │                                              │
│  ├── 开一个"店铺" ──→ 服务 (一直开着的店)       │
│  │                                              │
│  ├── 和同事聊天   ──→ 消息                      │
│  │                                              │
│  └── 问 AI 问题   ──→ 启新 (Explorer)           │
└─────────────────────────────────────────────────┘

### 2. 核心工作流程

> **流程 A：跑一个计算任务**
>
> - **第1步：准备“工具箱”（镜像）**  
>   你需要告诉平台：任务跑在什么软件环境里  
>   比如：PyTorch 2.5 + CUDA 12.4  
>   → 去「镜像」页面预热，平台会提前下载好
>
> - **第2步：准备“说明书”（代码）**  
>   你的代码放在 GitHub 仓库里  
>   提交任务时需要指定：哪个仓库、哪个分支、哪个版本
>
> - **第3步：下单（提交任务）**  
>   填写：任务名称、运行命令、需要几个GPU、多少内存  
>   → 平台把任务放进队列
>
> - **第4步：排队等资源**  
>   集群调度器检查：有没有空闲的 GPU？  
>   → 有空闲 → 开始准备环境  
>   → 没空闲 → 继续排队
>
> - **第5步：准备环境**  
>   平台自动：拉取镜像 → 下载代码 → 启动容器
>
> - **第6步：运行**  
>   你的代码开始跑了！你可以实时查看日志
>
> - **第7步：完成**  
>   任务跑完，查看结果

> **流程 B：发起 AI 对话（Explorer）**
>
> - 第1步：你输入问题（可以带图片）
> - 第2步：平台创建一个“会话”
> - 第3步：你的问题发给后端 AI 模型
> - 第4步：AI 的回答像打字一样一个字一个字“流”回来（流式响应）
> - 第5步：对话记录保存在会话中，可以继续追问

> **流程 C：实时聊天**
>
> - 第1步：创建会话（私聊或群聊）
> - 第2步：浏览器通过 WebSocket 和服务器保持长连接
> - 第3步：你发消息 → 服务器收到 → 通过 WebSocket 推给对方
> - 第4步：对方回消息 → 同样的路径推给你（不需要刷新页面，实时收到）

### 3. 关键技术概念（通俗版）

| 概念      | 是什么                                        | 类比                       |
| --------- | --------------------------------------------- | -------------------------- |
| Token     | 你的“通行证”，证明你是谁                      | 像门禁卡                   |
| API       | 平台提供的“服务窗口”，程序通过它和平台交互    | 像银行柜台                 |
| WebSocket | 浏览器和服务器之间的“电话线”，一直连着        | 像打电话，双方随时说话     |
| 流式响应  | AI 回答不是一次性给你，而是一个字一个字蹦出来 | 像看别人打字               |
| 镜像      | 打包好的软件环境                              | 像一台预装好软件的虚拟电脑 |
| 容器      | 用镜像启动的运行实例                          | 像用模具做出来的一个产品   |

---

## 第二部分：如何用代码调用平台资源

### 1. 第一步：获取你的 Token

Token 是你调用平台 API 的“钥匙”。获取方式：

**方式 A：从浏览器获取**  
1. 登录 Magnus Platform 网页  
2. 按 F12 打开开发者工具  
3. 切到 Console（控制台）  
4. 输入：`localStorage.getItem('magnus_token')`  
5. 复制输出的字符串，这就是你的 Token

**方式 B：从「人事」页面获取 API Token**  
1. 打开「人事」页面  
2. 点击你自己的头像  
3. 在详情面板中找到“Token”部分  
4. 点击复制（格式为 `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`）

### 2. 基本调用方式

所有 API 调用都遵循同一个模式：

```python
import requests

# 你的 Token
TOKEN = "你的token放这里"

# 平台地址
BASE_URL = "http://162.105.151.134:3011"

# 通用请求头
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}
```

### 3. 常用场景代码示例

#### 场景一：查看集群还有多少资源

> “我想知道现在还有没有空闲的 GPU”

```python
import requests

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

# 查询集群资源
resp = requests.get(f"{BASE_URL}/api/cluster/stats", headers=HEADERS)
stats = resp.json()

resources = stats["resources"]
print(f"CPU: 空闲 {resources['cpu_free']} / 总共 {resources['cpu_total']} 核")
print(f"内存: 空闲 {resources['mem_free_mb']}MB / 总共 {resources['mem_total_mb']}MB")
print(f"GPU ({resources['gpu_model']}): 空闲 {resources['free']} / 总共 {resources['total']} 张")
print(f"正在运行: {stats['total_running']} 个任务")
print(f"排队中: {stats['total_pending']} 个任务")
```

#### 场景二：提交一个训练任务

> “我想用代码自动提交一个 GPU 训练任务”

```python
import requests

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

# 提交任务
job_config = {
    "task_name": "我的训练任务",          # 任务名称
    "description": "测试用的小任务",        # 描述（可选）
    "namespace": "Rise-AGI",            # GitHub 组织名
    "repo_name": "my-project",          # GitHub 仓库名
    "branch": "main",                   # 分支
    "commit_sha": "HEAD",               # 用最新代码
    "entry_command": "python train.py", # 运行什么命令
    "gpu_count": 1,                     # 需要几个 GPU
    "gpu_type": "a100",                 # GPU 型号
    "job_type": "B2",                   # 优先级（见下方说明）
    "cpu_count": 4,                     # CPU 核数
    "memory_demand": "8G",              # 内存
    "ephemeral_storage": "10G",         # 临时存储
    "container_image": "docker://pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime",
    "runner": "magnus",
}

resp = requests.post(f"{BASE_URL}/api/jobs/submit", headers=HEADERS, json=job_config)

if resp.status_code == 200:
    job = resp.json()
    print(f"任务提交成功！任务 ID: {job['id']}")
else:
    print(f"提交失败: {resp.text}")
```

**任务优先级说明：**

| 类型 | 含义 | 会被抢占吗？ |
|------|------|--------------|
| A1 | 最高优先级 | 不会 |
| A2 | 高优先级 | 不会 |
| B1 | 普通优先级 | 可能被 A 类抢占 |
| B2 | 低优先级 | 可能被抢占 |

#### 场景三：查看任务状态和日志

> “任务提交后，我想用代码查看它跑到哪了”

```python
import requests
import time

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}
JOB_ID = "你的任务ID"

# 轮询任务状态
while True:
    resp = requests.get(f"{BASE_URL}/api/jobs/{JOB_ID}", headers=HEADERS)
    job = resp.json()
    status = job["status"]
    print(f"任务状态: {status}")
    
    if status in ["completed", "failed", "terminated"]:
        print("任务结束了！")
        break
        
    time.sleep(10)  # 每 10 秒查一次

# 查看日志
resp = requests.get(f"{BASE_URL}/api/jobs/{JOB_ID}/logs?page=1", headers=HEADERS)
logs = resp.json()
print(f"日志内容:\n{logs['logs']}")
print(f"总共 {logs['total_pages']} 页日志")
```

#### 场景四：列出我的所有任务

```python
import requests

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

# 获取任务列表（前 20 个）
resp = requests.get(
    f"{BASE_URL}/api/jobs",
    headers=HEADERS,
    params={"skip": 0, "limit": 20, "all_users": False}
)
data = resp.json()

print(f"共 {data['total']} 个任务\n")
for job in data["items"]:
    print(f" [{job['status']}] {job['task_name']} (ID: {job['id']})")
```

#### 场景五：终止一个任务

> “任务跑飞了，我想用代码紧急停止它”

```python
import requests

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}
JOB_ID = "要终止的任务ID"

resp = requests.post(f"{BASE_URL}/api/jobs/{JOB_ID}/terminate", headers=HEADERS)

if resp.status_code == 200:
    print("任务已终止")
else:
    print(f"终止失败: {resp.text}")
```

#### 场景六：和 AI 对话（Explorer）

> “我想在代码里调用平台的 AI 进行对话”

```python
import requests

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

# 第1步：创建一个会话
resp = requests.post(
    f"{BASE_URL}/api/explorer/sessions",
    headers=HEADERS,
    json={"title": "我的AI对话"}
)
session = resp.json()
session_id = session["id"]
print(f"会话创建成功，ID: {session_id}")

# 第2步：发送消息并接收流式回答
resp = requests.post(
    f"{BASE_URL}/api/explorer/sessions/{session_id}/chat",
    headers=HEADERS,
    json={"content": "请用一句话介绍什么是机器学习"},
    stream=True  # 关键：开启流式接收
)

# 第3步：逐块读取 AI 的回答
print("AI 回答: ", end="")
for chunk in resp.iter_content(chunk_size=None, decode_unicode=True):
    if chunk:
        print(chunk, end="", flush=True)
print() # 换行
```

#### 场景七：运行一个蓝图

> “我有一个配好的蓝图模板，想用代码填参数并运行”

```python
import requests

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}
BLUEPRINT_ID = "蓝图ID"

# 第1步：获取蓝图的参数定义（看看需要填什么）
resp = requests.get(f"{BASE_URL}/api/blueprints/{BLUEPRINT_ID}/schema", headers=HEADERS)
schema = resp.json()
print("需要填写的参数:")
for field in schema:
    print(f" - {field['key']}")

# 第2步：填写参数并运行
resp = requests.post(
    f"{BASE_URL}/api/blueprints/{BLUEPRINT_ID}/run",
    headers=HEADERS,
    json={
        "parameters": {
            "learning_rate": 0.001,  # 根据 schema 填写
            "epochs": 10,
            "batch_size": 32,
        },
        "use_preference": False,
        "save_preference": True  # 保存这次的参数，下次自动填充
    }
)

if resp.status_code == 200:
    print("蓝图运行成功，任务已创建！")
else:
    print(f"运行失败: {resp.text}")
```

#### 场景八：批量自动化（实用脚本）

> “我想自动检查资源 → 有空闲 GPU 就提交任务”

```python
import requests
import time

TOKEN = "你的token"
BASE_URL = "http://162.105.151.134:3011"
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

def check_gpu():
    """检查是否有空闲 GPU"""
    resp = requests.get(f"{BASE_URL}/api/cluster/stats", headers=HEADERS)
    resources = resp.json()["resources"]
    return resources["free"]  # 返回空闲 GPU 数量

def submit_job(name, command):
    """提交一个任务"""
    job = {
        "task_name": name,
        "namespace": "Rise-AGI",
        "repo_name": "my-project",
        "branch": "main",
        "commit_sha": "HEAD",
        "entry_command": command,
        "gpu_count": 1,
        "gpu_type": "a100",
        "job_type": "B2",
        "container_image": "docker://pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime",
        "runner": "magnus",
    }
    resp = requests.post(f"{BASE_URL}/api/jobs/submit", headers=HEADERS, json=job)
    return resp.json()

# 等待有空闲 GPU 再提交
experiments = [
    ("实验1-lr0.01", "python train.py --lr 0.01"),
    ("实验2-lr0.001", "python train.py --lr 0.001"),
    ("实验3-lr0.0001", "python train.py --lr 0.0001"),
]

for name, cmd in experiments:
    # 等待有空闲 GPU
    while check_gpu() < 1:
        print("没有空闲 GPU，等待 30 秒...")
        time.sleep(30)
        
    job = submit_job(name, cmd)
    print(f"已提交: {name} (ID: {job['id']})")
    time.sleep(2)  # 稍等一下再提交下一个

print("所有实验已提交！")
```

### 4. 完整 API 速查表

| 功能 | 方法 | 接口 | 说明 |
|------|------|------|------|
| 集群资源 | GET | `/api/cluster/stats` | 查看CPU/GPU/内存使用情况 |
| 我的活跃任务 | GET | `/api/cluster/my-active-jobs` | 当前用户运行中的任务 |
| 任务列表 | GET | `/api/jobs?skip=0&limit=20` | 分页查看任务 |
| 提交任务 | POST | `/api/jobs/submit` | 创建新任务 |
| 任务详情 | GET | `/api/jobs/{id}` | 查看单个任务 |
| 任务日志 | GET | `/api/jobs/{id}/logs?page=1` | 查看运行日志 |
| 终止任务 | POST | `/api/jobs/{id}/terminate` | 停止任务 |
| 蓝图列表 | GET | `/api/blueprints` | 查看所有蓝图 |
| 蓝图参数 | GET | `/api/blueprints/{id}/schema` | 获取蓝图的参数定义 |
| 运行蓝图 | POST | `/api/blueprints/{id}/run` | 填参数运行蓝图 |
| 服务列表 | GET | `/api/services` | 查看所有服务 |
| 创建/更新服务 | POST | `/api/services` | 创建或修改服务 |
| 删除服务 | DELETE | `/api/services/{id}` | 删除服务 |
| 技能列表 | GET | `/api/skills` | 查看所有技能 |
| 镜像列表 | GET | `/api/images` | 查看所有镜像 |
| 预热镜像 | POST | `/api/images` | 注册新镜像 `{uri}` |
| 用户列表 | GET | `/api/users` | 获取所有用户 |
| 用户花名册 | GET | `/api/users/roster` | 分页搜索用户 |
| AI对话-创建会话 | POST | `/api/explorer/sessions` | 创建Explorer会话 |
| AI对话-发消息 | POST | `/api/explorer/sessions/{id}/chat` | 发送消息（流式返回） |
| AI对话-上传文件 | POST | `/api/explorer/sessions/{id}/upload` | 上传文件给AI |
| 聊天-会话列表 | GET | `/api/conversations` | 查看聊天会话 |
| 聊天-发消息 | POST | `/api/conversations/{id}/messages` | 发送聊天消息 |
| 聊天-WebSocket | WS | `/ws/chat?token=xxx` | 实时聊天连接 |
| GitHub分支 | GET | `/api/github/{ns}/{repo}/branches` | 列出仓库分支 |
| GitHub提交 | GET | `/api/github/{ns}/{repo}/commits?branch=xxx` | 列出分支提交 |

### 5. 注意事项

1. **Token 安全**：Token 相当于你的密码，不要写在公开代码里。建议放在环境变量中：  
   
   ```python
   import os
   TOKEN = os.environ.get("MAGNUS_TOKEN")
   ```
   
2. **错误处理**：如果收到 `401` 响应，说明 Token 过期或无效，需要重新获取。

3. **不要频繁请求**：集群资源查询建议间隔 5 秒以上，避免给平台造成压力。

4. **任务资源**：提交任务时注意 GPU 数量不要超过集群总量，内存也别要太多，否则可能永远排不上队。

5. **流式响应**：调用 Explorer AI 对话时，一定要用 `stream=True` ，否则需要等 AI 全部说完才能收到回答。

---

*本指南基于 Magnus Platform v0.1.0 编写 Rise AGI*