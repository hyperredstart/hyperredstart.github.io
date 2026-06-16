---
title: "在 Mac 上跑 eBPF：用 Multipass 或 Lima，5 分鐘開好 Linux 開發環境"
description: "eBPF 只能跑在 Linux kernel，但你用的是 Mac。這篇給你一個乾淨、可重複的做法：開一台真正的 Linux VM（含 CO-RE 需要的 BTF），附一鍵安裝腳本。也順便說清楚為什麼別用 Docker Desktop。"
date: 2026-06-16 11:00:00 +0800
categories: [eBPF, 環境]
tags: [ebpf, macos, linux, devops]
render_with_liquid: false
---

## 前言：這個系列，始於 KubeSummit 2025 一場爆滿的演講

2025 年 10 月，我在 **iThome 主辦的 KubeSummit 2025** 上分享了一場演講——主題是《探索 eBPF 技術核心：運用 eBPF 與 Kubernetes 加速傳統產業 ERP 系統現代化監控》。當天現場坐滿了人，而事後最常被問到的，其實不是理論，而是各種版本的同一句話：

> 「講得很棒——但我自己到底要怎麼把這些跑起來？」

這個系列就是為了回答它：把那場演講，變成一篇篇你能在自己機器上**實際跑起來**的動手做指南。而第一道牆，每個 Mac 使用者都會撞上——**eBPF 不能跑在 macOS 上**。所以在進到 kernel tracing 與效能實測之前，先解決第零步：弄一個 eBPF 真的能動的 Linux 環境。

> **TL;DR：** eBPF 跑在 **Linux kernel** 裡，無法原生跑在 macOS。用 **Multipass**（最簡單）或 **Lima**（會自動掛載你的程式碼）開一台真正的 Ubuntu VM——兩者都附帶現代 CO-RE eBPF 需要的 **BTF**。**別用 Docker Desktop**：它的 LinuxKit kernel 通常沒有 BTF。文末有一鍵安裝腳本。

## 問題：你的 Mac 不是 Linux（而 eBPF 只說 Linux）

eBPF 是 **Linux kernel** 的技術。macOS 跑的是 XNU kernel——那裡沒有 eBPF。所以每一次「我來學 eBPF」的嘗試，在 Mac 上第一天就撞牆。

好消息是：在 Mac 上跑 Linux 玩 eBPF，是個已被解決的、5 分鐘的問題——**前提是你選對工具**。陷阱是反射性地去開 Docker Desktop（下面說為什麼）。

## 你會學到

- 為什麼你需要一台 Linux VM（以及為什麼 Docker Desktop 不適合跑 eBPF）
- Multipass vs Lima——該選哪個
- 一行指令的腳本，幫你開好一台「驗證過、能動」的 eBPF 開發 VM

## 一條能省下你好幾小時的鐵則：你需要 BTF

現代 eBPF（CO-RE，*compile once, run everywhere*）會讀 kernel 的 **BTF**（`/sys/kernel/btf/vmlinux`）來跨 kernel 版本保持可攜。如果你的 VM kernel 沒編進 BTF，第一步——產生 `vmlinux.h`——就會直接失敗。

- **Ubuntu 22.04 / 24.04 的 VM** 預設就有 BTF。✅
- **Docker Desktop 的 LinuxKit kernel** 常常**沒有**。❌——這就是「我在容器裡試 eBPF，結果 `vmlinux.h` 失敗」這麼常見的原因。

所以：要一台真正的 Ubuntu VM，不是容器。

## Multipass vs Lima——挑一個

| | **Multipass** | **Lima** |
|---|---|---|
| 安裝 | `brew install --cask multipass` | `brew install lima` |
| 風格 | 超級簡單，「給我一台 VM 就對了」 | 輕量、對開發者友善 |
| 把程式碼帶進 VM | `multipass mount` / `transfer` | **自動掛載你的家目錄** |
| 最適合 | 最快上手 | 在 Mac 編輯、在 VM 編譯的循環 |

兩者都很好用，也都有 BTF。**新手？用 Multipass。** 想讓檔案「直接就在 VM 裡」？用 Lima。

## 方案 A — Multipass（最快）

```bash
brew install --cask multipass
multipass launch 24.04 --name ebpf-dev --cpus 2 --memory 4G --disk 20G
multipass shell ebpf-dev
```

進到 VM 裡：

```bash
sudo apt update
sudo apt install -y clang llvm libbpf-dev libelf-dev zlib1g-dev \
                    make bpftrace git linux-tools-common linux-tools-$(uname -r)
```

把你的專案資料夾分享進去（在 Mac 編輯、在 VM 編譯）：

```bash
# 回到 Mac：
multipass mount "$PWD" ebpf-dev:/home/ubuntu/work
```

## 方案 B — Lima（自動掛載你的程式碼）

```bash
brew install lima
limactl start --name=ebpf-dev template://ubuntu-lts
limactl shell ebpf-dev
```

Lima 會把你的家目錄掛到 VM 裡的同一個路徑，所以你可以直接 `cd` 到你的專案——不用複製。工具鏈裝法同上。

## 或者，直接跑一個腳本

把上面兩種方式都包起來，加上 BTF 檢查與一個 eBPF smoke test：

```bash
#!/usr/bin/env bash
# setup-ebpf-mac.sh — ./setup-ebpf-mac.sh [multipass|lima]   (預設 multipass)
set -euo pipefail
PROVIDER="${1:-multipass}"; VM_NAME="ebpf-dev"

read -r -d '' VM_SETUP <<'EOF' || true
set -e
echo "[VM] kernel: $(uname -r)  arch: $(uname -m)"
[ -r /sys/kernel/btf/vmlinux ] && echo "[VM] BTF: OK" || { echo "[VM] BTF: MISSING"; exit 1; }
sudo apt-get update -y
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  clang llvm libbpf-dev libelf-dev zlib1g-dev make bpftrace git linux-tools-common
sudo apt-get install -y "linux-tools-$(uname -r)" || sudo apt-get install -y linux-tools-generic || true
echo "[VM] smoke test (3s): tracing execve()"
sudo timeout 3 bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%-6d %-16s %s\n", pid, comm, str(args->filename)); }' || true
echo "[VM] ✅ ready"
EOF

if [ "$PROVIDER" = multipass ]; then
  command -v multipass >/dev/null || brew install --cask multipass
  multipass info "$VM_NAME" >/dev/null 2>&1 || \
    multipass launch 24.04 --name "$VM_NAME" --cpus 2 --memory 4G --disk 20G
  multipass exec "$VM_NAME" -- bash -c "$VM_SETUP"
  echo "進入：multipass shell $VM_NAME   |   分享程式碼：multipass mount \"$PWD\" $VM_NAME:/home/ubuntu/work"
elif [ "$PROVIDER" = lima ]; then
  command -v limactl >/dev/null || brew install lima
  limactl list --format '{{.Name}}' 2>/dev/null | grep -qx "$VM_NAME" || \
    limactl start --name="$VM_NAME" --tty=false template://ubuntu-lts
  limactl shell "$VM_NAME" bash -c "$VM_SETUP"
  echo "進入：limactl shell $VM_NAME   （Lima 會自動掛載你的家目錄）"
else
  echo "用法：multipass | lima"; exit 1
fi
```

執行：

```bash
chmod +x setup-ebpf-mac.sh
./setup-ebpf-mac.sh            # 或：./setup-ebpf-mac.sh lima
```

（含 fallback 與更友善輸出的完整版，在文末 repo。）

## 驗證它真的能動

在 VM 裡：

```bash
# 1) BTF 在不在？（CO-RE 需要）
ls -l /sys/kernel/btf/vmlinux

# 2) eBPF 能動嗎？即時追蹤每一個被啟動的程式：
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve {
    printf("%-6d %-16s %s\n", pid, comm, str(args->filename)); }'
```

另開一個 shell、隨便跑個 `ls`——它會立刻出現。那就是 eBPF 在你 VM 的 kernel 裡跑起來了。

接著編譯本系列那支真正的 C + libbpf 範例：

```bash
cd hello-ebpf
make
sudo ./hello
```

## 為什麼不用 Docker Desktop？

它很誘人（反正已經裝了），但拿來跑 eBPF 會反咬你：

- 它的 **LinuxKit kernel 常常沒有 BTF** → `bpftool btf dump ... > vmlinux.h` 失敗 → CO-RE 從第一步就死。
- 那顆 kernel 對應的 headers 與 `linux-tools` 也很難裝。

容器很適合**部署（ship）** eBPF，不適合在 Mac 上**學習／開發**它。用真正的 VM。

## 重點整理

- **eBPF 只能跑 Linux**——在 Mac 上要用 Linux VM，不是容器。
- **你需要 BTF。** Ubuntu 22.04/24.04 的 VM 有；Docker Desktop 通常沒有。
- **Multipass＝最快、Lima＝開發循環最順。** 任一個都能讓你 5 分鐘內開跑；一個腳本全包。

## 延伸資源

- 安裝腳本 + `hello-ebpf` 範例：**github.com/hyperredstart/articles-ebpf**
- Multipass 文件：<https://multipass.run>
- Lima：<https://lima-vm.io>

---

*本文是「eBPF 用於傳統產業／雲原生」系列的一篇。VM 備好後，下一篇我們用 C + libbpf 追蹤真實的 syscall。*

*在 Mac 上卡在 eBPF 環境設定？留言告訴我卡在哪，我幫你看。*
