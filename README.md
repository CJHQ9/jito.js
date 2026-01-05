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
