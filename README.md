# Git Notebook

以 Quartz 5 發布的繁體中文學習筆記，主題是自架 Git 交付平台的 Git 內部、hook、失效復原、分散式鎖、儲存耐久性與 Linux／GCP。

網站網址：<https://ar36planet.github.io/git-notebook/>

## 內容

- Quartz 網站內容位於 content/
- 學習路線首頁位於 content/git-agent-platform-learning/index.md
- Quartz 設定位於 quartz.config.yaml

## 本機預覽

需要 Node.js 22+ 與 npm 10.9.2+。

- npm ci
- npx quartz plugin install
- npx quartz build --serve

## 發布

GitHub Pages workflow 監看 v5 分支。第一次發布前，請在 GitHub repository 的 Settings → Pages → Build and deployment 將 Source 設為 GitHub Actions。之後推送 v5 分支會自動建置與部署。
