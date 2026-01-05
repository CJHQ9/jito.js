const axios = require("axios");

/**
 * Jito Bundle API 封装
 * 支持高并发请求和自定义请求头
 */
class JitoAPI {
  constructor(options = {}) {
    this.baseURL =
      options.baseURL || "https://mainnet.block-engine.jito.wtf:443";
    this.timeout = options.timeout || 5000;
    this.defaultHeaders = {
      "Content-Type": "application/json",
      "User-Agent": "Jito-Bundle-Client/1.0",
      ...options.headers,
    };
  }

  /**
   * 发送单个Bundle请求
   * @param {Array} transactions - Base64编码的交易数组
   * @param {Object} options - 请求选项
   * @param {Object} options.headers - 自定义请求头
   * @param {string} options.encoding - 编码格式，默认'base64'
   * @returns {Promise} API响应
   */
  async sendBundle(transactions, options = {}) {
    const { headers = {}, encoding = "base64" } = options;

    const payload = {
      jsonrpc: "2.0",
      id: 1, // 确保每个请求有唯一ID
      method: "sendBundle",
      params: [
        transactions,
        {
          encoding: "base64",
        },
      ], // 直接传递交易数组，不需要encoding参数
    };

    const requestHeaders = {
      ...this.defaultHeaders,
      ...headers,
    };

    try {
      const response = await axios.post(
        `${this.baseURL}/api/v1/bundles`,
        payload,
        {
          headers: requestHeaders,
          timeout: this.timeout,
        }
      );
      console.log("✅ Bundle请求成功:", JSON.stringify(response.data, null, 2));
      return response.data;
    } catch (error) {
      console.error("❌ Bundle请求失败:");
      console.error("错误类型:", error.name);
      console.error("错误消息:", error.message);

      if (error.response) {
        // 服务器返回了错误响应
        console.error("响应状态码:", error.response.status);
        console.error("响应状态文本:", error.response.statusText);
        console.error(
          "响应头:",
          JSON.stringify(error.response.headers, null, 2)
        );
        console.error(
          "响应数据:",
          JSON.stringify(error.response.data, null, 2)
        );
      } else if (error.request) {
        // 请求已发出但没有收到响应
        console.error(
          "请求配置:",
          JSON.stringify(
            {
              url: error.config?.url,
              method: error.config?.method,
              headers: error.config?.headers,
              timeout: error.config?.timeout,
            },
            null,
            2
          )
        );
        console.error("网络错误或超时，未收到响应");
      } else {
        // 请求配置出错
        console.error("请求配置错误:", error.message);
      }

      throw new Error(`Jito Bundle API Error: ${error.message}`);
    }
  }

  /**
   * 发送单个交易（sendTransaction，启用 bundleOnly=true）
   * 说明: 根据官方文档，此端点会代理 Solana 的 sendTransaction，并强制 skip_preflight=true。
   *       通过 query 参数 bundleOnly=true 可开启 revert protection，以 Bundle 形式提交。
   * @param {string} transaction - Base64编码的交易（推荐）
   * @param {Object} options - 请求选项
   * @param {Object} options.headers - 自定义请求头
   * @param {string} options.encoding - 编码格式，默认'base64'
   * @returns {Promise} API响应
   */
  async sendTransactionBundleOnly(transaction, options = {}) {
    const { headers = {}, encoding = "base64" } = options;

    const payload = {
      jsonrpc: "2.0",
      id: Date.now(),
      method: "sendTransaction",
      params: [
        transaction,
        {
          encoding,
        },
      ],
    };

    const requestHeaders = {
      ...this.defaultHeaders,
      ...headers,
    };

    try {
      const url = `${this.baseURL}/api/v1/transactions?bundleOnly=true`;
      const response = await axios.post(url, payload, {
        headers: requestHeaders,
        timeout: this.timeout,
      });
      console.log("✅ sendTransaction(bundleOnly) 请求成功:", JSON.stringify(response.data, null, 2));
      return response.data;
    } catch (error) {
      console.error("❌ sendTransaction(bundleOnly) 请求失败:");
      console.error("错误类型:", error.name);
      console.error("错误消息:", error.message);

      if (error.response) {
        console.error("响应状态码:", error.response.status);
        console.error("响应状态文本:", error.response.statusText);
        console.error("响应头:", JSON.stringify(error.response.headers, null, 2));
        console.error("响应数据:", JSON.stringify(error.response.data, null, 2));
      } else if (error.request) {
        console.error(
          "请求配置:",
          JSON.stringify(
            {
              url: error.config?.url,
              method: error.config?.method,
              headers: error.config?.headers,
              timeout: error.config?.timeout,
            },
            null,
            2
          )
        );
        console.error("网络错误或超时，未收到响应");
      } else {
        console.error("请求配置错误:", error.message);
      }

      throw new Error(`Jito sendTransaction(bundleOnly) API Error: ${error.message}`);
    }
  }

// 定时策略：每 5 秒轮询 Jup 热榜，选择第一个标的，用 Ultra 下单并买入 0.1 SOL 
// 维护 position.json：避免重复开仓，记录开仓信息供后续风控（risk.js）使用 

常量 fs = require（"fs"）； 
const path = require（"path"）； 
const cron = require（"node-cron"）； 
const base58 = require（"bs58"）.default； 
const dayjs = require（"dayjs"）； 
const { Keypair， Connection， PublicKey } = require（"@solana/web3.js"）； 

// 依赖的模块 
const { getJupRankList } = require（"../适配器/jupter"）； 
常量sendToBot = require（"../tg/tg"）； 
const { swapUltra, getUltraOrder, getUsdPrice } = require("./jup"); 

// 常量配置 
常量 WSOL_MINT = "So111111111111111111111111111111112"； 
常量购买金额=0.01；//购买0.08索尔 
常数购买量索尔原始=数学地板（购买量索尔*1e9）；//工作索尔9位精度 
// 冷却时间：避免短时间内重复购买（默认：买入后 30 分钟、卖出后 15 分钟） 
常量 REENTRY_COOLDOWN_BUY_MS = Number（process.env.STRATEGY_COOLDOWN-BUY_MS || 20 * 60 * 1000）； 
常量 REENTRY_COOLDOWN_SELL_MS = Number（process.env.STRATEGY_COOLDOWN__SELL_MS || 60 * 60 * 1000）； 
// 启动冷静期：进程启动后 6 分钟内，市值超过 30w 的不买；6 分钟后不限制 
常量 START_TS = Date.now（）； 
cont STARTUP_GRACE_MS = 数字（process.env.STRAT_STARTUP_GRACE_MS || 6 * 60 * 1000）; 
常量 STARTUP_MAX_MCAP = Number（process.env.STRAT_STARTUP_MAX/MCAP || 300000）； 

// 文件路径 
常量 POSITION_FILE = path.resolve（__dirname，"./position.json"）； 
常量 POSITION_HISTORY_FILE = path.resolve（ 
__dirname， 
"./职位历史.json" 
); 

// 轻量 .env 加载，确保 PRIVATE_KEY_BASE58 可读 
函数 loadEnvFromFile（） { 
常量 envPath = path.resolve（__dirname，"../../。env"）； 
尝试 { 
常量 raw = fs.readFileSync（envPath， "utf8"）； 
raw.split/\r？\n/.forEachline={ 
常量 s = line.trim（）； 
如果 （！s || s.startsWith（“#”））返回; 
常量 i = s.indexOf（"="）； 
如果（i ===-1）返回； 
常量 key = s.slice（0， i）.trim（）； 
let val = s.slice（i+1）.trim（）； 
如果（ 
（val.startsWith（'''） & val.endsWith（'“'））|| 
（val.起始（“'”）和 val.endsWith（“'”）） 
     ) { 
val=val.slice（1，-1）； 
     } 
如果 （！process.env[key]） process.env [key] = val； 
   }); 
} catch （_） { 
   // 无 .env 时忽略 
 } 
} 

loadEnvFromFile(); 

// RPC 连接 & mint decimals 缓存 
让连接； 
const mintDecimalsCache = 新地图（）; 

函数 getRpcUrl（） { 
返回（ 
process.env.SOLANA_RPC_URL || 
process.env.HELIUS_RPC_URL || 
process.env.JITO_RPC_URL || 
」。https://api.mainnet-beta.solana.com」。 
 ); 
} 

函数 getConnection（） { 
如果（！连接） { 
连接 = new Connection（getRpcUrl（）， "已确认"）； 
 } 
返回连接； 
} 

async函数 getMintDecimal（mint） { 
如果（！mint）返回空值； 
如果 （mintDecimalsCache.has（mint）） 返回 mintDecimalsCache.get（mint）； 
尝试 { 
const conn = getConnection（）； 
常量 info = await conn.getParsedAccountInfo（new PublicKey（mint））； 
常量小数 = info？.value？.data？.parsed？.info？.小数； 
如果（typeof 小数点 === "number"） { 
mintDecimalsCache.set（mint，小数）; 
返回小数点； 
   } 
} catch （e） { 
   console.warn("【策略】获取薄荷小数失败：“, 电子邮件||e） 
 } 
返回空值； 
} 

// 根据链上交易签名，计算指定钱包在指定 mint 上的实际收到数量 
// 通过对比 preTokenBalances 与 postTokenBalances（以 owner+mint 过滤）求差值 
async 函数 getReceivedAmountFromTx（ 
签名， 
所有者Pubkey， 
薄荷， 
{ 重复尝试 =12， 延迟Ms = 1500 } = {} 
) { 
尝试 { 
const conn = getConnection（）； 
常量 ownerStr = 
typeof ownerPubkey === "字符串"？ownerPubkey:ownerPubkey.toBase58（）； 
let tx = null； 
对于（ let i = 0； i < retries； i++） { 
尝试 { 
tx = await conn.getTransaction（签名， { 
maxSupportedTransactionVersion:0， 
       }); 
} 捕捉 （_） {} 
如果 （tx && tx.meta） 中断； 
等待新的Promise（（r） => setTimeout（r， delayMs））； 
   } 
如果（！tx ||！tx.meta）返回{ raw: null， ui: null； 小数: null}； 

const preTB = Array.isArray（tx.meta.preTokenBalances） 
?tx.meta.preTokenBalances 
     : []; 
常量 postTB = Array.isArray（tx.meta.postTokenBalances） 
?tx.meta.postTokenBalances 
     : []; 

const normalizeAmount = （b） => { 
常量 amtStr = 
b & b.uiTokenAmount && typeof b.uiTokenAmount.amount === “string” 
？ b.uiTokenAmount.amount 
: typeof b.amount === "字符串" 
？ 金额 
:null； 
尝试 { 
返回 amtStr？ BigInt（amtStr）:0n； 
} catch （_） { 
返回 0n； 
     } 
   }; 
常量 byOwnerMint = （b） => { 
如果（！b）返回false； 
常量 ownerField = b.owner || b.uiTokenOwner || null； 
const ownerMatch = ownerField？ownerField === ownerStr:false； 
const mintMatch = mint？b.mint === mint: true； 
返回所有者匹配 && mintMatch； 
   }; 

const preSum = preTB 
.filter（byOwnerMint） 
.reduce（（acc， b） => acc + normalizeAmount（b）， 0n）； 
常量 postSum = postTB 
.filter（byOwnerMint） 
.reduce（（acc， b） => acc + normalizeAmount（b）， 0n）； 
常量 rawDiff = postSum - preSum； 

常量 decCandidate = 
postTB.find（byOwnerMint）？.uiTokenAmount？.小数点？？ 
preTB.find（byOwnerMint）？.uiTokenAmount？.小数？？ 
null； 
常量ui = 
decCandidate！= null 
？Number（rawDiff）/Math.pow（10，decCandidate） 
:null； 

返回 { raw:rawDiff.toString（）， ui， 小数点:decCandidate}； 
} catch （e） { 
console.warn 
     "[strategy] 解析链上实际到帐失败：", 
e && e.message？e.message:e 
   ); 
返回{raw:null，ui:null，小数:null}； 
 } 
} 

// 从 Jupiter Ultra Execute 回执中提取实际输出原始数量（字符串） 
// 优先 totalOutputAmount → outputAmountResult → swapEvents 最后一个匹配 outputMint 的 outputAmount 
函数 getOutputAmountFromExecute（execute， expectedMint） { 
尝试 { 
如果（！execute）返回空值； 
如果（ 
typeof execute.totalOutputAmount === "字符串" && 
执行.总输出金额 
   ) { 
返回execute.totalOutputAmount； 
   } 
如果（ 
typeof execute.outputAmountResult === "字符串" && 
执行.输出金额结果 
   ) { 
返回execute.outputAmountResult； 
   } 
const events = Array.isArray（execute.swapEvents）？execute.swapEvents:[]； 
如果 （events.length） { 
对于（ let i = events.length - 1； i >= 0； i--） { 
常量 ev = events[i]； 
如果（！ev）继续； 
如果（ 
预期Mint && 
ev.outputMint === 预期Mint && 
typeof ev.outputAmount === "字符串" 
       ) { 
返回 ev.outputAmount； 
       } 
     } 
     // 若未匹配到指定 mint，回退到最后一个事件的 outputAmount 
常量 lastEv = events[events.length - 1]； 
如果（lastEv && typeof lastEv.outputAmount === "字符串"） 
返回 lastEv.outputAmount； 
   } 
} 捕捉 （_） {} 
返回空值； 
} 

// 读/写仓位文件 
函数 ensurePositionFile（） { 
尝试 { 
如果 （！fs.existsSync（位置文件）） { 
fs.writeFileSync（POSITION_FILE， "[]"， "utf8"）； 
   } 
} catch （e） { 
   console.error("【策略】初始化位置.json失败：“, 电子邮件||e） 
 } 
} 

函数 ensurePositionHistoryFile（） { 
尝试 { 
如果 （！fs.existsSync（职位历史文件）） { 
fs.writeFileSync（职位历史文件， "[]"， "utf8"）； 
   } 
} catch （e） { 
console.error 
     "【策略】初始化位置_history.json失败：“, 
e.message || e 
   ); 
 } 
} 

函数 readPositions（） { 
确保位置文件（）； 
尝试 { 
常量 txt = fs.readFileSync（POSITION_FILE， "utf8"）； 
   const arr = JSON.parse(txt || "[]"); 
返回Array.isArray（arr）？arr:[]； 
} catch （e） { 
   console.error("【策略】读取position.json失败：“, 电子邮件||e） 
返回[]； 
 } 
} 

函数 readPositionHistory（） { 
确保位置历史文件（）； 
尝试 { 
常量 txt = fs.readFileSync（POSITION_HISTORY_FILE， "utf8"）； 
   const arr = JSON.parse(txt || "[]"); 
返回Array.isArray（arr）？arr:[]； 
} catch （e） { 
console.error 
     "【策略】读取position_history.json失败：“, 
e.message || e 
   ); 
返回[]； 
 } 
} 

函数 writePositions（positions） { 
尝试 { 
const txt = JSON.stringify（positions， null， 2）； 
fs.writeFileSync（POSITION_FILE，txt，“utf8”）； 
} catch （e） { 
   console.error("【策略】写入position.json失败：“, 电子邮件||e） 
 } 
} 

函数 hasOpenPosition（positions，tokenMint） { 
返回位置。某些（ 
（p） => p && p.tokenMint === tokenMint && p。status === "open" 
 ); 
} 

// 找到该代币在历史中的最近一条记录（按 closedAt 优先，其次 createdAt） 
函数 findLatestHistory（history，tokenMint） { 
常量列表 = history.filter（（h） => h && h.tokenMint === tokenMint）； 
如果（！list.length）返回空值； 
常量排序键=（h）={ 
cont t = h.closedAt ||h.createdAt ||无效; 
返回t？new Date（t）.getTime（）:0； 
 }; 
返回列表.sort（（a， b） => sortKey（b） - sortKey（a））[0] || null; 
} 

// 判断是否处于冷却期： 
// - 若最近记录为 open/未关闭，使用买入冷却； 
// - 若最近记录已关闭，使用卖出冷却； 
函数 checkCooldownForToken（history，tokenMint） { 
const latest = findLatestHistory（history，tokenMint）； 
如果（！latest）返回 { inCooldown: false}； 
const now = Date.now（）； 
const ts = 最新.closedAt？new Date（latest.closedAt）.getTime（） ： （latest.createdAt ？ new Date（latest.createdAt）.getTime（） ： null）; 
如果（ts == null）返回 { inCooldown:false}； 
常量 isClosed =！！latest.closedAt； 
常量窗口Ms = isClosed？ REENTRY_COOLDOWN_SELL_MS: REENTRY·COOLDOWN_BUY_MS； 
const elapsed = now - ts； 
如果（经过<窗口Ms）{ 
返回 { 
inCooldown: 真， 
原因：是否关闭？'卖出后冷却${数学.轮（再入_冷却_卖出_MS/60000）}分钟`：`买入后冷却${数学.轮（再进_冷却_买入_MS/60000）}分钟'， 
直到：new Date（ts + windowMs）.toISOString（）， 
剩余Ms:窗口Ms - 已过， 
   }; 
 } 
返回 { inCooldown:false }； 
} 

// 内存锁，避免并发重复下单 
let isBuying = false； 

async 函数 runCycle（） { 
如果（正在购买） { 
   return; // 避免并发 
 } 
isBuying = true； 
尝试 { 
const list=await getJupRankList（"1h"）； 
如果 （！Array.isArray（list） || list.length === 0） { 
     // console.log("[strategy] 热榜为空，等待下一轮"); 
返回； 
   } 
const first = list[0]; 
   //打印当前时间 
   console.log(`当前时间: ${dayjs().format("YYYY-MM-DD HH:mm:ss")}`); 
const topPrice = typeof first.usd价格 === 'number'？first.usd价格：null; 
常量 priceStr = topPrice！= null？ `$${Number（topPrice）.toFixed（6）} : 'N/A'； 
控制台.log 
【策略】热榜第一：${第一.符号||第一.合同地址}（${ 
first.contractAddress || first.id 
     }) 价格：${priceStr}` 
   ); 
const tokenMint = first.contractAddress； 
常量 tokenSymbol = first.symbol || "未知"； 
   // 启动冷静期市值限制（避免在高点追高触发止损） 
const elapsedMs = Date.now（） - START_TS； 
常量 marketCap = typeof first.marketCap === 'number'？ first.marketcap: null； 
如果 （elapsedMs < STARTUP_GRACE_MS && marketCap！= null && marketCAP > STARTUP_MAX_MCAP） { 
const minsLeft = （（STARTUP_GRACE_MS - elapsedMs） / 60000）。toFixed（1）； 
     console.log(`[strategy] 启动冷静期：市值>${STARTUP_MAX_MCAP}，跳过 ${tokenSymbol} (${tokenMint})，剩余约 ${minsLeft} 分钟`); 
返回； 
   } 

const positions = readPositions（）； 
常量 history = readPositionHistory（）； 
   // 有持仓直接跳过；否则检查历史冷却窗口，避免短时间内重复购买 
如果 （hasOpenPosition（positions，tokenMint）） { 
控件.日志（'【策略】已持有${标记符号}（${标记分钟}），跳过购买`）； 
返回； 
   } 
常量 cd = checkCooldownForToken（history，tokenMint）； 
如果 （cd.inCooldown） { 
const mins = （cd.remainingMs / 60000）.toFixed（1）； 
     console.log(`[strategy] 冷却中，跳过购买：${tokenSymbol} (${tokenMint}) 剩余约 ${mins} 分钟（${cd.reason}，至 ${cd.until}）`); 
返回； 
   } 

常量 sk58 = process.env.PRIVATE_KEY_BASE58 || ""； 
如果 （！sk58） { 
     console.error("【策略】缺少私人_密钥_基础58，无法执行购买“); 
const orderOnly = await getUltraOrder（{ 
输入Mint: WSOL_MINT， 
输出Mint:令牌Mint， 
金额:字符串（购买金额_SOL_RAW）， 
     }); 
     console.log("【超】获得订单响应：“, { 
请求ID:orderOnly.requestId， 
hasTransaction:！！orderOnly.transaction， 
模式:orderOnly.mode， 
路由器:orderOnly.router， 
       swapMode: orderOnly.swapMode, 
inputMint:orderOnly.inputMint， 
outputMint:orderOnly.outputMint， 
金额:仅限订单。金额， 
outAmount:orderOnly.outAmount， 
inUsdValue： orderOnly.inUsdValue， 
out UsdValue:orderOnly.out UsdValue， 
priceImpactPct： 仅订购.价格ImpactPct， 
feeBps:orderOnly.feeBps， 
平台费用:仅限订单。平台费用， 
slippageBps： 仅order.slippageBps， 
otherAmountThreshold:orderOnly.otherAmountThreshold， 
     }); 
如果 （！orderOnly.transaction） { 
console.warn 
         "[提示] 未提供 taker（钱包地址）时，Jupiter 会返回 transaction=null。要执行交易请提供 PRIVATE_KEY_BASE58。" 
       ); 
     } 

返回； 
   } 
const wallet = Keypair.fromSecretKey（base58.decode（sk58））； 

控制日志（'【策略】准备购买0.05索尔→${标记符号}（${标记分钟}）'）； 
const result = await swapUltra（{ 
输入Mint: WSOL_MINT， 
输出Mint:令牌Mint， 
金额:字符串（购买金额_SOL_RAW）， 
钱包， 
接受者:钱包.publicKey， 
   }); 

常量{order，execute} = result || {}； 
常量 status = execute && execute.status； 
如果（status！== "Success"） { 
console.error 
       "[strategy] 购买执行失败：", 
JSON.stringify（execute || {}， null， 2） 
     ); 
返回； 
   } 

常量签名= 
execute.signature || execute.txid || execute。transactionId || null； 
   // 计算实际入场价格（USD/代币） 
   // 优先使用执行回执中的实际输出；失败时回退到链上解析；最后才使用报价值 
常量 execOutRaw = getOutputAmountFromExecute（execute，tokenMint）； 
   // 为避免不必要的 RPC，仅当执行回执缺失输出时才解析链上交易 
const chainReceived = 
！execOutRaw && 签名 
？ 等待getReceivedAmountFromTx（签名，wallet.publicKey，tokenMint） 
{ raw: null， ui: null； 小数: null}； 
常量 outAmountRawStr = 
execOutRaw || 
chainReceived.raw || 
（order && order.outAmount？ String（order.outAmount）: null）； 
常量 outAmountRawNum = outAmountRawStr？ Number（outAmountRawStr）: null； 
   // 惰性获取 decimals：优先热榜，其次链上解析；仅在需要归一化且缺失时才请求 RPC 
let 小数点Used = 
typeof first.小数值 === "number"？ first.小数量值:null； 
如果（小数点Used == null && chainReceived.小数点！= null） { 
使用的小数点 = 链接收到的小数点； 
   } 
如果（ 
使用的小数点 == null && 
outAmountRawNum！= null && 
chainReceived.ui == null 
   ) { 
小数点Used = await getMintDecimal（tokenMint）； 
   } 
常量收到金额 = 
chainReceived.ui！= null 
？ chainReceived.ui 
: 小数点Used！= null && outAmountRawNum！= 空 
？ outAmountRawNum / Math.pow（10，小数点所用） 
:null； 
   // 入场 USD 总额：使用执行回执的实际输入 SOL × SOL USD 价格，避免使用波动较大的输出代币价格 
让 entryUsdTotal = null； 
const inRawStr = execute && typeof execute.totalInputAmount === 'string' ？execute.totalInputAmount ： null; 
常量 inRawNum = inRawStr？ Number（inRawStr）: null； 
如果（inRawNum！= null） { 
let wsolUsd = null； 
尝试 { 
常量 pIn = await getUsdPrice（WSOL_MINT）； 
wsolUsd = pIn && typeof pIn.usdPrice === 'number'？ pIn.usdPrice: null； 
} 捕捉 （_） {} 
如果 （wsolUsd！= null） { 
常量 inSol = inRawNum/1e9； 
entryUsdTotal = inSol * wsolUsd； 
     } 
   } 
如果（entryUsdTotal == null） { 
     // 回退到询价阶段的 USD 总额（优先 inUsdValue，其次 outUsdValue） 
entryUsdTotal = （order && typeof order.inUsdValue！== 'undefined'） 
？ 数量（order.inUsdValue） 
： （（（order && typeof order.outUsdValue ！== 'undefined'） ？Number（order.outUsdValue） ： null）; 
   } 
const computedEntryUsdPrice= 
收到金额 && 入账总金额 
？ 输入美元总额 / 收到金额 
:null； 

常量 entry = { 
tokenMint， 
标记符号， 
状态:"打开"， 
创建时间:new Date（）.toISOString（）， 
     // 成本与数量（原始值） 
项目SOL金额:购买金额SOL， 
条目SolAmountRaw:购买_金额_SOL_RAW， 
收到金额Raw:出金额RawStr， 
     receivedAmount, // 归一化后的收到数量（基于 mint decimals） 
     // 价格记录（来源于热榜/订单汇总） 
     entryUsdPrice: computedEntryUsdPrice, // 实际入场价格（USD/代币），若无法计算则回退热榜价格 
     entryUsdTotal, // Ultra 订单的 USD 总额（更接近实际成交） 
     // Jupiter 元信息 
请求ID:order && order.requestId， 
jupiterMode:order && order.mode， 
jupiterRouter:order && order.router， 
     // 交易签名 
签名， 
接受者:钱包.publicKey.toBase58（）， 
   }; 

positions.push（entry）； 
写入位置（位置）； 
常量消息 = [ 
     "开仓成功：", 
     `代币：${代币符号}（${代币明数}）'， 
     `交易签名：${signature || "N/A"}`, 
     `买入金额：${买入金额_结算日}结算日'， 
     `收到数量：${收到数量！=空？收到金额：“N/A”}`， 
     `入场价格(USD/代币)：${ 
计算入境美元价格！=null？计算入境美元价格:"无" 
     }`, 
     `GMGN-K线: https://gmgn.ai/sol/token/${tokenMint}`, 
].join（"\n"); 

   console.log("[strategy] 开仓成功，已记录仓位：\n" + message); 
尝试 { 
等待发送ToBot（{ text:{ content:message } }）； 
} catch （e） { 
console.error 
       "[strategy] 发送到 Telegram 失败：", 
e && e.message？e.message:e 
     ); 
   } 
} catch （e） { 
console.error 
     "[strategy] 轮询或购买出现错误：", 
e && e.stack？ e.Stack: e 
   ); 
}最后 { 
isBuying = false； 
 } 
} 

函数 startScheduler（） { 
 // 每 3 秒执行一次 
cron.schedule（"*/2 * * * *"， runCycle， { scheduled: true }）； 
 console.log("[strategy] 已启动 2 秒定时任务"); 
} 

// 允许作为独立脚本运行 
如果（require.main === 模块） { 
启动调度器（）； 
} 

模块.exports = { 
启动调度器， 
 runCycle, 
}; 
