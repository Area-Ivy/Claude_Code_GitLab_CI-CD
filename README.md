# Claude_Code_GitLab_CI-CD

## 背景

代码审查是保障代码质量的关键环节，但人工 Review 耗时且容易遗漏。Anthropic 推出的 [Claude Code](https://code.claude.com/) CLI 工具提供了 `--print` 非交互模式，可以在 CI 中自动对代码变更进行审查。

更进一步，我们希望在 GitLab 的 Issue 和 MR 中直接 `@claude` 来让 AI 分析问题、实现功能、修复 Bug——就像与一个 AI 队友协作一样。

本文将介绍两大能力的完整实现：

1. **Push 触发的自动代码审查**：每次提交/MR 自动审查代码变更
2. **`@claude` 交互式 AI 助手**：在 Issue/MR 评论中 `@claude` 触发 AI 执行任务

---

## 一、架构概览

```
Push / MR 事件
    │
    ▼
┌─────────────────────────┐
│ claude-code-review Job  │  ← Stage: review
│ 自动审查代码 diff       │
└─────────────────────────┘

用户在 Issue/MR 评论 @claude
    │
    ▼
┌─────────────────────────┐
│ Webhook Listener        │  ← Flask + Docker（独立服务）
│ 解析 @claude 指令       │
│ 获取 Issue/MR 上下文    │
│ 触发 Pipeline           │
│ 回复"收到，正在处理..." │
└─────────────┬───────────┘
              │ Pipeline Trigger API
              ▼
┌─────────────────────────┐
│ claude-on-issue Job     │  ← Stage: ai
│ Claude Code 执行任务    │
│ MCP Server → GitLab API │
│ 创建分支 / 提交 / MR   │
│ 回复评论                │
└─────────────────────────┘
```

---

## 二、前置条件

- GitLab Runner（Docker executor）
- Anthropic API Key（或兼容的代理地址）
- Node.js 20+ 镜像（代码审查用 `node:20`，AI 交互用 `node:24`）
- 一台可被 GitLab 访问的服务器（部署 Webhook Listener）

### 2.1 需要配置的 GitLab CI/CD Variables

在 GitLab 项目的 **Settings → CI/CD → Variables** 中添加（每个需要使用的项目都要配置）：

| 变量名                | 说明                                                       | Protected |
| --------------------- | ---------------------------------------------------------- | --------- |
| `ANTHROPIC_API_KEY`   | Anthropic API 密钥                                         | ❌ 关闭    |
| `ANTHROPIC_BASE_URL`  | API 地址（如使用代理）                                     | ❌ 关闭    |
| `GITLAB_ACCESS_TOKEN` | Personal Access Token（scope: `api` + `write_repository`） | ❌ 关闭    |
| `SKILLS_REPO_URL`     | Skill 仓库地址（可选）                                     | -         |
| `SKILLS_REPO_TOKEN`   | Skill 仓库访问 Token（可选）                               | -         |

> **重要**：`GITLAB_ACCESS_TOKEN` 的 Protected 标志**必须关闭**！因为通过 Trigger API 触发的 Pipeline 可能不被视为在 Protected branch 上运行，Protected 变量会导致 Job 中拿不到 Token 值。

---

## 三、Push 触发的自动代码审查

### 3.1 基础 `.gitlab-ci.yml`

```yaml
stages:
  - review

claude-code-review:
  stage: review
  tags:
    - claude
    - code-review
  image: node:20

  variables:
    ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
    ANTHROPIC_BASE_URL: $ANTHROPIC_BASE_URL
    CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS: "1"

  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

  cache:
    key: claude-cli
    paths:
      - /root/.npm

  before_script:
    - npm install -g @anthropic-ai/claude-code

    # 从外部 Git 仓库拉取 Skill（可选）
    - |
      mkdir -p /root/.claude/skills

      if [ -n "$SKILLS_REPO_URL" ]; then
        echo "📦 从外部仓库拉取 skills..."
        BRANCH="${SKILLS_REPO_BRANCH:-main}"

        if [ -n "$SKILLS_REPO_TOKEN" ]; then
          CLONE_URL=$(echo "$SKILLS_REPO_URL" | sed "s|https://|https://gitlab-ci-token:${SKILLS_REPO_TOKEN}@|")
        else
          CLONE_URL="$SKILLS_REPO_URL"
        fi

        git clone --depth=1 --branch "$BRANCH" "$CLONE_URL" /tmp/skills-repo 2>&1 && {
          find /tmp/skills-repo -maxdepth 2 -name "SKILL.md" -printf '%h\n' | while read skill_dir; do
            skill_name=$(basename "$skill_dir")
            mkdir -p /root/.claude/skills/"$skill_name"
            cp -r "$skill_dir"/* /root/.claude/skills/"$skill_name"/
            echo "  ✓ 已加载 skill: $skill_name"
          done
        } || echo "⚠️ 克隆外部 skills 仓库失败"
      fi

      echo "📋 /root/.claude/skills/ 中的 skills："
      find /root/.claude/skills -name "SKILL.md" -exec echo "  ✓ {}" \; 2>/dev/null || echo "  （无）"

  script:
    - |
      if [ "$CI_COMMIT_BEFORE_SHA" = "0000000000000000000000000000000000000000" ]; then
        git fetch origin $CI_DEFAULT_BRANCH
        CHANGED_FILES=$(git diff --name-only origin/$CI_DEFAULT_BRANCH...HEAD)
        DIFF_CONTENT=$(git diff origin/$CI_DEFAULT_BRANCH...HEAD)
      else
        CHANGED_FILES=$(git diff --name-only $CI_COMMIT_BEFORE_SHA...$CI_COMMIT_SHA)
        DIFF_CONTENT=$(git diff $CI_COMMIT_BEFORE_SHA...$CI_COMMIT_SHA)
      fi

    - echo "变更的文件：$CHANGED_FILES"

    - |
      if [ -z "$CHANGED_FILES" ]; then
        echo "没有代码变更，跳过审查"
        exit 0
      fi

    - |
      claude --print "
      请对以下代码变更进行代码审查：

      提交信息：$CI_COMMIT_MESSAGE
      提交者：$CI_COMMIT_AUTHOR
      分支：$CI_COMMIT_BRANCH

      变更文件列表：
      $CHANGED_FILES

      代码变更内容：
      $DIFF_CONTENT
      " | tee review_result.md

  artifacts:
    paths:
      - review_result.md
    expire_in: 1 week
```

这个基础版已经可以工作了——每次 MR 会触发 Claude 对 diff 内容进行审查，结果保存为 artifact。

**但问题是**：审查规则写死在 prompt 里，不够灵活，也无法跨项目复用。这就引出了 Skill 的概念（参见后面的 Skill 章节）。

---

## 四、`@claude` 交互式 AI 助手

这是本文的核心——在 GitLab Issue 或 MR 的评论中写 `@claude 帮我分析一下这个问题` 或 `@claude 实现这个功能`，Claude 就会自动执行任务。

### 4.1 工作流程

1. 用户在 Issue/MR 评论中写 `@claude <指令>`
2. GitLab 发送 Webhook（Note Hook）到我们的 Webhook Listener
3. Webhook Listener 解析 `@claude` 指令，获取 Issue/MR 上下文
4. 通过 GitLab Pipeline Trigger API 触发 `claude-on-issue` Job
5. CI Job 中 Claude Code 执行任务（分析代码、创建分支、提交代码、创建 MR、回复评论等）

### 4.2 `.gitlab-ci.yml` 中的 `claude-on-issue` Job

```yaml
stages:
  - review
  - ai

# ... (claude-code-review 保持不变) ...

# ============================================================
# @claude Issue / MR 交互作业
# 触发方式：Webhook 监听 @claude 评论后通过 Pipeline Trigger API 调用
# ============================================================
claude-on-issue:
  stage: ai
  tags:
    - claude
    - code-review

  image: node:24

  rules:
    - if: '$CI_PIPELINE_SOURCE == "web"'
    - if: '$CI_PIPELINE_SOURCE == "trigger"'
    - if: '$CI_PIPELINE_SOURCE == "api"'

  variables:
    ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
    ANTHROPIC_BASE_URL: $ANTHROPIC_BASE_URL
    GIT_STRATEGY: fetch

  before_script:
    - npm install -g @anthropic-ai/claude-code

    # 配置 Git 推送权限
    - |
      if [ -n "$GITLAB_ACCESS_TOKEN" ]; then
        git config --global user.email "claude-bot@example.com"
        git config --global user.name "Claude Bot"
        REPO_URL=$(echo "$CI_REPOSITORY_URL" | sed "s|.*@|https://oauth2:${GITLAB_ACCESS_TOKEN}@|" | sed "s|\.git$||").git
        git remote set-url origin "$REPO_URL"
        echo "✅ Git 推送权限已配置"

        # 验证 Token 是否有效
        echo "🔍 验证 GitLab Access Token 权限..."
        HTTP_CODE=$(curl -s -o /tmp/token_check.json -w "%{http_code}" \
          --header "PRIVATE-TOKEN: $GITLAB_ACCESS_TOKEN" \
          "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}")
        if [ "$HTTP_CODE" = "200" ]; then
          PROJECT_NAME=$(cat /tmp/token_check.json | python3 -c "import sys,json; print(json.load(sys.stdin).get('path_with_namespace','unknown'))" 2>/dev/null || echo "unknown")
          echo "✅ Token 有效，项目: $PROJECT_NAME"
        else
          echo "❌ Token 验证失败！HTTP $HTTP_CODE"
          echo "   响应内容: $(cat /tmp/token_check.json 2>/dev/null)"
          echo "   请检查："
          echo "   1. GITLAB_ACCESS_TOKEN 是否已在项目 CI/CD Variables 中配置"
          echo "   2. Protected 标志是否已关闭"
          echo "   3. Token 是否有 api + write_repository scope"
          echo "   4. Token 对应用户是否对此项目有 Developer 以上权限"
        fi
        rm -f /tmp/token_check.json
      else
        echo "⚠️ 未设置 GITLAB_ACCESS_TOKEN，Claude 将无法推送代码或调用 GitLab API"
        echo "   请在项目 CI/CD > Variables 中添加 GITLAB_ACCESS_TOKEN"
      fi

    # 配置 GitLab MCP Server
    - |
      echo "🔧 配置 GitLab MCP Server..."
      mkdir -p /root/.claude
      cat > /root/.claude/settings.json << 'MCPEOF'
      {
        "mcpServers": {
          "gitlab": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-gitlab"],
            "env": {
              "GITLAB_PERSONAL_ACCESS_TOKEN": "${GITLAB_ACCESS_TOKEN}",
              "GITLAB_API_URL": "${CI_API_V4_URL}"
            }
          }
        }
      }
      MCPEOF
      sed -i "s|\${GITLAB_ACCESS_TOKEN}|${GITLAB_ACCESS_TOKEN}|g" /root/.claude/settings.json
      sed -i "s|\${CI_API_V4_URL}|${CI_API_V4_URL}|g" /root/.claude/settings.json
      echo "✅ MCP Server 配置完成"

    # Skill 集成（同 claude-code-review）
    - |
      mkdir -p /root/.claude/skills
      if [ -n "$SKILLS_REPO_URL" ]; then
        echo "📦 从外部仓库拉取 skills..."
        BRANCH="${SKILLS_REPO_BRANCH:-main}"
        if [ -n "$SKILLS_REPO_TOKEN" ]; then
          CLONE_URL=$(echo "$SKILLS_REPO_URL" | sed "s|https://|https://gitlab-ci-token:${SKILLS_REPO_TOKEN}@|")
        else
          CLONE_URL="$SKILLS_REPO_URL"
        fi
        git clone --depth=1 --branch "$BRANCH" "$CLONE_URL" /tmp/skills-repo 2>&1 && {
          find /tmp/skills-repo -maxdepth 2 -name "SKILL.md" -printf '%h\n' | while read skill_dir; do
            skill_name=$(basename "$skill_dir")
            mkdir -p /root/.claude/skills/"$skill_name"
            cp -r "$skill_dir"/* /root/.claude/skills/"$skill_name"/
            echo "  ✓ 已加载 skill: $skill_name"
          done
        } || echo "⚠️ 克隆外部 skills 仓库失败"
      fi
      echo "📋 /root/.claude/skills/ 中的 skills："
      find /root/.claude/skills -name "SKILL.md" -exec echo "  ✓ {}" \; 2>/dev/null || echo "  （无）"

  script:
    # Token 诊断
    - |
      if [ -n "$GITLAB_ACCESS_TOKEN" ]; then
        echo "🔑 GITLAB_ACCESS_TOKEN 前缀: $(echo $GITLAB_ACCESS_TOKEN | cut -c1-8)..."
      else
        echo "❌ GITLAB_ACCESS_TOKEN 环境变量为空！Claude 将无法调用 GitLab API"
      fi
    # 打印触发上下文
    - echo "AI_FLOW_EVENT=$AI_FLOW_EVENT AI_FLOW_CONTEXT=$AI_FLOW_CONTEXT"
    - >
      claude
      -p "${AI_FLOW_INPUT:-'Review this MR and implement the requested changes'}"
      --permission-mode acceptEdits
      --allowedTools "Bash Read Edit Write mcp__gitlab"
      --debug
```

**关键说明：**

- `image: node:24`：必须使用 Debian 基础的镜像（不能用 Alpine），否则 Claude Code 二进制会因 musl 兼容性问题报 `posix_getdents: symbol not found`
- `rules`：只允许 `web`、`trigger`、`api` 三种触发方式，**不要加 `merge_request_event`**，否则 Claude 自己创建的 MR 会触发死循环
- `GIT_STRATEGY: fetch`：复用 Runner 缓存的仓库，加速构建
- `mcp__gitlab`：通过 MCP Server 让 Claude 直接调用 GitLab API（创建 MR、发评论等）

### 4.3 GitLab MCP Server 配置详解

MCP（Model Context Protocol）Server 是让 Claude 具备 GitLab API 操作能力的关键。**它不是 `@anthropic-ai/claude-code` 自带的**，而是独立的 npm 包 `@modelcontextprotocol/server-gitlab`。

配置方式是在 CI Job 的 `before_script` 中创建 `~/.claude/settings.json`：

```json
{
  "mcpServers": {
    "gitlab": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-gitlab"],
      "env": {
        "GITLAB_PERSONAL_ACCESS_TOKEN": "你的Token值",
        "GITLAB_API_URL": "https://your-gitlab.com/api/v4"
      }
    }
  }
}
```

**工作原理：**

1. `before_script` 中创建 `/root/.claude/settings.json`，通过 `sed` 用实际环境变量替换占位符
2. Claude Code 启动时自动读取 settings 文件
3. `npx -y @modelcontextprotocol/server-gitlab` 自动下载并启动 MCP Server
4. Claude 获得 `mcp__gitlab` 系列工具，可以创建 MR、发评论、管理 Issue 等

**注意事项：**

- `CI_API_V4_URL` 是 GitLab 自动提供的预定义变量，无需手动配置
- `GITLAB_PERSONAL_ACCESS_TOKEN` 的值来自 CI/CD Variables 中的 `GITLAB_ACCESS_TOKEN`
- 如果 MCP Server 没有正确配置，但 `--allowedTools` 中包含了 `mcp__gitlab`，Claude 的行为会不稳定——有时走 Bash+curl 成功，有时走 MCP 通道失败（401）

---

## 五、Webhook Listener 部署

Webhook Listener 是一个 Flask 应用，负责接收 GitLab Webhook 事件、解析 `@claude` 指令、触发 Pipeline。

### 5.1 项目结构

```
claude-webhook-listener/
├── app.py              # Flask 应用主文件
├── Dockerfile          # Docker 镜像构建文件
├── docker-compose.yaml # Docker Compose 编排文件
├── requirements.txt    # Python 依赖
├── .env                # 环境变量（从 env.example 复制）
└── env.example         # 环境变量模板
```

### 5.2 环境变量配置

复制 `env.example` 为 `.env` 并填入实际值：

```env
# ===== 必须配置 =====
GITLAB_URL=https://your-gitlab.com

# 单仓库时直接填这个（默认 Trigger Token）
GITLAB_TRIGGER_TOKEN=your_trigger_token_here

# ===== 多仓库配置（可选）=====
# JSON 格式：project_id → trigger_token
PROJECT_TRIGGER_TOKENS={"1388":"token_for_project_1388","1534":"token_for_project_1534"}

# ===== 推荐配置 =====
# Personal Access Token（需要 api scope，对所有监听项目有权限即可）
GITLAB_ACCESS_TOKEN=your_access_token_here
WEBHOOK_SECRET=your_webhook_secret_here

# ===== 可选配置 =====
DEFAULT_REF=master
PORT=8080
```

**关于 Trigger Token**：在 GitLab 项目的 **Settings → CI/CD → Pipeline trigger tokens** 中创建。每个项目的 Trigger Token 不同。

### 5.3 Docker 部署

**Dockerfile：**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "--timeout", "30", "app:app"]
```

**docker-compose.yaml：**

```yaml
version: "3.8"

services:
  claude-webhook-listener:
    build: .
    container_name: claude-webhook-listener
    restart: unless-stopped
    network_mode: host
    environment:
      GITLAB_URL: ${GITLAB_URL:-https://gitlab.com}
      GITLAB_TRIGGER_TOKEN: ${GITLAB_TRIGGER_TOKEN}
      PROJECT_TRIGGER_TOKENS: ${PROJECT_TRIGGER_TOKENS:-}
      GITLAB_ACCESS_TOKEN: ${GITLAB_ACCESS_TOKEN:-}
      WEBHOOK_SECRET: ${WEBHOOK_SECRET:-}
      DEFAULT_REF: ${DEFAULT_REF:-main}
      PORT: "8080"
    env_file:
      - .env
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

> **`network_mode: host`**：使用宿主机网络，解决容器内无法解析内网 GitLab 域名（如 `git.meetsocial.cn`）的 DNS 问题。副作用是不能用 `ports` 映射，容器直接监听宿主机的 8080 端口。

**启动命令：**

```bash
cd claude-webhook-listener
cp env.example .env
# 编辑 .env 填入实际值
docker compose up -d --build
```

### 5.4 配置 GitLab Webhook

在 GitLab 项目（或 Group）的 **Settings → Webhooks** 中添加：

| 配置项       | 值                                                           |
| ------------ | ------------------------------------------------------------ |
| URL          | `http://<服务器IP>:8080/webhook`                             |
| Secret token | 与 `.env` 中的 `WEBHOOK_SECRET` 一致                         |
| Trigger      | ✅ Comments (Note events) <br> ✅ Issues events <br> ✅ Pipeline events |

> **注意**：URL 路径是 `/webhook`，不要写错。

### 5.5 Webhook Listener 核心逻辑

Webhook Listener（`app.py`）的核心功能：

#### 5.5.1 `@claude` 提及检测

只匹配**行首**的 `@claude`，避免 Claude 回复内容中包含 `@claude` 文字触发死循环：

```python
def extract_claude_instruction(note_body: str) -> str | None:
    for line in note_body.splitlines():
        stripped = line.strip()
        if stripped.lower().startswith("@claude"):
            match = MENTION_PATTERN.search(stripped)
            if match:
                return match.group(1).strip()
    return None
```

#### 5.5.2 Bot 用户过滤

防止 Claude 自己的回复触发死循环：

```python
BOT_USER_PATTERN = re.compile(r"^project_\d+_bot\d*$", re.IGNORECASE)

def _is_bot_user(username: str) -> bool:
    if BOT_USER_PATTERN.match(username):
        return True
    bot_keywords = ("bot", "claude", "ci-bot", "gitlab-bot")
    lower_name = username.lower()
    return any(lower_name == kw or lower_name.endswith(f"_{kw}") for kw in bot_keywords)
```

#### 5.5.3 完整的 Prompt 构建

将 Issue/MR 上下文、讨论历史和用户指令打包成一个完整的 prompt：

```python
def _build_full_prompt(instruction, noteable_type, noteable_iid, title, description, username, context_url, discussion_context=""):
    parts = []
    # 来源信息 + 标题描述
    # ...
    # 当前指令（最重要）
    parts.append("你本次需要执行的唯一任务：")
    parts.append(f">>> {instruction}")
    # 讨论历史（仅供参考）
    # ...
    # 关键约束
    parts.append("1. 只执行上面「你本次需要执行的唯一任务」中的指令")
    parts.append("2. 在回复评论中不要出现 '@claude'")
    parts.append("3. 如果任务只是回答问题，直接在评论中回复即可，不需要创建分支或提交 MR")
    # 错误处理要求
    parts.append("遇到权限错误时必须输出完整的诊断信息（状态码、响应体、Token信息等）")
    return "\n".join(parts)
```

#### 5.5.4 多仓库支持

同一个 Webhook Listener 可以监听多个 GitLab 项目，通过 `PROJECT_TRIGGER_TOKENS` 环境变量配置：

```python
# .env
PROJECT_TRIGGER_TOKENS={"1388":"token_aaa","1534":"token_bbb"}

# app.py
def _get_trigger_token(project_id: int) -> str:
    return PROJECT_TRIGGER_TOKENS.get(str(project_id), GITLAB_TRIGGER_TOKEN)
```

获取项目 ID 的方法：打开项目主页，项目名称下方会显示 `Project ID: 1388`。或通过 API：

```bash
curl --header "PRIVATE-TOKEN: your_token" "https://your-gitlab.com/api/v4/projects?search=project-name"
```

### 5.6 更新部署

修改代码后，更新服务器上的 Webhook Listener：

```bash
# 用 scp 上传修改后的文件
scp claude-webhook-listener/app.py root@<服务器IP>:/path/to/claude-webhook-listener/app.py

# SSH 到服务器重建容器
ssh root@<服务器IP> "cd /path/to/claude-webhook-listener && docker compose up -d --build"
```

`.gitlab-ci.yml` 的改动直接 `git push` 到仓库即可自动生效。

---

## 六、什么是 Skill

Claude Code 的 **Skill** 是一种结构化的指令文件（`SKILL.md`），遵循 [Agent Skills](https://agentskills.io/) 开放标准。它的作用是为 Claude 提供**领域特定的规范和行为指引**。

### 6.1 SKILL.md 文件格式

```markdown
---
name: coding-standards
description: 项目代码审查规范和编码标准
auto-activates:
  - "代码审查"
  - "code review"
---

# 代码审查规范

你是一名资深代码审查专家。请严格按照以下规范进行审查...

## 审查维度
### 1. 代码正确性
- 逻辑是否正确，边界条件是否处理
- ...

## 输出格式
请按以下格式输出审查结果：
...
```

文件包含两部分：

- **YAML Frontmatter**：元数据（名称、描述、自动激活关键词）
- **Markdown 正文**：具体的指令和规范

### 6.2 Skill 的加载位置

Claude Code 按以下优先级自动发现 Skill：

| 级别   | 路径                                     | 说明           |
| ------ | ---------------------------------------- | -------------- |
| 个人级 | `~/.claude/skills/<skill-name>/SKILL.md` | 跨所有项目生效 |
| 项目级 | `.claude/skills/<skill-name>/SKILL.md`   | 仅当前项目生效 |

> **重要发现**：经过实测，`claude --print` 非交互模式下，**个人级路径 (`/root/.claude/skills/`) 可以被自动加载**。

---

## 七、Skill 集成方案对比

在 CI 中使用 Skill，关键问题是：**SKILL.md 从哪来？** 我们实践了以下方案：

### 方案一：GitLab Runner 挂载宿主机目录

**思路**：在服务器上放好 Skill 文件，通过 Docker volumes 挂载到容器中。

**服务器端**：

```bash
mkdir -p /root/.claude/skills/coding-standards/
scp SKILL.md root@server:/root/.claude/skills/coding-standards/SKILL.md
```

**Runner config.toml**：

```toml
[[runners]]
  [runners.docker]
    volumes = [
      "/root/.claude/skills:/root/.claude/skills:ro"
    ]
```

**优点**：CI 配置零改动，Claude Code 自动发现。

**缺点**：

- 更新 Skill 需要登录服务器手动操作
- 挂载为 `:ro` 时，CI 中无法动态修改
- 不同服务器需要分别部署

### 方案二：CI 中从外部 Git 仓库拉取（推荐）

**思路**：建一个专门的 Git 仓库管理 Skill，CI 运行时自动 clone 到 `/root/.claude/skills/`。

**Skill 仓库结构**：

```
claude-skills/              ← 仓库根目录
├── coding-standards/
│   └── SKILL.md
├── security-review/
│   └── SKILL.md
└── python-best-practices/
    └── SKILL.md
```

**优点**：

- Skill 有版本管理，可追溯变更
- 跨项目共享，统一维护
- 不依赖服务器手动部署

**注意**：如果 Runner 同时挂载了 `/root/.claude/skills`（`:ro`），Git clone 的写入会失败。两种方案不要混用。

### 方案对比总结

| 方案         | 优点                 | 缺点             | 适用场景                 |
| ------------ | -------------------- | ---------------- | ------------------------ |
| Runner 挂载  | 零配置，自动生效     | 更新需登录服务器 | Skill 稳定不常变         |
| Git 仓库拉取 | 版本管理，跨项目共享 | CI 多一步 clone  | 团队协作，Skill 持续迭代 |

---

## 八、防死循环机制

`@claude` 交互最大的风险是**死循环**：Claude 回复评论 → 评论触发 Webhook → 又检测到 `@claude` → 又触发 Pipeline → 无限循环。

我们实现了三层防护：

### 第一层：Bot 用户过滤

Webhook Listener 检查评论者的用户名，如果是 Bot 用户直接忽略：

```python
# GitLab project bot: project_1388_bot, project_1388_bot1 等
BOT_USER_PATTERN = re.compile(r"^project_\d+_bot\d*$", re.IGNORECASE)
```

### 第二层：行首匹配

只匹配行首的 `@claude`，Claude 回复中间出现的 `@claude` 文字不会被当作指令：

```python
for line in note_body.splitlines():
    stripped = line.strip()
    if stripped.lower().startswith("@claude"):
        # 只有这种情况才算真正的提及
```

### 第三层：Prompt 约束

在传给 Claude 的 prompt 中明确要求不要使用 `@claude`：

```
注意事项：
2. 在回复评论中不要出现 '@claude'，用「Claude」或「我」代替。
```

---

## 九、错误诊断增强

权限问题是最常见的故障。我们在三个层面增加了完整的错误诊断输出：

### 9.1 CI Job 中的 Token 验证

`before_script` 中主动调用 GitLab API 验证 Token：

```bash
HTTP_CODE=$(curl -s -o /tmp/token_check.json -w "%{http_code}" \
  --header "PRIVATE-TOKEN: $GITLAB_ACCESS_TOKEN" \
  "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}")
if [ "$HTTP_CODE" = "200" ]; then
  echo "✅ Token 有效"
else
  echo "❌ Token 验证失败！HTTP $HTTP_CODE"
  # 输出详细排查建议
fi
```

### 9.2 Webhook Listener 中的详细错误日志

API 调用失败时输出状态码、响应体、Token 前缀和排查建议：

```python
if e.response.status_code in (401, 403):
    logger.error(
        f"   ⚠️ 权限不足！请检查 GITLAB_ACCESS_TOKEN 是否有 api scope，"
        f"以及对应用户是否对该项目有足够权限。"
        f"\n   当前 Token 前缀: {GITLAB_ACCESS_TOKEN[:6]}..."
        f"\n   请求 URL: {url}"
    )
```

### 9.3 Claude 的错误输出要求

在 prompt 中要求 Claude 遇到错误时输出完整诊断信息：

```
错误处理要求：
当你遇到 API 调用失败或权限错误时，必须输出：
- 完整的错误信息（HTTP 状态码、响应体内容）
- 调用的 API 地址
- 使用的认证方式
- 可能的原因分析
- 建议的排查步骤
绝对不要只说「我没有权限」就结束。
```

---

## 十、踩坑记录

### 10.1 Docker 镜像选择：不能用 Alpine

`node:24-alpine3.21` 会导致 Claude Code 二进制运行失败：

```
Error relocating ... posix_getdents: symbol not found
```

**解决**：使用 Debian 基础的 `node:24`。

### 10.2 Docker 镜像拉取超时（TLS handshake timeout）

GitLab Runner 拉取 `node:20` 镜像时超时：

```
failed to pull image "node:20" ... TLS handshake timeout
```

**解决**：在 Runner 宿主机上配置 Docker 镜像加速（registry mirror），或在 Runner 的 `config.toml` 中设置 `pull_policy = "if-not-present"`。

### 10.3 MCP Server 不存在导致的间歇性 401

早期配置中写了 `/bin/gitlab-mcp-server || true`，但这个文件根本不存在（它不是 `@anthropic-ai/claude-code` 自带的）。`|| true` 吞掉了错误，而 `--allowedTools` 仍包含 `mcp__gitlab`。

Claude 每次执行时随机选择路径：

- 走 **Bash + curl** → 成功
- 走 **mcp__gitlab**（不存在的 MCP Server） → 401 失败

**解决**：正确配置 MCP Server（通过 `~/.claude/settings.json`），或从 `--allowedTools` 中移除 `mcp__gitlab`。

### 10.4 Webhook URL 拼写错误

GitLab Webhook 配置中 URL 写成了 `/wenhook`（少了一个 `b`），返回 404。

**解决**：确认 URL 是 `http://<IP>:8080/webhook`。

### 10.5 Pipeline 触发后没有 Job

通过 Trigger API 触发 Pipeline，返回 `No stages / jobs for this pipeline`。

**原因**：`.gitlab-ci.yml` 中的 `claude-on-issue` Job 还没有 push 到目标分支。Pipeline Trigger API 使用的是指定 `ref` 分支上的 CI 配置。

**解决**：确保包含 `claude-on-issue` Job 的 `.gitlab-ci.yml` 已 push 到默认分支。

### 10.6 Git 推送权限问题

Claude 执行完代码修改后无法 push，提示 `You are not allowed to upload code`。

**原因**：CI Job 默认使用的 `gitlab-ci-token` 只有只读权限。

**解决**：在 `before_script` 中用 `GITLAB_ACCESS_TOKEN` 替换 Git remote URL：

```bash
REPO_URL=$(echo "$CI_REPOSITORY_URL" | sed "s|.*@|https://oauth2:${GITLAB_ACCESS_TOKEN}@|" | sed "s|\.git$||").git
git remote set-url origin "$REPO_URL"
```

### 10.7 CI/CD Variable 的 Protected 陷阱

`GITLAB_ACCESS_TOKEN` 配置为 Protected Variable 后，通过 Trigger API 触发的 Pipeline 拿不到变量值（返回空）。

**原因**：Trigger API 触发的 Pipeline 可能不被视为在 Protected branch 上运行。

**解决**：在 CI/CD Variables 中将 `GITLAB_ACCESS_TOKEN` 的 **Protected 标志关闭**。

### 10.8 不同项目用了不同的 Token

两个项目的 CI/CD Variables 中配置了不同的 `GITLAB_ACCESS_TOKEN`，其中一个 Token 对新项目没有权限。

**现象**：项目 A 一切正常，项目 B 时灵时不灵或始终 401。

**解决**：确保所有项目使用的 `GITLAB_ACCESS_TOKEN` 对应的用户对这些项目都有 Developer 以上权限。

### 10.9 Docker 容器 DNS 解析失败

Webhook Listener 容器内无法解析内网 GitLab 域名：

```
Failed to resolve 'git.meetsocial.cn'
```

**解决**：在 `docker-compose.yaml` 中使用 `network_mode: host`，容器直接使用宿主机网络和 DNS。

### 10.10 `claude --print` 模式下 Skill 加载

- **项目级 `.claude/skills/`**：实测中存在不稳定情况
- **个人级 `/root/.claude/skills/`**：经实测**可以稳定加载**，推荐使用此路径

### 10.11 Runner 挂载与 Git 拉取冲突

如果 Runner `config.toml` 中挂载了 `/root/.claude/skills:ro`，CI 脚本中尝试写入该路径会报错：

```
cp: cannot create regular file '/root/.claude/skills/coding-standards/SKILL.md': Read-only file system
```

**解决**：二选一，不要混用。

### 10.12 Claude 对简单问题创建了不必要的 MR

用户只是让 Claude 介绍项目，Claude 却创建了分支和 MR。

**解决**：在 prompt 中明确约束：

```
如果任务只是回答问题（不涉及代码修改），直接在评论中回复即可，不需要创建分支或提交 MR。
```

---

## 十一、效果展示

### 代码审查

CI 日志中可以看到 Skill 加载成功：

```
📦 从外部仓库拉取 skills...
  ✓ 已加载 skill: coding-standards
📋 /root/.claude/skills/ 中的 skills：
  ✓ /root/.claude/skills/coding-standards/SKILL.md
```

审查输出示例（按 SKILL.md 中定义的格式）：

```
## 📋 代码审查报告

### 概览
- **提交信息**: test_CI_14
- **审查文件数**: 1
- **风险等级**: 🟡 中

### 🔴 必须修复 (Critical)
（无）

### 🟡 建议改进 (Suggestion)
1. **[test_14.py:19]** `is_palindrome` 仅过滤空格，未处理标点符号
   - 对于 "A man, a plan, a canal: Panama" 会返回 False
   - 建议使用 `re.sub(r'[^a-z0-9]', '', s.lower())`

### 🟢 代码亮点 (Good Practice)
1. 所有函数均有类型注解和 docstring

### 📊 总结
- 代码结构清晰，建议修复 `is_palindrome` 的标点处理缺陷
- 是否建议合并：⚠️ 修复后合并
```

### @claude 交互

```
用户: @claude 帮我分析一下这个 Issue 涉及的代码
Claude: 收到，正在处理... [Pipeline](https://your-gitlab.com/...)

(Pipeline 执行完成后)

Claude: 我已经分析了相关代码，以下是我的发现...
```

---

## 十二、后续优化方向

1. **审查结果自动发布为 MR 评论**：将 `review_result.md` 通过 GitLab API 自动发布到 MR Discussion
2. **多 Skill 组合**：针对不同语言/框架创建专属 Skill（如 `python-standards`、`react-standards`）
3. **审查结果评分**：让 Claude 输出量化评分，用于质量门禁
4. **增量审查**：只审查新增/修改的函数，减少 token 消耗
5. **Pipeline 命名优化**：通过 CI 变量让 `@claude` 触发的 Pipeline 有更明确的标识

---

*本文基于 Claude Code CLI v1.x + GitLab Runner 18.8 + @modelcontextprotocol/server-gitlab 实践，配置和行为可能随版本更新有所变化。*
