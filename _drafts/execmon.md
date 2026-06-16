---
title: "我寫了一支超小的 eBPF 工具，能即時看到主機上跑起來的每一支程式（還會幫你算風險分數）"
description: "一支大約 60 行的 eBPF 工具，即時追蹤主機上的每一次 exec()，並把可疑的那幾筆標出來。"
date: 2026-06-16 09:00:00 +0800
categories: [eBPF, 安全]
tags: [ebpf, security, linux, observability]
render_with_liquid: false
---

> 🖼️ **【需補：封面圖】** 建議直接放一張「工具實際跑起來的 JSON 風險輸出」截圖——`risk_level: HIGH` 配上 `reasons` 那種畫面，一眼就懂工具在幹嘛。dev.to 封面很顯眼，有圖點閱率明顯較高。

> **TL;DR：** 在 **execve** 上掛一個 **tracepoint**，再配一條 **ring buffer**，你就能即時看到一台主機上每一支被啟動的程式——而且可以幫每一筆算出風險分數。整支工具大約 60 行，這篇就帶你看我怎麼做出來的。

## 問題：你真的知道你的主機上正在跑什麼嗎？

傳統監控告訴你的，是 CPU、記憶體、磁碟 I/O——這些「狀態」類的數字。但它幾乎不會告訴你一件更要命的事：**剛剛是哪一支程式被啟動、用什麼參數、從哪個路徑跑起來的。**

✏️ **【需你補：痛點開場】** 用你自己的話把這段痛點講具體一點。可以帶到的角度：傳統 ERP／工廠主機常年累積一堆來路不明的排程與腳本、稽核時根本說不清「這台機器上到底跑過什麼」、安全事件發生後想回溯卻沒有 process 層級的紀錄。最好能放一個你實際遇過、會讓讀者點頭的小故事。

對現代 eBPF 來說，這其實是個 CP 值很高的切入點：不用改任何應用程式、不用裝侵入式的 agent，只要在 kernel 攔一個 syscall，主機上的程式啟動就全都看得見了。

## 你會學到

- 怎麼用 tracepoint 攔 `execve`，把 pid、ppid、cmdline 一起抓出來
- 怎麼用 ring buffer 把事件即時、低延遲地送到 user space
- 怎麼加一層簡單的風險評分（白名單／高風險工具／可疑路徑），把噪音變成有用的訊號

## 核心構想：掛上 execve，順手算風險

每當 Linux 要執行一支新程式，最後都會走到 `execve` 這個 syscall。換句話說，**只要攔得住 `execve`，主機上任何一次「啟動程式」的動作都逃不掉**——不管它是使用者手動下的指令、cron 排的工作，還是某支腳本偷偷 fork 出來的子程序。

我們的做法分成兩半：

- **kernel 端**：在 `tracepoint:syscalls:sys_enter_execve` 掛一個 eBPF 程式，把 pid、ppid、執行檔路徑、cmdline 這些欄位包成一筆 event，丟進 ring buffer。
- **user 端**：從 ring buffer 把 event 讀出來，套上一組簡單的規則算出風險等級，再印成方便機器處理的 JSON。

✏️ **【需你補：流程圖】** 建議放一張「kernel → user space」的事件流程小圖：`execve syscall → tracepoint → eBPF prog → ring buffer → user space reader → 風險評分 → JSON 輸出`。或直接匯出你投影片裡那張圖。

## kernel 端（最關鍵的部分）

✏️ **【需你補：kernel 端關鍵片段】** 貼上你 `tracepoint/sys_enter_execve` 裡填 `exec_event`、再 `bpf_ringbuf_*` 送出去的關鍵片段，控制在 20 行以內。重點讓讀者看到：怎麼拿 pid/ppid、怎麼用 `bpf_probe_read_user_str` 讀 filename、怎麼把 event 推進 ring buffer。

```c
// 範例骨架（請換成你 repo 裡的實際程式碼）
SEC("tracepoint/syscalls/sys_enter_execve")
int handle_execve(struct trace_event_raw_sys_enter *ctx)
{
    struct exec_event *e;

    // 從 ring buffer 預留一塊空間
    e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
    if (!e)
        return 0;

    e->pid = bpf_get_current_pid_tgid() >> 32;
    bpf_get_current_comm(&e->comm, sizeof(e->comm));
    bpf_probe_read_user_str(&e->filename, sizeof(e->filename),
                            (const char *)ctx->args[0]);

    bpf_ringbuf_submit(e, 0);
    return 0;
}
```

## user 端：把事件變成風險

user 端做的事很單純：開一個 ring buffer 的 reader、註冊一個 callback，每收到一筆 event 就跑一次風險評分，然後輸出。

風險評分不需要做得多複雜，光是幾條規則就很有用：

- **白名單**：常見、可信的執行檔（`/usr/bin` 底下那些日常工具）直接判為低風險，先把噪音壓下來。
- **高風險工具**：像 `nc`、`curl`、`wget`、`nmap`、各種 shell 這類「攻擊者很愛用」的工具，提高分數。
- **可疑路徑**：從 `/tmp`、`/dev/shm`、使用者家目錄這種「正常程式不該待著」的地方執行，加重風險。

✏️ **【需你補：user 端說明】** 用一兩段把你實際的 reader 怎麼讀 ring buffer、規則怎麼組合成 `risk_level` 講清楚（完整程式碼放 GitHub，這裡只要講概念）。如果你的評分是用「分數累加 → 對應到 LOW / MEDIUM / HIGH」的方式，把那個對應關係也帶一句。

## 實際輸出長這樣

✏️ **【需你補：實際輸出】** 這裡是整篇的主秀。下面是**示意範例**，請換成工具在你機器上跑出來的**真實 JSON 輸出**——至少要有一筆 `risk_level: HIGH`，並附上 `reasons` 說明為什麼被判為高風險。讓讀者看到「真的會動、真的抓得到東西」，可信度大增。

```json
{
  "timestamp": "2025-08-21 09:04:30",
  "pid": 2897205,
  "ppid": 2690127,
  "comm": "code",
  "filename": "/usr/local/bin/docker",
  "args": "docker context ls --format {{json .}}",
  "risk_level": "HIGH",
  "reasons": ["high-risk-tool", "suspicious-location"]
}
```

👉 完整、可直接跑的工具：🔗 **【需補：GitHub 連結】** 把這裡換成你 push 上去的 repo 網址。

## 什麼時候該用它（以及它的極限）

這支工具最適合的場景，是**輕量的主機可見性與稽核**：你想知道一台機器上到底跑過哪些程式、什麼時候跑的、誰啟動的——又不想為此扛一套又重又貴的商用方案。對傳統產業那種「跑了十幾年、沒人說得清裡面有什麼」的主機，這種 process 層級的紀錄特別值錢。

但也要把話說在前面，它**不是**一套完整的 EDR：

- 它只看 `execve`，看不到網路連線、檔案讀寫、權限提升那些面向。
- 風險評分是規則式的，會有漏網之魚、也會有誤報；它的定位是「縮小你要盯的範圍」，不是「自動幫你下結論」。
- 它應該是你縱深防禦裡的**其中一層**，要搭配其他防線一起用，而不是拿它取代誰。

✏️ **【需你補：適用情境】** 補上你自己心中最想推薦的使用場景，以及你實際踩過、覺得讀者該先知道的限制。

## 重點整理

- **execve tracepoint 是主機可見性的高 CP 值起點**——一個攔截點，就把所有程式啟動全收進來。
- **ring buffer 是 kernel → user space 的現代正解**：低延遲、不丟事件，比舊的 perf buffer 好用很多。
- **風險評分可以很簡單就很有用**：白名單 + 高風險工具 + 可疑路徑這幾條規則，就能把一堆雜訊濃縮成幾筆值得看的告警。

## 延伸資源

> 🔗 **【需補：GitHub 連結】** 把下面換成你 push 上去的 repo 網址。

- GitHub：**github.com/hyperredstart/articles-ebpf**
- 投影片：✏️ **【需你補：投影片連結】**（你 KubeSummit 2025 那場的投影片連結）
- 參考資料：Liz Rice，《Learning eBPF》（O'Reilly）

---

*在打造主機可見性的工具嗎？我很想知道你都追蹤什麼、為什麼追蹤——留言聊聊。*

*本文是「eBPF 用於傳統產業／雲原生」系列的一篇。喜歡的話追蹤一下，後面還有更多。*
