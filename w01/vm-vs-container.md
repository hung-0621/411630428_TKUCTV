# W01｜虛擬化與容器化技術分析

## 一、 VM 與 Container 多維度對照

根據虛擬化層級與資源管理機制，兩者差異如下表所示：

| 維度 | 虛擬機器 (VM) | 容器 (Container) |
| :--- | :--- | :--- |
| **核心技術** | Hypervisor (如 VMware, KVM) | Container Engine (如 Docker) |
| **隔離層級** | 硬體級隔離，擁有獨立 Guest OS | 作業系統級隔離，共用 Host Kernel |
| **資源佔用** | 較重，需預留 OS 運行空間 | 較輕，僅包含 App 與相依項 |
| **啟動速度** | 較慢，需經過完整系統開機程序 | 極快，僅需啟動應用程序 |
| **回復方式** | 透過 Snapshot回復磁碟與記憶體狀態 | 透過重新拉取映像檔部署 |

## 二、 本課選擇「VM 內跑 Docker」的理由

在實作環境中，主要基於以下考量：

1. **環境標準化與一致性：** 由於每個人的筆電 Host OS (Windows 或 macOS) 版本不一，透過 VM 可以統一底層作業系統為 Ubuntu 24.04，確保實驗結果具備可重現性，避免環境差異問題。
2. **具備安全防護網：** VM 提供的 Snapshot 機制讓我們在練習 Docker 設定或 Linux 系統配置不小心改壞時，可以還原到健康狀態，而不會影響到 Host OS 的穩定性。
3. **消除平台行為差異：** macOS 的 Docker Desktop 實際上是跑在輕量化 Linux 虛擬層上，其行為與原生 Linux 上的 Docker 有細微不同。在 VM 內操作能學習到最標準、原生的 Linux Docker 運維技能。

## 三、 Hypervisor 類型差異與本課選擇

Hypervisor 根據其安裝位置與運作機制分為兩類：

### 1. Type 1 (Bare-metal Hypervisor)
- **運作方式：** 直接安裝在實體硬體上，不需預裝作業系統。
- **範例：** VMware ESXi, Microsoft Hyper-V Server。
- **優點：** 效能高、延遲低，適合企業資料中心。

### 2. Type 2 (Hosted Hypervisor)
- **運作方式：** 安裝在現有的作業系統 (Host OS) 之上，如同一般應用程式。
- **範例：** VMware Workstation Pro, VirtualBox。
- **優點：** 安裝方便，硬體相容性高，適合個人開發與教學場景。

### 本課選擇：Type 2 (VMware Workstation Pro)
我們選擇 **Type 2** 的理由是因為它最符合學習情境。學生可以在不更動現有電腦系統的前提下，利用筆電快速建立實驗環境。雖然效能略低於 Type 1，但其提供的 Snapshot 管理功能對於教學環境的「一致性」與「可回復性」而言，是性價比最高的選擇。