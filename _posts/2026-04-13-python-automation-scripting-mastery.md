---
layout: post
title: "Python自动化脚本：从数据采集到定时任务的完整实战"
subtitle: "手把手教你用Python实现网页爬虫、API调用、文件处理、定时任务，配合Cron实现7×24小时自动化运行"
date: 2026-04-13
category: python
category_name: 🐍 Python
tags: [Python自动化, 爬虫, API开发, 脚本工具, 定时任务, 数据采集]
excerpt: "本文是Python自动化开发的实战指南，涵盖网页爬虫、RESTful API调用、文件自动化处理、Cron定时任务等核心技能，提供可直接使用的代码模板和最佳实践。"
keywords: Python自动化, 网络爬虫, BeautifulSoup, Requests, API调用, 定时任务, Cron, 数据采集, 脚本开发
---

# Python自动化脚本：从数据采集到定时任务的完整实战

Python 是自动化脚本开发的首选语言。其简洁的语法、丰富的库生态和跨平台特性，使其成为数据采集、API集成、文件处理、定时任务等场景的理想工具。本文将手把手教你构建一个生产级的自动化数据采集系统。

## 📋 目录

1. Python自动化应用场景
2. 开发环境与依赖
3. 网页爬虫开发
4. RESTful API调用
5. 文件自动化处理
6. 定时任务与Cron配置
7. 完整项目：加密货币行情监控
8. 错误处理与日志系统
9. 性能优化技巧

---

## 1. Python自动化应用场景

### 常见应用领域

| 场景 | 工具/库 | 用途 |
|------|---------|------|
| **网页爬虫** | Requests, BeautifulSoup, Playwright | 采集网页数据 |
| **API集成** | Requests, httpx, aiohttp | 对接第三方服务 |
| **文件处理** | pandas, openpyxl, python-docx | Excel/Word/PDF处理 |
| **邮件自动化** | smtplib, yagmail | 定时发送邮件 |
| **数据库操作** | SQLAlchemy, pymongo | 数据读写 |
| **GUI自动化** | pyautogui, selenium | 模拟人工操作 |
| **定时任务** | schedule, APScheduler | 定时执行脚本 |

---

## 2. 开发环境与依赖

### 2.1 创建虚拟环境

```bash
# 创建虚拟环境
python3 -m venv automation-env

# 激活环境
source automation-env/bin/activate  # macOS/Linux
# automation-env\Scripts\activate     # Windows

# 安装依赖
pip install requests beautifulsoup4 lxml httpx aiohttp
pip install pandas openpyxl python-docx pdfplumber
pip install schedule APScheduler python-dotenv loguru
pip install playwright && playwright install chromium
```

### 2.2 项目结构

```bash
automation_project/
├── config/
│   └── settings.py        # 配置管理
├── src/
│   ├── __init__.py
│   ├── crawler.py         # 爬虫模块
│   ├── api_client.py     # API客户端
│   ├── file_processor.py # 文件处理
│   ├── scheduler.py      # 定时任务
│   └── logger.py         # 日志系统
├── data/
│   └── output/            # 输出目录
├── logs/                  # 日志目录
├── requirements.txt
└── main.py               # 入口文件
```

### 2.3 配置管理

```python
# config/settings.py
import os
from dotenv import load_dotenv

load_dotenv()  # 加载 .env 文件

class Config:
    # 通用配置
    DEBUG = os.getenv("DEBUG", "False").lower() == "true"
    RETRY_TIMES = int(os.getenv("RETRY_TIMES", "3"))
    TIMEOUT = int(os.getenv("TIMEOUT", "30"))
    
    # API配置
    API_KEY = os.getenv("API_KEY", "")
    API_SECRET = os.getenv("API_SECRET", "")
    API_BASE_URL = os.getenv("API_BASE_URL", "https://api.example.com")
    
    # 数据库配置
    DB_HOST = os.getenv("DB_HOST", "localhost")
    DB_PORT = int(os.getenv("DB_PORT", "5432"))
    DB_NAME = os.getenv("DB_NAME", "automation")
    DB_USER = os.getenv("DB_USER", "postgres")
    DB_PASSWORD = os.getenv("DB_PASSWORD", "")
    
    # 文件路径
    DATA_DIR = os.path.join(os.path.dirname(__file__), "..", "data")
    LOG_DIR = os.path.join(os.path.dirname(__file__), "..", "logs")
    
    # 邮件配置
    SMTP_HOST = os.getenv("SMTP_HOST", "smtp.gmail.com")
    SMTP_PORT = int(os.getenv("SMTP_PORT", "587"))
    SMTP_USER = os.getenv("SMTP_USER", "")
    SMTP_PASSWORD = os.getenv("SMTP_PASSWORD", "")
    
    @classmethod
    def ensure_dirs(cls):
        """确保必要目录存在"""
        os.makedirs(cls.DATA_DIR, exist_ok=True)
        os.makedirs(cls.LOG_DIR, exist_ok=True)

config = Config()
config.ensure_dirs()
```

---

## 3. 网页爬虫开发

### 3.1 基础爬虫：Requests + BeautifulSoup

```python
# src/crawler.py
import requests
from bs4 import BeautifulSoup
from typing import List, Dict, Optional
from urllib.parse import urljoin
import time
from .logger import get_logger
from .file_processor import save_to_json

logger = get_logger(__name__)

class WebCrawler:
    """通用网页爬虫"""
    
    def __init__(
        self,
        timeout: int = 30,
        retry_times: int = 3,
        delay: float = 1.0,
        user_agent: str = None
    ):
        self.timeout = timeout
        self.retry_times = retry_times
        self.delay = delay
        
        self.session = requests.Session()
        self.session.headers.update({
            "User-Agent": user_agent or (
                "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/120.0.0.0 Safari/537.36"
            ),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",
        })
    
    def fetch(self, url: str) -> Optional[str]:
        """获取页面HTML"""
        for attempt in range(self.retry_times):
            try:
                response = self.session.get(url, timeout=self.timeout)
                response.raise_for_status()
                time.sleep(self.delay)  # 礼貌爬取
                return response.text
            except requests.RequestException as e:
                logger.warning(f"请求失败 (尝试 {attempt+1}/{self.retry_times}): {e}")
                if attempt < self.retry_times - 1:
                    time.sleep(2 ** attempt)  # 指数退避
        return None
    
    def parse_article_links(self, html: str, base_url: str) -> List[str]:
        """解析文章链接"""
        soup = BeautifulSoup(html, "lxml")
        links = []
        
        for a_tag in soup.select("a.article-link, a.post-link, h2 a"):
            href = a_tag.get("href")
            if href:
                full_url = urljoin(base_url, href)
                links.append(full_url)
        
        return list(set(links))  # 去重
    
    def parse_article(self, html: str) -> Dict:
        """解析文章内容"""
        soup = BeautifulSoup(html, "lxml")
        
        # 提取标题
        title = (
            soup.select_one("h1.title, h1.post-title, article h1") or
            soup.select_one("title")
        )
        title_text = title.get_text(strip=True) if title else ""
        
        # 提取正文
        content_tags = soup.select(
            "article .content, .post-content, .article-body, "
            "main p, article p"
        )
        paragraphs = [p.get_text(strip=True) for p in content_tags if p.get_text(strip=True)]
        content = "\n\n".join(paragraphs)
        
        # 提取发布时间
        date_tag = soup.select_one(
            "time, .date, .published, [datetime]"
        )
        date_text = date_tag.get("datetime") or (
            date_tag.get_text(strip=True) if date_tag else ""
        )
        
        # 提取标签
        tags = [
            tag.get_text(strip=True)
            for tag in soup.select(".tags a, .tag, [rel='tag']")
        ]
        
        return {
            "title": title_text,
            "content": content,
            "date": date_text,
            "tags": tags,
            "word_count": len(content)
        }
    
    def crawl_articles(
        self, 
        start_url: str, 
        max_pages: int = 10
    ) -> List[Dict]:
        """爬取多页文章"""
        all_articles = []
        current_url = start_url
        
        for page in range(max_pages):
            logger.info(f"正在爬取第 {page+1} 页: {current_url}")
            
            html = self.fetch(current_url)
            if not html:
                logger.error(f"页面获取失败: {current_url}")
                break
            
            # 解析文章链接
            article_links = self.parse_article_links(html, current_url)
            logger.info(f"发现 {len(article_links)} 篇文章")
            
            # 逐个爬取文章详情
            for link in article_links:
                article_html = self.fetch(link)
                if article_html:
                    article = self.parse_article(article_html)
                    article["url"] = link
                    all_articles.append(article)
                    logger.info(f"已爬取: {article.get('title', '')[:50]}")
            
            # 翻页逻辑（根据实际情况调整）
            soup = BeautifulSoup(html, "lxml")
            next_link = soup.select_one("a.next-page, a.next, .pagination .active + a")
            if next_link and (next_href := next_link.get("href")):
                current_url = urljoin(current_url, next_href)
            else:
                break
        
        logger.info(f"爬取完成，共 {len(all_articles)} 篇文章")
        return all_articles


class CryptoPriceCrawler:
    """加密货币价格爬虫"""
    
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            "Accept": "application/json",
            "User-Agent": "Mozilla/5.0 (compatible; CryptoBot/1.0)"
        })
    
    def get_binance_prices(self, symbols: List[str]) -> List[Dict]:
        """获取币安价格"""
        prices = []
        
        for symbol in symbols:
            url = f"https://api.binance.com/api/v3/ticker/24hr"
            params = {"symbol": f"{symbol}USDT".upper()}
            
            try:
                response = self.session.get(url, params=params, timeout=10)
                data = response.json()
                
                prices.append({
                    "symbol": symbol.upper(),
                    "price": float(data["lastPrice"]),
                    "change_24h": float(data["priceChangePercent"]),
                    "high_24h": float(data["highPrice"]),
                    "low_24h": float(data["lowPrice"]),
                    "volume": float(data["quoteVolume"]),
                    "timestamp": data["closeTime"]
                })
            except Exception as e:
                logger.error(f"获取 {symbol} 价格失败: {e}")
        
        return prices
```

### 3.2 高级爬虫：Playwright（JavaScript渲染页面）

```python
# src/crawler_advanced.py
from playwright.sync_api import sync_playwright
from typing import List, Dict
import time

class JSRenderCrawler:
    """处理JavaScript渲染页面的爬虫"""
    
    def __init__(self, headless: bool = True):
        self.playwright = None
        self.browser = None
        self.headless = headless
    
    def __enter__(self):
        self.playwright = sync_playwright().start()
        self.browser = self.playwright.chromium.launch(headless=self.headless)
        return self
    
    def __exit__(self, *args):
        if self.browser:
            self.browser.close()
        if self.playwright:
            self.playwright.stop()
    
    def crawl_dynamid_page(
        self, 
        url: str, 
        selectors: Dict[str, str],
        wait_for: str = None,
        timeout: int = 30000
    ) -> Dict[str, str]:
        """爬取动态渲染页面"""
        page = self.browser.new_page()
        
        try:
            # 导航到页面
            page.goto(url, timeout=timeout)
            
            # 等待元素加载（如果指定）
            if wait_for:
                page.wait_for_selector(wait_for, timeout=timeout)
            
            # 提取数据
            result = {}
            for key, selector in selectors.items():
                element = page.query_selector(selector)
                if element:
                    result[key] = element.inner_text()
            
            return result
            
        finally:
            page.close()
    
    def crawl_table_data(self, url: str, row_selector: str) -> List[Dict]:
        """爬取表格数据"""
        page = self.browser.new_page()
        
        try:
            page.goto(url, timeout=30000)
            page.wait_for_load_state("networkidle")
            
            rows = page.query_selector_all(row_selector)
            data = []
            
            for row in rows:
                cells = row.query_selector_all("td, th")
                row_data = [cell.inner_text() for cell in cells]
                data.append(row_data)
            
            return data
        finally:
            page.close()
```

---

## 4. RESTful API调用

### 4.1 通用API客户端

```python
# src/api_client.py
import requests
import httpx
import asyncio
from typing import Dict, Any, Optional, List
from abc import ABC, abstractmethod
from .logger import get_logger
import time

logger = get_logger(__name__)


class BaseAPIClient(ABC):
    """API客户端基类"""
    
    def __init__(
        self,
        base_url: str,
        api_key: str = None,
        timeout: int = 30,
        retry_times: int = 3
    ):
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.timeout = timeout
        self.retry_times = retry_times
    
    def _get_headers(self) -> Dict[str, str]:
        """获取请求头"""
        headers = {
            "Content-Type": "application/json",
            "Accept": "application/json"
        }
        if self.api_key:
            headers["Authorization"] = f"Bearer {self.api_key}"
        return headers
    
    def _request(
        self,
        method: str,
        endpoint: str,
        params: Dict = None,
        data: Dict = None,
        json: Dict = None
    ) -> Optional[Dict]:
        """通用请求方法（同步）"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        
        for attempt in range(self.retry_times):
            try:
                response = requests.request(
                    method=method.upper(),
                    url=url,
                    params=params,
                    data=data,
                    json=json,
                    headers=self._get_headers(),
                    timeout=self.timeout
                )
                response.raise_for_status()
                return response.json() if response.content else {}
            
            except requests.HTTPError as e:
                logger.error(f"HTTP错误 ({attempt+1}/{self.retry_times}): {e}")
                if response.status_code in [401, 403]:
                    raise  # 认证错误不重试
                if attempt < self.retry_times - 1:
                    time.sleep(2 ** attempt)
                    
            except requests.RequestException as e:
                logger.error(f"请求异常 ({attempt+1}/{self.retry_times}): {e}")
                if attempt < self.retry_times - 1:
                    time.sleep(2 ** attempt)
        
        return None
    
    def get(self, endpoint: str, params: Dict = None) -> Optional[Dict]:
        return self._request("GET", endpoint, params=params)
    
    def post(self, endpoint: str, json: Dict = None) -> Optional[Dict]:
        return self._request("POST", endpoint, json=json)
    
    def put(self, endpoint: str, json: Dict = None) -> Optional[Dict]:
        return self._request("PUT", endpoint, json=json)
    
    def delete(self, endpoint: str) -> Optional[Dict]:
        return self._request("DELETE", endpoint)


class AsyncAPIClient:
    """异步API客户端"""
    
    def __init__(
        self,
        base_url: str,
        api_key: str = None,
        timeout: int = 30,
        max_connections: int = 20
    ):
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self.timeout = timeout
        self.limits = httpx.Limits(max_connections=max_connections)
    
    def _get_headers(self) -> Dict[str, str]:
        headers = {"Accept": "application/json"}
        if self.api_key:
            headers["Authorization"] = f"Bearer {self.api_key}"
        return headers
    
    async def _request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> Optional[Dict]:
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        
        async with httpx.AsyncClient(
            headers=self._get_headers(),
            timeout=self.timeout,
            limits=self.limits
        ) as client:
            response = await client.request(method, url, **kwargs)
            response.raise_for_status()
            return response.json()
    
    async def get(self, endpoint: str, params: Dict = None) -> Optional[Dict]:
        return await self._request("GET", endpoint, params=params)
    
    async def post(self, endpoint: str, json: Dict = None) -> Optional[Dict]:
        return await self._request("POST", endpoint, json=json)
    
    async def batch_get(self, endpoints: List[str]) -> List[Optional[Dict]]:
        """批量并发请求"""
        tasks = [self.get(ep) for ep in endpoints]
        return await asyncio.gather(*tasks, return_exceptions=True)


# ======= 具体API客户端示例 =======

class CoinGeckoClient(BaseAPIClient):
    """CoinGecko免费加密货币API"""
    
    def __init__(self):
        super().__init__(
            base_url="https://api.coingecko.com/api/v3",
            timeout=30
        )
    
    def get_simple_price(
        self,
        ids: List[str],
        vs_currencies: List[str] = ["usd"]
    ) -> Optional[Dict]:
        """获取实时价格"""
        return self.get("/simple/price", params={
            "ids": ",".join(ids),
            "vs_currencies": ",".join(vs_currencies),
            "include_24hr_change": "true",
            "include_market_cap": "true"
        })
    
    def get_market_data(
        self,
        per_page: int = 100,
        page: int = 1
    ) -> Optional[List[Dict]]:
        """获取市场数据"""
        return self.get("/coins/markets", params={
            "vs_currency": "usd",
            "order": "market_cap_desc",
            "per_page": per_page,
            "page": page,
            "sparkline": "false"
        })
    
    def get_global_data(self) -> Optional[Dict]:
        """获取全球市场数据"""
        return self.get("/global")


class OpenAIAPIClient(BaseAPIClient):
    """OpenAI API客户端"""
    
    def __init__(self, api_key: str):
        super().__init__(
            base_url="https://api.openai.com/v1",
            api_key=api_key,
            timeout=60
        )
    
    def create_chat_completion(
        self,
        model: str = "gpt-4o",
        messages: List[Dict[str, str]],
        temperature: float = 0.7,
        max_tokens: int = 1000
    ) -> Optional[Dict]:
        """创建聊天完成"""
        return self.post("/chat/completions", json={
            "model": model,
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens
        })
    
    def create_embedding(
        self,
        input_text: str,
        model: str = "text-embedding-3-small"
    ) -> Optional[Dict]:
        """创建文本嵌入"""
        return self.post("/embeddings", json={
            "model": model,
            "input": input_text
        })
```

---

## 5. 文件自动化处理

```python
# src/file_processor.py
import os
import json
import csv
import zipfile
import shutil
from pathlib import Path
from typing import List, Dict, Any, Optional
from datetime import datetime
import pandas as pd
from openpyxl import load_workbook, Workbook
from .logger import get_logger

logger = get_logger(__name__)


class FileProcessor:
    """文件处理工具集"""
    
    @staticmethod
    def ensure_dir(path: str) -> Path:
        """确保目录存在"""
        p = Path(path)
        p.mkdir(parents=True, exist_ok=True)
        return p
    
    @staticmethod
    def save_to_json(data: Any, filepath: str, indent: int = 2) -> bool:
        """保存JSON文件"""
        try:
            FileProcessor.ensure_dir(os.path.dirname(filepath))
            with open(filepath, "w", encoding="utf-8") as f:
                json.dump(data, f, ensure_ascii=False, indent=indent)
            logger.info(f"已保存JSON: {filepath}")
            return True
        except Exception as e:
            logger.error(f"保存JSON失败: {e}")
            return False
    
    @staticmethod
    def load_from_json(filepath: str) -> Optional[Any]:
        """读取JSON文件"""
        try:
            with open(filepath, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception as e:
            logger.error(f"读取JSON失败: {e}")
            return None
    
    @staticmethod
    def save_to_csv(data: List[Dict], filepath: str) -> bool:
        """保存CSV文件"""
        try:
            FileProcessor.ensure_dir(os.path.dirname(filepath))
            if not data:
                return False
            
            keys = data[0].keys()
            with open(filepath, "w", newline="", encoding="utf-8-sig") as f:
                writer = csv.DictWriter(f, fieldnames=keys)
                writer.writeheader()
                writer.writerows(data)
            
            logger.info(f"已保存CSV: {filepath} ({len(data)} 行)")
            return True
        except Exception as e:
            logger.error(f"保存CSV失败: {e}")
            return False
    
    @staticmethod
    def read_csv(filepath: str) -> List[Dict]:
        """读取CSV文件"""
        try:
            with open(filepath, "r", encoding="utf-8-sig") as f:
                return list(csv.DictReader(f))
        except Exception as e:
            logger.error(f"读取CSV失败: {e}")
            return []
    
    @staticmethod
    def excel_to_json(excel_path: str, sheet_name: str = None) -> List[Dict]:
        """Excel转JSON"""
        try:
            df = pd.read_excel(excel_path, sheet_name=sheet_name)
            return df.to_dict(orient="records")
        except Exception as e:
            logger.error(f"Excel读取失败: {e}")
            return []
    
    @staticmethod
    def json_to_excel(
        data: List[Dict],
        excel_path: str,
        sheet_name: str = "Sheet1"
    ) -> bool:
        """JSON转Excel"""
        try:
            FileProcessor.ensure_dir(os.path.dirname(excel_path))
            df = pd.DataFrame(data)
            df.to_excel(excel_path, sheet_name=sheet_name, index=False)
            logger.info(f"已保存Excel: {excel_path}")
            return True
        except Exception as e:
            logger.error(f"Excel保存失败: {e}")
            return False
    
    @staticmethod
    def backup_file(filepath: str, backup_dir: str = None) -> Optional[str]:
        """备份文件"""
        try:
            if backup_dir is None:
                backup_dir = os.path.join(os.path.dirname(filepath), "backups")
            
            FileProcessor.ensure_dir(backup_dir)
            
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            filename = os.path.basename(filepath)
            backup_path = os.path.join(
                backup_dir, 
                f"{Path(filename).stem}_{timestamp}{Path(filename).suffix}"
            )
            
            shutil.copy2(filepath, backup_path)
            logger.info(f"已备份: {backup_path}")
            return backup_path
        except Exception as e:
            logger.error(f"备份失败: {e}")
            return None
    
    @staticmethod
    def compress_files(
        files: List[str],
        output_zip: str,
        compression: int = zipfile.ZIP_DEFLATED
    ) -> bool:
        """压缩多个文件"""
        try:
            FileProcessor.ensure_dir(os.path.dirname(output_zip))
            
            with zipfile.ZipFile(
                output_zip, "w", compression=compression
            ) as zf:
                for file in files:
                    if os.path.exists(file):
                        zf.write(file, os.path.basename(file))
            
            logger.info(f"已压缩 {len(files)} 个文件: {output_zip}")
            return True
        except Exception as e:
            logger.error(f"压缩失败: {e}")
            return False
    
    @staticmethod
    def list_files(
        directory: str,
        pattern: str = "*",
        recursive: bool = False
    ) -> List[str]:
        """列出文件"""
        path = Path(directory)
        if recursive:
            return [str(f) for f in path.rglob(pattern) if f.is_file()]
        return [str(f) for f in path.glob(pattern) if f.is_file()]
```

---

## 6. 定时任务与Cron配置

### 6.1 Python APScheduler

```python
# src/scheduler.py
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger
from apscheduler.executors.pool import ThreadPoolExecutor, ProcessPoolExecutor
from datetime import datetime
from typing import Callable, Optional
from .logger import get_logger
import pytz

logger = get_logger(__name__)

# 中国时区
CN_TZ = pytz.timezone("Asia/Shanghai")


class TaskScheduler:
    """任务调度器"""
    
    def __init__(self, scheduler_type: str = "blocking"):
        executors = {
            "default": ThreadPoolExecutor(10),
            "processpool": ProcessPoolExecutor(4)
        }
        job_defaults = {
            "coalesce": True,      # 合并错过的执行
            "max_instances": 1,    # 同一任务最大实例数
            "misfire_grace_time": 300  # 5分钟内容错
        }
        
        if scheduler_type == "blocking":
            self.scheduler = BlockingScheduler(
                executors=executors,
                job_defaults=job_defaults,
                timezone=CN_TZ
            )
        else:
            self.scheduler = AsyncIOScheduler(
                executors=executors,
                job_defaults=job_defaults,
                timezone=CN_TZ
            )
    
    def add_cron_job(
        self,
        func: Callable,
        job_id: str,
        hour: int = None,
        minute: int = None,
        day_of_week: str = "*",
        day: int = None,
        month: int = None
    ):
        """添加Cron任务"""
        trigger = CronTrigger(
            year="*",
            month=month or "*",
            day=day or "*",
            day_of_week=day_of_week,
            hour=hour or "*",
            minute=minute or "*",
            timezone=CN_TZ
        )
        
        self.scheduler.add_job(
            func,
            trigger=trigger,
            id=job_id,
            name=getattr(func, "__name__", job_id),
            replace_existing=True
        )
        logger.info(f"已添加Cron任务: {job_id}")
    
    def add_interval_job(
        self,
        func: Callable,
        job_id: str,
        seconds: int = None,
        minutes: int = None,
        hours: int = None
    ):
        """添加间隔任务"""
        trigger = IntervalTrigger(
            seconds=seconds or 0,
            minutes=minutes or 0,
            hours=hours or 0,
            timezone=CN_TZ
        )
        
        self.scheduler.add_job(
            func,
            trigger=trigger,
            id=job_id,
            replace_existing=True
        )
        logger.info(f"已添加间隔任务: {job_id} (每{hours}h{minutes}m{seconds}s)")
    
    def start(self):
        """启动调度器"""
        logger.info("任务调度器启动...")
        try:
            self.scheduler.start()
        except (KeyboardInterrupt, SystemExit):
            self.stop()
    
    def stop(self):
        """停止调度器"""
        logger.info("任务调度器停止...")
        self.scheduler.shutdown(wait=True)


# 预定义任务函数
def daily_report_job():
    """每日报告任务"""
    from .crawler import CryptoPriceCrawler
    from .api_client import CoinGeckoClient
    from .file_processor import FileProcessor
    import os
    
    logger.info("执行每日报告任务...")
    
    # 获取加密货币数据
    client = CoinGeckoClient()
    data = client.get_market_data(per_page=50)
    
    if data:
        output_dir = os.path.join(
            os.path.dirname(__file__), "..", "data", "reports"
        )
        timestamp = datetime.now().strftime("%Y%m%d")
        
        # 保存日报
        FileProcessor.save_to_json(
            data,
            os.path.join(output_dir, f"market_report_{timestamp}.json")
        )
        FileProcessor.save_to_csv(
            data,
            os.path.join(output_dir, f"market_report_{timestamp}.csv")
        )
        
        logger.info(f"日报已生成: market_report_{timestamp}")


def price_alert_job():
    """价格监控任务（每小时执行）"""
    from .api_client import CoinGeckoClient
    from .logger import get_logger
    
    logger = get_logger("price_alert")
    client = CoinGeckoClient()
    
    # 监控的币种
    watchlist = ["bitcoin", "ethereum", "solana", "bnb"]
    
    prices = client.get_simple_price(watchlist)
    
    if prices:
        for coin, price_data in prices.items():
            usd_price = price_data.get("usd", 0)
            change_24h = price_data.get("usd_24h_change", 0)
            
            # 价格波动超过5%时记录
            if abs(change_24h) > 5:
                logger.warning(
                    f"⚠️ {coin.upper()} 24h波动: {change_24h:.2f}% "
                    f"当前价格: ${usd_price:,.2f}"
                )
```

### 6.2 Linux Cron 配置

```bash
# 编辑 crontab
crontab -e

# 格式: 分 时 日 月 周 命令
# 每分钟执行一次Python脚本
* * * * * /path/to/venv/bin/python /path/to/main.py >> /path/to/logs/cron.log 2>&1

# 每天早上9点执行日报
0 9 * * * /path/to/venv/bin/python /path/to/main.py --task daily_report >> /path/to/logs/daily.log 2>&1

# 每小时执行一次价格监控
0 * * * * /path/to/venv/bin/python /path/to/main.py --task price_alert >> /path/to/logs/price.log 2>&1

# 每天凌晨3点执行数据备份
0 3 * * * /usr/bin/tar -czf /backup/data_$(date +\%Y\%m\%d).tar.gz /path/to/data/

# 查看cron日志
grep CRON /var/log/syslog
# macOS:
log show --predicate 'process == "cron"' --last 1d
```

### 6.3 systemd 服务（Linux生产环境）

```ini
# /etc/systemd/system/automation.service
[Unit]
Description=Python Automation Service
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/automation
Environment="PYTHONPATH=/opt/automation"
Environment="PATH=/opt/automation/venv/bin"
ExecStart=/opt/automation/venv/bin/python /opt/automation/main.py
Restart=always
RestartSec=10
StandardOutput=append:/var/log/automation/stdout.log
StandardError=append:/var/log/automation/stderr.log

[Install]
WantedBy=multi-user.target
```

```bash
# 启用服务
sudo systemctl daemon-reload
sudo systemctl enable automation.service
sudo systemctl start automation.service
sudo systemctl status automation.service
```

---

## 7. 完整项目：加密货币行情监控

```python
# main.py
"""
加密货币行情监控系统
功能：
1. 每小时自动采集Top 100加密货币行情
2. 检测异常波动（24h涨跌 > 10%）
3. 生成日报Excel
4. 价格异常时发送告警
"""
import os
import sys
import argparse
from datetime import datetime

# 添加项目根目录到路径
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

from config.settings import config
from src.crawler import CryptoPriceCrawler
from src.api_client import CoinGeckoClient
from src.file_processor import FileProcessor
from src.scheduler import TaskScheduler, daily_report_job, price_alert_job
from src.logger import get_logger, setup_logging

logger = get_logger(__name__)


def collect_market_data():
    """采集市场数据"""
    logger.info("=" * 50)
    logger.info("开始采集市场数据...")
    
    client = CoinGeckoClient()
    data = client.get_market_data(per_page=100)
    
    if not data:
        logger.error("数据采集失败")
        return
    
    # 保存原始数据
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    data_file = os.path.join(config.DATA_DIR, "raw", f"market_{timestamp}.json")
    FileProcessor.save_to_json(data, data_file)
    
    # 分析数据
    alerts = []
    for coin in data:
        change = coin.get("price_change_percentage_24h", 0)
        if abs(change) > 10:
            alerts.append({
                "symbol": coin.get("symbol", "").upper(),
                "name": coin.get("name", ""),
                "price": coin.get("current_price", 0),
                "change_24h": change
            })
    
    if alerts:
        logger.warning(f"⚠️ 发现 {len(alerts)} 个异常波动:")
        for alert in alerts:
            logger.warning(
                f"  {alert['symbol']}: {alert['change_24h']:+.2f}% "
                f"@ ${alert['price']:,.2f}"
            )
        
        # 保存告警记录
        alert_file = os.path.join(config.DATA_DIR, "alerts", f"alert_{timestamp}.json")
        FileProcessor.save_to_json(alerts, alert_file)
    
    # 生成日报
    report = {
        "生成时间": timestamp,
        "总币种数": len(data),
        "总市值": sum(c.get("market_cap", 0) for c in data),
        "24h总交易量": sum(c.get("total_volume", 0) for c in data),
        "Top 5涨幅": sorted(
            [c for c in data if c.get("price_change_percentage_24h", 0) > 0],
            key=lambda x: x.get("price_change_percentage_24h", 0),
            reverse=True
        )[:5],
        "Top 5跌幅": sorted(
            [c for c in data if c.get("price_change_percentage_24h", 0) < 0],
            key=lambda x: x.get("price_change_percentage_24h", 0)
        )[:5]
    }
    
    report_file = os.path.join(config.DATA_DIR, "reports", f"daily_{timestamp}.json")
    FileProcessor.save_to_json(report, report_file)
    
    # Excel格式日报
    excel_file = os.path.join(config.DATA_DIR, "reports", f"daily_{timestamp}.xlsx")
    FileProcessor.json_to_excel(data[:50], excel_file, sheet_name="Top50")
    
    logger.info(f"数据采集完成! 共 {len(data)} 个币种")
    return data


def main():
    parser = argparse.ArgumentParser(description="加密货币行情监控")
    parser.add_argument("--task", choices=["collect", "daemon"], default="collect")
    parser.add_argument("--once", action="store_true", help="仅执行一次")
    args = parser.parse_args()
    
    setup_logging()
    logger.info("加密货币行情监控系统启动")
    
    if args.task == "collect":
        collect_market_data()
    elif args.task == "daemon":
        scheduler = TaskScheduler(scheduler_type="blocking")
        
        # 每小时执行数据采集
        scheduler.add_cron_job(
            collect_market_data,
            "collect_market_data",
            minute=0  # 每小时整点执行
        )
        
        # 每5分钟检测价格异常
        scheduler.add_interval_job(
            price_alert_job,
            "price_alert",
            minutes=5
        )
        
        scheduler.start()


if __name__ == "__main__":
    main()
```

---

## 8. 错误处理与日志系统

```python
# src/logger.py
import logging
import os
from pathlib import Path
from logging.handlers import RotatingFileHandler
from datetime import datetime
import sys

LOG_DIR = Path(__file__).parent.parent / "logs"
LOG_DIR.mkdir(exist_ok=True)


def setup_logging(
    level: int = logging.INFO,
    log_file: str = None,
    max_bytes: int = 10 * 1024 * 1024,  # 10MB
    backup_count: int = 5
):
    """配置日志系统"""
    
    # 根日志器
    root_logger = logging.getLogger()
    root_logger.setLevel(level)
    root_logger.handlers.clear()
    
    # 格式化
    formatter = logging.Formatter(
        "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )
    
    # 控制台处理器
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(level)
    console_handler.setFormatter(formatter)
    root_logger.addHandler(console_handler)
    
    # 文件处理器（自动轮转）
    if log_file is None:
        log_file = LOG_DIR / f"automation_{datetime.now().strftime('%Y%m%d')}.log"
    
    file_handler = RotatingFileHandler(
        log_file,
        maxBytes=max_bytes,
        backupCount=backup_count,
        encoding="utf-8"
    )
    file_handler.setLevel(level)
    file_handler.setFormatter(formatter)
    root_logger.addHandler(file_handler)
    
    return root_logger


def get_logger(name: str) -> logging.Logger:
    """获取Logger实例"""
    return logging.getLogger(name)
```

---

## 9. 性能优化技巧

### 9.1 批量操作

```python
# ❌ 低效：逐个请求
for url in urls:
    response = requests.get(url)
    process(response)

# ✅ 高效：并发请求
import concurrent.futures

with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
    results = list(executor.map(fetch_url, urls))
```

### 9.2 数据库批量写入

```python
# ❌ 低效：逐条插入
for item in data:
    db.insert(item)

# ✅ 高效：批量插入
db.insert_many(data, batch_size=1000)
```

### 9.3 缓存策略

```python
from functools import lru_cache
import time

class RateLimitedClient:
    def __init__(self):
        self._cache = {}
        self._cache_ttl = 300  # 5分钟缓存
    
    def get_with_cache(self, key: str, fetch_func):
        now = time.time()
        if key in self._cache:
            cached_value, cached_time = self._cache[key]
            if now - cached_time < self._cache_ttl:
                return cached_value
        
        value = fetch_func()
        self._cache[key] = (value, now)
        return value
```

---

## 总结

本文覆盖了 Python 自动化开发的全链路技能：

1. **环境配置** - 虚拟环境、依赖管理、项目结构
2. **网页爬虫** - Requests + BeautifulSoup + Playwright
3. **API调用** - 同步/异步客户端设计
4. **文件处理** - JSON/CSV/Excel 读写转换
5. **定时任务** - APScheduler + Cron + systemd
6. **日志系统** - 结构化日志与自动轮转
7. **性能优化** - 并发、批量、缓存策略

---

## 💼 需要Python开发服务？

河北高软科技提供以下 **Python开发服务**：

- 🐍 Python 脚本定制开发
- 🌐 网页爬虫与数据采集
- 📊 数据处理与报表系统
- ⏰ 定时任务与自动化流程
- 🔌 API集成与后端服务

**联系方式：13315868800（微信同号）**
