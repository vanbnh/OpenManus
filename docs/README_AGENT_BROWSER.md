# Phân tích OpenManus: Cách Agent tương tác với Website

Tài liệu này mô tả cấu trúc project OpenManus và cơ chế agent tương tác với website (browser).

---

## 1. Tổng quan project OpenManus

OpenManus là framework agent với kiến trúc kế thừa:

- **BaseAgent** → **ReActAgent** (think → act) → **ToolCallAgent** (gọi tool qua LLM) → các agent cụ thể.
- Hai luồng chạy: **main.py** (chạy trực tiếp Manus) và **run_flow.py** (PlanningFlow với nhiều agent).
- Agent chính: **Manus** (đa năng, có browser + MCP + editor…), **BrowserAgent** (chuyên browser), **SandboxManus** (trong sandbox).

---

## 2. Cách agent tương tác với website

Tương tác với web được thực hiện qua **browser tools** và **browser agent**.

### 2.1. Hai hướng tương tác web

| Thành phần | Mô tả |
|------------|--------|
| **BrowserUseTool** (`app/tool/browser_use_tool.py`) | Tool dùng thư viện **browser_use** (Playwright), chạy browser thật (Chromium). |
| **SandboxBrowserTool** (`app/tool/sandbox/sb_browser_tool.py`) | Tool browser chạy trong **Daytona sandbox**, gọi API sandbox để thao tác browser. |

- **Manus** dùng `BrowserUseTool` như một tool trong bộ tool (PythonExecute, StrReplaceEditor, AskHuman, Terminate…).
- **BrowserAgent** dùng `BrowserUseTool` (hoặc `SandboxBrowserTool`) làm tool chính và có thêm **BrowserContextHelper** để đưa trạng thái browser vào prompt mỗi bước.

### 2.2. Luồng tương tác (ReAct + Tool Call)

1. **Think**: LLM nhận system prompt + memory (bao gồm kết quả tool trước) + (với browser) **trạng thái browser hiện tại** (URL, title, tabs, **cây phần tử tương tác**, ảnh màn hình). LLM quyết định **tool** và **thông số** (action, url, index, text…).
2. **Act**: Execute tool (ví dụ `browser_use` với `action="go_to_url"`, `url="..."`).
3. **Observe**: Kết quả tool (và ảnh screenshot nếu có) được đưa lại vào memory; bước tiếp theo lại **think** với context mới.

Với **BrowserAgent**, trước mỗi lần think, **BrowserContextHelper** gọi `get_current_state()` của browser tool để lấy state (DOM đã gán index cho phần tử clickable, screenshot…) rồi format vào `next_step_prompt` và đưa ảnh vào message; vì vậy agent “thấy” trang web hiện tại trước khi quyết định hành động tiếp.

### 2.3. Browser state và cách agent “nhìn” trang

Trong `BrowserUseTool.get_current_state()`:

- Gọi `context.get_state()` (browser_use) để lấy URL, title, tabs, **element tree** (các phần tử có thể click với **chỉ số index**).
- Chụp screenshot (base64).
- Trả về JSON gồm: `url`, `title`, `tabs`, `interactive_elements` (chuỗi mô tả phần tử kèm index, ví dụ `[33]<button>Submit</button>`), `scroll_info` (pixels above/below viewport).

Prompt browser (`app/prompt/browser.py`) quy định: chỉ các phần tử có **index trong []** mới được dùng để click/input; agent ra lệnh theo index (ví dụ `click_element` với `index=33`).

### 2.4. Các action browser (BrowserUseTool)

- **Điều hướng**: `go_to_url`, `go_back`, `web_search` (search rồi mở result đầu).
- **Tương tác**: `click_element`, `input_text`, `send_keys`, `get_dropdown_options`, `select_dropdown_option`.
- **Cuộn**: `scroll_down`, `scroll_up`, `scroll_to_text`.
- **Tab**: `switch_tab`, `open_tab`, `close_tab`.
- **Nội dung**: `extract_content` (HTML → markdown, rồi dùng LLM để extract theo `goal`).
- **Khác**: `wait`.

Thực thi thật sự dùng **browser_use** (Playwright): `page.goto`, `context.get_dom_element_by_index`, `_click_element_node`, `_input_text_element_node`, `execute_javascript` (scroll), v.v.

### 2.5. Tích hợp browser vào Manus

Trong **Manus**:

- Có `BrowserContextHelper` giống BrowserAgent.
- Trong `think()`, nếu phát hiện **browser đang được dùng** (trong vài message gần nhất có tool call tới `BrowserUseTool`), Manus sẽ gọi `browser_context_helper.format_next_step_prompt()` để bổ sung **trạng thái browser hiện tại** (và ảnh) vào prompt; sau đó gọi `super().think()`.
- Nhờ đó khi user bảo “vào trang X và điền form”, Manus vừa có tool `browser_use` vừa có context trang hiện tại để quyết định bước tiếp theo (ví dụ click index 5, input index 7…).

---

## 3. Sơ đồ luồng (tóm tắt)

```
User request
    ↓
BaseAgent.run() → vòng lặp step():
    ↓
ReActAgent.step() → think() → act()
    ↓
ToolCallAgent.think():
  - (Manus/BrowserAgent) BrowserContextHelper.format_next_step_prompt()
    → get_current_state() từ BrowserUseTool
    → URL, title, tabs, interactive_elements (index), screenshot
  - LLM.ask_tool(messages, tools=...) → tool_calls (vd: browser_use, action=..., index=...)
    ↓
ToolCallAgent.act():
  - execute_tool(browser_use, { action, url, index, text, ... })
    ↓
BrowserUseTool.execute():
  - _ensure_browser_initialized() (Playwright/BrowserUseBrowser)
  - Thực hiện action (go_to_url, click_element, input_text, extract_content, ...)
  - Trả về ToolResult (output, có thể kèm base64_image)
    ↓
Kết quả đưa vào memory → bước step tiếp theo (think lại với context mới)
```

---

## 4. Cấu trúc file liên quan

| File | Vai trò |
|------|--------|
| `app/agent/base.py` | BaseAgent: run loop, memory, state. |
| `app/agent/react.py` | ReActAgent: step = think + act. |
| `app/agent/toolcall.py` | ToolCallAgent: think (LLM + tool_calls), act (execute_tool). |
| `app/agent/browser.py` | BrowserAgent, BrowserContextHelper. |
| `app/agent/manus.py` | Manus: bộ tool có BrowserUseTool, tích hợp BrowserContextHelper khi dùng browser. |
| `app/tool/browser_use_tool.py` | BrowserUseTool: khởi tạo browser_use, execute(action,…), get_current_state(), cleanup. |
| `app/tool/sandbox/sb_browser_tool.py` | SandboxBrowserTool: browser trong Daytona sandbox. |
| `app/prompt/browser.py` | SYSTEM_PROMPT, NEXT_STEP_PROMPT cho browser agent. |

---

## 5. Tài liệu bổ sung

- **[Tools & Playwright Mapping](docs/TOOLS_PLAYWRIGHT_MAPPING.md)** – Chi tiết từng tool/action, code path và ánh xạ xuống API Playwright (qua browser-use). Dùng khi cần xoáy sâu vào lớp giao tiếp agent ↔ browser.

---

## 6. Kết luận

- **Cách agent tương tác với website**: qua **browser tool** (`BrowserUseTool` hoặc `SandboxBrowserTool`), với các **action** rõ ràng (go_to_url, click_element, input_text, scroll, extract_content, tab, …). Tool dùng thư viện **browser_use** (Playwright) để điều khiển browser thật.
- **Cách agent “hiểu” trang**: mỗi bước (hoặc khi đang dùng browser trong Manus), agent nhận **browser state** (URL, title, danh sách phần tử tương tác có **index**, ảnh màn hình) qua `get_current_state()` và prompt được format bởi **BrowserContextHelper**.
- **Quyết định hành động**: LLM dựa trên state đó để chọn **action** và **thông số** (index, text, url…); ToolCallAgent gọi tool tương ứng; kết quả và ảnh (nếu có) quay lại memory cho bước tiếp theo.

Xem thêm **[TOOLS_PLAYWRIGHT_MAPPING.md](docs/TOOLS_PLAYWRIGHT_MAPPING.md)** để biết từng action gọi hàm Playwright/browser-use cụ thể nào.
