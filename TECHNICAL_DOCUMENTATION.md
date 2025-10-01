# Fireworks AI BrowserUse - Technical Documentation

## 📋 Project Overview

**What it does**: Autonomous AI agent that controls a web browser to navigate, extract content, and perform web automation tasks.

**Key Innovation**: Combines LLM reasoning (Fireworks AI) with browser automation (Playwright) to create an agent that can "see" web pages via screenshots and make intelligent decisions.

---

## 🏗️ Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **AI Models** | Fireworks AI (DeepSeek-V3, FireLLaVA-13b) | High-performance inference, multimodal vision |
| **Browser** | Playwright + browser-use | Stealth automation, cross-browser support |
| **Backend** | FastAPI + Python 3.11-3.13 | Async support, modern API |
| **Validation** | Pydantic | Type safety, automatic validation |
| **Config** | TOML | Human-readable configuration |
| **Retry** | Tenacity | Exponential backoff for failures |
| **Tokens** | Tiktoken | Cost management, token counting |

---

## 🎯 Core Design Patterns

### 1. Agent Pattern (Autonomous AI)
```python
# File: app/agent/base.py
class BaseAgent:
    state: AgentState = AgentState.IDLE
    memory: Memory = Memory()
    
    async def run(self, request: str):
        self.update_memory("user", request)
        
        while self.current_step < self.max_steps:
            await self.step()  # Think + Act
            if self.state == AgentState.FINISHED:
                break
```

### 2. ReAct Pattern (Reasoning + Acting)
```python
# File: app/agent/react.py
async def step(self):
    should_act = await self.think()  # Decide
    if should_act:
        result = await self.act()    # Execute
    return result
```

### 3. Function Calling (Tool Use)
```python
# File: app/agent/toolcall.py
async def think(self):
    response = await self.llm.ask_tool(
        messages=self.messages,
        tools=self.available_tools.to_params(),
        tool_choice="auto"  # LLM decides which tool
    )
    self.tool_calls = response.tool_calls
```

### 4. Memory Management
```python
# File: app/schema.py
class Message(BaseModel):
    role: Literal["system", "user", "assistant", "tool"]
    content: Optional[str]
    base64_image: Optional[str]  # Screenshots!

class Memory:
    messages: List[Message] = []
    max_messages: int = 100
```

---

## 🔄 Execution Flow

```
User Request: "Go to amazon.com and find iPhone prices"
    ↓
1. THINK: LLM sees request → Decides to use browser_use(go_to_url)
2. ACT: Navigate to amazon.com → Take screenshot
3. THINK: LLM sees Amazon homepage → Decides to click search box
4. ACT: Click element → Take screenshot
5. THINK: LLM sees active search → Decides to type "iPhone"
6. ACT: Input text → Press Enter → Take screenshot
7. THINK: LLM sees results → Decides to extract prices
8. ACT: Extract content using LLM
9. THINK: LLM sees extracted data → Decides to terminate
10. ACT: Terminate → Return results
```

---

## 🛠️ Key Implementation Details

### Browser Initialization
```python
# File: app/tool/browser_use_tool.py
async def _ensure_browser_initialized(self):
    browser_config = {
        "headless": False,  # Visible browser (anti-detection)
        "disable_security": True,
        "extra_chromium_args": [
            "--disable-blink-features=AutomationControlled"  # Hide automation
        ]
    }
    
    self.browser = BrowserUseBrowser(BrowserConfig(**browser_config))
    self.context = await self.browser.new_context()
```

### Content Extraction
```python
# File: app/tool/browser_use_tool.py
async def _extract_content(self, goal: str):
    # Get page text
    page_text = await self.context.page.inner_text("body")
    
    # Build prompt
    prompt = f"Extract: {goal}\n\nPage content:\n{page_text[:5000]}"
    
    # Use LLM to extract
    result = await self.llm.ask(messages=[Message.user_message(prompt)])
    
    # Take screenshot
    screenshot = await self.context.page.screenshot()
    
    return ToolResult(output=result, base64_image=base64.b64encode(screenshot))
```

### LLM API Call
```python
# File: app/llm.py
@retry(wait=wait_random_exponential(min=1, max=60), stop=stop_after_attempt(6))
async def ask_tool(self, messages, tools, tool_choice="auto"):
    # Count tokens
    input_tokens = self.count_message_tokens(messages)
    if not self.check_token_limit(input_tokens):
        raise TokenLimitExceeded()
    
    # API call
    response = await self.client.chat.completions.create(
        model=self.model,
        messages=messages,
        tools=tools,
        tool_choice=tool_choice
    )
    
    # Track usage
    self.update_token_count(response.usage.prompt_tokens, response.usage.completion_tokens)
    
    return response.choices[0].message
```

---

## 🚨 Problem-Solving Techniques

### Challenge 1: Bot Detection

**Problem**: Websites block automated browsers (Cloudflare, etc.)

**Solutions**:
1. **Non-headless mode**: `headless = false` (full browser features)
2. **Remove automation flags**: `--disable-blink-features=AutomationControlled`
3. **Realistic user agent**: Custom UA string
4. **Proxy support**: Residential IPs to avoid datacenter detection
5. **Natural delays**: `await asyncio.sleep()` between actions
6. **Session persistence**: Maintain cookies across requests
7. **Real Chrome profile**: Use actual Chrome with existing cookies/logins

```toml
# config/config.toml
[browser]
headless = false
disable_security = true
extra_chromium_args = [
    "--disable-blink-features=AutomationControlled",
    "--user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)..."
]

[browser.proxy]
server = "http://residential-proxy.com:8080"
username = "user"
password = "pass"
```

### Challenge 2: Token Limits & Cost

**Solutions**:
1. **Token counting**: Pre-flight checks before API calls
2. **Observation limits**: Truncate long outputs (`max_observe = 10000`)
3. **Memory management**: Keep only recent 100 messages
4. **Token tracking**: Monitor cumulative usage

```python
# Check before API call
if not self.check_token_limit(input_tokens):
    raise TokenLimitExceeded()

# Truncate results
if self.max_observe:
    result = result[:self.max_observe]
```

### Challenge 3: API Failures

**Solutions**:
1. **Exponential backoff**: Retry with increasing delays (1s, 2s, 4s, 8s, 16s, 32s, 60s)
2. **Error handling**: Graceful degradation on failures
3. **Don't retry token errors**: Immediately fail on TokenLimitExceeded

```python
@retry(
    wait=wait_random_exponential(min=1, max=60),
    stop=stop_after_attempt(6),
    retry=retry_if_exception_type((OpenAIError, Exception))
)
async def ask_tool(self, ...):
    # API call with automatic retry
```

### Challenge 4: Agent Stuck in Loops

**Solutions**:
1. **Stuck detection**: Count duplicate responses
2. **Intervention**: Add prompt to change strategy
3. **Max steps**: Hard limit of 20 steps

```python
def is_stuck(self) -> bool:
    last_message = self.memory.messages[-1]
    duplicate_count = sum(
        1 for msg in self.memory.messages[:-1]
        if msg.content == last_message.content
    )
    return duplicate_count >= 2

def handle_stuck_state(self):
    stuck_prompt = "Observed duplicate responses. Try new strategies."
    self.next_step_prompt = f"{stuck_prompt}\n{self.next_step_prompt}"
```

---

## ⚙️ Configuration

### config/config.toml
```toml
[llm]
model = "accounts/fireworks/models/deepseek-v3"
base_url = "https://api.fireworks.ai/inference/v1"
api_key = "your-api-key"
max_tokens = 4096
temperature = 0.0

[llm.vision]
model = "accounts/fireworks/models/firellava-13b"

[browser]
headless = false
disable_security = true
extra_chromium_args = ["--disable-blink-features=AutomationControlled"]

[browser.proxy]
server = "http://proxy.com:8080"
username = "user"
password = "pass"
```

---

## 🚀 Usage

```bash
# Install dependencies
pip install -r requirements.txt
playwright install

# Configure
cp config/config.example.toml config/config.toml
# Edit config.toml with your Fireworks API key

# Run
python main.py
# Enter: "Go to amazon.com and find iPhone prices"
```

---

## 📊 Architecture Summary

```
User Input
    ↓
Manus Agent (orchestrates everything)
    ├─ BaseAgent: State management, execution loop
    ├─ ReActAgent: Think-Act cycle
    ├─ ToolCallAgent: LLM function calling
    └─ BrowserAgent: Browser state injection
    ↓
Tool Collection
    ├─ BrowserUseTool: Web automation
    ├─ PythonExecute: Code execution
    └─ Terminate: End execution
    ↓
Fireworks AI LLM
    ├─ Function calling (tool selection)
    ├─ Vision support (screenshots)
    └─ Token management
    ↓
Playwright Browser
    ├─ Stealth features
    ├─ Screenshot capture
    └─ Session persistence
    ↓
Target Website
```

---

## 🎯 Key Takeaways

1. **Autonomous**: Agent decides what to do based on context, not hardcoded logic
2. **Multimodal**: LLM can "see" web pages via screenshots
3. **Resilient**: Retry logic, error handling, stuck detection
4. **Stealthy**: Anti-detection techniques for accessing protected sites
5. **Cost-aware**: Token counting and limits prevent runaway costs
6. **Extensible**: Easy to add new tools via Strategy pattern

---

**End of Documentation**
