# MG 活動抽獎網頁 — 使用說明

這份文件包含兩個工具的使用方式：

1. **抽獎網頁**（`index.html`，部署在 GitHub Pages 上）
2. **Facebook 留言名單擷取工具**（瀏覽器主控台小程式，用來把留言者姓名抓出來貼進抽獎網頁）

---

## 一、抽獎網頁

### 1. 網頁位置

部署在 GitHub Pages，網址格式為：

```
https://mjmotorty.github.io/MG_lottery/
```

若要更新網頁內容，把新的 `index.html` 上傳覆蓋到 GitHub repo 即可（GitHub 網頁上「Add file → Upload files」直接拖檔案上傳、選擇覆蓋現有檔案、最後按 Commit changes）。部署通常在數十秒內生效，若沒看到變化，先硬性重新整理瀏覽器（`Ctrl + Shift + R`）。

### 2. 功能總覽

- **中壢廠 / 八德廠** 兩個分頁，各自獨立管理名單與抽獎結果
- **名單管理**：貼上人員名單（一行一位，也支援逗號分隔），可容納數百筆
- **獎項設定**：
  - 獎項名稱可直接點選編輯
  - 每個獎項可設定「每廠抽出幾人」（例如保溫隨行杯要抽 10 位，其他獎項各抽 1 位）
  - 可以「＋新增獎項」或用 ✕ 刪除獎項
- **抽獎**：按下「開始抽獎」，畫面會有短暫的滾動動畫，接著一次顯示該獎項全部得獎人（例如設定抽 10 人，按一次就直接抽出 10 位，不用重複按）
- **防重複中獎**：同一廠已經中獎的人，會自動從後續抽獎的候選名單中排除
- **得獎總表**：頁面下方彙總所有分廠、獎項、得獎人，可一鍵「複製得獎名單」到剪貼簿
- **資料保留**：名單與抽獎結果會存在瀏覽器的 `localStorage`，重新整理頁面不會遺失；換一台電腦或清瀏覽器資料則會消失

### 3. 操作步驟

1. 開啟網頁，切換到「中壢廠」或「八德廠」分頁
2. 展開「名單管理」，把該廠的人員名單貼進文字框，按「更新名單」
3. 視需求調整各獎項名稱、每廠抽出人數
4. 按「開始抽獎」，等動畫跑完即可看到得獎人
5. 抽錯或需要重抽：按「重新抽獎」（會清空該獎項這一廠的結果，重新抽一次）
6. 全部抽完後，到頁面下方「得獎總表」按「複製得獎名單」，貼到你要的地方（Excel、公告文件等）

### 4. 重置功能

- **重置抽獎結果**：清除所有已抽出的得獎人，但保留名單與獎項設定
- **清空全部資料**：連名單、獎項設定也一併清空，恢復成最初的空白狀態（無法復原，使用前會再次確認）

---

## 二、Facebook 留言名單擷取工具

因為 Facebook 的 Graph API 對第三方 App 讀取一般留言者身分有嚴格限制，實際測試後，**直接讀取瀏覽器畫面上顯示的留言（跟你自己肉眼看到的一樣）才是最準確可靠的方式**。以下工具是一段貼到瀏覽器「主控台」執行的小程式，會自動展開所有留言與回覆，並把留言者姓名抓出來去重。

### 1. 事前準備

1. 用瀏覽器打開要抓留言的那篇貼文或 Reel
2. 打開留言區的排序選單（通常寫著「最相關」），**切換成「所有留言」**——這樣才會載入完整留言，包含系統預設過濾掉的部分，不要選「最相關」

### 2. 打開開發者工具（F12）

1. 在網頁上按鍵盤的 **F12**（或滑鼠右鍵 → 選「檢查」/「Inspect」）
2. 畫面右側或下方會跳出一個工具面板，上面有一排分頁：Elements、**Console**、Sources、Network...
3. 點選 **Console**（主控台）這個分頁——接下來的程式碼都貼在這裡執行

### 3. 處理「self-XSS」安全警告

第一次在主控台貼東西時，Facebook 會自動印出一段紅色警告文字（提醒使用者不要亂貼陌生程式碼，這是 Facebook 自己的防護機制，不是錯誤）。

**解法**：在主控台裡用鍵盤直接打字（不要用貼上）輸入：

```
allow pasting
```

按 **Enter**，之後才能正常貼上程式碼。這個動作每次開新分頁、重新整理頁面後可能都要再做一次。

### 4. 貼上並執行擷取程式碼

把下面整段程式碼複製起來，貼進主控台，按 **Enter** 執行：

```js
(async function () {
  function sleep(ms) {
    return new Promise((r) => setTimeout(r, ms));
  }

  function clickExpanders() {
    const candidates = Array.from(
      document.querySelectorAll('div[role="button"], span[role="button"], a'),
    ).filter((el) => {
      const t = (el.innerText || "").trim();
      return (
        t.includes("查看更多留言") ||
        t.includes("查看更多則留言") ||
        t.includes("查看更多回覆") ||
        t.includes("顯示更多留言") ||
        /^\d+\s*則回覆/.test(t)
      );
    });
    candidates.forEach((el) => {
      try {
        el.click();
      } catch (e) {}
    });
    return candidates.length;
  }

  console.log("開始自動展開留言與回覆，請稍候...");
  let emptyStreak = 0;
  let rounds = 0;
  const maxRounds = 200;

  while (rounds < maxRounds && emptyStreak < 6) {
    window.scrollBy(0, 600);
    await sleep(400);
    const clicked = clickExpanders();
    if (clicked === 0) {
      emptyStreak++;
    } else {
      emptyStreak = 0;
    }
    rounds++;
    await sleep(600);
  }
  console.log("展開完成（共執行", rounds, "輪），開始擷取姓名...");

  const timePattern = /^\d+\s*(秒|分鐘|小時|天|週|個月|年)$/;

  const links = Array.from(document.querySelectorAll("a[href]"));
  const candidates = links.filter((a) => {
    const href = a.getAttribute("href") || "";
    const text = a.innerText.trim();
    if (!text || text.length > 20) return false;
    if (timePattern.test(text)) return false;
    return href.includes("facebook.com/") && href.includes("comment_id=");
  });

  const names = candidates.map((a) => a.innerText.trim());
  const uniqueNames = Array.from(new Set(names));

  console.log("共抓到", uniqueNames.length, "位不重複留言者：");
  console.log("=====COPY START=====");
  console.log(uniqueNames.join("\n"));
  console.log("=====COPY END=====");
})();
```

### 5. 等待與注意事項

- 留言數量多的話（例如上百則），程式會跑比較久（可能 1-2 分鐘），過程中**不要點擊或滑動頁面**，等主控台印出「展開完成」才算跑完
- 執行過程中如果又跳出一次 self-XSS 警告，一樣輸入 `allow pasting` 再重新貼上執行一次
- 完成後主控台會印出「共抓到 XX 位不重複留言者」，下面會有 `=====COPY START=====` 到 `=====COPY END=====` 之間夾著完整名單

### 6. 複製名單

用滑鼠在主控台裡，**框選 `COPY START` 和 `COPY END` 中間的名單文字**，`Ctrl + C` 複製，貼到抽獎網頁對應分廠的「名單管理」文字框，按「更新名單」即可。

### 7. 為什麼抓到的人數常常對不上畫面上顯示的留言總數？

這是正常現象，原因包括：

- 顯示的留言總數通常也把「按讚」「表情符號互動」等算進去，不完全等於「留言者人數」
- 已刪除帳號、停用帳號、極度私密設定的使用者，網頁本身就不會顯示完整資訊，任何方式都抓不到
- 極少數巢狀很深的「回覆的回覆」，偶爾需要多跑幾次才會完全展開

一般抓到 90% 以上（例如 196 則抓到 180 幾位）就已經是很完整的結果，抽獎的公平性看的是「所有留言的人都有機會被抓進名單」，不需要執著於數字完全對齊。

### 8. 如果抓到的清單怪怪的（人數落差很大、混進不是人名的文字）

Facebook 的網頁結構偶爾會調整，如果這段程式碼哪天突然不準了：

1. 在留言區找一個看得到的留言者姓名，滑鼠右鍵點在姓名上，選「檢查」/「檢查元素」（Inspect）
2. 會跳出開發者工具並自動反白對應的 HTML 那一行，把畫面截圖
3. 對照截圖裡留言者姓名的連結網址（`href`）長什麼樣子，調整程式碼裡篩選規則那幾行（目前是抓網址裡含有 `comment_id=` 的連結）

---

## 附錄：常見問題

**Q：抽獎網頁的資料存在哪裡？換電腦看得到嗎？**
A：存在你目前這台電腦、這個瀏覽器的本機儲存（localStorage），換電腦或用無痕視窗開會看不到之前的名單與結果，需要重新貼名單。

**Q：可以直接用 FB 帳號登入自動抓留言嗎？**
A：技術上可以做到登入流程，但 Facebook 的 Graph API 對第三方 App 會隱藏大部分一般留言者的真實姓名（僅對 App 本身的管理員/測試者才會顯示完整姓名），除非申請 App Review 通過進階存取權限（審核耗時、不保證通過）。因此本工具改用「主控台讀取畫面內容」的方式，準確度更高也更省事。

**Q：抽獎按鈕反灰按不下去？**
A：代表名單已經抽完（該廠所有人都已經在某個獎項中獎），或該廠名單還沒有匯入，請先確認「名單管理」裡有沒有貼名單。
