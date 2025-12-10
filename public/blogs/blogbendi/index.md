

2025 Blog 本地服务器部署与Webhook自动更新教程
=============================================

本人使用是实时更新 感觉不到延迟（香港服务器）其他场景自行研究

📌 前提条件
-----------

* 已完成 Github Fork 和 Github App 创建操作
* 本地服务器支持 Git 工具
* 本地服务器可访问 github.com
* 已安装 Node.js 环境（推荐 v24.11.1 稳定版）
* 已安装 pnpm 包管理器

📖 基本原理
-----------

将项目部署在本地服务器，通过 Webhook 机制监听 GitHub 仓库推送事件，自动执行 `git pull` 命令同步 GitHub 仓库内容，实现网站内容的自动更新。

📁 内容文件
-----------

| 内容类型 | 文件路径 |
| --- | --- |
| 博客文章 | `public/blogs/` 目录（每个博客一个子目录） |
| 项目列表 | `src/app/projects/list.json` |
| 分享内容 | `src/app/share/list.json` |
| 网站配置 | `src/config/site-content.json` |
| 卡片样式 | `src/config/card-styles.json` |

🚀 部署步骤
-----------

### 1\. 克隆仓库到本地服务器

```
# 创建项目目录
mkdir -p /www/wwwroot/2025Blog

# 进入项目目录
cd /www/wwwroot/2025Blog

# 克隆你的 GitHub 仓库（替换为你的仓库地址）
git clone https://github.com/your-username/2025-blog-public.git
```

### 2\. 修改 APP\_ID 环境变量

1. 进入仓库目录：`cd /www/wwwroot/2025Blog/2025-blog-public`
2. 编辑 `src/consts.ts` 文件，修改 `APP_ID` 为你的 GitHub App ID：
   
   export const GITHUB_CONFIG = {
   OWNER: process.env.NEXT_PUBLIC_GITHUB_OWNER || 'your-username',
   REPO: process.env.NEXT_PUBLIC_GITHUB_REPO || '2025-blog-public',
   BRANCH: process.env.NEXT_PUBLIC_GITHUB_BRANCH || 'main',
   APP_ID: process.env.NEXT_PUBLIC_GITHUB_APP_ID || 'your-app-id'
   } as const

### 3\. 宝塔面板部署步骤

1. **安装 Node.js 管理器**：
   
   * 进入宝塔面板 → 软件商店 → 安装 "Node.js管理器"
2. **安装 Node.js 版本**：
   
   * 进入 Node.js管理器 → 版本管理 → 下载并安装 v24.11.1 稳定版
3. **安装 pnpm 模块**：
   
   * 进入 Node.js管理器 → 模块管理 → 安装 pnpm 模块
4. **添加 Node.js 项目**：
   
   * 进入宝塔面板 → 网站 → Node.js项目 → 添加项目
   * **项目目录**：选择克隆的仓库目录（如 `/www/wwwroot/2025Blog/2025-blog-public`）
   * **名称**：自定义（如 "2025-blog"）
   * **启动选项**：`dev:next dev --turbopack -p 1234`（保持默认自动生成的选项）
   * **端口**：填写 `2025`
   * **域名**：填写你的域名（如 `blog.your-domain.com`）
   * **不安装node\_modules**：取消勾选（首次部署需下载依赖）
   * 点击 **确定**，等待依赖下载完成
5. **启动项目**：
   
   * 依赖下载完成后，项目会自动启动
   * 可在 Node.js项目列表中查看运行状态

### 4\. 其他面板/系统部署

基本步骤相同，核心是：

* 安装 Node.js v24.11.1 及 pnpm
* 克隆仓库
* 安装依赖：`pnpm install`
* 启动项目：`pnpm dev` 或使用进程管理工具（如 PM2）

🔧 添加 Webhook 自动更新功能
----------------------------

### 1\. 创建 Webhook 端点文件

创建 `2025-blog-public/src/app/webhook/route.ts` 文件，添加以下代码：

```
import { NextRequest, NextResponse } from 'next/server';
import crypto from 'node:crypto';
import { execSync } from 'node:child_process';

// 从环境变量获取GitHub Webhook密钥（可选，增强安全性）
const GITHUB_SECRET = process.env.GITHUB_WEBHOOK_SECRET || '';
// 项目根目录路径
const PROJECT_PATH = process.cwd();

// 简化的日志函数，只输出到控制台
function log(...args: any[]) {
  console.log(`[${new Date().toISOString()}] ${args.join(' ')}`);
}

/**
 * 验证GitHub Webhook签名
 */
function verifySignature(payload: string, signature: string): boolean {
  if (!GITHUB_SECRET) return true;
  
  const hmac = crypto.createHmac('sha256', GITHUB_SECRET);
  hmac.update(payload);
  const expectedSignature = `sha256=${hmac.digest('hex')}`;
  
  return signature === expectedSignature;
}

/**
 * 简化的命令执行函数，只返回执行结果
 */
function executeCommand(cmd: string, timeout: number = 5000): string {
  return execSync(cmd, { encoding: 'utf-8', timeout });
}

/**
 * 执行git pull的API端点
 */
export async function POST(request: NextRequest) {
  try {
    // 1. 验证GitHub签名
    const signature = request.headers.get('x-hub-signature-256') || '';
    const payload = await request.text();
    
    if (!verifySignature(payload, signature)) {
      return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
    }
    
    // 2. 设置Git安全目录（只执行一次，后续自动生效）
    try {
      executeCommand(`git config --global --add safe.directory "${PROJECT_PATH}"`, 5000);
    } catch (e) {
      // 忽略重复添加的错误
    }
    
    // 3. 检查并处理本地修改
    try {
      const status = executeCommand(`git -C "${PROJECT_PATH}" status --porcelain`, 5000);
      
      if (status.trim() !== '') {
        // 有未提交的修改，先保存
        executeCommand(`git -C "${PROJECT_PATH}" stash push -m "Webhook auto stash"`, 5000);
      }
      
      // 4. 执行git pull
      const gitPull = executeCommand(`git -C "${PROJECT_PATH}" pull --force`, 10000);
      
      return NextResponse.json({
        success: true,
        message: 'git pull executed successfully',
        result: gitPull,
        timestamp: new Date().toISOString()
      });
    } catch (pullError: any) {
      // 智能错误处理
      if (pullError.message.includes('Your local changes to the following files would be overwritten by merge')) {
        return NextResponse.json({
          success: false,
          message: 'git pull failed due to local changes conflict',
          error: pullError.message,
          suggestedSolution: `请手动执行以下命令解决冲突：\n1. git -C ${PROJECT_PATH} status\n2. git -C ${PROJECT_PATH} stash push -m "manual stash"\n3. git -C ${PROJECT_PATH} pull\n4. git -C ${PROJECT_PATH} stash pop\n5. 解决冲突并提交`
        }, { status: 500 });
      } else if (pullError.message.includes('Permission denied')) {
        return NextResponse.json({
          success: false,
          message: 'git pull failed due to permission error',
          error: pullError.message,
          suggestedSolution: `请手动执行以下命令解决权限问题：\n1. sudo chown -R www:www ${PROJECT_PATH}/.git\n2. sudo chmod -R 775 ${PROJECT_PATH}/.git`
        }, { status: 500 });
      } else {
        return NextResponse.json({
          success: false,
          message: 'git pull failed',
          error: pullError.message
        }, { status: 500 });
      }
    }
  } catch (error: any) {
    return NextResponse.json({
      success: false,
      message: 'webhook handler failed',
      error: error.message
    }, { status: 500 });
  }
}

/**
 * 健康检查端点
 */
export async function GET(request: NextRequest) {
  return NextResponse.json({
    success: true,
    message: 'Webhook endpoint is running',
    timestamp: new Date().toISOString(),
    status: 'healthy'
  });
}
```

### 2\. 配置 GitHub Webhook

1. 登录 GitHub，进入仓库 → **Settings** → **Webhooks**
2. 点击 **Add webhook**
3. 配置参数：
   * **Payload URL**：`https://你的域名/webhook`（替换为你的域名）
   * **Content type**：选择 `application/json`
   * **Secret**：设置一个安全的密钥（与 `route.ts` 中 `GITHUB_SECRET` 保持一致，可选）
   * **Which events would you like to trigger this webhook?**：选择 **Just the push event.**
   * **Active**：勾选
4. 点击 **Add webhook** 完成配置

🧪 测试 Webhook 功能
--------------------

1. **编辑网站内容**：
   
   * 使用 GitHub App 密钥登录网站，编辑一些内容
   * 保存编辑，等待 GitHub 仓库内容更新
2. **验证自动更新**：
   
   * 刷新网站，检查内容是否自动更新
   * 查看服务器日志，确认 Webhook 事件被触发

🚧 常见问题排查
---------------

### 1\. 内容未自动更新

* **检查 Webhook 日志**：
  
  * GitHub Webhook 页面 → 查看 **Recent Deliveries**，确认请求状态为 **200 OK**
* **检查服务器日志**：
  
  * 宝塔面板 → Node.js项目 → 日志，查看 Webhook 执行情况
* **检查 .git 目录权限**：
  
  * 确保网站运行用户对 `.git` 目录有读写权限
  * 执行命令：`chown -R www:www /www/wwwroot/2025Blog/2025-blog-public/.git`
  * 执行命令：`chmod -R 775 /www/wwwroot/2025Blog/2025-blog-public/.git`
* **手动测试 Webhook**：
  
  * 使用 `curl` 发送测试请求：`curl -X POST https://你的域名/webhook \ -H "Content-Type: application/json" \ -d '{"test": "webhook"}'`

### 2\. Git 命令执行失败

* **确保 Git 已安装**：
  
  * 执行命令：`git --version`
  * 如未安装，执行：`apt install git -y`（Debian/Ubuntu）或 `yum install git -y`（CentOS/RHEL）
* **检查 Git 安全目录配置**：
  
  * 执行命令：`git config --global --get-all safe.directory`
  * 确保项目目录已添加到安全目录列表

📋 总结
-------

通过以上步骤，你已成功部署了 2025 Blog 并配置了 Webhook 自动更新功能：

1. ✅ 本地服务器部署完成
2. ✅ 网站可正常访问和编辑
3. ✅ GitHub Webhook 已配置
4. ✅ 自动更新功能已启用

现在，当你编辑网站内容并保存后，GitHub 仓库会自动更新，Webhook 会触发本地服务器执行 `git pull`，实现网站内容的自动同步更新！

📚 相关资源
-----------

* 2025 Blog 使用引导：[https://www.yysuni.com/blog/readme](https://www.yysuni.com/blog/readme)
* Node.js 官网：[https://nodejs.org/](https://nodejs.org/)
* pnpm 官网：[https://pnpm.io/](https://pnpm.io/)

---

**优化建议**：

* 配置 HTTPS 证书，提高网站安全性
* 定期更新 Node.js 和依赖包，保持系统安全性



