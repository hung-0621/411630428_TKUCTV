# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統級設定檔 | daemon.json（Docker daemon 設定） |
| /var/lib/docker/ | 程式的持久性狀態資料 | 映像、容器、volumes |
| /usr/bin/docker | 使用者可執行檔 | Docker CLI 工具 |
| /run/docker.sock | 執行期暫存（PID/socket） | Docker daemon 的 Unix socket |

## Docker 系統資訊

- Storage Driver：
    ```
    Storage Driver: overlayfs
    ```
- Docker Root Dir：
    ```
    Docker Root Dir: /var/lib/docker
    ```
- 拉取映像前 /var/lib/docker/ 大小：
    ```bash
    hung@bastion:~$ sudo du -sh /var/lib/docker/
    276K	/var/lib/docker/
    ```
- 拉取映像後 /var/lib/docker/ 大小：
    ```bash
    hung@bastion:~$ sudo docker pull nginx:latest
    latest: Pulling from library/nginx
    5435b2dcdf5c: Pull complete 
    054715a6bffa: Pull complete 
    448ea5cac5d5: Pull complete 
    88d1d984b765: Pull complete 
    84e114c2bb36: Pull complete 
    7b5d674621c2: Pull complete 
    4a038fd18db1: Pull complete 
    3eff0a97d435: Download complete 
    cc0cf959117b: Download complete 
    Digest: sha256:7f0adca1fc6c29c8dc49a2e90037a10ba20dc266baaed0988e9fb4d0d8b85ba0
    Status: Downloaded newer image for nginx:latest
    docker.io/library/nginx:latest
    hung@bastion:~$ sudo du -sh /var/lib/docker/
    280K	/var/lib/docker/
    ```

## 權限結構

### Docker Socket 權限解讀
（貼上 `ls -la /var/run/docker.sock` 輸出，逐欄說明 owner/group/others 的權限）  
```
hung@bastion:~$ ls -la /var/run/docker.sock
srw-rw---- 1 root docker 0 Apr 19 22:52 /var/run/docker.sock
```
- s：檔案類型是 socket（不是普通檔案 -，也不是目錄 d）。
- rw-：owner（root）有讀寫權限。
- rw-：group（docker）有讀寫權限。
- ---：others 完全沒有權限。

### 使用者群組
```
hung@bastion:~$ id
uid=1000(hung) gid=1000(hung) groups=1000(hung),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)
```
- 不包含 docker 群組

### 安全意涵
- docker group 的使用者可以靠容器讀取 Host 的敏感檔案，在生產環境中，不應該隨意把使用者加入 docker group。

## 程序與服務管理

### systemctl status docker
```bash
hung@bastion:~$ systemctl status docker
Warning: The unit file, source configuration file or drop-ins of docker.service changed on disk. R>
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-04-19 22:52:18 CST; 50min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1700 (dockerd)
      Tasks: 10
     Memory: 136.2M (peak: 150.6M)
        CPU: 3.364s
     CGroup: /system.slice/docker.service
             └─1700 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

```

### journalctl 日誌分析
（貼上 `journalctl -u docker --since "1 hour ago"` 的重點摘錄，說明看到什麼事件）
```
hung@bastion:~$ journalctl -u docker --since "1 hour ago"
Apr 19 22:52:17 bastion systemd[1]: Starting docker.service - Docker Application Container Engine...
Apr 19 22:52:18 bastion dockerd[1700]: time="2026-04-19T22:52:18... level=info msg="Starting up"
Apr 19 22:52:18 bastion systemd[1]: Started docker.service - Docker Application Container Engine.
Apr 19 23:35:24 bastion dockerd[1700]: time="2026-04-19T23:35:24... level=info msg="..."
```
1. 服務啟動流程：在 22:52:17 時，由 systemd 發起啟動指令（Starting），隨即 Docker Daemon（dockerd，PID 為 1700）開始初始化並印出多筆載入資訊。

2. 啟動成功確認：在 22:52:18 時，出現了 Started docker.service，代表 Daemon 已成功進入運行狀態。

3. 穩定運行：後續在 23:35:24 仍有 level=info 的日誌紀錄，代表 Daemon 在這段期間內運作正常，沒有發生無預警的崩潰或重啟。

### CLI vs Daemon 差異
- 說明差異：Docker CLI (/usr/bin/docker) 是一個短暫執行的指令公用程式，負責接收使用者的輸入；而 Docker Daemon (dockerd) 是一個由 systemd 管理的長期背景服務，負責實際處理容器的生命週期。
- 兩者關係：CLI 必須透過 Unix Socket (/run/docker.sock) 發送 API 請求給 Daemon 才能執行功能。
- 為什麼 docker --version 正常不代表 Docker 能用：

  - 執行 --version 時，CLI 只是單純印出自己的檔案版本資訊，這個過程不需要跟 Daemon 通訊。

  - 即使 Daemon 服務掛掉（Inactive）或 Socket 權限錯誤，docker --version 依然會回傳結果。

  - 必須執行 docker ps 或 docker info 這類需要「通訊」的指令，才能確認後端的 Daemon 服務是否真的活著且可存取。

## 環境變數

- $PATH：
    ```bash
    hung@bastion:~$ echo $PATH
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin
    ```
- which docker：
    ```bash
    hung@bastion:~$ which docker
    /usr/bin/docker
    ```
- 容器內外環境變數差異觀察：
  - Host
    ```
    hung@bastion:~$ echo "--- Host ---"
    env | grep -E "^(PATH|HOME|USER|SHELL)="
    --- Host ---
    SHELL=/bin/bash
    HOME=/home/hung
    USER=hung
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin
    ```
  - Container
    ```
    hung@bastion:~$ echo "--- Container ---"
    docker run --rm alpine env
    --- Container ---
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    HOSTNAME=eb257c0ad087
    HOME=/root
    ```
    - 容器內有自己獨立的環境變數，\$HOME、\$PATH 都和 Host 不同。容器是隔離的執行環境，不會繼承 Host 的 shell 設定。

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| **systemctl status docker** | `active (running)` | `inactive (dead)` | `active (running)` |
| **docker --version** | 正常回傳版本 | 正常回傳版本 | 正常回傳版本 |
| **docker ps** | 正常執行 | `Cannot connect to the Docker daemon` | 正常執行 |
| **ps aux \| grep dockerd** | 有 dockerd 程序 | 無相關程序 | 有 dockerd 程序 |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| **ls -la docker.sock 權限** | `srw-rw----` | `srw-------` | `srw-rw----` |
| **docker ps（不加 sudo）** | 正常執行 | `permission denied` | 正常執行 |
| **sudo docker ps** | 正常執行 | 正常執行 (root 有權限) | 正常執行 |
| **systemctl status docker** | `active (running)` | `active (running)` | `active (running)` |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| **Cannot connect to the Docker daemon** | Docker Daemon 服務未啟動 | 使用 `systemctl status docker` 檢查背景服務 |
| **permission denied...docker.sock** | 使用者無權存取 Unix Socket | 檢查 `ls -la /run/docker.sock` 與 `id` 群組 |

### 差異說明
* **Cannot connect** 代表「服務端沒響應」：通常是整個 Docker 背景服務 (Daemon) 被關閉或崩潰，導致請求無法送達。
* **Permission denied** 代表「身分不符」：服務正在運作，但當前的 Linux 使用者權限不足以讀取或寫入 `/run/docker.sock`。

## 排錯紀錄
* **症狀**：輸入 `docker ps` 時出現 `permission denied while trying to connect to the Docker daemon socket`。
* **診斷**：
    1.  首先執行 `systemctl status docker` 確認服務為 `active`。
    2.  檢查 `ls -la /run/docker.sock` 發現群組權限為 `docker`。
    3.  執行 `id` 發現當前使用者不在 `docker` 群組中。
* **修正**：執行 `sudo usermod -aG docker $USER` 並重新登入 SSH 以更新 Session。
* **驗證**：再次執行 `id` 確認群組已更新，且 `docker ps` 不加 `sudo` 也能正常回傳。

## 設計決策
* **技術選擇**：在開發環境中，選擇將使用者加入 `docker` 群組而非每次使用 `sudo`。
* **取捨與風險**：
    * **取捨**：這提高了開發效率，避免頻繁輸入密碼，並方便自動化腳本執行。
    * **風險**：由於 Docker Daemon 以 root 權限執行，加入 `docker` 群組的使用者可以透過掛載磁碟等方式輕易取得主機的 root 權限，因此在生產環境中應嚴格控管此群組。