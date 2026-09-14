# 轻新课堂 · 课程签到

Sign-in-qxkt：国科大课程查询与签到助手。

UCAS 课程查询与签到二维码生成工具。

本仓库：https://github.com/windyBN/Sign-in-qxkt

本版本尚未提供独立部署地址。

支持以下功能：

1. 输入学号、密码和日期，查询当天课程
2. 选择课程，生成可实时刷新的签到二维码
3. 在可签到时间内直接发起签到
4. 手动输入课程 ID 或 UUID，生成对应签到码

## 界面与交互

- 中文标题、蓝灰配色、轻阴影卡片与分组导航。
- 支持亮色、暗色和初始跟随系统主题，以及移动端布局。
- 页面已移除 GitHub 链接、Star 数、仓库更新时间及相应请求、缓存逻辑。
- 二维码默认每 5 秒刷新；界面的倒计时是本地刷新策略，实际是否有效由学校服务器决定。

## 快速开始

需要 Node.js 20.9 或更新版本（建议使用受支持的 LTS 版本）。

### 本地运行

1. 克隆仓库并安装依赖

```bash
git clone https://github.com/windyBN/Sign-in-qxkt.git
cd Sign-in-qxkt
npm ci
```

2. 启动开发环境

```bash
npm run dev
```

默认访问地址：`http://localhost:3000`

3. 生产构建与启动

```bash
npm run build
npm run start
```

4. 代码检查

```bash
npm run lint
```

### 部署上线（可选）

推荐使用 Vercel 进行部署，步骤如下：

1. 在 Vercel 中选择本仓库 `windyBN/Sign-in-qxkt`
2. 在 Vercel 导入项目
3. Framework 自动识别为 Next.js
4. Build Command 使用默认的 `npm run build`
5. 部署完成后访问生成的域名

## 工作流程

```text
Browser
	-> POST /api/course-uuid/query
		-> login.action (上游登录)
		-> get_stu_course_sched.action (上游课表)
	<- 返回课程列表（筛选后的课程字段）
Browser
	-> 选择课程 / 手动输入课程 ID 或 UUID
	-> 本地生成签到 URL + QR Code
	-> POST /api/course-uuid/sign（可选，直接签到）
```

说明：

- 查询课程和直接签到时，学号与密码会经本站后端转交学校平台；当前代码不将其写入数据库或浏览器持久存储。
- 上游 `sessionId` 只在服务端请求链路中短暂使用，不回传前端。
- 前端二维码和下载二维码都由本地生成，不依赖额外前端存储。
- 直接签到时，服务端会先登录，再调用上游签到接口。

## 项目结构

```text
.
├─ src/
│  └─ app/
│     ├─ api/
│     │  └─ course-uuid/
│     │     ├─ query/
│     │     │  └─ route.ts      # 登录 + 课表查询接口
│     │     ├─ sign/
│     │     │  └─ route.ts      # 登录 + 直接签到接口
│     │     └─ timestamp/
│     │        └─ route.ts      # 学校服务器时间
│     ├─ globals.css            # 全局样式与主题变量
│     ├─ layout.tsx             # 字体、元信息、主题初始化
│     └─ page.tsx               # 主页面（查询、列表、二维码、签到）
├─ public/
├─ doc/
├─ package.json
└─ README.md
```

## API 说明

### POST /api/course-uuid/query

查询课程列表。

请求头：

- `Content-Type: application/json`

请求体：

```json
{
  "username": "2025xxxxxxxxxx",
  "password": "your-password",
  "date": "20260325"
}
```

字段说明：

- `username`：学号，必填
- `password`：密码，必填
- `date`：查询日期，支持 `yyyyMMdd` 或 `yyyy-MM-dd`

成功响应示例：

```json
{
  "date": "20260325",
  "total": 2,
  "courses": [
    {
      "id": "114xxxx",
      "uuid": "CADD27F17ACC44EDAFxxxxxxxxxxxxxx",
      "courseName": "xxxxxxx",
      "teacherName": "xxx",
      "weekDay": "周三",
      "classBeginTime": "2026-03-25 10:25:00",
      "classEndTime": "2026-03-25 12:00:00",
      "signStatus": "1"
    }
  ]
}
```

常见错误响应：

```json
{
  "message": "登录接口请求超时",
  "code": "UPSTREAM_LOGIN_TIMEOUT"
}
```

### POST /api/course-uuid/sign

发起直接签到。

请求头：

- `Content-Type: application/json`

请求体：

```json
{
  "username": "2025xxxxxxxxxx",
  "password": "your-password",
  "courseSchedId": "1234567",
  "timestamp": 1774405500000
}
```

字段说明：

- `username`：学号，必填
- `password`：密码，必填
- `courseSchedId`：7 位数字课程 ID，必填。当前直接签到接口不接受 UUID；UUID 仅用于手动生成二维码。
- `timestamp`：毫秒时间戳，前端根据学校服务器时间校准。示例值仅展示格式，不能直接用于签到。
- 页面将直接签到入口限制为开课前 30 分钟至下课时间，最终结果以学校服务器返回为准。

成功响应示例：

```json
{
  "success": true,
  "message": "签到成功",
  "upstreamStatus": "0",
  "result": {
    "stuSignId": "123456",
    "stuSignStatus": "1"
  }
}
```

可能的失败响应：

```json
{
  "success": false,
  "message": "签到失败，请稍后重试",
  "upstreamStatus": "1",
  "result": {
    "stuSignId": "123456",
    "stuSignStatus": "0"
  }
}
```

### GET /api/course-uuid/timestamp

返回 `{ "success": true, "timestamp": 1774405500000 }` 形式的服务器时间，用于校准二维码时间戳。时间请求失败时前端使用已有偏差缓存或本机时间。

## 错误码与排查

### HTTP 状态码

- `400`：请求体或参数格式错误
- `401`：登录失败（账号密码错误或上游鉴权失败）
- `403`：非同源请求
- `409`：签到请求已提交，但状态未完成
- `415`：Content-Type 不是 JSON
- `429`：触发限流
- `502`：上游接口异常或返回异常
- `504`：上游接口超时
- `500`：服务内部异常

### 常见 `code`

- `RATE_LIMITED`
- `UPSTREAM_LOGIN_HTTP`
- `UPSTREAM_LOGIN_BAD_JSON`
- `UPSTREAM_LOGIN_TIMEOUT`
- `UPSTREAM_LOGIN_NETWORK`
- `UPSTREAM_SCHEDULE_HTTP`
- `UPSTREAM_SCHEDULE_BAD_JSON`
- `UPSTREAM_SCHEDULE_TIMEOUT`
- `UPSTREAM_SCHEDULE_NETWORK`
- `UPSTREAM_SIGN_HTTP`
- `UPSTREAM_SIGN_BAD_JSON`
- `UPSTREAM_SIGN_TIMEOUT`
- `UPSTREAM_SIGN_NETWORK`
- `UNEXPECTED_ERROR`

## 技术栈

### 前端

- Next.js 16.2.1（App Router）
- React 19.2.4
- TypeScript 5
- Tailwind CSS 4
- next/font（Noto Sans SC / Noto Serif SC / IBM Plex Mono）

### 服务端

- Next.js Route Handler（Node.js runtime）
- 原生 Fetch + AbortController 超时控制
- 内存级限流（5 分钟窗口 + 每日上限）

### 工具链

- ESLint 9 + eslint-config-next
- qrcode 1.5.4（前端二维码生成）

## 来源与许可证

本项目基于 [lccipher/UCAS-Course-Sign-in](https://github.com/lccipher/UCAS-Course-Sign-in) 修改，保留上游提交历史。当前版本调整了前端展示并修订文档。

沿用 [AGPL-3.0 License](./LICENSE)。
