# Bài 02: Consuming Existing MCP Servers

> **Module**: 03 — Model Context Protocol  
> **Thời lượng**: ~4 giờ đọc + thực hành  
> **Level**: Intermediate

---

## Mục tiêu bài học

- Config MCP servers trong Claude Code và Claude Desktop
- Hands-on với 3 servers: filesystem, postgres, github
- Verify và debug MCP connections
- Apply MCP servers vào Spring Boot project workflow thực tế
- Nắm common issues và cách fix

---

## 1. MCP Config Files — Biết file nào dùng ở đâu

Trước khi làm bất cứ điều gì, cần hiểu có **hai config locations** khác nhau.

### 1.1 Claude Desktop — `~/.claude/claude.json`

Đây là global config cho Claude Desktop app (GUI). MCP servers configured ở đây available trong **mọi conversation** trên Claude Desktop.

```
Vị trí:
  macOS:   ~/Library/Application Support/Claude/claude.json
  Windows: %APPDATA%\Claude\claude.json
  Linux:   ~/.config/claude/claude.json
```

### 1.2 Claude Code — `.claude/settings.json`

Đây là project-level config cho Claude Code (CLI). MCP servers configured ở đây chỉ active khi bạn chạy Claude Code **trong project directory** đó.

```
Vị trí: <your-project>/.claude/settings.json
```

Ưu điểm của project-level config:
- Mỗi project có servers riêng (Spring Boot project dùng postgres server, không ảnh hưởng project khác)
- Config có thể commit vào git (không bao gồm secrets — dùng env vars)
- Team members có thể share config

### 1.3 Global Claude Code Config — `~/.claude/settings.json`

MCP servers ở đây available trong **mọi** Claude Code session, bất kể project nào.

```
Vị trí: ~/.claude/settings.json
```

**Recommendation**: Đặt servers liên quan đến specific project vào `.claude/settings.json` trong project. Đặt servers dùng cho mọi project (ví dụ: filesystem) vào `~/.claude/settings.json`.

---

## 2. Config Format Chi Tiết

### Basic Structure

```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "<executable>",
      "args": ["<arg1>", "<arg2>"],
      "env": {
        "ENV_VAR_NAME": "value"
      }
    }
  }
}
```

Từng field:
- `server-name`: Tên tùy chọn, dùng để reference và debug. Chọn tên có nghĩa.
- `command`: Executable để chạy server. Thường là `npx`, `node`, `python`, hoặc `java`.
- `args`: Array các arguments truyền vào command.
- `env`: Environment variables inject vào server process. **Đây là nơi đặt secrets**.

### Full Config Example — 3 Servers cho Spring Boot Project

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/projects/my-spring-app"
      ]
    },
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres"
      ],
      "env": {
        "DATABASE_URL": "postgresql://dev_user:dev_pass@localhost:5432/myapp_dev"
      }
    },
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

**Lưu ý về `-y` flag trong npx**: Flag này tự động install package nếu chưa có, không hỏi confirm. Quan trọng vì MCP servers chạy non-interactively.

---

## 3. Setup Filesystem MCP Server

### Mục đích

Cho phép Claude browse và đọc files ngoài project directory hiện tại. Ví dụ:
- Đọc shared libraries ở nơi khác
- Browse config files trong `/etc` hoặc `~/.config`
- Access documentation ở separate directory

### Installation

Không cần install riêng — npx handle tự động. Chỉ cần config:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/projects",
        "/Users/yourname/documents/tech-docs"
      ]
    }
  }
}
```

Bạn có thể truyền **nhiều directories** — tất cả đều được accessible.

### Các Tools Filesystem Server Expose

```
read_file(path)          → Đọc nội dung file
write_file(path, content) → Ghi file (cẩn thận!)
list_directory(path)     → Liệt kê files trong directory
create_directory(path)   → Tạo directory mới
move_file(src, dest)     → Di chuyển/rename file
search_files(pattern)    → Tìm files theo pattern
get_file_info(path)      → Metadata (size, modified date, etc.)
```

### Hands-on Test với Spring Boot Project

Sau khi config, thử trong Claude Code:

```
Bạn: "List tất cả Flyway migration files trong project của tôi"

Claude: [gọi list_directory và đọc files từ filesystem MCP server]
        "Tôi thấy 23 migration files:
        V1__create_users_table.sql
        V2__add_email_index.sql
        V3__create_orders_table.sql
        ..."
```

```
Bạn: "Đọc V15__add_payment_columns.sql và giải thích changes"

Claude: [gọi read_file]
        "Migration này thêm 3 columns vào bảng payments:
        - stripe_payment_intent_id VARCHAR(255)
        - payment_method_type VARCHAR(50)  
        - metadata JSONB
        
        Index được tạo trên stripe_payment_intent_id cho lookup performance."
```

---

## 4. Setup PostgreSQL MCP Server

### Mục đích

Đây là server bạn sẽ dùng nhiều nhất. Cho phép Claude:
- Query database trực tiếp
- Describe schema với full detail
- Analyze data patterns
- Generate và validate SQL queries

### Prerequisites

```bash
# PostgreSQL đang chạy
pg_isready -h localhost -p 5432

# Node.js installed (cho npx)
node --version  # cần 18+
```

### Config

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres"
      ],
      "env": {
        "DATABASE_URL": "postgresql://dev_user:dev_password@localhost:5432/myapp_dev"
      }
    }
  }
}
```

**Security quan trọng**: 
- Dùng **dedicated read-only user** cho MCP server, không phải application user
- Tạo user chỉ có SELECT permissions:

```sql
-- Tạo read-only user cho MCP
CREATE USER mcp_readonly WITH PASSWORD 'strong_random_password';

-- Grant kết nối vào database
GRANT CONNECT ON DATABASE myapp_dev TO mcp_readonly;

-- Grant usage trên schema
GRANT USAGE ON SCHEMA public TO mcp_readonly;

-- Grant SELECT trên tất cả tables hiện tại
GRANT SELECT ON ALL TABLES IN SCHEMA public TO mcp_readonly;

-- Grant SELECT trên tables được tạo trong tương lai
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT SELECT ON TABLES TO mcp_readonly;
```

### Các Tools PostgreSQL Server Expose

```
query(sql)          → Execute SQL query, trả về results
list_tables()       → Liệt kê tất cả tables
describe_table(name) → Schema của một table
```

### Hands-on: 5 Queries Thực Tế

**Query 1: Khám phá schema**
```
Bạn: "Describe schema của database, focus vào relationships giữa các tables"

Claude: [list_tables + describe_table cho mỗi table]
        "Database có 12 tables với structure sau:
        
        users (id, email, name, created_at, plan_id)
        plans (id, name, price, features JSONB)
        subscriptions (id, user_id→users, plan_id→plans, status, expires_at)
        orders (id, user_id→users, total, status, created_at)
        order_items (id, order_id→orders, product_id→products, quantity, price)
        ..."
```

**Query 2: Analyze data distribution**
```
Bạn: "Bao nhiêu users mỗi subscription plan? Có users nào không có subscription không?"

Claude: [query với GROUP BY và LEFT JOIN]
        SELECT p.name, COUNT(s.id) as subscribers, ...
        
        Results:
        Free:    12,453 users
        Pro:      3,821 users  
        Enterprise: 156 users
        No subscription: 892 users (legacy accounts)
```

**Query 3: Debug production issue**
```
Bạn: "Tìm orders có status 'PROCESSING' lâu hơn 1 tiếng, có gì bất thường không?"

Claude: [query orders table với time filter]
        "Tìm thấy 47 orders bị stuck trong PROCESSING:
        - 23 orders có payment_intent_id = NULL (payment chưa capture)
        - 15 orders có created_at trong window 14:30-15:45 hôm qua
          → Khớp với Stripe webhook outage bạn mention
        - 9 orders còn lại cần investigate manual"
```

**Query 4: Performance analysis**
```
Bạn: "Check xem có missing indexes nào ảnh hưởng đến performance không?"

Claude: [query pg_stat_user_tables, pg_indexes, pg_stat_statements]
        "Phát hiện vấn đề:
        - Table order_items không có index trên order_id
          → 847 seq scans trong 24h qua
          → Recommendation: CREATE INDEX idx_order_items_order_id ON order_items(order_id)
        - Table users: index trên email tồn tại nhưng không được dùng
          → Queries dùng ILIKE thay vì = → index không effective"
```

**Query 5: Data quality check**
```
Bạn: "Kiểm tra data integrity: có foreign key violations nào không, 
      có orphaned records không?"

Claude: [queries kiểm tra FK consistency]
        "Data integrity report:
        ✓ users.plan_id: tất cả valid
        ⚠ order_items.product_id: 3 records tham chiếu products đã bị xóa
          → IDs: 10234, 10235, 10236 (có thể do incomplete soft-delete)
        ✓ subscriptions.user_id: tất cả valid"
```

---

## 5. Setup GitHub MCP Server

### Mục đích

Cho phép Claude đọc context từ GitHub trong khi làm việc:
- Đọc issues để hiểu requirements
- Review PRs trong context
- Trace lịch sử code changes
- Cross-reference commits với bugs

### Prerequisites

```bash
# Tạo GitHub Personal Access Token
# GitHub → Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens
# Permissions cần: Contents (read), Issues (read), Pull Requests (read)
```

### Config

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_TOKEN": "github_pat_xxxxxxxxxxxx"
      }
    }
  }
}
```

**Security**: Token chỉ cần **read permissions**. Không cần write access cho development workflow.

### Các Tools GitHub Server Expose

```
search_repositories(query)          → Tìm repos
get_file_contents(owner, repo, path) → Đọc file từ repo
search_code(query)                  → Tìm code
list_issues(owner, repo, ...)       → Liệt kê issues
get_issue(owner, repo, number)      → Chi tiết một issue
search_issues(query)                → Tìm issues
list_pull_requests(owner, repo)     → Liệt kê PRs
get_pull_request(owner, repo, number) → Chi tiết một PR
list_commits(owner, repo, ...)      → Git log
get_commit(owner, repo, sha)        → Chi tiết một commit
```

### Hands-on: 5 Queries Thực Tế

**Query 1: Context khi fix bug**
```
Bạn: "Tôi đang fix issue #342. Đọc issue đó và các related PRs"

Claude: [get_issue + list_pull_requests filter by issue mention]
        "Issue #342: 'Payment processing fails for amounts > $999.99'
        Reporter: john.doe, Priority: Critical, Labels: bug, payments
        
        Description: Stripe charge fails với error 'amount_too_large' 
        khi total > 999.99. Có vẻ liên quan đến integer overflow.
        
        Related PR #345 (merged, 2 ngày trước):
        'Fix: Convert price to cents before Stripe call'
        → PR này fix phần Stripe call nhưng có thể miss edge case
        với discount codes làm giá vượt quá original amount"
```

**Query 2: Code review context**
```
Bạn: "Review PR #389: 'Add Redis caching for product catalog'"

Claude: [get_pull_request + get_file_contents các files changed]
        "PR #389 analysis:
        
        Files changed: 6 files, +234 -12 lines
        
        Key changes:
        1. ProductService.java: Added @Cacheable annotations
        2. CacheConfig.java: New Redis TTL configuration
        3. ProductController.java: Cache eviction on product update
        
        Issues tôi thấy:
        ⚠ Cache key dùng product ID nhưng không include version/locale
          → Nếu serve multi-language: users có thể thấy sai language
        ⚠ Không có cache warm-up strategy
          → Cold start sau deploy sẽ spike database load
        ✓ TTL configuration đúng, 15 phút cho product data"
```

**Query 3: Trace bug lịch sử**
```
Bạn: "File OrderService.java đã được thay đổi như thế nào trong 2 tuần qua?"

Claude: [list_commits filter by path + get_commit cho mỗi commit]
        "OrderService.java: 5 commits trong 14 ngày:
        
        Jun 01 - feat: Add retry logic for payment processing (PR #401)
        May 28 - fix: Prevent duplicate order creation under high load (PR #398)
        May 25 - refactor: Extract payment processing to PaymentService (PR #391)
        May 22 - fix: Handle null shipping address gracefully (PR #387)
        May 20 - feat: Add order status webhook notifications (PR #383)
        
        → Bug 'duplicate orders' (Issue #342) fix ngày May 28.
          Commit hash: a3f8c29"
```

**Query 4: Dependency analysis**
```
Bạn: "Kiểm tra pom.xml: có dependencies nào outdated có known vulnerabilities không?"

Claude: [get_file_contents pom.xml + search_issues/code về CVEs]
        "Phân tích pom.xml:
        
        ⚠ spring-boot version 3.1.5 → current LTS là 3.2.4
          Known CVE: CVE-2024-22234 (Spring Security bypass) trong 3.1.x
          
        ⚠ jackson-databind 2.15.2 → current là 2.17.0
          2 CVEs fixed trong 2.16.x range
          
        ✓ postgresql driver 42.6.0 → current, no known CVEs
        ✓ flyway-core 9.22.3 → acceptable, latest là 10.x (major upgrade)"
```

**Query 5: Cross-repo investigation**
```
Bạn: "Team có microservice 'notification-service'. 
      Tìm xem họ handle email retry như thế nào"

Claude: [search_code trong org/notification-service]
        "notification-service sử dụng:
        - Spring Retry với @Retryable(maxAttempts=3, backoff=@Backoff(delay=1000))
        - Dead letter queue sau 3 failures
        - Email provider: AWS SES với SDK v2
        
        Relevant file: src/main/java/com/company/notification/email/EmailService.java
        Lines 45-89: Retry logic implementation
        
        Bạn có thể adopt pattern tương tự cho payment notification 
        trong OrderService."
```

---

## 6. Verifying MCP Connections

### Dùng `/mcp` command trong Claude Code

```bash
# Trong Claude Code session
/mcp
```

Output:

```
MCP Servers:
  filesystem  ✓ Connected  (7 tools available)
  postgres    ✓ Connected  (3 tools available)
  github      ✓ Connected  (10 tools available)
```

Nếu có lỗi:

```
MCP Servers:
  filesystem  ✓ Connected  (7 tools available)
  postgres    ✗ Failed     Error: connect ECONNREFUSED 127.0.0.1:5432
  github      ✓ Connected  (10 tools available)
```

### List available tools của một server

```
/mcp postgres tools
```

Output:
```
postgres tools:
  - query(sql: string): Execute a SQL query
  - list_tables(): List all tables in the database  
  - describe_table(name: string): Describe a table schema
```

---

## 7. Common Issues và Cách Fix

### Issue 1: `npx: command not found`

**Triệu chứng**: Server không start, error "command not found".

**Nguyên nhân**: Node.js chưa install hoặc không trong PATH.

**Fix**:
```bash
# Install Node.js via nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 20
nvm use 20

# Verify
node --version  # v20.x.x
npx --version   # 10.x.x
```

Nếu dùng macOS và cài qua Homebrew:
```bash
brew install node
# Restart Claude Code sau khi install
```

**Root cause thường gặp**: Claude Code có thể không inherit shell PATH đầy đủ trên macOS. Fix bằng cách dùng absolute path:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "/usr/local/bin/npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    }
  }
}
```

Tìm absolute path: `which npx`

### Issue 2: PostgreSQL `connection refused`

**Triệu chứng**: `Error: connect ECONNREFUSED 127.0.0.1:5432`

**Fix steps**:
```bash
# 1. Check PostgreSQL đang chạy
pg_isready -h localhost -p 5432
# Expected: localhost:5432 - accepting connections

# 2. Nếu không chạy, start PostgreSQL
# macOS Homebrew:
brew services start postgresql@15

# Linux systemd:
sudo systemctl start postgresql

# Docker:
docker start my-postgres-container

# 3. Verify connection với psql
psql "postgresql://dev_user:dev_pass@localhost:5432/myapp_dev" -c "SELECT 1"
```

### Issue 3: GitHub `401 Unauthorized`

**Triệu chứng**: GitHub server connect được nhưng tool calls return 401.

**Fix**:
```bash
# 1. Verify token còn valid
curl -H "Authorization: Bearer ghp_yourtoken" https://api.github.com/user

# 2. Check token permissions
# Token cần: repo:read (hoặc Contents: read, Issues: read, Pull requests: read)

# 3. Check token expiry
# GitHub → Settings → Developer Settings → Personal Access Tokens → xem expiry date
```

### Issue 4: Server starts nhưng tools không available

**Triệu chứng**: Server show "Connected" nhưng `/mcp` show 0 tools.

**Nguyên nhân thường gặp**: Package version mismatch hoặc server initialization error.

**Fix**:
```bash
# Force update package
npx -y @modelcontextprotocol/server-postgres@latest

# Hoặc install globally
npm install -g @modelcontextprotocol/server-postgres

# Sau đó config dùng global install:
{
  "command": "mcp-server-postgres"
}
```

### Issue 5: Permission denied khi access files

**Triệu chứng**: Filesystem server connect được nhưng read_file return "Permission denied".

**Fix**: Verify Claude Code process có read access tới directories được config:

```bash
# Check permissions
ls -la /path/to/your/project

# Nếu cần, add read permissions
chmod -R +r /path/to/your/project
```

### Issue 6: Environment variables không được pass

**Triệu chứng**: Server start nhưng database connection fail vì wrong credentials.

**Fix**: Verify syntax trong config file. Common mistake:

```json
// WRONG: env vars dạng string có dấu $
"env": {
  "DATABASE_URL": "$DATABASE_URL"
}

// CORRECT: literal value
"env": {
  "DATABASE_URL": "postgresql://user:pass@localhost/db"
}

// HOẶC: dùng shell expansion trong command (không trong env object)
```

Nếu muốn dùng env vars từ shell, Claude Code **không** tự động expand shell variables trong config. Cần dùng literal values trong `env` object hoặc tham chiếu `${ENV_VAR}` syntax được support bởi Claude Code (check docs version hiện tại).

---

## 8. Best Practices cho Team Environment

### 8.1 Config nên commit, secrets không nên

Tạo `.claude/settings.json` với placeholders:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${MCP_DATABASE_URL}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${MCP_GITHUB_TOKEN}"
      }
    }
  }
}
```

Team members set env vars trong shell profile (`~/.zshrc` hoặc `~/.bashrc`):

```bash
export MCP_DATABASE_URL="postgresql://dev:pass@localhost/myapp_dev"
export MCP_GITHUB_TOKEN="ghp_yourpersonaltoken"
```

Thêm `.claude/settings.json` vào git, thêm actual secrets file vào `.gitignore`.

### 8.2 Separate DB users per environment

```
Dev DB:        mcp_dev_readonly    → full read access to dev schema
Staging DB:    mcp_staging_readonly → full read access (cẩn thận với PII)
Production DB: KHÔNG CONFIG cho MCP (too risky cho dev machines)
```

### 8.3 Document MCP setup trong project README

```markdown
## MCP Setup

This project uses MCP servers for AI-assisted development.

Required env vars (add to ~/.zshrc):
- `MCP_DATABASE_URL`: Dev database connection string
- `MCP_GITHUB_TOKEN`: GitHub PAT with read:repo scope

Setup: See `.claude/settings.json` for server configuration.
```

---

## 9. Exercise

### Exercise 2.1 — Full Setup

Setup 3 MCP servers cho một Spring Boot project:

**Bước 1**: Tạo file `.claude/settings.json` trong project root với config cho filesystem, postgres, github.

**Bước 2**: Verify connections với `/mcp` command.

**Bước 3**: Chạy 5 queries với mỗi server (15 queries total). Document output.

### Exercise 2.2 — Postgres Exploration

Với postgres MCP server connected, thực hiện:

1. List tất cả tables và relationships
2. Find table với nhiều rows nhất
3. Identify columns có nhiều NULL values nhất
4. Check có indexes nào missing trên foreign keys không
5. Tìm 3 queries phổ biến nhất (nếu pg_stat_statements enabled)

### Exercise 2.3 — GitHub Context

Với github MCP server, thực hiện:

1. Đọc 3 open issues gần đây nhất, summarize
2. Tìm commits liên quan đến một feature bạn đang làm
3. Review một PR đang open — identify potential issues
4. Tìm code trong repo theo pattern (ví dụ: tìm tất cả `@Scheduled` annotations)
5. Cross-reference một bug với git history

### Exercise 2.4 — Integrated Workflow

Simulate debug session thực tế:

> Production alert: API endpoint `/api/orders/{id}` returning 500 errors cho một số requests.

Dùng tất cả 3 MCP servers để investigate:
1. Filesystem: Đọc OrderController và OrderService code
2. GitHub: Tìm recent commits liên quan đến orders
3. Postgres: Query orders table tìm pattern trong failing cases

Document: Bạn có thể identify root cause chỉ bằng Claude + MCP servers không?

---

## Tóm tắt bài học

| Topic | Key Point |
|-------|-----------|
| Config locations | `.claude/settings.json` cho project-level, `~/.claude/settings.json` cho global |
| Config format | command + args + env object |
| Filesystem server | Browse files, multiple directories, use for project structure exploration |
| Postgres server | Query DB, schema exploration, data analysis — dùng read-only user |
| GitHub server | Issues, PRs, commits, code search — fine-grained token với read-only scopes |
| Verify connections | `/mcp` command trong Claude Code |
| Common issues | npx PATH, DB not running, invalid token, permission denied |
| Team best practices | Config in git, secrets in env vars, separate DB users per env |

---

## Bài tiếp theo

**Bài 03: Building MCP Server với Java** — Bây giờ bạn biết cách dùng MCP servers. Bước tiếp theo: build một MCP server của riêng mình bằng Spring Boot, expose Spring application internals (Actuator, database schema, logs) để Claude có thể query trực tiếp.

---

*Bài 02/03 — Module 03: Model Context Protocol*
