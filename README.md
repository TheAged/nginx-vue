# 部署指南 - 前端 Vue 與 Nginx 代理後端 Flask API

本文件介紹如何整合前端與後端系統，實現一個基於 Vue 的單頁應用（SPA），利用 Nginx 做反向代理，將 API 請求轉向基於 Flask 的後端服務。後端負責處理影像流（例如跌倒偵測應用），提供即時影像與狀態資料。以下說明各部分的準備工作、部署流程與常見問題排解。

---

## 目標架構

- **前端：Vue.js 應用**  
  - 實現即時影像串流顯示與狀態輪詢。
  - 使用 Vue CLI 建立專案，並支援跨域請求代理以方便開發。

- **反向代理：Nginx**  
  - 提供前端資源的靜態檔案服務。
  - 將 `/api/` 相關的 HTTP 請求引導至後端服務（例如 Flask API），確保前後端在同一網域下溝通。

- **後端：Flask API**  
  - 接收並處理從攝影裝置（例如 Raspberry Pi）發出的數據。
  - 提供兩個主要路由：
    - `/api/video_feed`：返回經處理的 MJPEG 影像流。
    - `/api/fall_status`：傳回目前檢測狀態，如「No Fall Detected」或「Fall Detected!」。

---

## 系統需求與準備工作

### 必備環境

- **Node.js & npm/yarn**  
  - 安裝 Node.js（建議使用最新 LTS 版本），以便使用 Vue CLI。

- **Vue CLI**  
  - 全域安裝 Vue CLI：
    ```bash
    npm install -g @vue/cli
    ```

- **Nginx**  
  - 請根據你的作業系統安裝 Nginx。參考官方文件：[Nginx Installation](https://nginx.org/en/docs/install.html)

- **Python 與 Flask**  
  - 後端服務使用 Python 3 與 Flask。請確保安裝 OpenCV、MediaPipe、Ultralytics YOLO、numpy 等 Python 模組，以支援影像處理。

### 範例網絡設定

- Flask API 假設運行在 `127.0.0.1:5000`
- Nginx 將作為前端靜態資源與 API 的反向代理服務
- Vue 前端專案在生產環境由 Nginx 提供

---

## 後端與 Nginx 配置說明

### Flask API 部署

- 後端服務需監聽所有網卡，啟動命令類似：
  ```python
  app.run(host='0.0.0.0', port=5000, debug=False, threaded=True)

