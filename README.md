# 湖南工业大学 教务系统全功能爬虫 (Playwright 版)

本爬虫用于抓取湖南工业大学强智教务系统的**个人课表**、**考试安排**、**课程成绩**及**学期与周次信息**，并输出为结构化的 JSON 数据供前端系统进行同步和展示。

---

## 1. 抓包接口设计

| 功能模块 | 请求地址 (基于 `http://jwxt.hut.edu.cn/jsxsd`) | 请求方式 | 关键参数 |
| :--- | :--- | :--- | :--- |
| **单点登录入口** | `/sso.jsp` | GET | - |
| **课表查询** | `/xskb/xskb_list.do` | GET | `xnxq01id` (学期), `zc` (周次), `viweType=0` |
| **考试安排** | `/xsks/xsksap_list` | GET | `xnxqid` (学期), `pageNum`, `pageSize` |
| **课程成绩** | `/kscj/cjcx_list` | GET | `kksj` (开课学期，空代表全部学期), `pageNum`, `pageSize` |
| **学期周次** | `/xskb/jxzlzc_xnxq_ajax` | GET | `xnxq01id` (学期) |
| **当前周次** | `/framework/xsMainV.htmlx` | GET | 解析网页正文中的 `"第N周"` 字样 |

---

## 2. 核心设计与特性

### 2.1 极速双模登录 (Requests + Playwright)
为了在效率与成功率之间取得平衡，爬虫实现了两阶段登录模式：
1. **Cookie 缓存复用**：登录成功后的 Cookie 会被保存至用户目录下的 `~/.hut_session.json`。后续运行会优先校验缓存，若未过期（TTL 4小时）则直接复用，**跳过所有登录步骤（耗时约 0.5s）**。
2. **Requests 直连登录 (P0 推荐)**：如果缓存失效，爬虫首先通过 Requests 获取教务系统的 RSA 公钥，在内存中完成密码加密并模拟登录。**无需打开浏览器，全程无感同步（耗时约 2s）**。
3. **Playwright 兜底登录**：若直连登录失败（如需图形验证码或滑块），则自动唤起 Chromium 浏览器让用户进行登录，成功后截获 Cookie 并更新缓存。

### 2.2 智能动态分页 (Dynamic Pagination)
由于成绩与考试安排可能存在多页数据，爬虫会**先请求第 1 页，解析响应 JSON 中的 `count`（总条数）与当前页数据长度，动态计算总页数，并自动循环拉取后续页面**进行合并。

### 2.3 学期自动识别 (Auto Term Detection)
* 查询学期传入 `auto` 时，爬虫将自动访问课表页面，解析页面顶部的 `<select id="xnxq01id">` 元素，提取出标有 `selected` 属性的选项作为**教务系统当前正活跃的学期**，实现零输入智能同步。
* 同时，通过解析主页顶部的当前周次与教学起止周，自动推算出该学期的**开学日期（第一周周一）**，便于日历课表精确定位。

---

## 3. 安装依赖

确保本机已安装 **Python 3.10+**。

```bash
# 安装 Python 依赖包
pip install -r requirements.txt

# 安装 Playwright 所需的浏览器内核 (Chromium)
playwright install chromium
```

> **主要依赖**：`requests` (网络请求)、`beautifulsoup4` (HTML解析)、`pycryptodome` (密码RSA加密)、`playwright` (浏览器兜底登录)。

---

## 4. 命令行使用说明

可以通过以下参数控制爬虫的行为：

```bash
# 同时抓取当前学期所有数据（课表、考试、成绩、学期信息），输出到 console
python hut_schedule.py --user "您的学号" --pwd "您的密码" --term auto --mode all

# 仅抓取课程成绩，指定学期，并输出到 result.json 文件中
python hut_schedule.py --user "您的学号" --pwd "您的密码" --term "2025-2026-1" --mode grade --out result.json

# 强制重新登录（忽略本地 Cookie 缓存）
python hut_schedule.py --user "您的学号" --pwd "您的密码" --relogin --mode schedule
```

### 命令行参数表
| 参数 | 默认值 | 可选值 | 说明 |
| :--- | :--- | :--- | :--- |
| `--user` | 空 | - | 学号/账号 |
| `--pwd` | 空 | - | 密码 |
| `--term` | 自动推算 | `auto` / `2025-2026-2` ... | 目标学期，`auto` 代表教务当前活跃学期 |
| `--week` | 空 | - | 周次，留空=整学期 |
| `--kbjcmsid`| 空 | - | 节次时间模式 id，留空走默认 |
| `--mode` | `all` | `schedule` / `exam` / `grade` / `info` / `all` | 抓取模块分类 |
| `--out` | 空 | - | 结果输出的 JSON 路径 |
| `--relogin`| `False` | 命令行加此参数即可 | 是否强制启动登录（不使用缓存） |
| `--headless`| `False` | 命令行加此参数即可 | 兜底启动 Playwright 时是否采用无头模式 |
