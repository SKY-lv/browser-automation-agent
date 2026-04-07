---
name: browser-automation-agent
description: "浏览器自动化Agent。使用Playwright/Puppeteer进行网页自动化、数据采集、表单填写。触发词：browser、浏览器自动化、爬虫、playwright、puppeteer、网页抓取。"
metadata: {"openclaw": {"emoji": "🌐"}}
---

# Browser Automation Agent

## 功能说明

AI驱动的浏览器自动化，执行复杂网页任务。

## 技术选型

| 库 | 特点 | 适用场景 |
|----|------|----------|
| Playwright | 跨浏览器、等待稳定 | 通用自动化 |
| Puppeteer | Chrome专用、Node原生 | Chrome深度控制 |
| Selenium | 多语言、老牌稳定 | 多浏览器兼容 |
| DrissionPage | Python、轻量 | 中国网站适配 |

## Playwright 完整实现

### 1. 基础框架

```python
from playwright.sync_api import sync_playwright, Page, Browser, BrowserContext
from dataclasses import dataclass
from typing import Optional
import json

@dataclass
class TaskResult:
    success: bool
    data: Optional[dict] = None
    error: Optional[str] = None
    screenshot: Optional[str] = None

class BrowserAgent:
    def __init__(self, headless: bool = True, slow_mo: int = 100):
        self.playwright = sync_playwright().start()
        self.browser: Browser = self.playwright.chromium.launch(
            headless=headless,
            slow_mo=slow_mo
        )
        self.context: BrowserContext = self.browser.new_context(
            viewport={"width": 1920, "height": 1080},
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        )
    
    def new_page(self) -> Page:
        return self.context.new_page()
    
    def execute(self, task: str) -> TaskResult:
        """执行AI指令"""
        page = self.new_page()
        try:
            # AI解析任务
            steps = self.plan_steps(task)
            
            for step in steps:
                self.execute_step(page, step)
            
            return TaskResult(success=True, data={"steps_completed": len(steps)})
        except Exception as e:
            return TaskResult(success=False, error=str(e))
        finally:
            page.close()
    
    def plan_steps(self, task: str) -> list[dict]:
        """AI规划步骤"""
        # 这里集成LLM进行任务规划
        # ...
        pass
    
    def execute_step(self, page: Page, step: dict):
        action = step["action"]
        
        if action == "goto":
            page.goto(step["url"], wait_until="domcontentloaded")
        elif action == "click":
            page.click(step["selector"])
        elif action == "type":
            page.fill(step["selector"], step["text"])
        elif action == "wait":
            page.wait_for_timeout(step["ms"])
        elif action == "screenshot":
            page.screenshot(path=step["path"])
        elif action == "extract":
            return page.query_selector(step["selector"]).inner_text()
    
    def close(self):
        self.browser.close()
        self.playwright.stop()
```

### 2. 智能元素定位

```python
class SmartLocator:
    """AI智能定位器"""
    
    LOCATOR_STRATEGIES = [
        "get_by_role",      # 无障碍角色（最可靠）
        "get_by_label",     # 表单标签
        "get_by_placeholder",  # 占位符
        "get_by_text",      # 文本内容
        "locator",          # CSS选择器
        "xpath",             # XPath（最后备选）
    ]
    
    def find(self, page: Page, description: str) -> Locator:
        """根据描述智能查找元素"""
        strategies = [
            # 按钮查找
            lambda: page.get_by_role("button", name=description),
            lambda: page.get_by_role("link", name=description),
            
            # 输入框查找
            lambda: page.get_by_label(description),
            lambda: page.get_by_placeholder(description),
            
            # 文本查找
            lambda: page.get_by_text(description, exact=True),
            lambda: page.get_by_text(description),
            
            # 通配符（放在最后）
            lambda: page.locator(f"text={description}").first,
        ]
        
        for strategy in strategies:
            try:
                locator = strategy()
                if locator.count() > 0:
                    return locator.first
            except:
                continue
        
        raise ElementNotFound(f"找不到元素: {description}")
    
    def find_many(self, page: Page, description: str) -> list:
        """批量查找元素"""
        # 尝试多种策略
        # ...
        pass
```

### 3. 表单自动填写

```python
class FormFiller:
    """智能表单填写"""
    
    def fill_form(self, page: Page, form_data: dict):
        """根据字段描述自动填写"""
        for field_name, value in form_data.items():
            try:
                # 尝试找对应label的输入框
                locator = page.get_by_label(field_name, exact=False)
                
                # 获取输入类型
                if locator.count() == 0:
                    # 尝试placeholder
                    locator = page.get_by_placeholder(field_name)
                
                if locator.count() == 0:
                    # 尝试名称属性
                    locator = page.locator(f'[name="{field_name}"]')
                
                if locator.count() > 0:
                    el = locator.first
                    tag = el.evaluate("el => el.tagName")
                    
                    if tag == "SELECT":
                        el.select_option(value)
                    elif tag == "INPUT":
                        input_type = el.get_attribute("type")
                        if input_type in ["checkbox", "radio"]:
                            if value:
                                el.check()
                        else:
                            el.fill(str(value))
                    else:
                        el.fill(str(value))
                        
            except Exception as e:
                print(f"填写字段 {field_name} 失败: {e}")
    
    def extract_form(self, page: Page, form_selector: str = "form") -> dict:
        """提取表单数据"""
        form = page.locator(form_selector).first
        data = {}
        
        inputs = form.locator("input, select, textarea")
        for inp in inputs.all():
            name = inp.get_attribute("name") or inp.get_attribute("id")
            if not name: continue
            
            input_type = inp.get_attribute("type") or "text"
            if input_type in ["submit", "button", "reset", "image"]: continue
            
            if input_type == "checkbox":
                data[name] = inp.is_checked()
            elif input_type == "radio":
                checked = form.locator(f'input[name="{name}"]:checked')
                data[name] = checked.get_attribute("value") if checked.count() > 0 else None
            else:
                data[name] = inp.input_value()
        
        return data
```

### 4. 数据采集

```python
class WebScraper:
    """网页数据采集"""
    
    def scrape_table(self, page: Page, table_selector: str) -> list[dict]:
        """采集表格数据"""
        table = page.locator(table_selector).first
        
        # 获取表头
        headers = table.locator("thead th, th").all_text_contents()
        
        # 采集每行
        rows = []
        for row in table.locator("tbody tr, tr").all():
            cells = row.locator("td, th").all_text_contents()
            if len(cells) == len(headers):
                rows.append(dict(zip(headers, cells)))
        
        return rows
    
    def scrape_cards(self, page: Page, card_selector: str, fields: dict) -> list[dict]:
        """采集卡片式列表"""
        cards = page.locator(card_selector).all()
        results = []
        
        for card in cards:
            item = {}
            for field_name, selector in fields.items():
                try:
                    el = card.locator(selector).first
                    item[field_name] = el.inner_text().strip()
                except:
                    item[field_name] = None
            results.append(item)
        
        return results
    
    def scroll_scrape(self, page: Page, item_selector: str, max_items: int = 100) -> list:
        """滚动加载采集"""
        items = []
        last_count = 0
        
        while len(items) < max_items:
            # 滚动
            page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
            page.wait_for_timeout(1000)
            
            # 采集新数据
            new_items = page.locator(item_selector).all_text_contents()
            items.extend(new_items[last_count:])
            last_count = len(items)
            
            # 检查是否到底
            if len(items) == last_count:
                break
        
        return items[:max_items]
```

### 5. 反爬对抗

```python
class AntiDetection:
    """反检测"""
    
    @staticmethod
    def stealth(context: BrowserContext):
        """Stealth模式"""
        # 随机User-Agent
        ua_list = [
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120.0.0.0",
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 Safari/17.2",
        ]
        context.set_extra_http_headers({"User-Agent": random.choice(ua_list)})
        
        # 注入反检测脚本
        context.add_init_script("""
            Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
            window.chrome = { runtime: {} };
        """)
    
    @staticmethod
    def human_delay(page: Page, min_ms=50, max_ms=200):
        """模拟人类延迟"""
        import random
        import time
        time.sleep(random.uniform(min_ms, max_ms) / 1000)
```

## Puppeteer 实现

```javascript
const { chromium } = require('puppeteer');

class BrowserAutomation {
  async init() {
    this.browser = await chromium.launch({ 
      headless: true,
      args: ['--disable-blink-features=AutomationControlled']
    });
    
    // Stealth
    const context = await this.browser.newContext({
      viewport: { width: 1920, height: 1080 },
      userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
    });
    
    // Remove webdriver flag
    await context.addInitScript(() => {
      Object.defineProperty(navigator, 'webdriver', { get: () => false });
    });
    
    this.page = await context.newPage();
  }
  
  async goto(url) {
    await this.page.goto(url, { waitUntil: 'networkidle2' });
    await this.page.waitForTimeout(1000);
  }
  
  async click(selector) {
    await this.page.locator(selector).first().click();
  }
  
  async type(selector, text) {
    await this.page.locator(selector).fill(text);
  }
  
  async extract(selector) {
    return await this.page.locator(selector).allTextContents();
  }
  
  async screenshot(path) {
    await this.page.screenshot({ path, fullPage: true });
  }
  
  async close() {
    await this.browser.close();
  }
}
```

## 常见任务模板

### 登录 + 数据采集
```python
with BrowserAgent() as agent:
    page = agent.new_page()
    
    # 登录
    page.goto("https://example.com/login")
    page.fill("#username", "user@example.com")
    page.fill("#password", "password123")
    page.click("button[type='submit']")
    page.wait_for_url("**/dashboard")
    
    # 采集数据
    page.goto("https://example.com/data")
    data = WebScraper().scrape_table(page, "table.data")
    
    print(data)
```

### 批量截图
```python
urls = ["https://site1.com", "https://site2.com", "https://site3.com"]
with BrowserAgent() as agent:
    for i, url in enumerate(urls):
        page = agent.new_page()
        page.goto(url, wait_until="networkidle")
        page.screenshot(path=f"screenshot_{i}.png")
        page.close()
```

## 最佳实践

1. **等待策略**：用 `wait_for_selector` 而不是固定sleep
2. **重试机制**：网络不稳定时自动重试
3. **异常处理**：每个操作都要try-catch
4. **资源清理**：总是关闭page和context
5. **隐身模式**：每个任务用独立的context避免cookies污染
