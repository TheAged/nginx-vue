# Vue 前端與 Nginx 部署教學

本教學將指導你如何從零開始建立一個 Vue 前端專案，並將打包後的靜態檔案利用 Nginx 反向代理部署，同時將 API 請求代理到後端 Flask 服務。

---

## 目錄

- [前置需求](#前置需求)
- [建立 Vue 專案](#建立-vue-專案)
- [修改前端 API 配置](#修改前端-api-配置)
- [打包 Vue 專案](#打包-vue-專案)
- [Nginx 部署配置](#nginx-部署配置)
- [測試與除錯](#測試與除錯)
- [其他注意事項](#其他注意事項)

---

## 前置需求

請確認你已安裝以下工具與環境：
- **Node.js 與 npm/yarn**  
  建議使用 Node.js LTS 版本。
- **Vue CLI**  
  使用命令：
  ```bash
  npm install -g @vue/cli
Nginx
已安裝並設定 Nginx 作為反向代理與靜態檔案服務器。

後端 API 服務
例如 Flask 部署的 API（本專案示例中部署在 127.0.0.1:5000）。

建立 Vue 專案
使用 Vue CLI 建立專案：

bash
複製
vue create fall-detection-frontend
按照需要選擇預設或自定義配置。完成後進入專案目錄：

bash
複製
cd fall-detection-frontend
修改 src/App.vue 內容，範例如下：

html
複製
<template>
  <div id="app">
    <h1>跌倒偵測監控</h1>
    <!-- 影像串流來源依據反向代理配置，這裡使用相對路徑 -->
    <img :src="videoUrl" alt="影像串流" width="640" height="480" />
    <h2>跌倒狀態：{{ fallStatus }}</h2>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      videoUrl: "/api/video_feed",  // 使用代理時，這裡填入相對路徑
      fallStatus: "Loading..."
    };
  },
  mounted() {
    this.fetchFallStatus();
    setInterval(this.fetchFallStatus, 1000);
  },
  methods: {
    async fetchFallStatus() {
      try {
        const response = await fetch("/api/fall_status");
        const data = await response.json();
        this.fallStatus = data.status;
      } catch (error) {
        console.error("Error fetching fall status:", error);
        this.fallStatus = "Error";
      }
    }
  }
};
</script>

<style scoped>
/* 可根據需求新增樣式 */
</style>
修改前端 API 配置
若前後端不在同一個域名下，可在專案根目錄建立或修改 vue.config.js，設定代理：

js
複製
module.exports = {
  devServer: {
    proxy: {
      "/api": {
        target: "http://127.0.0.1:5000",
        changeOrigin: true
      }
    }
  }
};
這樣在開發期間，所有對 /api 路徑的請求會自動代理到 http://127.0.0.1:5000。

打包 Vue 專案
當前端專案測試通過後，使用以下命令打包靜態資源：

bash
複製
npm run build
打包完成後，產生的靜態檔案位於 dist/ 目錄中。

Nginx 部署配置
部署靜態檔案：
將 dist/ 目錄中的檔案上傳至 Nginx 伺服器指定的目錄，例如 /var/www/fall_frontend。

撰寫 Nginx 配置文件：
以下為一份範例配置，將靜態檔案與 API 請求進行整合：

nginx
複製
server {
    listen 80;
    server_name yourdomain.com;  # 替換為你的域名或伺服器 IP

    # 靜態檔案根目錄
    root /var/www/fall_frontend;
    index index.html;

    # 所有 URL 都嘗試先找到對應檔案，找不到則返回 index.html (適用於 SPA)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 反向代理 API 請求
    location /api/ {
        proxy_pass http://127.0.0.1:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
設置完成後，重新啟動 Nginx：

bash
複製
sudo systemctl restart nginx
（選擇性）SSL 與公開域名：
如果需要 HTTPS，請參考 Let's Encrypt 和 Certbot 的相關文件，配置 SSL 憑證後更新 Nginx 配置。

測試與除錯
使用瀏覽器存取 http://yourdomain.com 或直接輸入伺服器 IP，確認 Vue 前端頁面是否正確載入。

前端頁面中的影像串流應顯示從 /api/video_feed 獲得的即時影像，同時跌倒狀態會定時更新（從 /api/fall_status 獲取）。

若發現 API 請求錯誤，請檢查 Nginx 日誌與後端服務日誌排查問題。

其他注意事項
跨域問題：
使用反向代理後，所有請求均通過同一域名，有效解決跨域問題。開發環境中可使用 Vue 的代理功能進行調試。

安全性：
除了基本部署外，建議在生產環境中配置防火牆、SSL/TLS 與必要的安全驗證機制。

環境區分：
開發過程中請根據需求調整 API URL。生產環境中建議使用反向代理整合前後端資源，並確保域名解析正確。

以上步驟說明了如何從建立 Vue 專案、修改 API 配置、打包靜態檔案到最終利用 Nginx 反向代理部署，助你順利完成前端與後端整合部署。

go
複製

你可以直接將上面的內容複製並儲存為 `README.md` 文件，並根據你自己的專案需求進行進一步調整
