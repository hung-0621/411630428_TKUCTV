# W02 虛擬網路拓樸圖 (Network Topology)

本圖表描述了 dev-a 與 server-b 的網路配置，包含網段劃分、網卡模式及流量方向。

## 1. 邏輯拓樸圖 (Mermaid)

```mermaid
flowchart TD
    %% 定義外部網路
    subgraph Internet ["【 外部網路 (Internet) 】"]
        INET[Google / External Services]
    end

    %% 定義 VMware 虛擬環境
    subgraph VMware_Net ["【 VMware 虛擬網路環境 】"]
        
        %% 區域標記：NAT 網段
        subgraph Net_NAT ["VMnet8 (NAT 網段: 192.168.78.0/24)"]
            direction LR
        end

        %% 區域標記：Host-only 網段
        subgraph Net_HO ["VMnet1 (Host-only 網段: 192.168.174.0/24)"]
            direction LR
        end

        %% VM: dev-a 配置 (角色：開發機/跳板)
        subgraph VM_dev_a ["dev-a (開發機 / 管理者)"]
            eth0_NAT["ens33 (NAT 模式)<br/>IP: 192.168.78.138"]
            eth1_HO["ens37 (Host-only 模式)<br/>IP: 192.168.174.130"]
        end

        %% VM: server-b 配置 (角色：受控伺服器)
        subgraph VM_server_b ["server-b (受控後端 / 隔離)"]
            eth2_HO["ens33 (Host-only 模式)<br/>IP: 192.168.174.129"]
        end

        %% 虛擬交換器連線
        vSwitch8{{VMnet8 NAT Switch}}
        vSwitch1{{VMnet1 Host-only Switch}}
    end

    %% --- 流量路徑與方向 (Traffic Direction) ---
    
    %% 外網流量路徑 (雙向)
    eth0_NAT <--> vSwitch8 <--> INET
    
    %% 內網管理流量路徑 (dev-a 管理 server-b)
    eth1_HO -- "SSH/Ping 流量" --> vSwitch1 --> eth2_HO

    %% 樣式設定
    style VM_dev_a fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style VM_server_b fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style Internet fill:#f1f8e9,stroke:#33691e