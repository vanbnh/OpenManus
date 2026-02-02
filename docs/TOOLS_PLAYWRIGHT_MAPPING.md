# Tools & Functions: Giao tiếp Agent ↔ Playwright

Tài liệu này mô tả chi tiết **các tool/action** mà agent gọi và **cách chúng ánh xạ xuống Playwright** (qua thư viện **browser-use**). Dùng khi cần xoáy sâu vào lớp giao tiếp browser.

---

## 1. Tầng thư viện

| Tầng | Thư viện / Class | Vai trò |
|------|-------------------|--------|
| Agent | `BrowserUseTool`, `BrowserAgent`, `Manus` | Quyết định action (LLM) và gọi tool. |
| Tool | `app/tool/browser_use_tool.py` | Nhận `action` + params → gọi **browser_use** Context/Page. |
| Browser automation | **browser-use** (`browser_use`) | Context, DOM index, click/input; bên trong dùng **Playwright**. |
| Runtime | **Playwright** (`playwright`) | Điều khiển Chromium: Page, Locator, Keyboard, screenshot, v.v. |

- **OpenManus** không gọi Playwright trực tiếp; mọi thao tác browser đều đi qua **BrowserUseTool** → **browser_use** → Playwright.
- **browser-use** cung cấp: `Browser`, `BrowserConfig`, `BrowserContext`, `BrowserContextConfig`, `DomService`. Context có `get_current_page()` trả về **Playwright Page**.

---

## 2. Khởi tạo browser & context (Playwright được tạo bên trong)

**File:** `app/tool/browser_use_tool.py`

| Bước | Code OpenManus | API dùng | Ghi chú |
|------|----------------|----------|--------|
| 1 | `BrowserUseBrowser(BrowserConfig(**kwargs))` | `browser_use.Browser`, `BrowserConfig` | Cấu hình headless, proxy, chromium args, v.v. |
| 2 | `await self.browser.new_context(context_config)` | `BrowserContextConfig` | Tạo context (tương đương Playwright `browser.new_context()`). |
| 3 | `DomService(await self.context.get_current_page())` | `browser_use.dom.service.DomService` | Phục vụ DOM / element tree (dùng nội bộ browser_use). |

Sau bước 2, `self.context` là **browser_use.browser.context.BrowserContext**, bên trong giữ **Playwright BrowserContext** và **Page**. Mọi thao tác trang đều qua `context` hoặc `page = await context.get_current_page()`.

---

## 3. Bảng ánh xạ: Tool Action → Code → Playwright

Tool mà agent gọi là **`browser_use`**, với tham số `action` + các tham số kèm theo. Dưới đây là từng action, code tương ứng trong `BrowserUseTool.execute()`, và API Playwright/browser_use được dùng.

---

### 3.1. Điều hướng

| Action | Tham số tool | Hàm trong `BrowserUseTool.execute()` | API browser_use / Playwright |
|--------|--------------|--------------------------------------|------------------------------|
| **go_to_url** | `url` | `page = await context.get_current_page()` | **Playwright Page** |
| | | `await page.goto(url)` | `Page.goto(url)` |
| | | `await page.wait_for_load_state()` | `Page.wait_for_load_state()` |
| **go_back** | — | `await context.go_back()` | **browser_use Context** (gọi lại history/Playwright) |
| **refresh** | — | `await context.refresh_page()` | **browser_use Context** (refresh page hiện tại) |
| **web_search** | `query` | `search_response = await self.web_search_tool.execute(...)` | Không dùng Playwright cho search. |
| | | `page = await context.get_current_page()` | **Playwright Page** |
| | | `await page.goto(url_to_navigate)` | `Page.goto(url)` |
| | | `await page.wait_for_load_state()` | `Page.wait_for_load_state()` |

---

### 3.2. Tương tác phần tử (click, input, dropdown, keyboard)

| Action | Tham số tool | Hàm trong `execute()` | API browser_use / Playwright |
|--------|--------------|------------------------|------------------------------|
| **click_element** | `index` | `element = await context.get_dom_element_by_index(index)` | **browser_use Context**: lấy node DOM tương ứng index (có `.xpath`). |
| | | `download_path = await context._click_element_node(element)` | **browser_use Context**: thực hiện click (bên trong dùng Playwright Locator/click). |
| **input_text** | `index`, `text` | `element = await context.get_dom_element_by_index(index)` | **browser_use Context**: cùng cơ chế index → element. |
| | | `await context._input_text_element_node(element, text)` | **browser_use Context**: focus + nhập text (Playwright fill/type). |
| **send_keys** | `keys` | `page = await context.get_current_page()` | **Playwright Page** |
| | | `await page.keyboard.press(keys)` | `Page.keyboard.press(keys)` |
| **get_dropdown_options** | `index` | `element = await context.get_dom_element_by_index(index)` | **browser_use Context** (element có `.xpath`). |
| | | `page = await context.get_current_page()` | **Playwright Page** |
| | | `options = await page.evaluate(js, element.xpath)` | `Page.evaluate(script, arg)`: JS dùng `document.evaluate(xpath, ...)` lấy `<select>`, map `options` → `{ text, value, index }`. |
| **select_dropdown_option** | `index`, `text` | `element = await context.get_dom_element_by_index(index)` | **browser_use Context**. |
| | | `page = await context.get_current_page()` | **Playwright Page** |
| | | `await page.select_option(element.xpath, label=text)` | `Page.select_option(selector, label=...)` (Playwright). |

---

### 3.3. Cuộn (scroll)

| Action | Tham số tool | Hàm trong `execute()` | API browser_use / Playwright |
|--------|--------------|------------------------|------------------------------|
| **scroll_down** / **scroll_up** | `scroll_amount` (optional) | `await context.execute_javascript("window.scrollBy(0, {pixels});")` | **browser_use Context**: chạy JS trên page hiện tại (tương đương `page.evaluate()` hoặc inject script). |
| **scroll_to_text** | `text` | `page = await context.get_current_page()` | **Playwright Page** |
| | | `locator = page.get_by_text(text, exact=False)` | `Page.get_by_text(text, exact=False)` → **Locator** |
| | | `await locator.scroll_into_view_if_needed()` | `Locator.scroll_into_view_if_needed()` |

---

### 3.4. Nội dung trang (extract)

| Action | Tham số tool | Hàm trong `execute()` | API browser_use / Playwright |
|--------|--------------|------------------------|------------------------------|
| **extract_content** | `goal` | `page = await context.get_current_page()` | **Playwright Page** |
| | | `content = markdownify.markdownify(await page.content())` | `Page.content()` → HTML full page; chuyển sang Markdown. |
| | | (sau đó dùng LLM với `extraction_function` để extract theo `goal`) | Không gọi thêm Playwright. |

---

### 3.5. Tab

| Action | Tham số tool | Hàm trong `execute()` | API browser_use / Playwright |
|--------|--------------|------------------------|------------------------------|
| **switch_tab** | `tab_id` | `await context.switch_to_tab(tab_id)` | **browser_use Context** (chuyển page hiện tại sang tab tương ứng). |
| | | `page = await context.get_current_page()` | **Playwright Page** (page sau khi đã switch). |
| | | `await page.wait_for_load_state()` | `Page.wait_for_load_state()` |
| **open_tab** | `url` | `await context.create_new_tab(url)` | **browser_use Context** (tạo tab mới và goto url). |
| **close_tab** | — | `await context.close_current_tab()` | **browser_use Context** (đóng tab hiện tại). |

---

### 3.6. Utility

| Action | Tham số tool | Hàm trong `execute()` | API browser_use / Playwright |
|--------|--------------|------------------------|------------------------------|
| **wait** | `seconds` | `await asyncio.sleep(seconds_to_wait)` | Không dùng Playwright. |

---

## 4. Lấy trạng thái browser (get_current_state) – ảnh hưởng tới prompt agent

**Hàm:** `BrowserUseTool.get_current_state()`. Dùng khi **BrowserContextHelper** cần state + screenshot để format prompt (BrowserAgent / Manus khi đang dùng browser).

| Bước | Code | API browser_use / Playwright |
|------|------|------------------------------|
| 1 | `state = await ctx.get_state()` | **browser_use Context**: trả về state (url, title, tabs, element_tree, viewport_info, pixels_above, pixels_below). |
| 2 | `page = await ctx.get_current_page()` | **Playwright Page** (tab hiện tại). |
| 3 | `await page.bring_to_front()` | `Page.bring_to_front()` |
| 4 | `await page.wait_for_load_state()` | `Page.wait_for_load_state()` |
| 5 | `screenshot = await page.screenshot(full_page=True, animations="disabled", type="jpeg", quality=100)` | `Page.screenshot(...)` → bytes, sau đó base64. |

State trả về cho agent gồm: `url`, `title`, `tabs`, `interactive_elements` (từ `state.element_tree.clickable_elements_to_string()`), `scroll_info`, `viewport_height`, và ảnh base64. Không gọi thêm Playwright ngoài các bước trên.

---

## 5. Tóm tắt API Playwright được dùng (qua browser_use / OpenManus)

Từ code hiện tại, **Playwright** được dùng gián tiếp qua **browser_use** như sau:

| Đối tượng | API Playwright (được gọi qua browser_use hoặc qua `page` lấy từ context) |
|-----------|----------------------------------------------------------------------------|
| **Page** | `goto`, `wait_for_load_state`, `content`, `bring_to_front`, `screenshot`, `keyboard.press`, `evaluate`, `select_option` |
| **Locator** | `scroll_into_view_if_needed` (từ `page.get_by_text`) |
| **BrowserContext** | Không gọi trực tiếp từ OpenManus; browser_use quản lý context và tab (go_back, refresh, switch_to_tab, create_new_tab, close_current_tab). |

**browser_use** bổ sung:

- `context.get_dom_element_by_index(index)` → element có `.xpath` (tương ứng cây DOM đã đánh index).
- `context._click_element_node(element)`, `context._input_text_element_node(element, text)` → bên trong dùng selector/xpath để click và nhập text.
- `context.execute_javascript(js)` → chạy JS trên page hiện tại.
- `context.get_state()` → url, title, tabs, element_tree, viewport, scroll info.

---

## 6. Schema tool `browser_use` (tham chiếu nhanh)

Agent gọi tool với tên **`browser_use`** và body JSON gồm:

- **action** (bắt buộc): một trong các giá trị liệt kê dưới đây.
- **url**: cho `go_to_url`, `open_tab`.
- **index**: cho `click_element`, `input_text`, `get_dropdown_options`, `select_dropdown_option`.
- **text**: cho `input_text`, `scroll_to_text`, `select_dropdown_option`.
- **scroll_amount**: cho `scroll_down`, `scroll_up`.
- **tab_id**: cho `switch_tab`.
- **query**: cho `web_search`.
- **goal**: cho `extract_content`.
- **keys**: cho `send_keys`.
- **seconds**: cho `wait`.

**Danh sách action:**
`go_to_url`, `click_element`, `input_text`, `scroll_down`, `scroll_up`, `scroll_to_text`, `send_keys`, `get_dropdown_options`, `select_dropdown_option`, `go_back`, `web_search`, `wait`, `extract_content`, `switch_tab`, `open_tab`, `close_tab`.

(refresh không nằm trong enum trong code hiện tại nhưng có branch xử lý trong `execute()`.)

---

## 7. Luồng giao tiếp (tóm tắt)

```
LLM (agent) quyết định action + params
    ↓
ToolCallAgent.execute_tool("browser_use", { action, url, index, ... })
    ↓
BrowserUseTool.execute(action=..., url=..., index=..., ...)
    ↓
context = await self._ensure_browser_initialized()   # browser_use BrowserContext
page = await context.get_current_page()               # Playwright Page (khi cần)
    ↓
• Điều hướng: context.go_back() / refresh_page() hoặc page.goto() / wait_for_load_state()
• Phần tử: context.get_dom_element_by_index() → context._click_element_node() / _input_text_element_node()
• Keyboard: page.keyboard.press(keys)
• Dropdown: page.evaluate(js, xpath) hoặc page.select_option(xpath, label=text)
• Scroll: context.execute_javascript("window.scrollBy(...)") hoặc page.get_by_text().scroll_into_view_if_needed()
• Nội dung: page.content() → markdownify + LLM
• Tab: context.switch_to_tab() / create_new_tab() / close_current_tab()
• State/screenshot: ctx.get_state() + page.bring_to_front() + page.wait_for_load_state() + page.screenshot()
    ↓
ToolResult(output=..., base64_image=...) → đưa lại vào memory cho bước think tiếp theo.
```

File tham chiếu chính: **`app/tool/browser_use_tool.py`** (method `execute()` và `get_current_state()`).
