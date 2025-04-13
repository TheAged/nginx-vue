# 跌倒偵測監控平台 - 部署指南

本文件說明如何部署一個整合前端與後端的系統，此系統包含：

1.  **前端**：一個基於 Vue.js 的單頁應用（SPA），用於顯示即時影像串流和偵測狀態。
2.  **反向代理**：使用 Nginx 提供前端靜態資源，並將 API 請求代理至後端服務。
3.  **後端**：一個基於 Flask 的 API 服務，負責處理影像（例如來自攝影裝置的數據進行跌倒偵測），並提供影像流和狀態數據。

---

## 目標架構

* **前端 (Vue.js)**：
    * 顯示 `/api/video_feed` 的 MJPEG 影像流。
    * 定期輪詢 `/api/fall_status` 以更新偵測狀態。
    * 使用 Vue CLI 建立與管理。
* **反向代理 (Nginx)**：
    * 託管 Vue 應用打包後的靜態檔案（HTML, CSS, JS）。
    * 將所有 `/api/` 路徑的請求轉發給運作中的 Flask 後端服務。
    * 確保前後端可透過同一個網域（或 IP）與端口進行通訊，避免跨域問題。
* **後端 (Flask API)**：
    * 運行在例如 `100.0.0.1:5000`。
    * 提供 API 端點：
        * `/api/video_feed`：輸出 MJPEG 格式的即時影像流。
        * `/api/fall_status`：返回 JSON 格式的目前狀態（例如 `{"status": "No Fall Detected"}` 或 `{"status": "Fall Detected!"}`）。
    * 需處理影像輸入及執行偵測邏輯。

---

## 系統需求與準備工作

### 環境要求

1.  **Node.js & npm/yarn**:
    * 安裝 Node.js (建議使用最新的 LTS 版本)。npm 或 yarn 會隨之安裝。
    * 用於運行 Vue CLI 和管理前端專案依賴。
2.  **Vue CLI**:
    * 需全域安裝。打開終端機或命令提示字元執行：
      ```bash
      npm install -g @vue/cli
      # 或 yarn global add @vue/cli
      ```
3.  **Nginx**:
    * 根據您的作業系統安裝 Nginx。
    * 參考官方文件：[Nginx Installation](https://nginx.org/en/docs/install.html)
4.  **Python & Flask**:
    * 需要 Python 3 環境。
    * 安裝 Flask 及其他後端所需的 Python 模組，例如：
        * `Flask`
        * `opencv-python` (用於影像處理)
        * `mediapipe` (若使用 MediaPipe 進行姿態估計)
        * `ultralytics` (若使用 YOLO 進行物件偵測)
        * `numpy`
        * 建議使用 `pip install -r requirements.txt` (如果專案提供此檔案)。

### 範例網絡設定

* 假設 Flask API 運行於本機的 5000 埠 (`100.0.0.1:5000`)。
* Nginx 監聽標準的 80 埠 (HTTP)，並作為唯一的對外入口。

---

## 部署流程

### 1. 後端 Flask API 部署

1.  **準備程式碼**：確保您的 Flask 應用程式已完成，包含 `/api/video_feed` 和 `/api/fall_status` 路由。
2.  **安裝依賴**：在後端專案目錄中，安裝所有必要的 Python 套件。
3.  **啟動服務**：
    * 為了讓 Nginx 能夠代理請求，Flask 應用需要監聽 `0.0.0.0` 而非僅 `100.0.0.1`，這樣它才能接收來自容器外部或 Nginx 代理的請求。
    * 使用類似以下的命令啟動 Flask (建議使用 Gunicorn 或 uWSGI 在生產環境中運行，但此處以開發伺服器為例)：
      ```python
      # 在您的 app.py 或主檔案中
      if __name__ == '__main__':
          # threaded=True 允許處理多個請求，對影像流很重要
          # debug=False 建議用於生產環境
          app.run(host='0.0.0.0', port=5000, debug=False, threaded=True)
      ```
    * 確保後端服務在伺服器上持續運行（可使用 `systemd`, `supervisor`, 或 `docker` 等工具）。

### 2. 前端 Vue 應用部署

1.  **建立專案** (若尚未建立):
    ```bash
    vue create frontend
    cd frontend
    ```
2.  **開發與測試**:
    * 在 `src/App.vue` 或其他組件中，撰寫與後端 API 互動的程式碼。影像來源設為相對路徑 `/api/video_feed`，狀態請求發送至 `/api/fall_status`。
    * **本地開發代理設定**：為了在本地開發時 (`npm run serve`) 能順利調用後端 API，在專案根目錄建立 `vue.config.js`：
      ```javascript
      // vue.config.js
      module.exports = {
        devServer: {
          proxy: {
            '/api': { // 將所有 /api 開頭的請求
              target: '[http://100.0.0.1:5000](http://100.0.0.1:5000)', // 代理到後端 Flask 服務地址
              changeOrigin: true, // 允許跨域
              // 可選：如果後端 API 路徑不含 /api 前綴，需要重寫路徑
              // pathRewrite: { '^/api': '' }
            }
          }
        }
      };
      ```
    * 使用 `npm run serve` 啟動開發伺服器進行測試。
3.  **打包應用**:
    * 確認應用功能正常後，執行打包指令：
      ```bash
      npm run build
      ```
    * 此命令會在專案內產生一個 `dist/` 目錄，其中包含所有用於部署的靜態檔案 (HTML, CSS, JavaScript)。

### 3. Nginx 配置與部署

1.  **複製前端檔案**：
    * 將 Vue 專案 `dist/` 目錄下的 *所有內容* 複製到 Nginx 設定的網站根目錄。例如，如果 Nginx 配置中使用 `/var/www/vue_app`，則執行：
      ```bash
      sudo cp -r dist/* /var/www/vue_app/
      ```
    * 確保 Nginx 有讀取這些檔案的權限。
2.  **配置 Nginx**：
    * 編輯 Nginx 的配置文件（通常在 `/etc/nginx/sites-available/` 目錄下，然後創建符號連結到 `/etc/nginx/sites-enabled/`）。
    * 以下是一個配置範例 (`yourdomain.com` 需替換為您的伺服器 IP 或域名，`/var/www/vue_app` 需替換為您實際存放前端檔案的路徑)：
      ```nginx
      server {
          listen 80;
          server_name yourdomain.com; # 替換成你的域名或 IP

          # 前端靜態檔案根目錄
          root /var/www/vue_app;
          index index.html;

          # 處理 Vue Router 的 History 模式 (如果使用)
          # 任何找不到的檔案請求都重定向到 index.html，讓 Vue Router 處理
          location / {
              try_files $uri $uri/ /index.html;
          }

          # 反向代理 API 請求到 Flask 後端
          location /api/ {
              # 將請求轉發給後端 Flask 服務
              proxy_pass [http://100.0.0.1:5000/](http://100.0.0.1:5000/); # 注意後面的斜線！

              # 設定必要的 Header，讓後端能獲取原始請求資訊
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto $scheme;

              # 針對 WebSocket 或長時間連接的額外設定 (如果需要)
              # proxy_http_version 1.1;
              # proxy_set_header Upgrade $http_upgrade;
              # proxy_set_header Connection "upgrade";
          }

          # 可選：設定錯誤頁面、日誌等
          # error_page 500 502 503 504 /50x.html;
          # access_log /var/log/nginx/vue_app.access.log;
          # error_log /var/log/nginx/vue_app.error.log;
      }
      ```
3.  **測試與重啟 Nginx**：
    * 檢查 Nginx 配置語法是否有誤：
      ```bash
      sudo nginx -t
      ```
    * 如果語法正確，重新啟動 Nginx 服務以應用變更：
      ```bash
      sudo systemctl restart nginx
      # 或者 sudo service nginx restart
      ```

4.  **驗證**：
    * 在瀏覽器中訪問您的 `http://yourdomain.com` (或伺服器 IP)。
    * 應能看到 Vue 應用界面。
    * 檢查瀏覽器的開發者工具 (Network tab)，確認：
        * 前端靜態資源 (HTML, CSS, JS) 是否成功載入。
        * `/api/video_feed` 是否正確顯示影像串流。
        * `/api/fall_status` 的請求是否成功發出並獲得回應，狀態是否按預期更新。

---

## 常見問題排解 (Placeholder)

* **API 請求 404 Not Found**:
    * 檢查 Nginx 的 `proxy_pass` 地址是否正確指向後端 Flask 服務。
    * 檢查 Flask 後端是否已啟動且正常監聽。
    * 確認 Nginx 配置中的 `location /api/` 路徑與前端請求的路徑匹配。注意 `proxy_pass` 後的斜線 `/` 可能影響路徑轉發。
* **靜態檔案 404 Not Found**:
    * 檢查 Nginx 配置中的 `root` 指令是否指向正確的前端檔案目錄。
    * 確認 `dist/` 目錄下的檔案已正確複製到 Nginx 的 `root` 目錄。
    * 檢查檔案權限，確保 Nginx 進程有讀取權限。
* **影像流無法顯示**:
    * 檢查後端 `/api/video_feed` 是否能直接透過 `curl` 或瀏覽器訪問 (例如 `http://<server_ip>:5000/api/video_feed`)。
    * 檢查瀏覽器控制台是否有錯誤訊息。
    * 確認 Nginx 配置沒有阻擋長時間連接。
* **狀態更新失敗或顯示錯誤**:
    * 檢查後端 `/api/fall_status` 是否能正常返回 JSON 資料。
    * 檢查前端 JavaScript 的 `Workspace` 或 `axios` 請求是否正確處理回應和錯誤。
    * 查看 Nginx 和 Flask 的日誌以獲取詳細錯誤資訊。

---
