# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlayfs
- Cgroup Version：2
- Cgroup Driver：systemd
- Default Runtime：runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：隔離 Process ID 空間，讓容器擁有獨立的 PID 序列（如容器內的 PID 1），且看不到 Host 或其他容器的 Process。
- NET：隔離網路資源，包括網路卡、IP 地址、路由表、防火牆規則等，讓每個容器有獨立的網路棧。
- MNT：隔離掛載點，讓容器擁有獨立的檔案系統視圖（如自己的 / 根目錄），與 Host 的掛載點互不干擾。
- UTS：隔離主機名稱（Hostname）與域名（Domain Name），讓容器可以設定自己的名稱而不影響 Host。
- IPC：隔離進程間通訊（Inter-Process Communication）資源，如共享記憶體、信號量等，防止不同容器間的通訊干擾。
- USER：隔離使用者與群組 ID，允許容器內的 root 使用者對應到 Host 上的非 root 使用者，增強安全性（本環境預設未開啟）。

### Host vs 容器 inode 對照
（參考 `namespace-table.md`）

| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|---|---|---|---|
| pid | 4026531836 | 4026532668 | No |
| net | 4026531833 | 4026532670 | No |
| mnt | 4026531832 | 4026532665 | No |
| uts | 4026531838 | 4026532666 | No |
| ipc | 4026531839 | 4026532667 | No |
| user | 4026531837 | 4026531837 | Yes |

### 容器內 `ps aux` 輸出
```
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 3600
   13 root      0:00 ps aux
```
（只看到兩支 process，因為 PID Namespace 隔離了視角，sleep 3600 成為了容器內的 PID 1。）

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456 (256MB)
- cpu.max：50000 100000 (0.5 CPU)

### Host 端對照（路徑：`/sys/fs/cgroup/system.slice/docker-<ID>.scope/`）
- memory.max：268435456
- cpu.max：50000 100000
- memory.current（執行時某一刻）：409600

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | (實驗確認為 OOM Kill) | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
8 層

### 兩個同源 image 共享 layer 的證據
雖然 `inspect` 的 `sha256` 雜湊值可能因版本差異未完全重疊，但在 `docker pull` 過程中可觀察到多個 layer 顯示 "Already exists" 或快速跳過，證明底層結構被共享。

### `docker diff` 輸出範例與解讀
```
C /etc
C /etc/nginx
C /etc/nginx/conf.d
A /etc/nginx/conf.d/custom.conf
D /etc/nginx/conf.d/default.conf
C /tmp
A /tmp/hello.txt
```
- **A (Added)**: 新增檔案（如 `/tmp/hello.txt`）。
- **C (Changed)**: 目錄或檔案被修改（如 `/etc/nginx` 目錄下的變更）。
- **D (Deleted)**: 檔案被刪除（如 `default.conf`）。
這些變更僅存在於容器的可寫層（upperdir），不影響底層唯讀映像。

## OCI 呼叫鏈

- **dockerd**: 提供高層 API，處理使用者請求、網路與 Volume 管理。
- **containerd**: 負責映像管理與容器生命週期，將 dockerd 的指令轉化為底層操作。
- **containerd-shim**: 每個容器獨立的 Process，負責接管容器的 stdio 與 exit code，並在 runtime 退出後持續管理容器。
- **runc**: OCI Runtime 的具體實作，真正負責與 Kernel 溝通（clone, unshare, cgroup 等）來建立容器。

OCI Runtime Spec 的 `config.json` 檔案中，`linux.namespaces` 欄位定義了要開啟的 Namespace（如 pid, network），而 `linux.resources` 欄位則定義了 Cgroup 的各項限制（如 memory.max）。

## 排錯紀錄
- 症狀：執行 `docker run` 時出現 `dial tcp: lookup registry-1.docker.io: i/o timeout`。
- 診斷：App VM 暫時性網路不穩或 DNS 解析失敗，導致無法從 Docker Hub 拉取映像。
- 修正：稍候片刻重新執行指令，或確認 Host 端的網路連接。
- 驗證：成功拉取 `alpine` 映像並啟動容器。

## 想一想（回答 3 題）
1. **容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？**
   不是。容器內的 PID 1 是我們啟動的應用（如 sleep），它在 Host 視角下是一個大數字 PID。如果在容器內 `kill -9 1`，該 Process 會終止，由於它是容器的 Init Process，容器也會隨之停止。

2. **兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？**
   共用底層 layer，僅各自的可寫層（upperdir）佔用額外空間。可透過 `docker system df` 觀察 Image 佔用，或查看 `/var/lib/docker/overlay2/` 下的實體檔案大小來驗證。

3. **如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？**
   如果 Kernel 爆漏洞，攻擊者可能透過漏洞逃逸到 Host 權限，因此隔離是不完整的。與 VM 的差異在於 VM 虛擬化了整個硬體並擁有獨立的 Kernel，其攻擊面較小（僅 Hypervisor）；而容器直接共用 Host Kernel，安全性高度依賴 Kernel 的補丁與配置。
