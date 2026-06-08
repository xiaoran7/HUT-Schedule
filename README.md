# 湖南工业大学教务系统全功能爬虫 (双引擎开源版)

本爬虫项目用于抓取湖南工业大学强智教务系统的**个人课表**、**考试安排**、**课程成绩**及**学期与周次信息**，并支持将数据输出为结构化的 JSON 文件。

项目同时提供了 **Playwright** 和 **Selenium** 两种浏览器自动化引擎的实现，以满足不同运行环境和部署偏好的需求。本项目支持以 CLI 命令行方式被 Electron 前端或后端主进程调用。

> [!NOTE]
> 爬虫与前端系统的对接标准及数据格式规范，请参考项目外置的：[对接开发指南 (crawler_integration_guide.md)](file:///C:/Users/tanner/Desktop/crawler_integration_guide.md)。

---

## 1. 核心设计与特性

* **双动力引擎支持**：提供 Playwright（现代、极速、免配置）和 Selenium（传统、成熟）两个版本。
* **极速双模登录 (Requests + Browser)**：
  1. **Cookie 缓存复用**：登录成功后的 Cookie 会持久化存入用户目录下的 `~/.hut_session.json`。后续运行优先校验缓存，若未过期（TTL 4小时）则直接复用，**跳过所有登录步骤（耗时约 0.5s）**。
  2. **Requests 直连登录 (P0 推荐)**：如果缓存失效，爬虫首先通过 Requests 获取教务系统的 RSA 公钥，在内存中完成密码加密并模拟登录。**无需打开浏览器，全程无感同步（耗时约 2s）**。
  3. **浏览器自动兜底登录**：若直连登录失败（如需图形验证码或滑块），则自动唤起 Chromium/Chrome 浏览器让用户进行登录，成功后截获 Cookie 并更新缓存。
* **智能动态分页 (Dynamic Pagination)**：自动分页拉取多页成绩与考试安排，无需硬编码 `pageSize`。
* **学期与周次自动识别**：支持学期参数 `auto`，自动提取活跃学期，并根据主页当前周次反推学期开学日期（第一周周一）。

---

## 2. 版本对比与目录结构

本项目包含两个版本的爬虫实现：

| 维度 | Playwright 版本 (推荐) | Selenium 版本 |
| :--- | :--- | :--- |
| **文件位置** | [`playwright/hut_schedule.py`](file:///c:/Users/tanner/Desktop/selenium/playwright/hut_schedule.py) | [`hut_schedule.py`](file:///c:/Users/tanner/Desktop/selenium/hut_schedule.py) |
| **环境依赖** | Python 3.10+，支持一键自检与自动修复 | Python 3.10+，需系统预装 Chrome 浏览器 |
| **浏览器驱动** | 首次运行自动检测安装 (Chromium 内核) | 通过 `webdriver-manager` 自动配置 Chrome 驱动 |
| **启动速度** | 极快，运行开销小 | 较快 |
| **反爬规避** | 强 (Playwright 默认防检测能力优异) | 中 (需配置 Chrome Options 绕过 CDP 检测) |

---

## 3. Playwright 版本详解 (`/playwright`)

Playwright 版本位于 [`playwright/`](file:///c:/Users/tanner/Desktop/selenium/playwright/) 文件夹中，提供了**完全自动化的依赖检测和静默安装服务**。

### 3.1 环境要求
- **Python 版本**：Python 3.10+
- **系统环境**：Windows / macOS / Linux (无需手动配置任何浏览器环境，Playwright 会全自动下载 Chromium 内核)

### 3.2 依赖介绍
主要依赖列在 [`playwright/requirements.txt`](file:///c:/Users/tanner/Desktop/selenium/playwright/requirements.txt) 中：
- `requests>=2.31` (网络请求与直连登录)
- `beautifulsoup4>=4.12` & `lxml>=5.0` (HTML 解析)
- `playwright>=1.40.0` (Chromium 浏览器登录兜底)
- `pycryptodome>=3.19.0` (CAS 密码 RSA 加密)

### 3.3 零配置快速开始 (推荐)
Playwright 版本已内置**自检机制**。你不需要手动运行任何 `pip` 或 `playwright install` 命令。直接运行脚本，脚本会自动检测缺失项并静默安装：
```bash
# 激活 Conda 或 Python 虚拟环境，直接运行
python playwright/hut_schedule.py --user "您的学号" --pwd "您的密码" --term auto --mode all
```

*若想手动安装依赖，也可通过以下步骤：*
```bash
pip install -r playwright/requirements.txt
python -m playwright install chromium
```

---

## 4. Selenium 版本详解 (根目录)

Selenium 版本位于项目根目录下，是传统的浏览器自动化解决方案。

### 4.1 环境要求
- **Python 版本**：Python 3.10+
- **系统环境**：Windows / macOS / Linux
- **硬性前提**：**必须**在系统上安装 Google Chrome 浏览器。

### 4.2 依赖介绍
主要依赖列在根目录 [`requirements.txt`](file:///c:/Users/tanner/Desktop/selenium/requirements.txt) 中：
- `requests>=2.31`
- `beautifulsoup4>=4.12` & `lxml>=5.0`
- `selenium>=4.15` (Chrome 浏览器登录兜底)
- `webdriver-manager>=4.0` (自动下载匹配当前 Chrome 版本的 ChromeDriver)
- `pycryptodome>=3.19.0`

### 4.3 快速开始
1. 安装 Python 依赖包：
   ```bash
   pip install -r requirements.txt
   ```
2. 运行脚本：
   ```bash
   python hut_schedule.py --user "您的学号" --pwd "您的密码" --term auto --mode all
   ```

---

## 5. 命令行使用说明

两版本爬虫接受相同的 CLI 参数，支持完全一致的功能：

```bash
# 1. 抓取整学期所有课表、考试、成绩，并保存为 JSON 文件
python playwright/hut_schedule.py --user "123456" --pwd "mypwd" --term "2025-2026-2" --mode all --out "result.json"

# 2. 仅抓取课程成绩 (单模块运行)
python playwright/hut_schedule.py --user "123456" --pwd "mypwd" --term "2025-2026-2" --mode grade

# 3. 强制重新登录（跳过本地 Cookie 缓存，重新调起浏览器或 Requests）
python playwright/hut_schedule.py --user "123456" --pwd "mypwd" --relogin
```

### CLI 参数列表

| 参数 | 默认值 | 可选值 | 说明 |
| :--- | :--- | :--- | :--- |
| `--user` | 空 | - | 学号/账号 |
| `--pwd` | 空 | - | 统一身份认证密码 |
| `--term` | 自动推算 | `auto` / `2025-2026-1` ... | 目标学期，`auto` 代表教务当前活跃学期 |
| `--week` | 空 | - | 目标周次，格式如 `"第13周"`。留空代表整学期 |
| `--kbjcmsid` | 空 | - | 节次时间模式 ID（特殊学期调整大节时间时使用） |
| `--mode` | `all` | `schedule` / `exam` / `grade` / `info` / `all` | 抓取模块分类。`all` 包含全部四个模块 |
| `--out` | 空 | - | 结果输出的 JSON 路径，留空则仅控制台打印 |
| `--relogin` | `False` | 命令行直接添加即可 | 是否忽略本地 Cookie 缓存强制执行重新登录 |
| `--headless` | `False` | 命令行直接添加即可 | 调起浏览器兜底登录时是否使用无头模式 (Headless) |

---
