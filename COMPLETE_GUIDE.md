# Fireworks AI BrowserUse - Complete Technical Guide
**A Comprehensive Deep Dive with All Code Snippets and Workflows**

Generated from detailed technical conversation - October 2025

---

## 📋 Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture & Tech Stack](#2-architecture--tech-stack)  
3. [Core Design Patterns](#3-core-design-patterns)
4. [Complete Execution Flow](#4-complete-execution-flow)
5. [How Content Extraction Works](#5-how-content-extraction-works)
6. [Bot Detection Solutions](#6-bot-detection-solutions)
7. [Problem-Solving Techniques](#7-problem-solving-techniques)
8. [Configuration](#8-configuration)

---

## 1. Project Overview

### What It Does
Autonomous AI agent that controls a web browser to:
- Navigate any website (Amazon, Google, etc.)
- Extract content intelligently using AI
- Perform SEO analysis
- Automate web interactions

### Key Innovation
Combines **LLM reasoning** (Fireworks AI) with **browser automation** (Playwright) to create an agent that can "see" web pages via screenshots and make intelligent decisions.

---

## 2. Architecture & Tech Stack

### Technology Stack

| Component | Technology | Why Used |
|-----------|-----------|----------|
| **AI Models** | Fireworks AI (DeepSeek-V3, FireLLaVA-13b) | High-performance, cost-effective, multimodal |
| **Browser** | Playwright + browser-use | Stealth features, reliable automation |
| **Backend** | FastAPI + Python 3.11-3.13 | Async support, modern framework |
| **Validation** | Pydantic | Type safety, automatic validation |
| **Config** | TOML | Human-readable configuration |
| **Retry** | Tenacity | Exponential backoff for failures |
| **Tokens** | Tiktoken | Cost management, token counting |

### Architecture Flow

```
User Input: "Go to amazon.com and find iPhone prices"
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
Target Website (Amazon, Google, etc.)
```

---

## 3. Core Design Patterns

### 3.1 Agent Pattern

**Autonomous entity that perceives, reasons, and acts**

```python
# File: app/agent/base.py
class BaseAgent(BaseModel, ABC):
    state: AgentState = AgentState.IDLE
    memory: Memory = Memory()
    current_step: int = 0
    max_steps: int = 10
    
    async def run(self, request: str):
        self.update_memory("user", request)
        
        async with self.state_context(AgentState.RUNNING):
            while self.current_step < self.max_steps:
                self.current_step += 1
                await self.step()  # Think + Act
                
                if self.is_stuck():
                    self.handle_stuck_state()
                
                if self.state == AgentState.FINISHED:
                    break
```

### 3.2 ReAct Pattern

**Alternates between reasoning and acting**

```python
# File: app/agent/react.py
async def step(self):
    should_act = await self.think()  # Decide
    if should_act:
        result = await self.act()    # Execute
    return result
```

### 3.3 Function Calling

**LLM returns structured function calls**

```python
# File: app/agent/toolcall.py
async def think(self):
    response = await self.llm.ask_tool(
        messages=self.messages,
        tools=self.available_tools.to_params(),
        tool_choice="auto"  # LLM decides
    )
    self.tool_calls = response.tool_calls
```

**Tool Definition:**

```python
# File: app/tool/browser_use_tool.py
class BrowserUseTool(BaseTool):
    name = "browser_use"
    description = "Control web browser..."
    parameters = {
        "type": "object",
        "properties": {
            "action": {"type": "string", "enum": ["go_to_url", "click_element", ...]},
            "url": {"type": "string"},
            "index": {"type": "integer"}
        }
    }
```

### 3.4 Memory Management

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

## 4. Complete Execution Flow

### Example: "Go to amazon.com and find iPhone prices"

#### Step 1: User Input
```python
# File: main.py
agent = Manus()
await agent.run("Go to amazon.com and find iPhone prices")
```

#### Step 2: First THINK Phase
```python
# File: app/agent/browser.py
async def think(self):
    # Get browser state
    browser_state = await self.get_browser_state()
    
    # Add screenshot to memory
    if self._current_base64_image:
        self.memory.add_message(
            Message.user_message(
                content="Current screenshot:",
                base64_image=self._current_base64_image
            )
        )
    
    # Ask LLM what to do
    response = await self.llm.ask_tool(
        messages=self.messages,
        tools=self.available_tools.to_params()
    )
    
    # LLM decides: "Use browser_use to go to amazon.com"
    self.tool_calls = response.tool_calls
```

#### Step 3: First ACT Phase
```python
# File: app/agent/toolcall.py
async def act(self):
    for command in self.tool_calls:
        result, terminated = await self.execute_tool(command)
        
        tool_msg = Message.tool_message(
            content=result,
            tool_call_id=command.id,
            base64_image=self._current_base64_image
        )
        self.memory.add_message(tool_msg)
```

#### Step 4: Browser Navigation
```python
# File: app/tool/browser_use_tool.py
async def _go_to_url(self, url: str):
    await self.context.page.goto(url)
    await self.context.page.wait_for_load_state("networkidle")
    
    screenshot = await self.context.page.screenshot()
    base64_image = base64.b64encode(screenshot).decode()
    
    return ToolResult(
        output=f"Successfully navigated to {url}",
        base64_image=base64_image
    )
```

#### Step 5-10: Repeat Think-Act Cycle

1. THINK: See Amazon homepage → Click search box
2. ACT: Click element → Screenshot
3. THINK: See active search → Type "iPhone"
4. ACT: Input text → Screenshot
5. THINK: See results → Extract prices
6. ACT: Extract content
7. THINK: See data → Terminate
8. ACT: End execution

---

## 5. How Content Extraction Works

### Two-Stage Process

**Stage 1: Browser Captures Page State**

```python
# File: app/tool/browser_use_tool.py
async def _extract_content(self, goal: str):
    # Get page text
    page_text = await self.context.page.inner_text("body")
    
    # Get element tree (DOM with indices)
    state = await self.context.get_state()
    element_tree = state.element_tree
    # Example:
    # [1] <input> Search box
    # [2] <button> Search
    # [3] <a> iPhone 15 Pro - $999
```

**Stage 2: LLM Analyzes and Extracts**

```python
    # Build extraction prompt
    prompt = f"""
    Extract: {goal}
    
    Page content:
    {page_text[:5000]}
    
    Interactive elements:
    {element_tree[:2000]}
    """
    
    # Use LLM to extract
    result = await self.llm.ask(
        messages=[Message.user_message(prompt)]
    )
    
    # LLM returns:
    # "Found 3 products:
    #  1. iPhone 15 Pro - $999.00
    #  2. iPhone 15 - $799.00
    #  3. iPhone 14 - $699.00"
    
    screenshot = await self.context.page.screenshot()
    
    return ToolResult(
        output=result,
        base64_image=base64.b64encode(screenshot)
    )
```

### Why This Works

- **LLM as intelligent parser**: Understands content semantically
- **Multimodal**: Can "see" screenshots for visual context
- **Natural language goals**: No brittle CSS selectors
- **Adaptive**: Works on any website

---

## 6. Bot Detection Solutions

### Problem: Websites Block Bots

Detection techniques:
- Headless browser detection
- `navigator.webdriver` flag
- Missing browser properties
- Behavioral patterns
- IP/fingerprinting

### Solutions Implemented

#### 1. Non-Headless Mode
```toml
# config/config.toml
[browser]
headless = false  # Visible browser window
```

#### 2. Remove Automation Flags
```toml
extra_chromium_args = [
    "--disable-blink-features=AutomationControlled",
    "--user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)..."
]
```

#### 3. Proxy Support
```toml
[browser.proxy]
server = "http://residential-proxy.com:8080"
username = "user"
password = "pass"
```

```python
# File: app/tool/browser_use_tool.py
if config.browser_config.proxy:
    browser_config_kwargs["proxy"] = ProxySettings(
        server=config.browser_config.proxy.server,
        username=config.browser_config.proxy.username,
        password=config.browser_config.proxy.password
    )
```

#### 4. Natural Delays
```python
async def _click_element(self, index: int):
    await self.context.page.click(element_selector)
    await self.context.page.wait_for_load_state("networkidle")
    await asyncio.sleep(0.5)  # Human-like pause
```

#### 5. Use Real Chrome Profile
```toml
[browser]
chrome_instance_path = "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

---

## 7. Problem-Solving Techniques

### Challenge 1: Token Limits

**Solutions:**

```python
# 1. Token counting before API call
input_tokens = self.count_message_tokens(messages)
if not self.check_token_limit(input_tokens):
    raise TokenLimitExceeded()

# 2. Truncate long outputs
if self.max_observe:
    result = result[:self.max_observe]

# 3. Memory management
if len(self.messages) > self.max_messages:
    self.messages = self.messages[-self.max_messages:]
```

### Challenge 2: API Failures

**Solutions:**

```python
# Exponential backoff retry
@retry(
    wait=wait_random_exponential(min=1, max=60),
    stop=stop_after_attempt(6),
    retry=retry_if_exception_type((OpenAIError, Exception))
)
async def ask_tool(self, ...):
    response = await self.client.chat.completions.create(...)
    return response
```

### Challenge 3: Agent Stuck in Loops

**Solutions:**

```python
def is_stuck(self) -> bool:
    last_message = self.memory.messages[-1]
    duplicate_count = sum(
        1 for msg in self.memory.messages[:-1]
        if msg.content == last_message.content
    )
    return duplicate_count >= 2

def handle_stuck_state(self):
    stuck_prompt = "Try new strategies. Avoid repeating."
    self.next_step_prompt = f"{stuck_prompt}\n{self.next_step_prompt}"
```

---

## 8. Configuration

### Complete config.toml

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
extra_chromium_args = [
    "--disable-blink-features=AutomationControlled",
    "--user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"
]

[browser.proxy]
server = "http://proxy.com:8080"
username = "user"
password = "pass"
```

### Usage

```bash
# Install
pip install -r requirements.txt
playwright install

# Configure
cp config/config.example.toml config/config.toml
# Edit with your API key

# Run
python main.py
```

---

## 🎯 Key Takeaways

1. **Autonomous**: Agent decides based on context, not hardcoded logic
2. **Multimodal**: LLM can "see" web pages via screenshots
3. **Resilient**: Retry logic, error handling, stuck detection
4. **Stealthy**: Anti-detection for accessing protected sites
5. **Cost-aware**: Token counting prevents runaway costs
6. **Extensible**: Easy to add new tools

---

**End of Complete Technical Guide**
