

# 依赖
node_modules/
# 运行时生成的临时文件
qr/
qrkeys.json
# 环境变量
.env
.env.*
# 系统文件
.DS_Store
Thumbs.db
# IDE
.vscode/
.idea/

MIT LicenseMIT 许可证

Copyright (c) 2026 qfmc7040版权所有 © 2026 qfmc7040

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

import { printBlue, printGreen, printMagenta, printRed, printYellow } from "./utils/colorOut.js";
import { hasSecretWriteToken, setRepoSecret } from "./utils/githubSecrets.js";
import { maskDisplayName, maskIdentifier, sanitizeForLog, summarizeResponse } from "./utils/safeLog.js";
import { sendNotify } from "./utils/notify.js";
import { close_api, delay, send, startService, waitForApi } from "./utils/utils.js";

async function main() {

  const USERINFO = process.env.USERINFO
  let needRefresh = false
  if (!USERINFO) {
    throw new Error("未配置")
  }
  const userinfo = JSON.parse(USERINFO)

  // 启动服务并等待就绪（避免冷启动竞态导致首个请求失败）
  const api = startService()
  try {
    await waitForApi()
  } catch (e) {
    close_api(api)
    throw e
  }

  const today = new Date();
  // 服务器时间比国内慢8小时
  today.setTime(today.getTime() + 8 * 60 * 60 * 1000)
  //日期
  const DD = String(today.getDate()).padStart(2, '0'); // 获取日
  const MM = String(today.getMonth() + 1).padStart(2, '0'); //获取月份，1 月为 0
  const yyyy = today.getFullYear(); // 获取年份
  const date = yyyy + '-' + MM + '-' + DD

  const errorMsg = {}
  // 通知结果收集
  const notifyResults = []
  let hasError = false

  try {
    for (const user of userinfo) {
      // 单账号异常隔离：任何一个账号的请求/解析出错，只记录该账号失败，
      // 不影响其余账号继续执行，也保证后续通知与 secret 刷新一定能触发。
      try {
        let headers = { 'cookie': 'token=' + user.token + '; userid=' + user.userid }
        const userDetail = await send(`/user/detail?timestrap=${Date.now()}`, "GET", headers)
        if (userDetail?.data?.nickname == null) {
          const safeUserId = maskIdentifier(user.userid)
          printRed(`token过期或账号不存在, userid: ${safeUserId}`)
          errorMsg[safeUserId] = {
            msg: `token过期或账号不存在, userid: ${safeUserId}`,
            data: summarizeResponse(userDetail)
          }
          notifyResults.push({
            nickname: safeUserId,
            status: '失败',
            listen: '账号不存在',
            vipClaim: '0/8',
            vipExpiry: '未知',
            error: 'token过期或账号不存在'
          })
          hasError = true
          continue
        }
        const safeNickname = maskDisplayName(userDetail.data.nickname)
        printMagenta(`账号 ${safeNickname} 开始领取VIP...`)

        // 周日刷新token
        if (today.getDay() === 0) {
          const refreshToken = await send(`/login/token?timestrap=${Date.now()}`, "POST", headers)
          if (refreshToken?.status == 1) {
            if (refreshToken?.data?.token !== user.token) {
              needRefresh = true
              printYellow(`账号 ${safeNickname} 需要刷新token`)
              user.token = refreshToken.data.token
              // 用新 token 重建本次请求的 headers，使后续听歌/VIP 领取使用刷新后的凭证
              headers = { 'cookie': 'token=' + user.token + '; userid=' + user.userid }
            }
          }
        }

        // 开始听歌
        printYellow(`开始听歌领取VIP...`)
        // 听歌获取vip
        const listen = await send(`/youth/listen/song?timestrap=${Date.now()}`, "GET", headers)

        let listenStatus = '未知'
        if (listen.status === 1) {
          printGreen("听歌领取成功")
          listenStatus = '成功'
        } else if (listen.error_code === 130012) {
          printGreen("今日已领取")
          listenStatus = '今日已领取'
        } else {
          errorMsg[`${safeNickname} listen`] = summarizeResponse(listen)
          printRed("听歌领取失败")
          listenStatus = '失败'
          hasError = true
        }

        printYellow("开始领取VIP...")
        let claimCount = 0
        let claimTotal = 0
        for (let i = 1; i <= 8; i++) {
          const ad = await send(`/youth/vip?timestrap=${Date.now()}`, "GET", headers)
          claimTotal = i
          if (ad.status === 1) {
            printGreen(`第${i}次领取成功`)
            claimCount++
            if (i != 8) {
              await delay(30 * 1000)
            }
          } else if (ad.error_code === 30002) {
            printGreen("今天次数已用光")
            break
          } else {
            printRed(`第${i}次领取失败`)
            errorMsg[`${safeNickname} ad`] = summarizeResponse(ad)
            hasError = true
            break
          }
        }

        let vipExpiry = '未知'
        const vip_details = await send(`/user/vip/detail?timestrap=${Date.now()}`, "GET", headers)
        if (vip_details.status === 1 && Array.isArray(vip_details.data?.busi_vip) && vip_details.data.busi_vip.length > 0) {
          vipExpiry = vip_details.data.busi_vip[0].vip_end_time
          printBlue(`今天是：${date}`)
          printBlue(`VIP到期时间：${vipExpiry}\n`)
        } else {
          printRed("获取失败\n")
          errorMsg[`${safeNickname} vip_details`] = summarizeResponse(vip_details)
          hasError = true
        }

        notifyResults.push({
          nickname: safeNickname,
          status: listenStatus === '失败' || claimCount === 0 ? '部分失败' : '成功',
          listen: listenStatus,
          vipClaim: `${claimCount}/${claimTotal}`,
          vipExpiry,
          error: ''
        })
      } catch (err) {
        const safeUserId = maskIdentifier(user.userid || '未知')
        printRed(`账号 ${safeUserId} 处理异常：${err && err.message ? err.message : String(err)}`)
        errorMsg[safeUserId] = { msg: '处理异常', error: err && err.message ? err.message : String(err) }
        notifyResults.push({
          nickname: safeUserId,
          status: '失败',
          listen: '异常',
          vipClaim: '0/8',
          vipExpiry: '未知',
          error: err && err.message ? err.message : String(err)
        })
        hasError = true
        continue
      }
    }

  } finally {
    close_api(api)
  }

  // 更新secret <USERINFO>（使用完整 userinfo 数组，保留所有用户包括过期账号）
  let secretError = null
  if (needRefresh) {
    if (hasSecretWriteToken()) {
      const userinfoJSON = JSON.stringify(userinfo)
      try {
        setRepoSecret("USERINFO", userinfoJSON)
        printGreen("secret <USERINFO> token刷新成功")
      } catch (error) {
        printRed("token刷新失败")
        console.dir(sanitizeForLog({ message: error.message }), { depth: null })
        secretError = new Error("secret <USERINFO> token刷新失败")
      }
    } else {
      printYellow("存在账号需要刷新token，但是未配置PAT，未刷新token最多两个月后过期")
    }
  }

  // 构建通知内容（放在 secret 更新之后、错误抛出之前，确保始终执行）
  const title = `酷狗签到${hasError ? '异常' : '成功'} ${date}`
  let content = `📅 日期: ${date}\n`
  content += `📊 账号数: ${notifyResults.length}\n`
  const successCount = notifyResults.filter(r => r.status === '成功').length
  const failCount = notifyResults.length - successCount
  content += `✅ 成功: ${successCount}  ❌ 失败: ${failCount}\n`

  for (const r of notifyResults) {
    content += `\n【${r.nickname}】\n`
    content += `  🎵 听歌领取: ${r.listen}\n`
    content += `  🎁 VIP领取: ${r.vipClaim} 次\n`
    content += `  ⏰ VIP到期: ${r.vipExpiry}\n`
    if (r.error) {
      content += `  ⚠️ 错误: ${r.error}\n`
    }
  }

  // 发送通知（确保即使 secret 更新失败也能发出）
  try {
    await sendNotify(title, content)
  } catch (e) {
    printYellow(`通知发送异常: ${e.message}`)
  }

  if (Object.keys(errorMsg).length > 0) {
    printRed("异常信息如下:")
    console.dir(sanitizeForLog(errorMsg), { depth: null })
    throw new Error("领取异常")
  }

  if (secretError) {
    throw secretError
  }

}

main().then(() => process.exit(0)).catch(e => { console.error(e); process.exit(1) })

{
  "name": "kgcheckin",
  "type": "module",
  "scripts": {
    "main": "node main.js",
    "install": "cd api && npm ci",
    "apiService": "cd api && npm run start",
    "phoneLogin": "node phoneLogin.js",
    "sent": "node sent.js",
    "qrcodeLogin": "node qrcodeLogin.js"
  },
  "dependencies": {
    "qrcode": "^1.5.3"
  }
}

import { printGreen, printRed, printYellow } from "./utils/colorOut.js";
import { sanitizeForLog, summarizeResponse } from "./utils/safeLog.js";
import { upsertUser, saveUserinfo } from "./utils/userinfo.js";
import { close_api, delay, send, startService, waitForApi } from "./utils/utils.js";

async function login() {

  const phone = process.env.PHONE
  const code = process.env.CODE
  const USERINFO = process.env.USERINFO
  const APPEND_USER = process.env.APPEND_USER
  const userinfo = (USERINFO && APPEND_USER == "是") ? JSON.parse(USERINFO) : []

  // 不使用二维码登录并且没有手机号或验证码
  if (!phone || !code) {
    throw new Error("未配置")
  }
  // 启动服务并等待就绪（避免冷启动竞态）
  const api = startService()
  try {
    await waitForApi()
  } catch (e) {
    close_api(api)
    throw e
  }

  try {
    const result = await send(`/login/cellphone?mobile=${phone}&code=${code}`, "GET", {})
    if (result.status === 1) {
      printGreen("登录成功！")
      upsertUser(userinfo, { userid: result.data.userid, token: result.data.token }, APPEND_USER == "是")
      saveUserinfo(userinfo)
    } else if (result.error_code === 34175) {
      throw new Error("暂不支持多账号绑定手机登录")
    } else {
      printRed("响应内容")
      console.dir(summarizeResponse(result), { depth: null })
      throw new Error("登录失败！请检查")
    }
  } finally {
    close_api(api)
  }
}

login().then(() => process.exit(0)).catch(e => { console.error(e); process.exit(1) })

import { createRequire } from 'module'
import fs from 'node:fs'
import { close_api, delay, send, startService, waitForApi } from "./utils/utils.js";
import { printGreen, printMagenta, printRed, printYellow } from "./utils/colorOut.js";
import { summarizeResponse } from "./utils/safeLog.js";
import { upsertUser, saveUserinfo } from "./utils/userinfo.js";

const require = createRequire(import.meta.url)
// 优先从常规 node_modules 解析（本地/全局安装场景），失败再回退到 Actions 构建产物中的 api/node_modules 硬编码路径
let QRCode
try {
  QRCode = require('qrcode')
} catch {
  QRCode = require('./api/node_modules/qrcode')
}

// GitHub Actions 运行环境下自动注入的 Step Summary 文件路径
const SUMMARY_FILE = process.env.GITHUB_STEP_SUMMARY || ''
const QR_DIR = './qr'
const KEYS_FILE = './qrkeys.json'

/**
 * 向 GitHub Step Summary 追加 Markdown 内容。
 * 非 Actions 环境（本地运行）时 SUMMARY_FILE 为空，自动跳过。
 * @param {string} markdown
 */
function appendSummary(markdown) {
  if (!SUMMARY_FILE) return
  try {
    fs.appendFileSync(SUMMARY_FILE, markdown + '\n')
    const size = fs.statSync(SUMMARY_FILE).size
    if (size > 0) {
      console.log(`[Summary] 已追加 ${Buffer.byteLength(markdown)} 字节，总计 ${size} 字节`)
    }
  } catch (err) {
    console.warn(`[Summary] 写入失败：${err.message}`)
  }
}

/**
 * 生成并展示单个二维码 — 展示渠道：
 *
 *   ① PNG 文件（qr/qr-N.png）：Release 直链 + HTML 内嵌双用途
 *   ② 自包含 HTML 页面（qr/login.html）：浏览器打开即见大图，手机直接扫
 *   ③ base64 data URI：HTML <img> 共用
 *
 * @param {string} url   酷狗扫码登录完整 URL
 * @param {number} index 账号序号（从 1 开始）
 * @param {number} total 总账号数
 * @returns {{ dataUrl: string, url: string, header: string, index: number }} 供 HTML 聚合用
 */
async function buildQr(url, index, total) {
  const header = total > 1 ? `（第 ${index}/${total} 个账号）` : ''

  // ── 1) PNG 文件（Release 直链 + HTML 内嵌双用途）──
  await QRCode.toFile(`${QR_DIR}/qr-${index}.png`, url, { width: 320, margin: 2 })

  // ── 2) base64 data URI（HTML <img> 共用）──
  const dataUrl = await QRCode.toDataURL(url, { width: 320, margin: 2 })

  // ── 3) 日志输出：指引去直链步骤 ──
  printMagenta(`\n═══ 第 ${index}/${total} 个二维码已生成 ═══`)
  console.log('')
  console.log('  🔗 请查看下一步「发布二维码图片直链」输出的链接，浏览器打开即可直接扫码')
  console.log('')

  return { dataUrl, url, header, index }
}

/**
 * 生成自包含 HTML 登录页（所有二维码的大图集中展示）
 * 用户从 artifact 下载后双击/手机打开即可直接扫码，无需任何依赖。
 */
function generateHtmlPage(qrItems) {
  const cards = qrItems.map(item => `
    <div class="card">
      <h2>账号 ${item.index}/${qrItems.length} ${item.header}</h2>
      <div class="qrcode">
        <img src="${item.dataUrl}" alt="账号${item.index} 二维码" />
      </div>
      <p class="url"><code>${item.url}</code></p>
      <p class="tip">⏳ 有效期约 2 分钟</p>
    </div>
  `).join('\n')

  return `<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>酷狗音乐扫码登录</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
body{
  font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif;
  background:#0d1117;color:#e6edf3;display:flex;flex-direction:column;
  align-items:center;min-height:100vh;padding:20px;
}
.header{text-align:center;margin-bottom:30px}
.header h1{font-size:24px;color:#58a6ff}
.header p{color:#8b949e;font-size:14px;margin-top:8px}
.cards{display:flex;flex-wrap:wrap;justify-content:center;gap:24px;width:100%;max-width:960px}
.card{
  background:#161b22;border:1px solid #30363d;border-radius:16px;
  padding:28px 20px;text-align:center;width:320px;
}
.card h2{font-size:16px;color:#e6edf3;margin-bottom:16px}
.qrcode img{
  width:280px;height:auto;border-radius:12px;
  border:3px solid #30363d;background:#fff;padding:12px;
}
.url{margin-top:14px;word-break:break-all;font-size:13px;color:#8b949e}
.tip{margin-top:8px;color:#f0883e;font-size:13px;font-weight:600}
.footer{margin-top:40px;color:#484f58;font-size:12px}
@media(max-width:400px){
  .card{width:100%;padding:20px 12px}
  .qrcode img{width:240px}
}
</style>
</head>
<body>
<div class="header">
  <h1>🎵 酷狗音乐扫码登录</h1>
  <p>使用「酷狗音乐 APP」扫描下方二维码完成登录</p>
</div>
<div class="cards">${cards}</div>
<p class="footer">此页面由 kgcheckin 自动生成 · 二维码有效期约 2 分钟 · 请尽快扫描</p>
</body>
</html>`
}

/** 解析账号数量，无效输入回退为 1 */
function resolveNumber() {
  const args = process.argv.slice(3)
  const n = parseInt(process.env.NUMBER || args[0] || "1")
  return (Number.isNaN(n) || n < 1) ? 1 : n
}

/**
 * 模式一：生成二维码（PNG + HTML），随后立即结束 step。
 * step 结束后 Release 直链即可使用，用户浏览器打开直接扫码。
 */
async function genMode() {
  const api = startService()
  try {
    await waitForApi()
  } catch (e) {
    close_api(api)
    throw e
  }
  const number = resolveNumber()
  const keys = []

  // 清理上次运行残留的 QR 文件，避免旧二维码混入本次 Release
  fs.rmSync(QR_DIR, { recursive: true, force: true })
  fs.mkdirSync(QR_DIR, { recursive: true })

  if (!SUMMARY_FILE) {
    console.log('[INFO] 非 Actions 环境（$GITHUB_STEP_SUMMARY 未设置），Summary 将跳过')
  }

  try {
    const qrItems = []

    for (let n = 0; n < number; n++) {
      const result = await send(`/login/qr/key?timestrap=${Date.now()}`, "GET", {})
      if (result.status === 1) {
        const qrcode = result.data.qrcode
        const qrUrl = `https://h5.kugou.com/apps/loginQRCode/html/index.html?qrcode=${qrcode}`
        keys.push(qrcode)
        const item = await buildQr(qrUrl, n + 1, number)
        qrItems.push(item)
      } else {
        printRed("响应内容")
        console.dir(summarizeResponse(result), { depth: null })
        throw new Error(`获取二维码密钥失败：接口返回 status=${result.status}`)
      }
    }

    if (qrItems.length > 0) {
      const htmlContent = generateHtmlPage(qrItems)
      fs.writeFileSync(`${QR_DIR}/login.html`, htmlContent, 'utf8')

      // 也把每个二维码单独做成一个 HTML 方便多账号时逐个处理
      for (const item of qrItems) {
        fs.writeFileSync(
          `${QR_DIR}/qr-${item.index}.html`,
          `<!DOCTYPE html><html><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>扫码登录 ${item.header}</title>` +
          `<style>*{margin:0;padding:0}body{display:flex;justify-content:center;align-items:center;min-height:100vh;background:#0d1117}` +
          `img{border-radius:16px;border:3px solid #30363d;padding:20px;background:#fff;max-width:90vw}</style></head>` +
          `<body><img src="${item.dataUrl}" alt="扫码登录${item.header}" /></body></html>`,
          'utf8'
        )
      }
    }

    fs.writeFileSync(KEYS_FILE, JSON.stringify({ number, keys }))
    printMagenta(`\n✅ 已生成 ${number} 个二维码。`)
    printMagenta(`🔗 请查看下一步「发布二维码图片直链」输出的可点击链接，浏览器打开即可直接扫码！`)

    // 写入 Summary 提示
    appendSummary(`## 🎵 酷狗音乐扫码登录\n\n✅ 已生成 ${number} 个二维码，请查看下一步「发布二维码图片直链」输出的链接进行扫码。\n\n⏳ 二维码有效期约 2 分钟，请尽快扫描。`)
  } catch (e) {
    const msg = e && e.message ? e.message : String(e)
    console.error(`::error::二维码生成失败：${msg}`)
    appendSummary(`## ❌ 二维码生成失败\n\n错误信息：${msg}`)
    throw e
  } finally {
    close_api(api)
  }
}

/**
 * 模式二：读取已生成的二维码密钥，轮询等待用户扫码确认
 */
async function waitMode() {
  const api = startService()
  try {
    await waitForApi()
  } catch (e) {
    close_api(api)
    throw e
  }
  let parsed
  try {
    parsed = JSON.parse(fs.readFileSync(KEYS_FILE, 'utf8'))
  } catch {
    throw new Error('未找到二维码密钥文件，请确认已先运行「生成登录二维码图片」步骤')
  }
  const { number, keys } = parsed
  const USERINFO = process.env.USERINFO
  const APPEND_USER = process.env.APPEND_USER
  const userinfo = (USERINFO && APPEND_USER == "是") ? JSON.parse(USERINFO) : []

  const results = []

  try {
    for (let n = 0; n < number; n++) {
      const qrcode = keys[n]
      if (!qrcode) {
        printRed(`第 ${n + 1}/${number} 个账号的二维码密钥缺失，跳过`)
        results.push({ index: n + 1, status: '密钥缺失' })
        continue
      }
      printMagenta(`\n正在等待第 ${n + 1}/${number} 个账号扫码登录...`)
      let loggedIn = false
      let expireFlag = false
      for (let i = 0; i < 30; i++) {
        const timestrap = Date.now();
        const res = await send(`/login/qr/check?key=${qrcode}&timestrap=${timestrap}`, "GET", {})
        const status = res?.data?.status
        switch (status) {
          case 0:
            printYellow("二维码已过期，请重新运行工作流生成新二维码")
            expireFlag = true
            break
          case 1:
            // 未扫描二维码
            break
          case 2:
            // 二维码未确认，请点击确认登录
            break
          case 4:
            printGreen("登录成功！")
            upsertUser(userinfo, { userid: res.data.userid, token: res.data.token }, APPEND_USER == "是")
            loggedIn = true
            break
          default:
            printRed("请求出错")
            console.dir(summarizeResponse(res), { depth: null })
        }
        if (loggedIn || expireFlag) {
          break
        }
        if (i === 29) {
          printRed("等待超时\n")
        }
        // 前 10 次用 3 秒间隔（快速响应），之后用 5 秒间隔
        await delay(i < 10 ? 3000 : 5000)
      }
      results.push({
        index: n + 1,
        status: loggedIn ? '✅ 登录成功' : (expireFlag ? '❌ 二维码过期' : '❌ 等待超时')
      })
    }
    saveUserinfo(userinfo)

    const resultLines = results.map(r => `- 账号 ${r.index}/${number}：${r.status}`).join('\n')
    appendSummary(`### 扫码结果\n\n${resultLines}`)
  } finally {
    close_api(api)
  }
}

const mode = process.argv[2] || 'gen'
if (mode === 'wait') {
  waitMode().then(() => process.exit(0)).catch(e => { console.error(e); process.exit(1) })
} else {
  genMode().then(() => process.exit(0)).catch(e => { console.error(e); process.exit(1) })
}

# 酷狗概念版签到
【对齐上游 KuGouMusicApi v1.6.2】

GitHub Actions 实现 `酷狗概念VIP` 自动签到，每天领取总计 `两天酷狗概念VIP`

登录后即可使用，目前提供二维码登录(推荐)和手机号登录(一个手机号绑定多个账号无法登录，见 [多账号登录问题](https://github.com/MakcRe/KuGouMusicApi/issues/51))

## 免责声明

> [!important]
>
> 1. 本项目仅供学习使用，请尊重版权，请勿利用此项目从事商业行为及非法用途!
> 2. 使用本项目的过程中可能会产生版权数据。对于这些版权数据，本项目不拥有它们的所有权。为了避免侵权，使用者务必在 24小时内清除使用本项目的过程中所产生的版权数据。
> 3. 由于使用本项目产生的包括由于本协议或由于使用或无法使用本项目而引起的任何性质的任何直接、间接、特殊、偶然或结果性损害（包括但不限于因商誉损失、停工、计算机故障或故障引起的损害赔偿，或任何及所有其他商业损害或损失）由使用者负责。
> 4. **禁止在违反当地法律法规的情况下使用本项目。** 对于使用者在明知或不知当地法律法规不允许的情况下使用本项目所造成的任何违法违规行为由使用者承担，本项目不承担由此造成的任何直接、间接、特殊、偶然或结果性责任。
> 5. 音乐平台不易，请尊重版权，支持正版。
> 6. 本项目仅用于对技术可行性的探索及研究，不接受任何商业（包括但不限于广告等）合作及捐赠。
> 7. 如果官方音乐平台觉得本项目不妥，可联系本项目更改或移除。

## 使用说明

> [!warning]
> **注意事项**
> 
> 若登录后听歌领取失败，请到APP 活动中心->天天签到领VIP(这个活动新用户好像没有) 查看当日是否已经领取VIP。

> [!CAUTION]
> **常见错误**：
> 
> 执行签到报错未配置，为`PAT`秘钥权限设置问题，需正确设置`PAT Secret`权限并再次执行登录，**Repository secrets**处除了配置的**PAT Secret**外，成功写入**USERINFO**即可正常执行签到
> 
> **即 先确保权限无误`Only select Repository`仅选中所需仓库**
> 
> **权限①：`Metadata` 保持只读**
> 
> **权限②：`Secrets` 设置为读写**
> 
> **在仓库`Settings`设置`PAT Secret`，再执行登录 确保`Secrets and variables` - `Actions`中成功写入`USERINFO`再执行签到**

$${\color{red}避免将PAT秘钥复制于Windows记事本中，可能存在的字号字体问题会导致秘钥中所有下划线消失导致秘钥出错}$$

<details>

<summary>⚠️部署教程(点击展开)⚠️</summary>

1. Fork 本仓库

1. 创建添加令牌
   - **创建令牌**  
     复制下方官网链接，在浏览器中打开

     ```html copy
     https://github.com/settings/personal-access-tokens/new
     ```

   - **登录 GitHub 官网**  
     若登陆后未跳转至token生成页，请再次粘贴链接进行访问
   - **在设置页面配置权限**  
      **Token name 备注**：随意填写
      **Expiration (有效期)**：建议自定义有效期，长期无人维护时不要选择过长
      **Repository access (仓库范围)**：只选择当前 fork 的仓库
      **Repository permissions (仓库权限)**：`Metadata` 保持只读，`Secrets` 设置为读写
      ![精细化个人访问令牌权限](imgs/精细化个人访问令牌权限.png)
   - 滑动到底部，点击绿色的 Generate token 保存按钮
   - 复制生成的字符串，回到本仓库添加到[Secret](https://github.com/qfmc7040/KGM-AUTO-CHECKIN#secret-%E4%BD%8D%E7%BD%AE)，变量名 `PAT`，value 为复制的令牌

1. 登录（两种独立的登录方式，任选其一）

   3.1 二维码登录(推荐)

   运行 Actions `二维码登录`，点击 Run → 在运行摘要页面（Summary）查看二维码图片，使用酷狗音乐 APP 扫码并确认登录即可。

   3.2 手机号登录

   添加手机号到 Secret `PHONE`，运行 Actions `手机号登录`，操作步骤选择「发送验证码」获取验证码，把验证码添加到 Secret `CODE`；再次运行 Actions `手机号登录`，操作步骤选择「登录」即可。

   ⬆️ $${\color{red}与上文给仓库添加PAT秘钥同一位置}$$ ⬆️

1. 启用 Actions `签到`，每天北京时间 01:10 自动签到（可在 `签到.yml` 中设置 cron）。启用 Actions `仓库保活` 以保证签到可以长期执行。

1. （可选）配置运行结果通知

   在仓库 Settings → Secrets and variables → Actions 中添加对应渠道的 Secret，签到完成后将自动推送结果通知。支持以下渠道（全部可选，配置多个将同时发送）：

   | 通知渠道 | Secret 变量名 | 说明 |
   |---------|-------------|------|
   | 企业微信机器人 | `WECOM_BOT_KEY` | 企业微信群机器人 webhook 的 key |
   | 钉钉机器人 | `DINGTALK_BOT_KEY` | 钉钉机器人 access_token |
   | 钉钉加签 | `DINGTALK_SECRET` | 钉钉机器人加签密钥（可选） |
   | 飞书机器人 | `FEISHU_BOT_KEY` | 飞书自定义机器人 webhook 的 key |
   | 云湖机器人 | `YUNHU_BOT_KEY` | 云湖机器人 webhook 的 key |
   | Server酱 | `SERVERCHAN_SENDKEY` | Server酱 SendKey |
   | PushPlus | `PUSHPLUS_TOKEN` | PushPlus token |
   | PushPlus群组 | `PUSHPLUS_TOPIC` | PushPlus 群组编码（可选） |
   | Telegram | `TG_BOT_TOKEN` | Telegram Bot Token |
   | Telegram | `TG_CHAT_ID` | Telegram 接收消息的 Chat ID |
   | Bark (iOS) | `BARK_KEY` | Bark key 或完整 URL |
   | Bark分组 | `BARK_GROUP` | Bark 消息分组（可选） |
   | Discord | `DISCORD_WEBHOOK` | Discord Webhook 完整 URL |
   | 邮箱 SMTP | `MAIL_HOST` | SMTP 服务器地址（如 `smtp.qq.com`） |
   | 邮箱 SMTP | `MAIL_PORT` | SMTP 端口（默认 465） |
   | 邮箱 SMTP | `MAIL_USER` | 发件邮箱账号 |
   | 邮箱 SMTP | `MAIL_PASS` | 发件邮箱授权码（非登录密码） |
   | 邮箱 SMTP | `MAIL_TO` | 收件邮箱地址 |

   通知内容包含：运行日期、账号数量、成功/失败统计、各账号听歌领取状态、VIP 领取次数、VIP 到期时间、错误信息等。

API源代码来自 [MakcRe/KuGouMusicApi](https://github.com/MakcRe/KuGouMusicApi) ~~图省事直接搬来~~

## 令牌（Token）机制说明

项目中包含两类令牌：

1. **GitHub Personal Access Token (PAT)**：用于自动将酷狗登录信息写入仓库 Secret `USERINFO`，以及每周日自动刷新酷狗登录 Token。

2. **酷狗登录 Token**：存储在 `USERINFO` Secret 中，用于酷狗 API 身份认证。通过登录获取，每周日自动刷新。

## Secret 位置

  ### 步骤一
   
   ![步骤一](./imgs/步骤一.jpg)
  ### 步骤二
   
   ![步骤二](./imgs/步骤二.jpg)
  ### 步骤三
   
   ![步骤三](./imgs/步骤三.jpg)
  ### 步骤四
   
   ![步骤四](./imgs/步骤四.jpg)

</details>

## 致谢

- 感谢 [@MakcRe](https://github.com/MakcRe) 提供 API 源代码
- 感谢 [@itfw](https://github.com/itfw) 提供二维码显示问题的解决方案
- 感谢 [@klaas8](https://github.com/klaas8) 提供自动写入secret的方法
- 感谢 [@develop202](https://github.com/develop202/kgcheckin) 原项目

import { close_api, delay, send, startService, waitForApi } from "./utils/utils.js";
import { summarizeResponse } from "./utils/safeLog.js";

async function login() {

  const phone = process.env.PHONE

  if (!phone) {
    throw new Error("参数错误！请检查")
  }
  // 启动服务并等待就绪（避免冷启动竞态）
  const api = startService()
  try {
    await waitForApi()
  } catch (e) {
    close_api(api)
    throw e
  }

  console.log("开始发送验证码")
  try {
    const result = await send(`/captcha/sent?mobile=${phone}`, "GET", {})
    if (result.status === 1) {
      console.log("发送成功")
    } else {
      console.log("响应内容")
      console.dir(summarizeResponse(result), { depth: null })
      throw new Error("发送失败！请检查")
    }
  } finally {
    close_api(api)
  }
}

login().then(() => process.exit(0)).catch(e => { console.error(e); process.exit(1) })
