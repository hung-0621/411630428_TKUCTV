# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：`Windows 11`
- VM 名稱：`vct-w01-411630428`
- Ubuntu 版本：`Ubuntu 24.04.4 LTS`
- Docker 版本：`29.3.0`
- Docker Compose 版本：`v5.1.1`

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | CPU(s): 4 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | Mem:           3.8Gi       1.3Gi       1.0Gi        35Mi       1.7Gi       2.5Gi
 |
| 磁碟 | 40 GB | `df -h /` | Filesystem    Size  Used Avail Use% Mounted on </br> /dev/sda2        49G   12G   36G  25% /
 |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | Hypervisor vendor: VMware |

## 四層驗收證據
- [x] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出  
    
    **輸出內容** 
    ```bash
    hung@vct-w01-411630428:~$ cat /etc/apt/sources.list.d/docker.list
    deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) noble stable
    ```
- [x] ② Engine：`dpkg -l | grep docker-ce` 輸出  
    **輸出內容** 
    ```bash
    hung@vct-w01-411630428:~$ dpkg -l | grep docker-ce
    ii  docker-ce                   5:29.3.0-1~ubuntu.24.04~noble  amd64  Docker: the open-source application container engine
    ii  docker-ce-cli               5:29.3.0-1~ubuntu.24.04~noble  amd64  Docker CLI: the open-source application container engine
    ii  docker-ce-rootless-extras   5:29.3.0-1~ubuntu.24.04~noble  amd64  Rootless support for Docker.
    ```

- [x] ③ Daemon：`sudo systemctl status docker` 顯示 active     
    **輸出內容** 
    ```bash
    hung@vct-w01-411630428:~$  sudo systemctl status docker
    ● docker.service - Docker Application Container Engine
        Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: e>
        Active: active (running) since Sun 2026-03-22 16:54:01 CST; 24min ago
    TriggeredBy: ● docker.socket
        Docs: https://docs.docker.com
    Main PID: 1647 (dockerd)
        Tasks: 10
        Memory: 132.5M (peak: 133.4M)
            CPU: 966ms
        CGroup: /system.slice/docker.service
                └─1647 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/cont>
    ```
- [x] ④ 端到端：`sudo docker run hello-world` 成功輸出   
    **輸出內容**
    ```bash
    hung@vct-w01-411630428:~$ sudo docker run hello-world

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
    1. The Docker client contacted the Docker daemon.
    2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
        (amd64)
    3. The Docker daemon created a new container from that image which runs the
        executable that produces the output you are currently reading.
    4. The Docker daemon streamed that output to the Docker client, which sent it
        to your terminal.

    To try something more ambitious, you can run an Ubuntu container with:
    $ docker run -it ubuntu bash

    Share images, automate workflows, and more with a free Docker ID:
    https://hub.docker.com/

    For more examples and ideas, visit:
    https://docs.docker.com/get-started/
    ```
- [x] Compose：`docker compose version` 可執行  
    **輸出內容**
    ```bash
    hung@vct-w01-411630428:~$ docker compose version
    Docker Compose version v5.1.1
    ```

    **輸出內容**
    ```bash
    ```

## 容器操作紀錄
- [x] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出  
    **輸出內容**
    ```bash
    hung@vct-w01-411630428:~$ sudo docker run -d -p 8080:80 nginx
    ef1cc753a13a82ab9c8c49e6c0c4a3478423f3e6459cac6a0ba051d040002507
    hung@vct-w01-411630428:~$ curl localhost:8080
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, nginx is successfully installed and working.
    Further configuration is required for the web server, reverse proxy, 
    API gateway, load balancer, content cache, or other features.</p>

    <p>For online documentation and support please refer to
    <a href="https://nginx.org/">nginx.org</a>.<br/>
    To engage with the community please visit
    <a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
    For enterprise grade support, professional services, additional 
    security features and capabilities please refer to
    <a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    ```
- [x] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出  
    **輸出內容**
    ```bash
    hung@vct-w01-411630428:~$ sudo docker run -it --rm alpine /bin/sh
    / # hostname
    44ce115309ff
    / # cat /etc/os-release
    NAME="Alpine Linux"
    ID=alpine
    VERSION_ID=3.23.3
    PRETTY_NAME="Alpine Linux v3.23"
    HOME_URL="https://alpinelinux.org/"
    BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
    / # ls /
    bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
    / # whoami
    root
    / # exit
    ```
- [x] 映像列表：`sudo docker images` 輸出  
    **輸出內容**
    ```bash
    hung@vct-w01-411630428:~$ sudo docker images
                                                                                                        i Info →   U  In Use
    IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
    alpine:latest        25109184c71b       13.1MB         3.95MB        
    hello-world:latest   85404b3c5395       25.9kB         9.52kB        
    nginx:latest         dec7a90bd097        240MB         65.8MB    U   
    ```
    
## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
| :--- | :--- | :--- | :--- |
| clean-baseline | 2026-03-22 16:22 | Ubuntu 系統初裝完成，尚未安裝 Docker。 | 執行 `hostnamectl` 與 `ping google.com` 確認系統名稱與網路連線正常。 |
| docker-ready | 2026-03-22 17:04 | Docker Engine 安裝完畢且通過四層驗證後的穩定狀態。 | 通過 `hello-world` 執行、`nginx` 容器啟動測試及映像檔拉取驗證。 |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
| --- |---|---|---|
| docker.list 存在 | 是 | 否 | 是 |
| apt-cache policy 有候選版本 | 是 | 否 | 是 |
| docker 重裝可行 | 是 | 否 | 是 |
| hello-world 成功 | 是 | N/A | 是 |
| nginx curl 成功 | 是 | N/A | 是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
| :--- | :--- | :--- |
| **所需時間** | 約 2-5 分鐘 (需查找指令、移動檔案或重新設定) | 約 30-60 秒 (僅需等待虛擬機狀態切換) |
| **適用情境** | 確切知道哪一行設定改錯，且變動範圍極小時。 | 系統大範圍損壞、中毒、或不明原因導致服務無法啟動。 |
| **風險** | 人為輸入錯誤可能導致二次故障，修復後環境可能不純淨。 | 遺失從「Snapshot 建立點」到「現在」之間的所有未存檔資料。 |

## Snapshot 保留策略
- **新增條件：** 每次完成重大軟體安裝（如 Docker, Kubernetes）或進行具有破壞性的實驗前。
- **保留上限：** 最多保留 3 個活躍 Snapshot，避免差異磁碟鏈（Delta Disk）過長導致磁碟 I/O 效能下降。
- **刪除條件：** 當新節點已通過 48 小時穩定測試，且確認舊節點的環境配置已過時不再需要時。

## 最小可重現命令鏈
```bash
# 1. 注入故障 (將 source list 移走)
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update

# 2. 驗證故障 (會發現找不到 docker-ce 套件)
apt-cache policy docker-ce

# 3. 執行回復 (透過 VMware 介面回復至 docker-ready 點)

# 4. 回復後驗證
ls /etc/apt/sources.list.d/docker.list
sudo docker run --rm hello-world
```

## 排錯紀錄
- 症狀：執行 `sudo apt update` 時出現 `GPG error` 或是 `NO_PUBKEY` 錯誤。
- 診斷：檢查 `/etc/apt/keyrings/` 是否存在 `docker.gpg` 檔案，並確認權限是否為 `644`。
- 修正：重新執行 `curl` 下載 `GPG Key` 並使用 `chmod a+r` 修正讀取權限。
- 驗證：再次執行 `sudo apt update`，確認輸出中包含 `download.docker.com` 且無報錯。

## 設計決策
**選擇「VM 內跑 Docker」而非直接在 Host OS 執行**：
雖然在 Windows 直接安裝 Docker Desktop 較方便，但考慮到後續課程環境需一致（消除 Windows 與 Mac 使用者的系統差異），且 Docker Desktop 的行為在某些進階網路設定上與 Linux 原生 Docker 不同，故選擇在 VMware Ubuntu VM 內建立純淨 Linux 環境，以達到最高的技術可控性。