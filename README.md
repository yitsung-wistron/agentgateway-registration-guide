# AgentGateway 部署與 MCP Server 註冊指南

<div align="center">

**完整的 AgentGateway 平台部署與多 MCP Server 統一管理指南**

[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://www.docker.com/)
[![AgentGateway](https://img.shields.io/badge/AgentGateway-Latest-green.svg)](https://github.com/agentgateway/agentgateway)
[![MCP](https://img.shields.io/badge/MCP-Compatible-purple.svg)](https://modelcontextprotocol.io/)

</div>

---

## 📖 目錄

- [專案簡介](#-專案簡介)
- [核心特性](#-核心特性)
- [技術架構](#-技術架構)
- [前置需求](#-前置需求)
- [快速開始](#-快速開始)
- [專案結構](#-專案結構)
- [完整配置指南](#-完整配置指南-以-mcp-server-development-guide-為例)
  - [Docker Compose 配置詳解](#1-docker-compose-配置詳解)
  - [AgentGateway 配置文件](#2-agentgateway-配置文件-以-mcp-server-development-guide-為例)
  - [網路配置詳解](#3-網路配置詳解核心-以-mcp-server-development-guide-為例)
- [使用流程](#-使用流程)
- [測試和使用已註冊的 MCP Server](#-測試和使用已註冊的-mcp-server-以-mcp-server-development-guide-為例)
  - [使用 Python MCP 客戶端](#使用-python-mcp-客戶端-推薦)
    - [準備 Python 環境](#1-準備-python-環境)
    - [創建測試客戶端腳本](#2-創建測試客戶端腳本)
    - [使用測試客戶端](#3-使用測試客戶端)
    - [測試示例輸出](#4-測試示例輸出)
  - [常見問題與故障排除](#5-常見問題與故障排除)
  - [診斷工具](#6-診斷工具)
  - [網路配置驗證清單](#7-網路配置驗證清單)
  - [最佳實踐總結](#8-最佳實踐總結)
- [API 和端點說明](#-api-和端點說明)
- [故障排除](#-故障排除)
- [最佳實踐](#-最佳實踐)
- [與 mcp-server-development-guide 的整合](#-與-mcp-server-development-guide-的整合)
- [常見問題](#-常見問題-以使用-mcp-server-development-guide-為例)
- [參考資源](#-參考資源)

---

## 🎯 專案簡介

本專案提供完整的 **AgentGateway** 平台部署指南，協助您建立統一的 MCP Server 管理和代理服務。

### 什麼是 AgentGateway？

**AgentGateway** 是一個高性能的 MCP (Model Context Protocol) 代理和管理平台，用於：

- **統一入口點**: 為多個 MCP Server 提供單一的訪問端點
- **集中管理**: 通過 Web UI 集中管理和監控所有 MCP Server
- **路由和負載均衡**: 智能路由請求到適當的 MCP Server
- **安全和認證**: 統一的安全策略和 CORS 配置
- **可觀測性**: 完整的日誌記錄和監控功能

### AgentGateway 與 MCP Server 的關係

```
AI Client (Claude, etc)
         ↓
   AgentGateway (統一入口)
    ↙    ↓    ↘
 MCP-1  MCP-2  MCP-3 (多個 MCP Server)
```

AgentGateway 作為中間層：
- AI Client 只需連接到 AgentGateway
- AgentGateway 負責將請求路由到正確的 MCP Server
- 支援動態註冊和移除 MCP Server
- 提供統一的監控和管理介面

### 適用場景

- ✅ 管理多個 MCP Server 的企業環境
- ✅ 需要統一入口點的 AI 應用
- ✅ 需要負載均衡和容錯的生產環境
- ✅ 需要集中監控和日誌的運維場景
- ✅ 開發和測試環境的快速切換

---

## ✨ 核心特性

- 🌐 **統一 MCP 代理**: 單一端點訪問所有 MCP Server
- 📊 **Web 管理介面**: 直觀的 UI 管理和監控
- 🔄 **動態配置**: 支援配置熱更新，無需重啟
- 🐳 **Docker 化部署**: 完整的容器化解決方案
- 🔗 **靈活網路配置**: 支援同主機和跨主機部署
- 🛡️ **CORS 支援**: 內建跨域請求處理
- 📝 **詳細日誌**: Debug 級別日誌支援故障診斷
- ⚡ **高性能**: 基於 Rust 語言開發的高效能代理
- 🔌 **多 Server 支援**: 同時註冊和管理多個 MCP Server
- 🎯 **路由策略**: 靈活的路由和策略配置

---

## 🏗️ 技術架構

### 整體架構圖

```
┌─────────────────────────────────────────────────────────┐
│                      AI Client                          │
│                  (Claude, LLM Apps)                     │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP/MCP Protocol
                     v
┌─────────────────────────────────────────────────────────┐
│                   AgentGateway                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   MCP Proxy  │  │   Router     │  │  Web UI      │  │
│  │  Port:32010  │  │   Engine     │  │  Port:32011  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└────────┬────────────┬────────────┬────────────┬─────────┘
         │            │            │            │
         v            v            v            v
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│ MCP Server │  │ MCP Server │  │ MCP Server │  │    ...     │
│  Service-1 │  │  Service-2 │  │  Service-3 │  │            │
│ Port:1024  │  │ Port:2024  │  │ Port:3024  │  │            │
└────────────┘  └────────────┘  └────────────┘  └────────────┘
```

### Docker 網路拓撲（同主機部署）

```
┌─────────────────────────────────────────────────────────┐
│           Docker Network: agentgateway-network          │
│                       (Bridge Mode)                     │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  agentgateway-registration-guide Container       │  │
│  │  - AgentGateway Process                          │  │
│  │  - Port: 32010 (MCP Proxy)                       │  │
│  │  - Port: 32011 (Admin UI)                        │  │
│  └──────────────────────────────────────────────────┘  │
│                           ↕                             │
│  ┌──────────────────────────────────────────────────┐  │
│  │  mcp-server-development-guide Container         │  │
│  │  - FastAPI Service (Port: 1023)                  │  │
│  │  - MCP Server (Port: 1024)                       │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  容器間通過容器名稱進行 DNS 解析                           │
│  示例: http://mcp-server-development-guide:1024/mcp   │
└─────────────────────────────────────────────────────────┘
```

### 核心技術棧

| 技術 | 版本 | 用途 |
|------|------|------|
| **AgentGateway** | 0.8.2 | MCP 代理服務器 (Rust) |
| **Docker** | Latest | 容器化環境 |
| **Docker Compose** | Latest | 容器編排 |
| **YAML** | - | 配置文件格式 |
| **MCP Protocol** | - | Model Context Protocol |

---

## 📋 前置需求

### 必要環境

- **Docker** (20.10+) 與 **Docker Compose** (v2.0+)
- **網路存取**: 確保防火牆允許相關端口通訊

### 選用工具

- **curl**: 用於 API 測試
- **瀏覽器**: 訪問 AgentGateway Web UI

### 環境驗證

```bash
# 檢查 Docker 版本
docker --version
docker compose version

# 檢查 Docker 網路
docker network ls | grep agentgateway-network

# 如果網路不存在，會在啟動時自動創建
```

---

## 🚀 快速開始

### 1. 克隆專案

```bash
git clone git@gitlab-k8s.wzs.wistron.com.cn:llm-application/agentgateway-registration-guide.git
cd agentgateway-registration-guide
```

### 2. 確認 MCP Server 已運行 (以 mcp-server-development-guide 為例)

在啟動 AgentGateway 之前，確保您的 MCP Server 已經運行：

```bash
# 檢查 MCP Server 容器狀態
docker ps | grep mcp-server-development-guide

# 測試 MCP Server 可訪問性
curl http://localhost:1024/mcp
```

### 3. 配置 config.yaml (以 mcp-server-development-guide 為例)

根據您的部署場景選擇配置方式：

**場景 A: 同主機部署**

```yaml
backends:
- mcp:
    targets:
    - name: mcp-server-development-guide
      mcp:
        host: http://mcp-server-development-guide:1024/mcp
```

**場景 B: 跨主機部署**

```yaml
backends:
- mcp:
    targets:
    - name: mcp-server-development-guide
      mcp:
        host: http://10.28.33.15:1024/mcp  # 替換為實際 IP
```

### 4. 啟動 AgentGateway

```bash
# 啟動服務（背景執行）
docker compose up -d

# 查看啟動日誌
docker compose logs -f agentgateway-registration-guide
```

成功啟動後會看到：

```
agentgateway-registration-guide  | 2026-03-23T07:49:14.478Z	info	agentgateway::app	serving UI at http://0.0.0.0:15000/ui
agentgateway-registration-guide  | 2026-03-23T07:49:14.479Z	info	agentgateway::proxy::gateway	started bind	bind/32010
```

### 5. 驗證服務

**方式 1: 訪問 Web UI**

開啟瀏覽器訪問: http://localhost:32011/ui

您應該能看到：
- AgentGateway 管理介面
- 已註冊的 MCP Server 列表
- 服務健康狀態

**方式 2: 測試 MCP 代理端點**

根據您的測試環境選擇對應的測試方式：

```bash
# 情境 A: 在主機上直接測試（非容器環境）
curl http://localhost:32010

# 情境 B: 在同主機的容器內測試（例如在 mcp-server-development-guide 容器內）
curl http://agentgateway-registration-guide:32010

# 情境 C: 從遠端主機測試（跨主機部署）
curl http://<YOUR_HOST>:32010  # 請將 <YOUR_HOST> 替換為 AgentGateway 主機的實際 IP 地址


# 通過 AgentGateway 調用 MCP Server
# (具體測試方法請參閱本文檔的「測試和使用已註冊的 MCP Server」章節)
```

**測試說明**：
- **情境 A**: 適用於在 AgentGateway 主機上直接使用 curl 測試
- **情境 B**: 適用於在同主機的其他容器內測試（需在同一 Docker 網路中）
- **情境 C**: 適用於從其他主機訪問 AgentGateway（需確保防火牆允許 32010 端口）

**方式 3: 檢查容器狀態**

```bash
# 查看運行狀態
docker compose ps

# 應該顯示
# NAME                              STATUS
# agentgateway-registration-guide   Up
```

### 6. 停止服務

```bash
# 停止服務
docker compose down

# 停止並刪除 volumes（謹慎使用）
docker compose down -v
```

---

## 📁 專案結構

```
agentgateway_registration_guide/
├── docker-compose.yml              # Docker Compose 配置
├── config.yaml                     # AgentGateway 主配置文件
├── config-mcp-server.yaml.example  # 配置範例（mcp server 註冊範例）
└── README.md                       # 本文件
```

### 核心文件說明

#### `docker-compose.yml`
Docker Compose 配置文件，定義：
- AgentGateway 服務容器
- 端口映射配置
- Volume 掛載
- Docker 網路設定

#### `config.yaml`
AgentGateway 的主配置文件，包含：
- 管理介面設定
- 日誌配置
- MCP Server 註冊資訊
- 路由和 CORS 策略

#### 配置範例文件
- `config-mcp-server.yaml.example`: mcp server 的配置範本

---

## 📚 完整配置指南 (以 mcp-server-development-guide 為例)

### 1. Docker Compose 配置詳解

#### 完整配置範例

`docker-compose.yml`:

```yaml
services:
  agentgateway-registration-guide:
    container_name: agentgateway-registration-guide
    image: ghcr.io/agentgateway/agentgateway:0.8.2
    command: ["-f", "/config/config.yaml"]

    ports:
      - "32010:32010"          # MCP 代理端點
      - "0.0.0.0:32011:15000" # 管理 UI

    volumes:
      - ./config.yaml:/config/config.yaml:ro  # 唯讀掛載配置

    restart: unless-stopped

    networks:
      - agentgateway-network  # 加入自定義網路

networks:
  agentgateway-network:
    name: agentgateway-network
    driver: bridge
```

#### 配置項目說明

| 配置項 | 說明 | 注意事項 |
|--------|------|----------|
| `container_name` | 容器名稱 | 用於 Docker DNS 解析 |
| `image` | AgentGateway 映像 | 使用版本 tag 確保跨平台一致性 |
| `command` | 啟動命令 | 指定配置文件路徑 |
| `ports` | 端口映射 | 32010 (代理), 32011 (UI) |
| `volumes` | 文件掛載 | `:ro` 表示唯讀掛載 |
| `restart` | 重啟策略 | unless-stopped 確保高可用 |
| `networks` | 網路配置 | **關鍵**: 必須與 MCP Server 同網路 |

#### 端口配置

| 端口 | 用途 | 訪問方式 |
|------|------|----------|
| **32010** | MCP 代理端點 | AI Client 連接此端口 |
| **32011** | Web 管理 UI | 瀏覽器訪問 http://localhost:32011/ui |

---

### 2. AgentGateway 配置文件 (以 mcp-server-development-guide 為例)

#### 完整配置範例

`config.yaml`:

```yaml
# yaml-language-server: $schema=../../schema/local.json
config:
  adminAddr: "0.0.0.0:15000"  # 管理介面監聽地址
  logging:
    filter: debug              # 日誌級別 (debug, info, warn, error)

binds:
- port: 32010                  # MCP 代理監聽端口
  listeners:
  - routes:
    - policies:
        cors:
          allowOrigins:       # CORS 允許的來源
            - "*"
          allowHeaders:       # CORS 允許的標頭
            - content-type
            - mcp-protocol-version
            - cache-control
          exposeHeaders:      # CORS 暴露的標頭
            - Mcp-Session-Id
      backends:
      - mcp:
          targets:
          # 同主機部署: 使用容器名稱
          - name: mcp-server-development-guide
            mcp:
              host: http://mcp-server-development-guide:1024/mcp

          # 跨主機部署: 使用 IP 地址（註解掉上面的配置）
          # - name: mcp-server-development-guide
          #   mcp:
          #     host: http://10.28.33.15:1024/mcp
```

#### 配置區塊詳解

**1. config 區塊**

```yaml
config:
  adminAddr: "0.0.0.0:15000"  # 管理介面綁定地址
                               # 0.0.0.0 表示監聽所有網路介面
  logging:
    filter: debug               # 生產環境可改為 info
```

**2. binds 區塊**

```yaml
binds:
- port: 32010                   # MCP 代理服務監聽的端口
  listeners:                   # 可配置多個監聽器
  - routes:                    # 路由規則
    - policies:                # 策略配置
        cors:                  # CORS 策略
          allowOrigins:
            - "*"              # 開發環境可用 *，生產環境應指定具體域名
          allowHeaders:
            - content-type
            - mcp-protocol-version
            - cache-control
          exposeHeaders:
            - Mcp-Session-Id
```

**3. backends 區塊（關鍵）**

```yaml
backends:
- mcp:                         # Backend 類型為 MCP
    targets:                   # MCP Server 目標列表
    - name: server-name        # Server 識別名稱
      mcp:
        host: http://...       # MCP Server 的完整 URL
```

#### 註冊多個 MCP Server

```yaml
backends:
- mcp:
    targets:
    # Server 1
    - name: mcp-server-development-guide
      mcp:
        host: http://mcp-server-development-guide:1024/mcp

    # Server 2
    - name: another-mcp-server
      mcp:
        host: http://another-mcp-server:2024/mcp

    # Server 3 (跨主機)
    - name: remote-mcp-server
      mcp:
        host: http://192.168.1.100:3024/mcp
```

---

### 3. 網路配置詳解（核心）(以 mcp-server-development-guide 為例)

這是同主機部署成功的關鍵配置。

#### 為什麼需要自定義 Docker 網路？

**問題場景**:
- AgentGateway 和 MCP Server 都在同一台主機上運行
- 使用 `localhost` 或 `127.0.0.1` 無法連接（容器網路隔離）
- 使用主機 IP 可能遇到防火牆或路由問題

**解決方案**：
使用 Docker 自定義網路 `agentgateway-network`，讓容器間可以通過容器名稱互相訪問。

#### 同主機部署配置

**步驟 1: 確保 MCP Server 加入網路**

在 MCP Server 的 `docker-compose.yml` 中：

```yaml
services:
  wih-aiteam-base:
    container_name: mcp-server-development-guide
    # ... 其他配置 ...
    networks:
      - agentgateway-network  # 加入相同網路

networks:
  agentgateway-network:
    name: agentgateway-network
    driver: bridge
```

**步驟 2: AgentGateway 加入相同網路**

在 AgentGateway 的 `docker-compose.yml` 中：

```yaml
services:
  agentgateway-registration-guide:
    # ... 其他配置 ...
    networks:
      - agentgateway-network  # 加入相同網路

networks:
  agentgateway-network:
    name: agentgateway-network
    driver: bridge
```

**步驟 3: 配置使用容器名稱**

在 `config.yaml` 中：

```yaml
backends:
- mcp:
    targets:
    - name: mcp-server-development-guide
      mcp:
        # 使用容器名稱，Docker 會自動 DNS 解析
        host: http://mcp-server-development-guide:1024/mcp
```

#### 網路連通性驗證

```bash
# 1. 檢查網路是否存在
docker network ls | grep agentgateway-network

# 2. 查看網路詳細資訊
docker network inspect agentgateway-network

# 3. 確認兩個容器都在同一網路中
docker network inspect agentgateway-network | grep -A 5 "Containers"

# 應該看到:
# - agentgateway-registration-guide
# - mcp-server-development-guide
```

#### 跨主機部署配置 (以 mcp-server-development-guide 為例)

當 MCP Server 在不同主機時：

**步驟 1: 獲取 MCP Server 主機 IP**

```bash
# 在 MCP Server 主機上
ip addr show | grep inet
# 或
hostname -I
```

**步驟 2: 配置使用 IP 地址**

在 `config.yaml` 中：

```yaml
backends:
- mcp:
    targets:
    - name: mcp-server-development-guide
      mcp:
        host: http://10.28.33.15:1024/mcp  # 使用實際 IP
```

**步驟 3: 確保網路可達**

```bash
# 在 AgentGateway 主機測試連通性
curl http://10.28.33.15:1024/mcp

# 檢查防火牆規則
sudo ufw status
# 或
sudo firewall-cmd --list-all

# 如需開放端口
sudo ufw allow 1024
```

#### 網路配置對比

| 配置項 | 同主機部署 | 跨主機部署 |
|--------|-----------|-----------|
| **網路類型** | Docker 自定義網路 | 物理網路 |
| **主機地址** | 容器名稱 | IP 地址 |
| **配置示例** | `http://container-name:1024/mcp` | `http://10.28.33.15:1024/mcp` |
| **DNS 解析** | Docker 內建 DNS | 系統 DNS 或直接 IP |
| **防火牆** | 不需要配置 | 需要開放端口 |
| **複雜度** | 低 | 中 |
| **推薦場景** | 開發/測試環境 | 生產環境 |

#### 網路配置最佳實踐

1. **優先使用同主機部署**
   - 更簡單、更安全
   - 延遲更低
   - 無需配置防火牆

2. **網路命名一致**
   ```yaml
   networks:
     agentgateway-network:
       name: agentgateway-network  # 使用 name 確保網路名稱一致
   ```

3. **避免使用 localhost**
   ```yaml
   # ❌ 錯誤
   host: http://localhost:1024/mcp

   # ❌ 錯誤
   host: http://127.0.0.1:1024/mcp

   # ✅ 正確（同主機）
   host: http://mcp-server-development-guide:1024/mcp

   # ✅ 正確（跨主機）
   host: http://10.28.33.15:1024/mcp
   ```

4. **驗證再部署**
   - 先啟動 MCP Server
   - 驗證網路連通性
   - 再啟動 AgentGateway

---

## 🔄 使用流程

### 完整的端到端流程

#### 1. 確保開啟 MCP Server

#### 3. 啟動 AgentGateway

```bash
# 啟動服務
docker compose up -d

# 查看啟動日誌
docker compose logs -f agentgateway-registration-guide

# 等待看到成功訊息
# 2026-03-23T07:49:14.479Z	info	agentgateway::proxy::gateway	started bind	bind/32010
```

#### 4. 驗證註冊狀態

**方法 1: Web UI 檢查**

1. 開啟瀏覽器: http://localhost:32011/ui
2. 查看 "Backends" 或 "MCP Servers" 區塊
3. 確認你的 mcp server 顯示為 "Active" 或 "Healthy"

**方法 2: 日誌檢查**

```bash
# 查看 AgentGateway 日誌
docker compose logs agentgateway-registration-guide | grep -i mcp

# 成功時會看到類似:
# 2026-03-23T07:49:14.480Z	debug	Connected to MCP server	mcp-server-development-guide
```

#### 5. 調用 MCP 工具

通過 AgentGateway 調用已註冊的 MCP Server 工具：

```
AI Client → AgentGateway:32010 → MCP Server:1024
```

AgentGateway 會自動路由請求到正確的 MCP Server。

#### 6. 監控和日誌

**實時日誌**:

```bash
# AgentGateway 日誌
docker compose logs -f agentgateway-registration-guide

# 過濾特定級別
docker compose logs -f | grep 'error'
```

**Web UI 監控**:
- 訪問 http://localhost:32011/ui
- 查看 Server 狀態、請求統計、錯誤率等

---

## 🧪 測試和使用已註冊的 MCP Server (以 mcp-server-development-guide 為例)

本章節介紹如何通過 SSE (Server-Sent Events) 方式測試和調用 AgentGateway 上已註冊的 MCP Server。

### 測試方法概述

AgentGateway 使用 MCP (Model Context Protocol) 協議通過 SSE 方式與 MCP Server 進行通訊。要正確測試，需要使用 MCP Python SDK。

**可用的測試方法**:
- **Python MCP 客戶端**: 使用 MCP SDK 進行程式化測試

**測試功能**:
- **test-connection**: 測試與 MCP Server 的連接
- **list-tools**: 列出 MCP Server 提供的所有工具
- **call-tool**: 調用特定的 MCP 工具
- **validate**: 執行完整的驗證測試

### 重要網路配置說明 ⚠️

在測試之前，請先了解不同部署場景下的網路配置要求：

| 場景 | Client 位置 | AgentGateway 位置 | 連接方式 | 範例 URL |
|------|------------|------------------|---------|---------|
| **場景 A** | 容器內 (同主機) | 容器內 (同主機) | 容器名稱 + 端口 | `http://agentgateway-registration-guide:32010` |
| **場景 B** | 遠端主機 | AgentGateway 主機 | IP + 端口 | `http://10.28.33.15:32010` |
| **場景 C** | 本機主機 (非容器) | 本機主機 | localhost + 端口 | `http://localhost:32010` |

**關鍵注意事項**：
- ❌ **錯誤**：如果 Client 和 AgentGateway 在同一台主機的不同容器中，不能使用 IP 地址和端口來訪問
- ✅ **正確**：在同主機容器環境中，必須使用容器名稱和端口（例如：`http://agentgateway-registration-guide:32010`）
- ✅ **正確**：如果 Client 在主機上（非容器），可以使用 `http://localhost:32010`
- ✅ **正確**：如果 Client 和 AgentGateway 在不同主機上，使用 IP 地址和端口（例如：`http://10.28.33.15:32010`）

---

### 使用 Python MCP 客戶端 (推薦)

#### 1. 準備 Python 環境

首先需要安裝 MCP Python SDK 和相關依賴。

**在容器環境中**:
```bash
# 進入主機的相同網絡中的任意容器 (例如: mcp-server-development-guide)
docker exec -it mcp-server-development-guide bash

# 添加必要的依賴套件
uv add mcp httpx click python-dotenv
```

**在本機環境中**:
```bash
# 使用 pip 安裝
pip install mcp httpx click python-dotenv

# 或使用 uv
uv pip install mcp httpx click python-dotenv
```

---

#### 2. 創建測試客戶端腳本

創建一個名為 `agentgateway_client.py` 的測試腳本:

```python
"""AgentGateway MCP 測試客戶端"""

import asyncio
import json
import logging
from contextlib import AsyncExitStack
from typing import Optional

import click
from mcp import ClientSession
from mcp.client.sse import sse_client

# 配置日誌
logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger(__name__)


class AgentGatewayClient:
    """AgentGateway MCP 客戶端"""

    def __init__(self, gateway_url: str):
        self.gateway_url = gateway_url
        self.session: Optional[ClientSession] = None
        self.exit_stack = AsyncExitStack()

    async def connect(self):
        """連接到 AgentGateway 的 SSE 端點"""
        sse_url = f"{self.gateway_url}/sse"
        logger.info(f"正在連接到 {sse_url}...")

        try:
            streams = await self.exit_stack.enter_async_context(
                sse_client(url=sse_url)
            )
            self.session = await self.exit_stack.enter_async_context(
                ClientSession(*streams)
            )
            await self.session.initialize()
            await self.session.list_tools()
            logger.info("✓ 連接成功!")
        except Exception as e:
            raise ConnectionError(f"✗ 連接失敗: {str(e)}")

    async def list_tools(self):
        """列出所有可用的工具"""
        if not self.session:
            raise RuntimeError("未連接。請先調用 connect()")

        tools_response = await self.session.list_tools()
        return tools_response.tools

    async def call_tool(self, tool_name: str, arguments: dict):
        """調用特定工具"""
        if not self.session:
            raise RuntimeError("未連接。請先調用 connect()")

        result = await self.session.call_tool(tool_name, arguments)
        return result

    async def close(self):
        """關閉連接"""
        await self.exit_stack.aclose()
        logger.info("連接已關閉")


# ===== CLI 命令 =====

@click.group()
def cli():
    """AgentGateway MCP 測試客戶端"""
    pass


@cli.command()
@click.option("--url", required=True, help="AgentGateway URL")
def test_connection(url):
    """測試連接"""
    asyncio.run(_test_connection(url))


async def _test_connection(url):
    client = AgentGatewayClient(url)
    try:
        await client.connect()
        click.echo(f"\n✓ 成功連接到 {url}")
    except Exception as e:
        click.echo(f"\n✗ 連接失敗: {e}", err=True)
    finally:
        await client.close()


@cli.command()
@click.option("--url", required=True, help="AgentGateway URL")
def list_tools(url):
    """列出所有可用工具"""
    asyncio.run(_list_tools(url))


async def _list_tools(url):
    client = AgentGatewayClient(url)
    try:
        await client.connect()
        tools = await client.list_tools()

        click.echo("\n=== 可用的工具 ===")
        if tools:
            for tool in tools:
                click.echo(f"\n🔧 工具: {tool.name}")
                click.echo(f"   描述: {tool.description}")
                if hasattr(tool, "inputSchema"):
                    click.echo(f"   參數: {json.dumps(tool.inputSchema, indent=2, ensure_ascii=False)}")
        else:
            click.echo("沒有可用的工具")
    except Exception as e:
        click.echo(f"\n✗ 失敗: {e}", err=True)
    finally:
        await client.close()


@cli.command()
@click.option("--url", required=True, help="AgentGateway URL")
@click.option("--tool", required=True, help="工具名稱")
@click.option("--args", default="{}", help="JSON 格式的參數")
def call_tool(url, tool, args):
    """調用工具"""
    asyncio.run(_call_tool(url, tool, args))


async def _call_tool(url, tool_name, args_json):
    client = AgentGatewayClient(url)
    try:
        await client.connect()

        # 解析參數
        try:
            arguments = json.loads(args_json)
        except json.JSONDecodeError as e:
            click.echo(f"✗ 無效的 JSON 參數: {e}", err=True)
            return

        click.echo(f"\n📞 調用工具: {tool_name}")
        click.echo(f"   參數: {json.dumps(arguments, indent=2, ensure_ascii=False)}")

        result = await client.call_tool(tool_name, arguments)

        click.echo("\n=== 結果 ===")
        if hasattr(result, "content"):
            for content_item in result.content:
                if hasattr(content_item, "text"):
                    click.echo(content_item.text)
                else:
                    click.echo(str(content_item))
        else:
            click.echo(str(result))
    except Exception as e:
        click.echo(f"\n✗ 調用失敗: {e}", err=True)
    finally:
        await client.close()


@cli.command()
@click.option("--url", required=True, help="AgentGateway URL")
def validate(url):
    """執行完整驗證測試"""
    asyncio.run(_validate(url))


async def _validate(url):
    click.echo("=== AgentGateway 完整驗證 ===\n")

    client = AgentGatewayClient(url)

    # 1. 測試連接
    click.echo("[1/3] 測試連接...")
    try:
        await client.connect()
        click.echo("  ✓ 連接成功")
    except Exception as e:
        click.echo(f"  ✗ 連接失敗: {e}")
        return

    # 2. 列出工具
    click.echo("\n[2/3] 列出可用工具...")
    try:
        tools = await client.list_tools()
        click.echo(f"  ✓ 找到 {len(tools)} 個工具")
        if tools:
            click.echo("  前 3 個工具:")
            for tool in tools[:3]:
                click.echo(f"    - {tool.name}")
    except Exception as e:
        click.echo(f"  ✗ 失敗: {e}")

    # 3. 測試工具調用
    click.echo("\n[3/3] 測試工具調用...")
    try:
        if tools:
            first_tool = tools[0]
            click.echo(f"  嘗試調用: {first_tool.name}")
            try:
                result = await client.call_tool(first_tool.name, {})
                click.echo(f"  ✓ 調用成功")
            except Exception as e:
                click.echo(f"  ⚠ 調用失敗 (可能需要參數): {e}")
        else:
            click.echo("  ⚠ 沒有可用的工具進行測試")
    except Exception as e:
        click.echo(f"  ✗ 測試失敗: {e}")

    await client.close()
    click.echo("\n=== 驗證完成 ===")


if __name__ == "__main__":
    cli()
```

---

#### 3. 使用測試客戶端

根據您的部署場景選擇正確的 URL：

**場景 A：同主機容器環境（Client 在容器內）**
```bash
# 進入主機的相同網路中的任意容器（例如：mcp-server-development-guide）
docker exec -it mcp-server-development-guide bash

# 測試連接
uv run agentgateway_client.py test-connection \
  --url http://agentgateway-registration-guide:32010

# 列出工具
uv run agentgateway_client.py list-tools \
  --url http://agentgateway-registration-guide:32010

# 調用工具(注意你的tool名稱, 當同時註冊多個 MCP Server 時, 你原先的 tool 名稱可能會改變)
uv run agentgateway_client.py call-tool \
  --url http://agentgateway-registration-guide:32010 \
  --tool get_greeting \
  --args '{"name": "Alice"}'

# 完整驗證
uv run agentgateway_client.py validate \
  --url http://agentgateway-registration-guide:32010
```

**場景 B/C：跨主機或本機測試**
```bash
# 跨主機使用 IP 地址
python agentgateway_client.py test-connection \
  --url http://10.28.33.15:32010

# 本機測試使用 localhost
python agentgateway_client.py test-connection \
  --url http://localhost:32010

# 其他命令類似，只需修改 --url 參數
```

---

#### 4. 測試示例輸出

**測試連接**：
```
$ uv run agentgateway_client.py test-connection --url http://localhost:32010

正在連接到 http://localhost:32010/sse...
✓ 連接成功!

✓ 成功連接到 http://localhost:32010
連接已關閉
```

**列出工具**：
```
$ uv run agentgateway_client.py list-tools --url http://localhost:32010

正在連接到 http://localhost:32010/sse...
✓ 連接成功!

=== 可用的工具 ===

🔧 工具: get_greeting
   描述: Get a personalized greeting message
   參數: {
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The name to greet"
    }
  },
  "required": [
    "name"
  ]
}

🔧 工具: calculate_sum
   描述: Calculate the sum of two numbers
   參數: {
  "type": "object",
  "properties": {
    "a": {
      "type": "integer",
      "description": "First number"
    },
    "b": {
      "type": "integer",
      "description": "Second number"
    }
  },
  "required": [
    "a",
    "b"
  ]
}

連接已關閉
```

**調用工具**：
```
$ uv run agentgateway_client.py call-tool --url http://localhost:32010 --tool get_greeting --args '{"name": "Alice"}'

正在連接到 http://localhost:32010/sse...
✓ 連接成功!

📞 調用工具: get_greeting
   參數: {
  "name": "Alice"
}

=== 結果 ===
Hello, Alice! You know what? It's greeted by service example form mcp-server-development-guide! LOL

連接已關閉
```

---

### 5. 常見問題與故障排除

#### 問題 1: 連接失敗 - "Could not resolve host"

**症狀**：
```
ConnectionError: ✗ 連接失敗: Could not resolve host: agentgateway-registration-guide
```

**原因**： Client 不在 Docker 網路中，無法解析容器名稱。

**解決方案**：

**A. 確認您的運行環境**
```bash
# 檢查您是在容器內還是主機上
hostname  # 如果顯示容器 ID，則在容器內

# 在容器內: 使用容器名稱
--url http://agentgateway-registration-guide:32010

# 在主機上: 使用 localhost
--url http://localhost:32010

# 在遠端主機: 使用 IP
--url http://10.28.33.15:32010
```

**B. 確保容器在同一網路** (容器間通訊)
```bash
# 檢查網路
docker network inspect agentgateway-network

# 如果容器不在網路中，手動連接
docker network connect agentgateway-network <your-container-name>
```

---

#### 問題 2: 連接被拒絕 - "Connection refused"

**症狀**：
```
ConnectionError: ✗ 連接失敗: Connection refused
```

**原因**： AgentGateway 未運行或端口配置錯誤。

**解決方案**：

```bash
# 1. 檢查 AgentGateway 是否運行
docker ps | grep agentgateway-registration-guide

# 2. 如果未運行，啟動服務
cd agentgateway-registration-guide-demo
docker compose up -d

# 3. 檢查日誌
docker compose logs agentgateway-registration-guide

# 4. 驗證端口映射
docker compose ps
# 應該看到: 0.0.0.0:32010->32010/tcp
```

---

### 6. 診斷工具

使用內建的 `validate` 命令進行完整診斷:

```bash
# 執行完整驗證
python agentgateway_client.py validate --url http://localhost:32010
```

這將測試:
1. 連接狀態
2. 可用工具列表
3. 工具調用功能

**示例輸出**：
```
=== AgentGateway 完整驗證 ===

[1/3] 測試連接...
正在連接到 http://localhost:32010/sse...
✓ 連接成功!
  ✓ 連接成功

[2/3] 列出可用工具...
  ✓ 找到 3 個工具
  前 3 個工具:
    - get_greeting
    - calculate_sum
    - get_note

[3/3] 測試工具調用...
  嘗試調用: get_greeting
  ⚠ 調用失敗 (可能需要參數): Missing required argument: name

連接已關閉

=== 驗證完成 ===
```

---

### 7. 網路配置驗證清單

在進行測試之前，請確認以下項目:

- [ ] **AgentGateway 容器正在運行**
  ```bash
  docker ps | grep agentgateway-registration-guide
  ```

- [ ] **MCP Server 容器正在運行**
  ```bash
  docker ps | grep mcp-server-development-guide
  ```

- [ ] **兩個容器在同一 Docker 網路** (同主機部署)
  ```bash
  docker network inspect agentgateway-network | grep -E "(agentgateway|mcp-server)"
  ```

- [ ] **MCP Server 可從主機訪問**
  ```bash
  curl http://localhost:1024/mcp
  ```

- [ ] **AgentGateway 可從主機訪問**
  ```bash
  curl http://localhost:32010
  ```

- [ ] **配置文件中 MCP Server 註冊正確**
  ```bash
  cat config.yaml | grep -A 3 "targets"
  ```

- [ ] **Web UI 顯示 MCP Server 狀態為 Healthy**
  ```
  訪問 http://localhost:32011/ui
  ```

---

### 8. 最佳實踐總結

1. **選擇正確的 URL**
   - 容器內測試: `http://agentgateway-registration-guide:32010`
   - 主機上測試: `http://localhost:32010`
   - 跨主機測試: `http://<IP>:32010`

2. **使用 validate 命令**
   - 快速診斷連接和配置問題
   - 驗證整個鏈路是否正常

3. **查看詳細日誌**
   ```bash
   # AgentGateway 日誌
   docker compose logs -f agentgateway-registration-guide

   # MCP Server 日誌
   docker logs -f mcp-server-development-guide
   ```

4. **先測試連接，再測試功能**
   - 使用 `test-connection` 確保基本連接正常
   - 使用 `list-tools` 確認工具已註冊
   - 使用 `call-tool` 測試具體功能

5. **保存測試腳本**
   - 將 `agentgateway_client.py` 保存到專案中
   - 創建常用測試命令的 shell 腳本
   - 便於快速重複測試

---

## 📖 API 和端點說明

### MCP 代理端點

**基礎 URL**: `http://localhost:32010`

這是 AI Client 連接的主要端點。所有 MCP 協議請求都應發送到此端點。

```bash
# 健康檢查
curl http://localhost:32010/health

# MCP 協議端點（由 AI Client 使用）
# 具體 API 取決於 MCP Protocol 規範
```

### 管理 UI

**URL**: `http://localhost:32011/ui`

Web 管理介面提供：
- 📊 已註冊 MCP Server 列表和狀態
- 📈 請求統計和性能指標
- 🔍 日誌查看和過濾
- ⚙️ 配置檢視（部分版本支援）
- 🛠️ 故障診斷工具

---

## 🔧 故障排除

### 問題 1: Web UI 無法訪問

**症狀**：
- 瀏覽器無法開啟 http://localhost:32011/ui
- 連接被拒絕或超時

**診斷步驟**:

```bash
# 1. 檢查容器運行狀態
docker compose ps

# 2. 檢查端口映射
docker compose ps agentgateway-registration-guide
# 應該看到: 0.0.0.0:32011->15000/tcp

# 3. 檢查容器日誌
docker compose logs agentgateway-registration-guide | grep -i admin

# 4. 測試本地連接
curl http://localhost:32011/ui
```

**解決方案**：

**情況 A: 容器未運行**
```bash
docker compose up -d
```

**情況 B: 端口衝突**
```bash
# 檢查端口佔用
sudo lsof -i :32011
# 或
netstat -tuln | grep 32011

# 修改 docker-compose.yml 使用其他端口
ports:
  - "16000:15000"  # 主機端口改為 16000
```

**情況 C: 防火牆阻擋**
```bash
# Linux
sudo ufw allow 32011

# 或臨時關閉防火牆測試
sudo ufw disable
```

---

### 問題 2: Docker 網路問題

**症狀**：
- 網路 `agentgateway-network` 不存在
- 容器無法加入網路

**診斷步驟**:

```bash
# 列出所有網路
docker network ls

# 檢查網路詳情
docker network inspect agentgateway-network
```

**解決方案**：

**情況 A: 網路不存在**
```bash
# 手動創建網路
docker network create agentgateway-network --driver bridge

# 或透過 docker-compose 創建
docker compose up -d
```

**情況 B: 網路配置衝突**
```bash
# 刪除舊網路（確保沒有容器在使用）
docker network rm agentgateway-network

# 重新創建
docker compose up -d
```

**情況 C: 容器連接失敗**
```bash
# 手動連接容器到網路
docker network connect agentgateway-network <container-name>

# 驗證連接
docker network inspect agentgateway-network
```

---

## 💡 最佳實踐

### 1. 配置管理

**使用版本控制**:
```bash
# 備份當前配置
cp config.yaml config.yaml.backup.$(date +%Y%m%d)

# 使用 Git 管理配置變更
git add config.yaml
git commit -m "Update MCP server configuration"
```

**環境分離**:
```
agentgateway_registration_guide/
├── config.dev.yaml      # 開發環境
├── config.staging.yaml  # 測試環境
└── config.prod.yaml     # 生產環境
```

使用時指定配置:
```yaml
# docker-compose.yml
services:
  agentgateway:
    command: ["-f", "/config/config.${ENV}.yaml"]
```

---

### 2. 安全性考量

**CORS 配置**:

```yaml
# ❌ 開發環境可用，生產環境不安全
cors:
  allowOrigins:
    - "*"
  allowHeaders:
    - content-type
    - mcp-protocol-version
    - cache-control
  exposeHeaders:
    - Mcp-Session-Id

# ✅ 生產環境應指定具體域名
cors:
  allowOrigins:
    - "https://your-app.example.com"
    - "https://api.example.com"
  allowHeaders:
    - content-type
    - mcp-protocol-version
    - cache-control
  exposeHeaders:
    - Mcp-Session-Id
```

---

## 🔗 與 mcp-server-development-guide 的整合

### 端到端完整示例

這裡展示如何將 mcp-server-development-guide的 MCP Server 完整註冊到 AgentGateway。

#### 步驟 1: 確保 MCP Server 配置正確

```bash
# 進入 MCP Server 專案目錄
cd mcp-server-development-guide
```

**主專案 docker-compose.yml** (`/workspace/docker-compose.yml`):

```yaml
services:
  wih-aiteam-base:
    build: .
    container_name: mcp-server-development-guide
    image: mlops-harbor-wzs.wistron.com/k8swzsdat/wih-aiteam-base:jmd6e0

    ports:
      - "1023:1023"  # FastAPI 服務
      - "1024:1024"  # MCP Server
      - "6274:6274"  # MCP Inspector
      - "6277:6277"

    volumes:
      - .:/workspace

    working_dir: /workspace

    environment:
      - TZ=Asia/Taipei

    restart: unless-stopped

    networks:
      - agentgateway-network  # ⭐ 關鍵: 加入網路

networks:
  agentgateway-network:
    name: agentgateway-network
    driver: bridge
```

#### 步驟 2: 啟動 MCP Server 專案服務

```bash
# 啟動容器
docker compose up -d

# 進入容器
docker exec -it mcp-server-development-guide bash

# 啟動 FastAPI Service
uv run src/service_example.py &

# 啟動 MCP Server
uv run src/mcp_server_example.py
```

#### 步驟 3: 驗證 MCP Server 可訪問

```bash
# 在主機上測試
curl http://localhost:1024/mcp

# 應該返回 MCP Server 資訊
```

#### 步驟 4: 配置 AgentGateway

```bash
# 進入 agentgateway 專案目錄
cd agentgateway-registration-guide
```

**agentgateway_demo/config.yaml**:

```yaml
config:
  adminAddr: "0.0.0.0:15000"
  logging:
    filter: debug

binds:
- port: 32010
  listeners:
  - routes:
    - policies:
        cors:
          allowOrigins:
            - "*"
          allowHeaders:
            - content-type
            - mcp-protocol-version
            - cache-control
          exposeHeaders:
            - Mcp-Session-Id
      backends:
      - mcp:
          targets:
          - name: mcp-server-development-guide
            mcp:
              # 使用容器名稱（同主機部署）
              host: http://mcp-server-development-guide:1024/mcp
```

#### 步驟 5: 啟動 AgentGateway

```bash
# 啟動 AgentGateway
docker compose up -d

# 查看日誌確認連接成功
docker compose logs -f
```

#### 步驟 6: 驗證整合

**Web UI**
- 訪問 http://localhost:32011/ui
- 確認看到 `mcp-server-development-guide` 且狀態為 Healthy

---

### 整合架構圖

```
┌─────────────────────────────────────────────────────────┐
│                AI Client / MCP Inspector                │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ MCP Protocol
                     ↓
┌─────────────────────────────────────────────────────────┐
│              AgentGateway (Port 32010)                   │
│              容器: agentgateway-registration-guide      │
│                                                         │
│  [路由] → [CORS] → [MCP Proxy]                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Docker Network: agentgateway-network
                     ↓
┌─────────────────────────────────────────────────────────┐
│          MCP Server (Port 1024)                         │
│          容器: mcp-server-development-guide            │
│                                                         │
│  ┌──────────────────┐     ┌──────────────────┐         │
│  │  MCP Server      │ →   │  FastAPI Service │         │
│  │  (FastMCP)       │     │  (Port 1023)     │         │
│  └──────────────────┘     └──────────────────┘         │
│                                                         │
│  提供工具:                                               │
│  - get_greeting                                         │
│  - calculate_sum                                        │
│  - get_note                                             │
└─────────────────────────────────────────────────────────┘
```

---

## ❓ 常見問題 (以使用 mcp-server-development-guide 為例)

### Q1: AgentGateway 和直接使用 MCP Server 的區別？

**A**:

| 特性 | 直接連接 MCP Server | 通過 AgentGateway |
|------|-------------------|------------------|
| **端點數量** | 每個 Server 一個端點 | 單一統一端點 |
| **管理複雜度** | 需要分別管理多個 Server | 集中管理 |
| **負載均衡** | 需要自行實現 | 內建支援 |
| **監控** | 分散在各 Server | 統一監控介面 |
| **CORS 配置** | 每個 Server 單獨配置 | 統一配置 |
| **適用場景** | 簡單應用、單一 Server | 多 Server、生產環境 |

**建議**:
- 開發和測試: 可以直接連接 MCP Server
- 生產環境: 建議使用 AgentGateway

---

### Q2: 同主機部署時的網路配置為何重要？ 

**A**:

**問題背景**:
- Docker 容器有自己的網路空間
- `localhost` 在容器內指向容器自身，而非主機
- 容器間預設無法直接通訊

**沒有自定義網路時的問題**:

```yaml
# ❌ 這不會工作
host: http://localhost:1024/mcp
# 原因: localhost 指向 AgentGateway 容器本身

# ❌ 這也可能不工作
host: http://127.0.0.1:1024/mcp
# 原因: 同上

# ❌ 使用主機 IP 可能遇到防火牆問題
host: http://192.168.1.100:1024/mcp
```

**使用自定義網路的優勢**:

```yaml
# ✅ Docker 會自動 DNS 解析容器名稱
host: http://mcp-server-development-guide:1024/mcp

# 優勢:
# 1. 簡單: 不需要知道 IP 地址
# 2. 穩定: IP 改變不影響連接
# 3. 安全: 網路隔離
# 4. 高效: 容器間直接通訊，無需經過主機網路棧
```

---

### Q3: 如何診斷連接問題？

**A**: 按照以下步驟逐一排查：

**1. 檢查 MCP Server**

```bash
# MCP Server 容器是否運行？
docker ps | grep mcp-server-development-guide

# MCP Server 端口是否監聽？
curl http://localhost:1024/mcp
```

**2. 檢查網路**

```bash
# 網路是否存在？
docker network ls | grep agentgateway-network

# 容器是否在網路中？
docker network inspect agentgateway-network
```

**3. 檢查 DNS 解析**

```bash
# 進入 AgentGateway 容器
docker exec -it agentgateway-registration-guide sh

# 測試 DNS 解析
nslookup mcp-server-development-guide
ping mcp-server-development-guide
```

**4. 檢查連通性**

```bash
# 在 AgentGateway 容器內
wget -O- http://mcp-server-development-guide:1024/mcp

# 或使用 curl
curl http://mcp-server-development-guide:1024/mcp
```

**5. 查看日誌**

```bash
# AgentGateway 日誌
docker compose logs agentgateway-registration-guide | grep -i error
```

---

### Q6: 如何更新 AgentGateway 到新版本？

**A**:

由於我們使用版本 tag 固定版本，更新只需修改 `docker-compose.yml` 中的映像版本 tag：

```bash
# 1. 查看最新可用的版本 tag
#    參考: https://github.com/agentgateway/agentgateway/pkgs/container/agentgateway

# 2. 將新的版本 tag 更新到 docker-compose.yml 的 image 欄位
#    image: ghcr.io/agentgateway/agentgateway:<新版本號>

# 3. 拉取新映像並重啟服務
docker compose pull
docker compose up -d

# 4. 查看版本資訊
docker compose logs | grep -i version
```

**注意**: 更新前建議備份配置文件。使用版本 tag 可確保跨平台部署的一致性。

---

## 📚 參考資源

### 官方文檔

- **AgentGateway**: https://github.com/agentgateway/agentgateway
- **MCP Protocol**: https://modelcontextprotocol.io/
- **Docker**: https://docs.docker.com/
- **Docker Compose**: https://docs.docker.com/compose/
- **Docker 網路**: https://docs.docker.com/network/

### 相關專案

- **主專案 MCP Server 開發指南**: [../README.md](../README.md)
- **FastMCP**: https://github.com/jlowin/fastmcp
- **MCP Inspector**: https://github.com/modelcontextprotocol/inspector

### 教學資源

- **YAML 語法**: https://yaml.org/spec/
- **Docker 網路教學**: https://docs.docker.com/network/bridge/
- **Docker Compose 網路**: https://docs.docker.com/compose/networking/

### 工具

- **YAML Lint**: https://www.yamllint.com/
- **JSON Formatter**: https://jsonformatter.org/

---

## 📄 授權

本專案為 Wistron 內部使用指南。

---

## 📞 聯絡方式

如有問題或建議，請聯繫：

- **專案維護者**: Wistron AI Team (人工智慧三部 JMD6E0)
- **Email**: YiTsung_Chang@wistron.com

---

<div align="center">

**感謝使用 AgentGateway 部署指南！**

Made with ❤️ by Wistron AI Team

</div>
