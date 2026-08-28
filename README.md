チャットを PDF に変換
チャットを PDF に変換
このチャットを Acrobat で開く
Gemini との会話
教室で使用する、ストック状況(コピー用紙など)を管理するサイトを作成したい。以下のリンクのスプレッドシートは自由に使っていいので、作成してください。



やりたいこと

ストックを使用したらdiscord通知

そのサイトから直接ストック追加

通知にはコピー用紙のリンクを添付



https://docs.google.com/spreadsheets/d/1G7rDRKN97M8J3UngDBD40DouVK4takmnWFrKCjgu7pA/edit?usp=sharing

Google スプレッドシート（Google Apps Script / HTML Service）をバックエンドおよびWebアプリのホスティング環境として利用し、備品の増減（利用・補充）とDiscord通知を行うWebアプリの構成案・実装コードを作成しました。

1. スプレッドシートの準備
指定いただいたスプレッドシート（シート名: シート1）の1行目に以下のヘッダーを設定してください。

A列	B列	C列	D列	E列
品名	現在在庫数	単位	発注/商品リンク	最終更新日時
サンプルデータ（2行目以降）:

A2: A4コピー用紙 (500枚/包)

B2: 5

C2: 包

D2: [https://www.amazon.co.jp/dp/xxxxxxxxx](https://www.amazon.co.jp/dp/xxxxxxxxx)（購入先URLなど）

E2: （自動更新されます）

2. Google Apps Script の設定
スプレッドシート上部メニューの 「拡張機能」 > 「Apps Script」 を開き、以下の2つのファイルを作成・記述します。

① コード.gs
DISCORD_WEBHOOK_URL に通知先DiscordチャンネルのWebhook URLを設定してください。

JavaScript
const SPREADSHEET_ID = '1G7rDRKN97M8J3UngDBD40DouVK4takmnWFrKCjgu7pA';
const SHEET_NAME = 'シート1';
const DISCORD_WEBHOOK_URL = 'YOUR_DISCORD_WEBHOOK_URL_HERE'; // 実際のDiscord Webhook URLを入力

function doGet() {
  return HtmlService.createHtmlOutputFromFile('index')
    .setTitle('教室ストック管理')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL)
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}

// 在庫一覧の取得
function getStockList() {
  const sheet = SpreadsheetApp.openById(SPREADSHEET_ID).getSheetByName(SHEET_NAME);
  const data = sheet.getDataRange().getValues();
  const items = [];

  for (let i = 1; i < data.length; i++) {
    if (data[i][0]) {
      items.push({
        rowIndex: i + 1,
        name: data[i][0],
        count: Number(data[i][1]) || 0,
        unit: data[i][2] || '個',
        link: data[i][3] || '',
        updatedAt: data[i][4] ? Utilities.formatDate(new Date(data[i][4]), 'Asia/Tokyo', 'yyyy/MM/dd HH:mm') : '-'
      });
    }
  }
  return items;
}

// 在庫数の更新（消費 or 補充）
function updateStock(rowIndex, delta, userName) {
  const sheet = SpreadsheetApp.openById(SPREADSHEET_ID).getSheetByName(SHEET_NAME);
  const currentCount = Number(sheet.getRange(rowIndex, 2).getValue()) || 0;
  const newCount = Math.max(0, currentCount + delta);
  const itemName = sheet.getRange(rowIndex, 1).getValue();
  const itemUnit = sheet.getRange(rowIndex, 3).getValue() || '個';
  const itemLink = sheet.getRange(rowIndex, 4).getValue();
  const now = new Date();

  // スプレッドシート更新
  sheet.getRange(rowIndex, 2).setValue(newCount);
  sheet.getRange(rowIndex, 5).setValue(now);

  // 消費時（delta < 0）にDiscordへ通知
  if (delta < 0) {
    sendDiscordNotification({
      itemName: itemName,
      amount: Math.abs(delta),
      unit: itemUnit,
      remaining: newCount,
      link: itemLink,
      user: userName || '教室利用者'
    });
  }

  return { success: true, newCount: newCount };
}

// 新規品目の追加
function addNewItem(name, initialCount, unit, link) {
  const sheet = SpreadsheetApp.openById(SPREADSHEET_ID).getSheetByName(SHEET_NAME);
  const now = new Date();
  sheet.appendRow([name, Number(initialCount) || 0, unit, link, now]);
  return { success: true };
}

// Discord Webhook通知
function sendDiscordNotification(payload) {
  if (!DISCORD_WEBHOOK_URL || DISCORD_WEBHOOK_URL === 'YOUR_DISCORD_WEBHOOK_URL_HERE') return;

  const description = `**${payload.user}** が **${payload.itemName}** を **${payload.amount}${payload.unit}** 使用しました。\n\n` +
                      `📦 **残り在庫:** ${payload.remaining}${payload.unit}\n` +
                      (payload.link ? `🔗 **発注・商品リンク:** [こちらをクリック](${payload.link})` : '🔗 **発注・商品リンク:** 未登録');

  const message = {
    embeds: [
      {
        title: '📢 備品ストック使用通知',
        description: description,
        color: payload.remaining <= 1 ? 0xE74C3C : 0x3498DB, // 残り1以下で赤、通常時は青
        timestamp: new Date().toISOString()
      }
    ]
  };

  UrlFetchApp.fetch(DISCORD_WEBHOOK_URL, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(message),
    muteHttpExceptions: true
  });
}
② index.html（ファイル > 新規作成 > HTML）
HTML
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <style>
    body { background-color: #f8f9fa; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
    .card-stock { border-radius: 12px; border: none; box-shadow: 0 2px 8px rgba(0,0,0,0.06); margin-bottom: 16px; }
    .badge-count { font-size: 1.25rem; font-weight: 600; }
    .btn-action { min-width: 44px; }
  </style>
</head>
<body class="p-3 p-md-4">
  <div class="container" style="max-width: 720px;">
    <div class="d-flex justify-content-between align-items-center mb-3">
      <h3 class="m-0">📦 教室ストック管理</h3>
      <button class="btn btn-outline-primary btn-sm" onclick="loadStock()">更新</button>
    </div>

    <!-- 利用者名入力 -->
    <div class="mb-3">
      <label class="form-label text-muted small">操作者名（通知用）</label>
      <input type="text" id="userName" class="form-control" placeholder="例: 講師名 / 教室スタッフ">
    </div>

    <!-- 新規追加アコーディオン -->
    <div class="accordion mb-4" id="accordionAdd">
      <div class="accordion-item">
        <h2 class="accordion-header">
          <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseAdd">
            ＋ 新しい品目を登録する
          </button>
        </h2>
        <div id="collapseAdd" class="accordion-collapse collapse" data-bs-parent="#accordionAdd">
          <div class="accordion-body">
            <div class="mb-2">
              <label class="form-label small">品名</label>
              <input type="text" id="newName" class="form-control form-control-sm" placeholder="例: A4コピー用紙">
            </div>
            <div class="row g-2 mb-2">
              <div class="col-6">
                <label class="form-label small">初期数量</label>
                <input type="number" id="newCount" class="form-control form-control-sm" value="1" min="0">
              </div>
              <div class="col-6">
                <label class="form-label small">単位</label>
                <input type="text" id="newUnit" class="form-control form-control-sm" placeholder="包, 箱, 本 など" value="個">
              </div>
            </div>
            <div class="mb-3">
              <label class="form-label small">発注・商品URL</label>
              <input type="url" id="newLink" class="form-control form-control-sm" placeholder="https://...">
            </div>
            <button class="btn btn-primary btn-sm w-100" onclick="addItem()">追加する</button>
          </div>
        </div>
      </div>
    </div>

    <!-- ストック一覧 -->
    <div id="stockList">
      <div class="text-center py-5 text-muted">読み込み中...</div>
    </div>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <script>
    function loadStock() {
      const container = document.getElementById('stockList');
      container.innerHTML = '<div class="text-center py-5 text-muted">読み込み中...</div>';

      google.script.run
        .withSuccessHandler(renderStock)
        .withFailureHandler(() => {
          container.innerHTML = '<div class="alert alert-danger">データの取得に失敗しました。</div>';
        })
        .getStockList();
    }

    function renderStock(items) {
      const container = document.getElementById('stockList');
      if (!items || items.length === 0) {
        container.innerHTML = '<div class="text-center py-4 text-muted">登録されている備品はありません。</div>';
        return;
      }

      let html = '';
      items.forEach(item => {
        const isLow = item.count <= 1;
        const badgeColor = isLow ? 'bg-danger' : 'bg-secondary';
        const linkBtn = item.link 
          ? `<a href="${item.link}" target="_blank" rel="noopener noreferrer" class="btn btn-sm btn-outline-info text-decoration-none">🔗 発注リンク</a>` 
          : '';

        html += `
          <div class="card card-stock p-3">
            <div class="d-flex justify-content-between align-items-start mb-2">
              <div>
                <h5 class="mb-1">${escapeHtml(item.name)}</h5>
                <div class="text-muted small">最終更新: ${item.updatedAt}</div>
              </div>
              <span class="badge ${badgeColor} badge-count">${item.count} <span style="font-size: 0.8rem;">${escapeHtml(item.unit)}</span></span>
            </div>
            <div class="d-flex justify-content-between align-items-center mt-2 pt-2 border-top">
              <div>${linkBtn}</div>
              <div class="d-flex gap-2">
                <button class="btn btn-outline-danger btn-sm btn-action" onclick="changeCount(${item.rowIndex}, -1)">-1 使用</button>
                <button class="btn btn-outline-success btn-sm btn-action" onclick="changeCount(${item.rowIndex}, 1)">+1 補充</button>
              </div>
            </div>
          </div>
        `;
      });
      container.innerHTML = html;
    }

    function changeCount(rowIndex, delta) {
      const userName = document.getElementById('userName').value.trim();
      google.script.run
        .withSuccessHandler(() => loadStock())
        .withFailureHandler((err) => alert('更新に失敗しました: ' + err))
        .updateStock(rowIndex, delta, userName);
    }

    function addItem() {
      const name = document.getElementById('newName').value.trim();
      const count = document.getElementById('newCount').value;
      const unit = document.getElementById('newUnit').value.trim() || '個';
      const link = document.getElementById('newLink').value.trim();

      if (!name) {
        alert('品名を入力してください');
        return;
      }

      google.script.run
        .withSuccessHandler(() => {
          document.getElementById('newName').value = '';
          document.getElementById('newLink').value = '';
          const collapseEl = document.getElementById('collapseAdd');
          const bsCollapse = bootstrap.Collapse.getInstance(collapseEl) || new bootstrap.Collapse(collapseEl);
          bsCollapse.hide();
          loadStock();
        })
        .withFailureHandler((err) => alert('追加に失敗しました: ' + err))
        .addNewItem(name, count, unit, link);
    }

    function escapeHtml(str) {
      if (!str) return '';
      return String(str).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
    }

    window.onload = loadStock;
  </script>
</body>
</html>
3. デプロイ手順
Apps Scriptエディタ右上の 「デプロイ」 > 「新しいデプロイ」 をクリックします。

種類で 「ウェブアプリ」 を選択します。

設定を以下のように指定します:

次のユーザーとして実行: 自分

アクセスできるユーザー: 全員（または教室内のGoogleアカウント組織内）

「デプロイ」 をクリックし、発行された ウェブアプリのURL にアクセスするとストック管理サイトが開きます。

https://docs.google.com/spreadsheets/d/1G7rDRKN97M8J3UngDBD40DouVK4takmnWFrKCjgu7pA/edit?usp=sharing

リンクをありがとうございます。スプレッドシートの中身を確認いたしました。

大変申し訳ありませんが、AI側から直接お客様のスプレッドシートを編集したり、Apps Script（GAS）のプログラムを書き込んだりすることはシステム上できません。

お手数ですが、以下の手順でご自身で設定を進めていただく必要があります。

1. スプレッドシートの準備
スプレッドシートのA1セルに以下の内容をコピー＆ペーストして、見出しを作成してください。
品名,現在在庫数,単位,発注/商品リンク,最終更新日時

2. Apps Scriptへのコード貼り付け

スプレッドシート上部のメニューから 「拡張機能」 > 「Apps Script」 をクリックします。

開いた画面に、先ほど提示した コード.gs と index.html のコードをコピーして貼り付けます。

コード.gs の上部にある DISCORD_WEBHOOK_URL に、通知を送りたいDiscordのWebhook URLを貼り付けます。

3. デプロイ（Webアプリの公開）
右上の「デプロイ」ボタンから「新しいデプロイ」を選択し、「ウェブアプリ」として公開します。

もし、「DiscordのWebhook URLの取得方法がわからない」「コードを貼り付けたがエラーが出る」「武田塾の他の自動化ツールと連携させたい」など、設定の途中でつまずいた部分や追加したい機能があれば、詳しくサポートいたしますのでお知らせください。

では、このアカウントのスプシで作成してください。

大変申し訳ありませんが、AIのシステム上、私自身のアカウントでGoogleスプレッドシートを新規作成し、公開用のアプリとして初期設定を行うことはできない仕組みとなっております。

その代わりとして、スプレッドシートやGASの面倒な初期設定が一切不要で、ブラウザで開くだけですぐに使えるストック管理アプリを作成しますね。

このアプリはブラウザの保存機能（ローカルストレージ）を利用してデータを管理し、画面上からDiscordのWebhook URLを設定するだけで、ご要望の「使用時のDiscord通知（リンク付き）」も完全に動作します。

それでは、アプリケーションを構築します。

教室ストック管理アプリ
8月26日 21:53
こちらのファイルの使い方は非常に簡単です。

上記のコードブロックの右上にある「コピー」ボタンを押してコードをコピーするか、拡張子を .html （例: stock_manager.html）としてファイルをパソコン上に保存してください。

保存したファイルをダブルクリックしてブラウザ（ChromeやSafariなど）で開きます。

画面右上の「歯車マーク」をクリックし、DiscordのWebhook URLを貼り付けて保存します。

これで準備完了です！
「新規追加」からコピー用紙などを登録し、「-1 使用」ボタンを押すと、在庫が減ると同時にDiscordへリンク付きの通知が届きます。データはご利用のブラウザ内に自動保存されるため、毎回設定し直す必要はありません。ぜひお試しください！

これの操作マニュアルを作成。

README用をまず作成してくだしあ

Your Markdown file is ready

MD のアイコン
README
 MD 
開く
操作マニュアルとしてMarkdown形式で作成しました。Notion等で管理されているドキュメントにそのままコピー＆ペーストして表示することも可能です。

シフトに入っているスタッフが迷わず使えるよう、ブラウザでの保存の仕様や運用上の注意点も記載しておきました。ご活用ください。







Gemini は AI であり、不正確な情報を表示することがあります。

Finalizing the Content
# 教室ストック管理アプリ (Classroom Stock Manager)

ブラウザのみで動作する、軽量な備品（コピー用紙、インク、文房具など）の在庫管理ツールです。
在庫を使用・消費した際に、自動的にDiscordへ通知を送信します（発注用の商品リンク付き）。

## 特徴
- **サーバー/データベース不要**: HTMLファイル1つで動作し、データはブラウザのローカルストレージに保存されます。
- **Discord連携**: 備品を使用（マイナス）した際、指定したチャンネルに通知と残り在庫数を自動送信します。
- **シンプルUI**: PCやタブレットから直感的に操作可能。

## セットアップ手順

1. **ファイルの準備**
   - 作成したHTMLファイル（例: `stock_manager.html`）を教室のPCやタブレットに保存します。
2. **アプリの起動**
   - 保存したHTMLファイルをブラウザ（Chrome, Edge, Safariなど）で開きます。
   - ※ ブックマークバーに追加したり、Notion等のドキュメント内にファイルパスのリンクを貼っておくと、講師やスタッフからのアクセスがスムーズになります。
3. **Discord Webhookの設定**
   - アプリ画面右上の「⚙️（歯車マーク）」をクリックします。
   - 通知を送信したいDiscordチャンネルのWebhook URLを入力し、「保存」をクリックします。

## 使い方

### 1. 備品の登録
- 画面内の「＋ 新しい品目を登録する」をクリックします。
- 以下の項目を入力し、「追加する」を押します。
  - **品名**: （例）A4コピー用紙 500枚
  - **初期数量**: 現在の在庫数
  - **単位**: 包、箱、本など
  - **発注・商品URL**: Amazonやアスクルなどの商品リンク（通知から直接発注できるようにするため）

### 2. 在庫の管理
- **「-1 使用」**: 在庫を1つ減らします。**この操作を行った時のみ、Discordに通知が飛びます。**
- **「+1 補充」**: 在庫を1つ増やします（発注した商品が届いた際などに使用）。補充時は通知されません。

### 3. 操作者名の設定
- 画面上部の「操作者名（通知用）」欄に名前（例: 自身の名前や講師名）を入れておくと、Discord通知に「誰が使用したか」が記載されます。
- シフトに入っているスタッフが、各自入力してから操作する運用を推奨します。

## 運用上の注意事項（重要）
- 本アプリのデータは、**開いているブラウザのローカルストレージ**に保存されます。
- そのため、**教室内の共通PCや専用タブレットなど、1つの「固定端末（固定ブラウザ）」で運用**してください。別のPCで開くとデータは同期されません。
- ブラウザの「閲覧履歴・キャッシュの完全消去」を行うと登録データも消えてしまうため、ご注意ください（通常の再起動は問題ありません）。
README.md
「README.md」を表示しています。
