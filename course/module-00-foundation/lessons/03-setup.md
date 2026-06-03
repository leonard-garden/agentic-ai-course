# Bài 03 — Thiết lập môi trường phát triển

**Module:** 00 — Foundation  
**Thời lượng:** ~1 giờ  
**Prerequisite:** Bài 01 (AI là gì), Bài 02 (Agentic AI overview)  
**Mục tiêu:** Có một môi trường hoàn chỉnh, chạy được `HelloClaude.java` và gọi Claude API thành công trước khi kết thúc bài.

---

## Mục lục

1. [Tổng quan — Checklist đầy đủ](#1-tổng-quan--checklist-đầy-đủ)
2. [Anthropic API Key](#2-anthropic-api-key)
3. [Java Setup](#3-java-setup)
4. [Claude Code CLI](#4-claude-code-cli)
5. [Node.js & MCP Setup](#5-nodejs--mcp-setup)
6. [Python Setup](#6-python-setup)
7. [IDE Setup — IntelliJ IDEA](#7-ide-setup--intellij-idea)
8. [Verify toàn bộ setup](#8-verify-toàn-bộ-setup)
9. [Troubleshooting thường gặp](#9-troubleshooting-thường-gặp)
10. [Exercise](#10-exercise)

---

## 1. Tổng quan — Checklist đầy đủ

Trước khi bắt đầu, đây là tất cả những gì bạn cần cài trong module này. Không cần làm tất cả một lúc — mỗi section dưới đây sẽ hướng dẫn chi tiết từng bước.

| Tool | Phiên bản tối thiểu | Dùng cho | Bắt buộc? |
|------|---------------------|----------|-----------|
| Java (JDK) | 17+ (LTS) | Toàn bộ khóa học | ✅ Bắt buộc |
| Maven hoặc Gradle | Maven 3.8+ / Gradle 8+ | Build Java projects | ✅ Bắt buộc |
| Anthropic API Key | — | Gọi Claude API | ✅ Bắt buộc |
| Claude Code CLI | Latest | Agentic coding, MCP | ✅ Bắt buộc |
| Node.js | 18+ (LTS) | Chạy MCP servers | ✅ Bắt buộc (Module 03) |
| Python | 3.11+ | Claude Agent SDK | ✅ Bắt buộc (Module 02) |
| IntelliJ IDEA | 2023.1+ | IDE chính | Recommended |

> **Lưu ý cho Windows users:** Các lệnh trong bài này viết cho macOS/Linux. Phần Windows sẽ được chú thích riêng. Nếu dùng Windows, khuyến khích dùng WSL2 (Windows Subsystem for Linux) để tránh friction.

---

## 2. Anthropic API Key

API key là thứ đầu tiên bạn cần — không có nó, không có gì chạy được.

### 2.1 Tạo account và lấy API key

1. Truy cập [console.anthropic.com](https://console.anthropic.com)
2. Sign up hoặc log in bằng Google/email
3. Vào **API Keys** → **Create Key**
4. Đặt tên descriptive, ví dụ: `agentic-ai-course-dev`
5. Copy key ngay — **key chỉ hiện một lần**, mất là phải tạo lại

> **Về billing:** Free tier có $5 credit khi sign up. Đủ để học toàn bộ khóa này nếu dùng model Haiku cho development. Khi hết credit, cần thêm payment method.

### 2.2 Set Usage Limits (quan trọng)

Trước khi code, set spending limit để tránh bill bất ngờ:

1. Console → **Settings** → **Limits**
2. Set **Monthly Spend Limit**: $10–$20 cho môi trường dev
3. Set **Notification Threshold**: 80% để nhận email cảnh báo

### 2.3 Set Environment Variable

API key không được hardcode trong code. Luôn đọc từ environment variable.

**macOS / Linux — thêm vào shell config:**

```bash
# Kiểm tra bạn đang dùng shell gì
echo $SHELL
# Output: /bin/zsh hoặc /bin/bash

# Nếu dùng zsh (macOS default từ Catalina):
echo 'export ANTHROPIC_API_KEY="sk-ant-api03-..."' >> ~/.zshrc
source ~/.zshrc

# Nếu dùng bash:
echo 'export ANTHROPIC_API_KEY="sk-ant-api03-..."' >> ~/.bashrc
source ~/.bashrc
```

**Windows — System Environment Variables:**

```powershell
# PowerShell (run as Administrator)
[System.Environment]::SetEnvironmentVariable(
    "ANTHROPIC_API_KEY",
    "sk-ant-api03-...",
    "User"
)

# Hoặc qua GUI: 
# Windows Search → "Environment Variables" → User variables → New
# Variable name: ANTHROPIC_API_KEY
# Variable value: sk-ant-api03-...
```

**IntelliJ IDEA — Run Configuration:**

Khi chạy từ IDE, environment variable từ shell có thể không được inherit tự động:

1. **Run** → **Edit Configurations**
2. Chọn configuration của bạn (hoặc tạo mới Application)
3. Tab **Environment Variables** → thêm `ANTHROPIC_API_KEY=sk-ant-api03-...`
4. Hoặc dùng `.env` file (xem Section 7)

### 2.4 Verify API Key với curl

```bash
# Verify API key hoạt động — gọi Claude API trực tiếp
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-haiku-4-5",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Say hi"}]
  }'
```

Response thành công trông như thế này:

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "Hi there! How can I help you today?"}],
  "model": "claude-haiku-4-5",
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 10, "output_tokens": 12}
}
```

Nếu thấy `"type": "message"` và `"role": "assistant"` — API key của bạn đã hoạt động.

---

## 3. Java Setup

### 3.1 Cài Java 17+

```bash
# Kiểm tra Java đã cài chưa
java -version
# Expected: openjdk version "17.x.x" hoặc "21.x.x"

javac -version
# Expected: javac 17.x.x
```

Nếu chưa có Java hoặc version thấp hơn 17:

**macOS — dùng Homebrew (recommended):**

```bash
# Cài Homebrew nếu chưa có
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Cài Java 21 (LTS mới nhất)
brew install openjdk@21

# Thêm vào PATH
echo 'export PATH="/opt/homebrew/opt/openjdk@21/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

java -version
# openjdk version "21.x.x" ...
```

**Linux (Ubuntu/Debian):**

```bash
sudo apt update
sudo apt install openjdk-21-jdk
java -version
```

**Windows:**

Tải JDK 21 từ [adoptium.net](https://adoptium.net) (Eclipse Temurin). Installer sẽ tự set PATH.

### 3.2 Cài Maven

```bash
# macOS
brew install maven
mvn -version
# Apache Maven 3.x.x

# Linux
sudo apt install maven

# Verify
mvn -version
```

Nếu dùng Gradle:

```bash
# macOS
brew install gradle
gradle -version

# Hoặc dùng Gradle Wrapper (không cần cài global)
./gradlew -version
```

### 3.3 Anthropic Java SDK — Maven Dependency

Thêm vào `pom.xml`:

```xml
<dependencies>
    <!-- Anthropic Java SDK -->
    <dependency>
        <groupId>com.anthropic</groupId>
        <artifactId>anthropic-java</artifactId>
        <version>0.8.0</version>
    </dependency>

    <!-- SLF4J logging (SDK cần) -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>2.0.9</version>
    </dependency>
</dependencies>
```

**Gradle alternative** — thêm vào `build.gradle`:

```groovy
dependencies {
    implementation 'com.anthropic:anthropic-java:0.8.0'
    implementation 'org.slf4j:slf4j-simple:2.0.9'
}
```

Hoặc `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.anthropic:anthropic-java:0.8.0")
    implementation("org.slf4j:slf4j-simple:2.0.9")
}
```

> **Kiểm tra version mới nhất:** [mvnrepository.com/artifact/com.anthropic/anthropic-java](https://mvnrepository.com/artifact/com.anthropic/anthropic-java)

### 3.4 HelloClaude.java — First Working Program

Tạo project Maven mới:

```bash
mvn archetype:generate \
  -DgroupId=com.yourname.agenticai \
  -DartifactId=hello-claude \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

cd hello-claude
```

Tạo file `src/main/java/com/yourname/agenticai/HelloClaude.java`:

```java
package com.yourname.agenticai;

import com.anthropic.client.Anthropic;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.messages.Message;
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.Model;

public class HelloClaude {

    public static void main(String[] args) {
        // Client tự động đọc ANTHROPIC_API_KEY từ environment
        Anthropic client = AnthropicOkHttpClient.fromEnv();

        MessageCreateParams params = MessageCreateParams.builder()
                .model(Model.CLAUDE_HAIKU_4_5)          // Dùng Haiku để tiết kiệm cost
                .maxTokens(256)
                .addUserMessage("Hello! I'm a Java backend engineer starting to learn agentic AI. "
                        + "In one sentence, what's the most exciting thing I can build with Claude API?")
                .build();

        Message message = client.messages().create(params);

        System.out.println("=== Response from Claude ===");
        System.out.println(message.content().get(0).text().get().text());
        System.out.println("============================");
        System.out.println("Model    : " + message.model());
        System.out.println("Input    : " + message.usage().inputTokens() + " tokens");
        System.out.println("Output   : " + message.usage().outputTokens() + " tokens");
    }
}
```

Chạy:

```bash
# Download dependencies và compile
mvn compile

# Chạy (ANTHROPIC_API_KEY phải có trong environment)
mvn exec:java -Dexec.mainClass="com.yourname.agenticai.HelloClaude"
```

Output mong đợi:

```
=== Response from Claude ===
You can build intelligent autonomous agents that analyze your Java codebase, 
suggest architectural improvements, and automatically generate production-ready 
microservices with just a natural language description.
============================
Model    : claude-haiku-4-5-20251001
Input    : 47 tokens
Output   : 38 tokens
```

Nếu bạn thấy output tương tự — **Java setup hoàn tất!**

---

## 4. Claude Code CLI

Claude Code là CLI tool chính bạn sẽ dùng xuyên suốt khóa học. Không chỉ là chatbot — đây là agentic coding assistant có thể đọc/viết file, chạy lệnh, và làm việc với codebase thực.

### 4.1 Cài đặt

```bash
# Yêu cầu Node.js 18+ (xem Section 5 nếu chưa cài)
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
# claude v1.x.x
```

> **Windows note:** Nếu gặp permission error, chạy PowerShell as Administrator.  
> **macOS note:** Nếu dùng nvm (xem Section 5), không cần sudo.

### 4.2 Login lần đầu

```bash
claude
# Lần đầu chạy sẽ mở browser để authenticate với Anthropic account
# Sau khi auth xong, quay lại terminal — bạn đang trong Claude Code session
```

Thoát session: gõ `exit` hoặc Ctrl+C.

### 4.3 Basic Commands

```bash
# Mở interactive session (không có prompt — Claude đợi bạn nhập)
claude

# Chạy với prompt thẳng từ command line (non-interactive)
claude "Explain what this Java class does"

# Chạy và pipe output
claude "Write a Maven pom.xml for a Spring Boot app" > pom.xml
```

**Slash commands bên trong session:**

```
/help          # Xem tất cả available commands
/model         # Đổi model (haiku/sonnet/opus)
/clear         # Clear conversation context
/status        # Xem trạng thái session hiện tại
/mcp           # Xem MCP servers đang connected
/cost          # Xem token usage và estimated cost của session
exit           # Thoát session
```

### 4.4 Recommended Settings

Tạo file `~/.claude/settings.json` với cấu hình cơ bản:

```json
{
  "defaultModel": "claude-sonnet-4-5",
  "theme": "dark",
  "autoUpdates": true,
  "preferredLanguage": "vi"
}
```

> **Giải thích model choice:** `claude-sonnet-4-5` là balance tốt nhất giữa capability và cost cho daily development. Dùng `claude-opus-4-5` cho complex architectural decisions, `claude-haiku-4-5` cho quick lookups.

### 4.5 Test Claude Code trong Java Project

```bash
# Di chuyển vào hello-claude project vừa tạo
cd hello-claude

# Mở Claude Code session
claude

# Trong session, thử hỏi:
> Describe the structure of this Java project
> What does HelloClaude.java do?
> Suggest improvements to the error handling in this code
```

Claude Code sẽ đọc các file trong project và trả lời context-aware — đây là điểm khác biệt hoàn toàn so với chatbot thông thường.

---

## 5. Node.js & MCP Setup

Node.js cần cho hai mục đích: (1) chạy Claude Code CLI, và (2) chạy MCP (Model Context Protocol) servers trong Module 03.

### 5.1 Cài Node.js với nvm (Recommended)

Dùng `nvm` (Node Version Manager) thay vì cài trực tiếp — dễ switch version và không cần sudo.

**macOS / Linux:**

```bash
# Cài nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Reload shell
source ~/.zshrc  # hoặc ~/.bashrc

# Cài Node.js 22 (LTS)
nvm install 22
nvm use 22
nvm alias default 22

# Verify
node --version
# v22.x.x
npm --version
# 10.x.x
```

**Windows:**

Dùng [nvm-windows](https://github.com/coreybutler/nvm-windows/releases) — tải installer `.exe` từ releases page.

```powershell
nvm install 22
nvm use 22
node --version
```

### 5.2 Quick Test: Chạy MCP Server

Ngay bây giờ, thử chạy một MCP server để verify Node.js hoạt động và xem MCP là gì:

```bash
# Chạy filesystem MCP server — expose /tmp directory cho Claude
npx @modelcontextprotocol/server-filesystem /tmp
```

Terminal sẽ hiển thị server đang running. Đây là MCP server — một process expose tools cho Claude sử dụng. Chi tiết về MCP sẽ học trong Module 03.

Ctrl+C để dừng server.

### 5.3 Verify MCP trong Claude Code

```bash
# Trong một terminal khác, chạy MCP server
npx @modelcontextprotocol/server-filesystem ~/Documents

# Mở Claude Code session
claude

# Check MCP status
> /mcp
```

---

## 6. Python Setup

Python cần cho Anthropic Agent SDK (dùng trong Module 02). Java SDK và Python SDK có feature parity, nhưng một số examples và tools trong khóa học viết bằng Python.

### 6.1 Cài Python với pyenv (Recommended)

```bash
# macOS — cài pyenv qua Homebrew
brew install pyenv

# Thêm vào shell config
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc
source ~/.zshrc

# Cài Python 3.12
pyenv install 3.12.3
pyenv global 3.12.3

# Verify
python --version
# Python 3.12.3
```

**Linux (Ubuntu/Debian):**

```bash
sudo apt update
sudo apt install python3.12 python3.12-pip python3.12-venv
python3.12 --version
```

**Windows:**

Tải Python 3.12 từ [python.org](https://python.org/downloads). Tick vào "Add Python to PATH" khi install.

### 6.2 Cài Anthropic Python SDK

```bash
# Tạo virtual environment (best practice — không cài global)
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Cài Anthropic SDK
pip install anthropic

# Verify
python -c "import anthropic; print('Anthropic SDK:', anthropic.__version__)"
# Anthropic SDK: 0.x.x
```

### 6.3 Quick Python Test

```python
# test_claude.py
import anthropic
import os

client = anthropic.Anthropic()  # Tự đọc ANTHROPIC_API_KEY từ env

message = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=100,
    messages=[
        {"role": "user", "content": "Say 'Python setup successful!' and nothing else."}
    ]
)

print(message.content[0].text)
```

```bash
python test_claude.py
# Python setup successful!
```

---

## 7. IDE Setup — IntelliJ IDEA

### 7.1 Recommended Plugins

Mở IntelliJ → **Settings** → **Plugins** → Marketplace, search và cài:

- **EnvFile** (Borys Pierov) — load `.env` files tự động vào Run Configurations
- **Maven Helper** — analyze dependency conflicts
- **.ignore** — template cho `.gitignore`

> **Claude Code Extension:** Anthropic không có official IntelliJ plugin riêng biệt. Tuy nhiên, Claude Code CLI chạy trong terminal tích hợp của IntelliJ hoàn toàn tốt. Mở **View** → **Tool Windows** → **Terminal** và dùng như bình thường.

### 7.2 Environment Variables trong Run Configuration

**Cách 1 — Trực tiếp trong Run Configuration:**

1. **Run** → **Edit Configurations** → chọn Application configuration
2. **Modify options** → **Add environment variables**
3. Thêm: `ANTHROPIC_API_KEY=sk-ant-api03-...`

**Cách 2 — Dùng .env file với EnvFile plugin (Recommended):**

Tạo file `.env` ở root project:

```bash
# .env — KHÔNG commit file này lên Git
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here
APP_ENV=development
LOG_LEVEL=DEBUG
```

Thêm vào `.gitignore` ngay:

```bash
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
```

Tạo `.env.example` để team members biết cần set gì (không có values thực):

```bash
# .env.example — commit file này lên Git
ANTHROPIC_API_KEY=your-anthropic-api-key-here
APP_ENV=development
LOG_LEVEL=DEBUG
```

**Cách 3 — Load .env trong Java code với dotenv-java:**

Thêm dependency:

```xml
<dependency>
    <groupId>io.github.cdimascio</groupId>
    <artifactId>dotenv-java</artifactId>
    <version>3.0.0</version>
</dependency>
```

Sử dụng trong code:

```java
import io.github.cdimascio.dotenv.Dotenv;

public class Config {
    private static final Dotenv dotenv = Dotenv.configure()
            .ignoreIfMissing()  // Không throw error nếu .env không tồn tại (production)
            .load();

    public static String get(String key) {
        // Ưu tiên environment variable thực > .env file
        String value = System.getenv(key);
        return value != null ? value : dotenv.get(key);
    }
}
```

### 7.3 Gitignore Checklist

`.gitignore` tối thiểu cho Java project với AI:

```gitignore
# Build outputs
target/
build/
*.class
*.jar

# IDE files
.idea/
*.iml
.vscode/

# Environment & Secrets — QUAN TRỌNG
.env
.env.local
.env.*.local

# API keys và credentials
secrets/
credentials/
*.pem
*.key

# OS files
.DS_Store
Thumbs.db
```

---

## 8. Verify Toàn Bộ Setup

Chạy lần lượt từng lệnh sau. Tất cả phải pass trước khi tiếp tục sang bài tiếp theo.

```bash
# ✅ 1. Java version (phải >= 17)
java -version
# openjdk version "21.x.x" ...

# ✅ 2. Maven hoặc Gradle
mvn -version
# Apache Maven 3.x.x

# ✅ 3. Claude Code CLI
claude --version
# claude v1.x.x

# ✅ 4. Node.js (phải >= 18)
node --version
# v22.x.x

# ✅ 5. npm
npm --version
# 10.x.x

# ✅ 6. Python (phải >= 3.11)
python --version
# Python 3.12.x

# ✅ 7. Anthropic API Key được set
echo $ANTHROPIC_API_KEY
# sk-ant-api03-... (phải có output, không được blank)

# ✅ 8. Anthropic API trả về 200
curl -s -o /dev/null -w "%{http_code}" https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
# Output: 200

# ✅ 9. HelloClaude.java chạy thành công
cd hello-claude && mvn exec:java -Dexec.mainClass="com.yourname.agenticai.HelloClaude" -q
# === Response from Claude ===
# ...

# ✅ 10. Claude Code mở được
claude --version && echo "Claude Code: OK"
```

**Tất cả 10 checks pass? Bạn sẵn sàng cho Module 01.**

---

## 9. Troubleshooting Thường Gặp

### Problem: API key không được nhận — `AuthenticationError`

```
com.anthropic.errors.AuthenticationException: 401 Unauthorized
```

**Nguyên nhân và cách fix:**

```bash
# Kiểm tra key có trong environment không
echo $ANTHROPIC_API_KEY
# Nếu blank → key chưa được export

# Fix: export lại (chú ý dùng export, không chỉ gán)
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# Kiểm tra format key — phải bắt đầu bằng "sk-ant-"
echo $ANTHROPIC_API_KEY | cut -c1-7
# Output: sk-ant-

# Nếu đã add vào ~/.zshrc nhưng vẫn không thấy → reload shell
source ~/.zshrc

# Hoặc restart terminal hoàn toàn
```

### Problem: Claude Code lỗi permission khi npm install

```
npm error code EACCES
npm error permission denied, access '/usr/local/lib/node_modules'
```

**Nguyên nhân:** Cài Node.js trực tiếp (không dùng nvm) → npm global directory thuộc về root.

```bash
# Fix 1 (Recommended): Dùng nvm — không cần sudo
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.zshrc
nvm install 22 && nvm use 22
npm install -g @anthropic-ai/claude-code  # Không cần sudo nữa

# Fix 2: Đổi npm global directory
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.zshrc
source ~/.zshrc
npm install -g @anthropic-ai/claude-code
```

### Problem: Maven không tìm thấy anthropic-java artifact

```
Could not find artifact com.anthropic:anthropic-java:jar:0.8.0
```

**Nguyên nhân:** Maven Central chưa sync, hoặc version sai.

```bash
# Kiểm tra version mới nhất trên Maven Central
curl -s "https://search.maven.org/solrsearch/select?q=g:com.anthropic+a:anthropic-java&rows=5&wt=json" \
  | python3 -m json.tool | grep '"latestVersion"'

# Force update Maven local repository
mvn dependency:resolve -U

# Clear Maven cache nếu vẫn lỗi
rm -rf ~/.m2/repository/com/anthropic
mvn dependency:resolve
```

### Problem: Rate limit ngay từ lần đầu — `RateLimitError`

```
com.anthropic.errors.RateLimitException: 429 Too Many Requests
```

**Nguyên nhân:** Free tier có rate limit thấp: 5 requests/minute, 10,000 tokens/minute.

```bash
# Kiểm tra usage trong console
# console.anthropic.com → Usage → xem requests/tokens trong ngày

# Fix trong code — thêm retry với exponential backoff:
```

```java
// Thêm vào HelloClaude.java
import java.time.Duration;

// Trong main():
int maxRetries = 3;
for (int attempt = 0; attempt < maxRetries; attempt++) {
    try {
        Message message = client.messages().create(params);
        // process message...
        break;
    } catch (com.anthropic.errors.RateLimitException e) {
        if (attempt < maxRetries - 1) {
            long waitMs = (long) Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
            System.out.println("Rate limited. Waiting " + waitMs + "ms...");
            Thread.sleep(waitMs);
        } else {
            throw e;
        }
    }
}
```

### Problem: `java.lang.ClassNotFoundException` khi chạy mvn exec

```bash
# Đảm bảo compile trước khi run
mvn compile exec:java -Dexec.mainClass="com.yourname.agenticai.HelloClaude"

# Hoặc package trước
mvn package -DskipTests
java -cp target/hello-claude-1.0-SNAPSHOT.jar:target/dependency/* \
  com.yourname.agenticai.HelloClaude
```

### Problem: Python `ModuleNotFoundError: No module named 'anthropic'`

```bash
# Kiểm tra virtual environment đã activate chưa
which python
# Phải là: /path/to/project/.venv/bin/python
# Không phải: /usr/bin/python

# Activate lại
source .venv/bin/activate  # macOS/Linux
.venv\Scripts\activate     # Windows

# Cài lại
pip install anthropic
```

---

## 10. Exercise

### SystemCheck.java — Bài tập tổng kết setup

Viết chương trình Java `SystemCheck.java` thực hiện các yêu cầu sau:

**Yêu cầu:**
1. Gọi Claude API với prompt về agentic AI cho Java engineers
2. Parse và format response đẹp
3. Handle exceptions properly (không để crash thô)
4. Commit lên GitHub với `.env` trong `.gitignore`

**Template để bắt đầu:**

```java
package com.yourname.agenticai;

import com.anthropic.client.Anthropic;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.errors.AnthropicException;
import com.anthropic.models.messages.Message;
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.Model;

public class SystemCheck {

    public static void main(String[] args) {
        System.out.println("=".repeat(60));
        System.out.println("  Agentic AI Course — System Check");
        System.out.println("=".repeat(60));

        // 1. Verify environment
        String apiKey = System.getenv("ANTHROPIC_API_KEY");
        if (apiKey == null || apiKey.isBlank()) {
            System.err.println("❌ ERROR: ANTHROPIC_API_KEY not set");
            System.err.println("   Run: export ANTHROPIC_API_KEY='sk-ant-...'");
            System.exit(1);
        }
        System.out.println("✅ API Key     : " + apiKey.substring(0, 12) + "...[hidden]");
        System.out.println("✅ Java Version: " + System.getProperty("java.version"));

        // 2. Gọi Claude API
        try {
            Anthropic client = AnthropicOkHttpClient.fromEnv();

            String prompt = """
                    I am a Java backend engineer with 5 years of experience, 
                    just starting to learn agentic AI development.
                    
                    List the 3 most important things I need to understand 
                    as a Java engineer learning agentic AI.
                    
                    Format your response as a numbered list with a brief 
                    explanation for each point. Be specific and practical.
                    """;

            MessageCreateParams params = MessageCreateParams.builder()
                    .model(Model.CLAUDE_HAIKU_4_5)
                    .maxTokens(512)
                    .addUserMessage(prompt)
                    .build();

            System.out.println("\n⏳ Calling Claude API...\n");
            Message message = client.messages().create(params);

            // 3. Format và print response
            String response = message.content().get(0).text().get().text();

            System.out.println("📋 Claude's advice for Java engineers learning Agentic AI:");
            System.out.println("-".repeat(60));
            System.out.println(response);
            System.out.println("-".repeat(60));

            // 4. Print usage stats
            System.out.printf("%n📊 Token Usage:%n");
            System.out.printf("   Input  : %d tokens%n", message.usage().inputTokens());
            System.out.printf("   Output : %d tokens%n", message.usage().outputTokens());
            System.out.printf("   Model  : %s%n", message.model());

            System.out.println("\n✅ System check PASSED — bạn sẵn sàng cho khóa học!");

        } catch (AnthropicException e) {
            System.err.println("\n❌ API Error: " + e.getMessage());
            System.err.println("   Status code: " + e.statusCode());
            System.err.println("   Kiểm tra API key và network connection");
            System.exit(1);
        } catch (Exception e) {
            System.err.println("\n❌ Unexpected error: " + e.getMessage());
            e.printStackTrace();
            System.exit(1);
        }
    }
}
```

**Chạy:**

```bash
mvn compile exec:java -Dexec.mainClass="com.yourname.agenticai.SystemCheck"
```

**Commit lên GitHub:**

```bash
# Khởi tạo Git repo
git init
git add .gitignore
git add pom.xml src/
# KHÔNG add .env

# Verify .env không bị track
git status
# Phải thấy .env trong "untracked files" hoặc không hiện — KHÔNG phải trong "Changes to be committed"

git commit -m "feat: add HelloClaude and SystemCheck setup exercises"

# Tạo repo trên GitHub và push
gh repo create agentic-ai-course --public --source=. --push
# Hoặc tạo manual trên github.com rồi:
git remote add origin https://github.com/yourusername/agentic-ai-course.git
git push -u origin main
```

### Bonus Challenge

Nếu xong sớm, extend `SystemCheck.java`:

1. **Multi-turn conversation:** Sau câu trả lời đầu tiên, hỏi follow-up: "Give me a concrete Java code example for point #1"
2. **Streaming response:** Dùng `client.messages().stream()` thay vì `.create()` để print response từng chunk như ChatGPT
3. **Model comparison:** Gọi cùng prompt với cả `claude-haiku-4-5` và `claude-sonnet-4-5`, so sánh response quality và token cost

---

## Tóm tắt

Trong bài này, bạn đã:

- ✅ Set up Anthropic API key an toàn qua environment variable
- ✅ Cài Java 17+, Maven/Gradle và Anthropic Java SDK
- ✅ Viết và chạy `HelloClaude.java` — Java program đầu tiên gọi Claude API
- ✅ Cài Claude Code CLI và làm quen với basic commands
- ✅ Cài Node.js (cần cho MCP servers ở Module 03)
- ✅ Cài Python (cần cho Agent SDK ở Module 02)
- ✅ Configure IntelliJ IDEA với `.env` file workflow
- ✅ Nắm troubleshooting các lỗi thường gặp nhất

**Bài tiếp theo:** Chuyển sang [Module 01 — Claude API Java](../../module-01-claude-api-java/README.md) để bắt đầu gọi Claude API từ Java.

---

*Module 00 — Foundation | Bài 03/05*
