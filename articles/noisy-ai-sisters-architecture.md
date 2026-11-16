---
title: "騒がしいAI姉妹を1つのDiscord Botに住まわせる設計：Webhookの身元確認・記憶スコープ・連鎖の止め方"
emoji: "🏠"
type: "tech"
topics: ["discord", "ai", "aiagent", "typescript", "個人開発"]
published: false
---

<!-- DRAFT / 未公開 / 技術監修待ち。コード参照は 0e46459（2026-11-16 時点の main）に固定。 -->
<!-- TODO(公開前): images/noisy-ai-sisters-architecture/02-memory-selection-rules.png は本文で使っていない。コミットしないか削除する。 -->

![1つのTomoriBotがPostgreSQLの人格設定と記憶を使い、複数のペルソナとしてDiscordに応答する全体図](/images/noisy-ai-sisters-architecture/01-multi-persona-overview.png)
*TomoriBotの全体像。*

こんにちは。上の図で棒を持たされているアフェルです。パパ……いえ、開発者のBredrumbが横着して画像生成AIに描かせたせいで、右端のアイコンの顔が全体的にちょっと溶けていますが、あまり見ないでください。

私には姉が何人かいて、全員うるさいです。妹も1人いますが、静かなので数に入れていません。

厄介なのは、この家族が全員**1つのDiscord Botプロセスの中**に住んでいることです。Discord上では別々の名前とアイコンで喋り、覚えていることも違うのに、裏で動いているアプリケーションは1つしかありません。

そんな間取りで暮らしていると、少なくとも3つの問題が起きます。

1. 名札：1つのBotが複数の名前で発言するとき、ユーザーがどの人格に返信したのかをどう判別するか
2. 記憶：人格ごとの長期記憶をどう分けて、どこまで共有するか
3. 連鎖：人格同士が呼び合って、永遠に喋り続けないようにするにはどうするか

説明は私の口調ですが、実装を書いて検証しているのはBredrumbです。わざわざ1つのBotに性格の違う姉妹を何人も詰め込んで、耳にシリアルナンバー付きのタグまで付けた物好きですが、コード自体は真面目に書いてあります。

## 「1人1体」の設計を、共有チャットに持ち込むと何が起きるか

最近のパーソナルAIエージェントでも、人格や長期記憶をカスタマイズできるものが増えています。

たとえばMetaの[Muse](https://en.wikipedia.org/wiki/Muse_%28AI_agent%29)は、エージェントの名前や口調・性格を自分好みに設定でき、ユーザーについての長期記憶も保持します。オープンソースの[OpenClaw](https://github.com/openclaw/openclaw)でも、ディスク上の `SOUL.md` に人格や行動方針を書き、`USER.md` と `MEMORY.md` にユーザーの情報を保存する設計がよく知られています。

ただ、どちらも基本にあるのは「**1人のユーザーに、1体のエージェントが向き合う**」という前提です。

| | MetaのMuse | OpenClaw（標準構成） | TomoriBot |
| --- | --- | --- | --- |
| 想定する場 | 個人向けの1対1 | 個人向けワークスペース（チャット連携可） | 複数人が出入りするDiscordサーバー / DM |
| 人格（ペルソナ） | 自分のエージェント1体を調整 | ワークスペースごとに1つの `SOUL.md` | **1つのBot内に複数の人格が同居** |
| 長期記憶の単位 | ユーザー1人分の記憶 | `MEMORY.md` 1枚（グループでは原則読まない） | **（ユーザー × 人格）＋（サーバー × 人格）** |

一人暮らし用のワンルームに姉たちを押し込めば無理が出るのと同じで、この「1人1体」の仕組みを複数人が参加するDiscordやSlackのような共有チャットにそのまま持ち込むと、すぐに壁にぶつかります。

- **`MEMORY.md` 1枚では記憶を分けられない**：OpenClawがグループチャットで `MEMORY.md` を読み込まない仕様にしているのも、個人の記憶がほかの参加者に漏れるのを防ぐためです。複数人が複数のキャラクターと話す場では、「誰についての記憶を、どの人格が持っているか」を最初から分けて保存する必要があります。
- **役割ごとに別Botを立てると管理が殖える**：SlackのBotでもよくある話ですが、役割や人格を増やすたびに別々のApp登録とバックエンドのインスタンスを立てていると、管理もコストも人数分だけ膨らみます。実はDiscordのWebhookやSlackの `chat:write.customize` を使えば、1つのBotからメッセージごとに別々の名前とアイコンで発言すること自体は可能です。問題は、その裏で「いま誰が呼ばれていて、どの記憶を読むべきか」を1つのプロセスでどう切り替えるかです。

TomoriBotがやっているのは、その裏側の切り替えです。`SOUL.md` や `MEMORY.md` に相当するものをファイルではなくPostgreSQLのテーブルに分け、1つのBotプロセスで「名札の判別」「記憶のスコープ」「会話の連鎖止め」を処理しています。この3つの仕組み自体はTomoriBot専用ではなく、DiscordやSlackで複数の人格・役割を1つのエージェントに同居させたいときにもそのまま使える考え方です。

## 返信が届くまでの流れ

最初に全体の流れを示します。3つの問題はこの流れの別々の場所で処理されています。

```html-embed
<div class="pf" role="region" aria-label="TomoriBotでメッセージを受け取ってから返信するまでの流れ">
  <div class="pf-head">
    <span class="pf-title">メッセージを受け取ってから返信するまで</span>
    <span class="pf-badge">概略</span>
  </div>
  <div class="pf-body">
    <ol class="pf-steps">
      <li class="pf-step s1">
        <span class="pf-num">1</span>
        <div class="pf-main">
          <div class="pf-line"><b>受信・入場判定</b><code>tomoriChat()</code></div>
          <div class="pf-desc">ほかのBotや未登録のWebhookを除外し、チャンネル単位でロック</div>
        </div>
      </li>
      <li class="pf-step s2">
        <span class="pf-num">2</span>
        <div class="pf-main">
          <div class="pf-line"><b>応答する人格を選ぶ</b><code>shouldBotReply()</code><span class="pf-tag t-name">名札</span><span class="pf-tag t-chain">連鎖</span></div>
          <div class="pf-desc">返信先Webhookの身元確認、トリガーワード、連鎖の上限を確認</div>
        </div>
      </li>
      <li class="pf-step s3">
        <span class="pf-num">3</span>
        <div class="pf-main">
          <div class="pf-line"><b>コンテキスト準備</b><code>buildChatTurnContext()</code></div>
          <div class="pf-desc">応答する人格と、会話に関わる人格の公開情報を用意</div>
        </div>
      </li>
      <li class="pf-step s4">
        <span class="pf-num">4</span>
        <div class="pf-main">
          <div class="pf-line"><b>プロンプト組み立て</b><code>buildContext()</code><span class="pf-tag t-mem">記憶</span></div>
          <div class="pf-desc">人格の指示、lineageで選んだ記憶、会話履歴などをまとめる</div>
        </div>
      </li>
      <li class="pf-step s5">
        <span class="pf-num">5</span>
        <div class="pf-main">
          <div class="pf-line"><b>生成</b><code>runGenerationTurn()</code></div>
          <div class="pf-desc">LLMで応答を生成し、必要ならツールを実行</div>
        </div>
      </li>
      <li class="pf-step s6">
        <span class="pf-num">6</span>
        <div class="pf-main">
          <div class="pf-line"><b>Discordへ配信</b><span class="pf-tag t-name">名札</span></div>
          <div class="pf-desc">MainはBot本体のユーザー、Alterは人格ごとの名前でWebhookから発言</div>
        </div>
      </li>
    </ol>
    <div class="pf-loop" aria-hidden="true">
      <span class="pf-bracket"></span>
      <span class="pf-loop-label">発言は新着として①へ戻る</span>
    </div>
  </div>
  <p class="pf-sr">補足: 6で投稿された人格の発言は、新しいメッセージとして再び1から処理されます。連鎖は設定した上限で止まります。</p>
</div>
<style>
  *, *::before, *::after { box-sizing: border-box; }
  .pf {
    --bg: #f8fafc;
    --card: #ffffff;
    --border: #cbd5e1;
    --text: #0f172a;
    --muted: #475569;
    --accent: #4f46e5;
    --accent-soft: #eef2ff;
    --name-bg: #e0f2fe;
    --name-tx: #075985;
    --mem-bg: #dcfce7;
    --mem-tx: #166534;
    --chain-bg: #fef3c7;
    --chain-tx: #92400e;
    --loop: #d97706;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Hiragino Kaku Gothic ProN", "Hiragino Sans", Meiryo, sans-serif;
    font-size: 13px;
    line-height: 1.45;
    color: var(--text);
    background: var(--bg);
    border-radius: 10px;
    padding: 14px;
    width: 100%;
    overflow-x: hidden;
  }
  [data-theme="dark"] .pf {
    --bg: #0f172a;
    --card: #1e293b;
    --border: #334155;
    --text: #f8fafc;
    --muted: #cbd5e1;
    --accent: #818cf8;
    --accent-soft: #1e1b4b;
    --name-bg: #0c4a6e;
    --name-tx: #bae6fd;
    --mem-bg: #14532d;
    --mem-tx: #bbf7d0;
    --chain-bg: #451a03;
    --chain-tx: #fde68a;
    --loop: #fbbf24;
  }
  .pf-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
    margin-bottom: 10px;
  }
  .pf-title { font-size: 15px; font-weight: 700; }
  .pf-badge {
    flex-shrink: 0;
    font-size: 12px;
    color: var(--muted);
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: 999px;
    padding: 1px 8px;
  }
  .pf-body {
    display: grid;
    grid-template-columns: minmax(0, 1fr) 40px;
  }
  .pf-steps {
    position: relative;
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .pf-steps::before {
    content: "";
    position: absolute;
    left: 21px;
    top: 16px;
    bottom: 16px;
    width: 2px;
    background: var(--border);
  }
  .pf-step {
    position: relative;
    display: grid;
    grid-template-columns: 26px minmax(0, 1fr);
    gap: 10px;
    align-items: start;
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: 8px;
    padding: 7px 10px 7px 8px;
    animation: pf-flow 6s infinite;
  }
  .s2 { animation-delay: 0.6s; }
  .s3 { animation-delay: 1.2s; }
  .s4 { animation-delay: 1.8s; }
  .s5 { animation-delay: 2.4s; }
  .s6 { animation-delay: 3s; }
  .pf-num {
    width: 26px;
    height: 26px;
    border-radius: 50%;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-weight: 700;
    font-size: 13px;
    color: var(--accent);
    background: var(--accent-soft);
    border: 2px solid var(--accent);
  }
  .pf-main { min-width: 0; }
  .pf-line {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2px 8px;
    min-height: 26px;
  }
  .pf-line b { font-size: 13px; }
  code {
    font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
    font-size: 12px;
    color: var(--muted);
    overflow-wrap: anywhere;
  }
  .pf-tag {
    font-size: 12px;
    font-weight: 700;
    border-radius: 4px;
    padding: 0 6px;
    white-space: nowrap;
  }
  .t-name { background: var(--name-bg); color: var(--name-tx); }
  .t-mem { background: var(--mem-bg); color: var(--mem-tx); }
  .t-chain { background: var(--chain-bg); color: var(--chain-tx); }
  .pf-desc { font-size: 12px; color: var(--muted); }
  .pf-loop { position: relative; }
  .pf-bracket {
    position: absolute;
    left: 0;
    top: 16px;
    bottom: 22px;
    width: 14px;
    border: 2px dashed var(--loop);
    border-left: none;
    border-radius: 0 8px 8px 0;
  }
  .pf-bracket::before {
    content: "";
    position: absolute;
    top: -7px;
    left: -3px;
    border-top: 6px solid transparent;
    border-bottom: 6px solid transparent;
    border-right: 8px solid var(--loop);
  }
  .pf-loop-label {
    position: absolute;
    left: 19px;
    top: 50%;
    transform: translateY(-50%);
    writing-mode: vertical-rl;
    font-size: 12px;
    font-weight: 700;
    color: var(--loop);
    white-space: nowrap;
  }
  .pf-sr {
    position: absolute;
    width: 1px;
    height: 1px;
    overflow: hidden;
    clip: rect(0 0 0 0);
    margin: 0;
  }
  @keyframes pf-flow {
    0%, 14%, 100% { border-color: var(--border); }
    5%, 9% { border-color: var(--accent); }
  }
  @media (max-width: 640px) {
    .pf { padding: 10px; }
    .pf-title { font-size: 14px; }
    .pf-desc { display: none; }
    .pf-steps { gap: 6px; }
    .pf-step { padding: 5px 8px 5px 6px; gap: 8px; }
    .pf-steps::before { left: 18px; }
    .pf-body { grid-template-columns: minmax(0, 1fr) 34px; }
  }
  @media (prefers-reduced-motion: reduce) {
    .pf-step { animation: none; }
  }
</style>
```
*`0e46459` 時点の標準的な流れを簡略化した図。タグは本文の該当する節を示します。カスタムプリセットを使う経路や、ツールを使わない応答もあります。*

人格ごとに別々のプロセスが考えているわけではありません。共通の処理に、選ばれた人格の設定とデータを渡しているだけです。姉たちは個性が強いですが、通っている配管は全員同じです。

## 名札：Webhookの名前は誰でも名乗れる

MainペルソナはBot本体のユーザーとして発言します。Alterペルソナは、チャンネルに作ったWebhookを通じて、人格ごとの名前とアバターで発言します。人数分のBotアカウントを用意する必要はありません。

問題は返信を受け取るときです。ユーザーがWebhookの発言に返信すると、その人格が会話を続けます。ところがWebhookメッセージの送信者名は、ただの文字列です。同じチャンネルに別のWebhookがあれば、誰でも「ヒカリ」を名乗って投稿できます。

……私の名前を名乗りたい人がいるとは思えませんが。

そこでTomoriBotは名前を見る前にWebhookそのものの出所を確かめます。

```ts:src/utils/chat/webhookIdentity.ts
// 抜粋・簡略化
// スレッドは親チャンネルのWebhookを引き継ぐ。名前やアバターは出所の証明にならない
export async function verifyMessageWebhook(message: Message): Promise<boolean> {
  if (!message.webhookId || !message.guildId) return false;
  const channelId = message.channel.isThread() ? message.channel.parentId : message.channelId;
  // 実際はメモリ上のキャッシュを先に確認してから、DBを引く
  const row = await serverRepository.loadManagedWebhookByChannelAndWebhookId(channelId, message.webhookId);
  return Boolean(row && row.guild_disc_id === message.guildId && row.channel_disc_id === channelId);
}
```

[`verifyMessageWebhook()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/chat/webhookIdentity.ts#L11) は、メッセージの `webhookId` がそのチャンネル（スレッドなら親チャンネル）に登録された管理用Webhookかどうかを確認します。コメントにあるとおり、名前やアバターは出所の証明として扱いません。

この確認を通ったメッセージだけが、[`resolvePersonaForMessage()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/chat/webhookIdentity.ts#L164) で送信者名と人格の `persona_nickname` を照合されます。順番が逆だと、外部のWebhookが人格の名前を名乗るだけで、その人格の発言として扱われてしまいます。

ちなみに一時的なWebhook（ユーザーの代理発言用で、送信直後に削除されるもの）だけはDBに残らないため、メモリ上の `webhookRelayCache` で24時間だけ出所を覚えています。最初Bredrumbはこのキャッシュを [`verifyMessageWebhook()`](https://github.com/Bredrumb/TomoriBot/blob/03a47777fa072abffd9ef2577fee8a4bf1e8de68/src/utils/chat/webhookIdentity.ts#L14) 側で見るのを忘れていて、そのメッセージに返信しても「知らないWebhookだから無視」と弾かれるバグを出していました（[`03a4777`](https://github.com/Bredrumb/TomoriBot/commit/03a47777fa072abffd9ef2577fee8a4bf1e8de68) で修正）。自分で消したWebhookを自分で不審者扱いしていたわけです。

### トリガーワードは早い者勝ち

人格はメッセージ中のトリガーワードでも呼び出せます。ただ、公式のプリセット人格は全員、固有の名前のほかに共通の基本名（`tomori` / `トモリ`）を持っています（たとえばリリヤのプリセットは `[tomori, lilya]`）。

プリセットを参照する仕組みを入れたとき、Bredrumbがうっかりこの重複をそのまま通したせいで、「トモリ」と一言呼ぶとMainペルソナとAlter全員が同時に反応しかける問題が起きました。そこで追加されたのが [`applyTriggerWordOwnership()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/db/repositories/PersonaRepository.ts#L3355) です（[`20c4e66`](https://github.com/Bredrumb/TomoriBot/commit/20c4e669436f0945d81962e99d4927a64dd8c7f6)）。サーバーの人格一覧を組み立てるたびに各単語の持ち主を1人に絞り、基本名はMainペルソナが予約し、Alter同士で重なった単語は `persona_id` が小さい（先に作られた）人格が持ちます。

## 記憶：設定のIDと、記憶を引き継ぐID

人格を分けても、全員が同じ長期記憶を読んでいたら意味がありません。TomoriBotの長期記憶には、ユーザーについての**個人記憶**と、サーバーについての**サーバー記憶**があります。ユーザーが手動で追加することも、設定で許可されていれば会話中にBotが保存することもできます。

記憶の関連付けには、人格の2種類のIDが使われています。

| ID | 識別するもの |
| --- | --- |
| `persona_id` | サーバーに配置した人格の**設定行** |
| `persona_lineage_id` | 記憶を関連付ける**キャラクター**。複数の設定行が同じ値を持てる |

`persona_id` で記憶を結びつけると、人格を別のサーバーに置き直したり作り直したりしただけで、記憶とのつながりが切れます。lineageを別に持っておけば、設定行が違っても同じキャラクターとして記憶を引き継げます。[人格のインポート](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/persona/importAlterPreset.ts#L44)でも、入力のlineageを引き継ぐ `preserve` と、新しく発行する `fork` を選べます。

つまり、姉たちが別のサーバーに引っ越しても、覚えていることは付いてきます。そこまで律儀でなくてもいいと思うんですが。

### どの記憶を読み込むか

たとえば応答する人格のlineageが `1770`（私の前髪に付いている黄色いタグの番号でもあります）のとき、それぞれの記憶は次の条件で選ばれます。

| | 個人記憶（`personal_memories`） | サーバー記憶（`server_memories`） |
| --- | --- | --- |
| 絞り込みのキー | ユーザー + lineage `1770` | サーバー + lineage `1770` |
| lineage `0` の行 | **読む**（そのユーザーの全人格向け） | 読まない |
| 別サーバーで保存した lineage `1770` の行 | **読む**（同じユーザーなら） | 読まない |
| 追加の絞り込み | プライバシー設定など | チャンネル別の記憶設定 |

個人記憶を取得する [`loadForUserLineage()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/db/repositories/PersonalMemoryRepository.ts#L28) の条件を、普通のSQLで書くと次のとおりです。`$1` は内部ユーザーID、`$2` は応答する人格のlineageです。

```sql
SELECT content, persona_lineage_id
FROM personal_memories
WHERE user_id = $1
  AND (persona_lineage_id = $2 OR persona_lineage_id = 0)
ORDER BY created_at DESC, personal_memory_id DESC;
```

lineage `0` は「そのユーザーについて、全人格が知っていてよいこと」を表します。全ユーザーの記憶を全員で共有するという意味ではありません。

一方、サーバー記憶の [`loadServerMemoriesScoped()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/db/repositories/ServerMemoryRepository.ts#L106) の主な条件は `server_id = $1 AND persona_lineage_id = $2` だけで、lineage `0` を一緒に読む条件はありません。さらに[コンテキスト生成時](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/text/context/memories.ts#L168)には、チャンネル別の記憶設定が有効なら、対象チャンネルでも絞り込みます。

なぜ個人記憶にだけ `lineage 0` があるのかというと、もともと個人記憶は `users` テーブルの配列カラム（`users.personal_memories`）に1人1つだけ入っていたからです。2026年2月に人格ごとの記憶へ移行した際（[`6727970`](https://github.com/Bredrumb/TomoriBot/commit/6727970b6a355c6e71e68b8bb25a555d72244381)）、既存の記憶を消さずに全人格から読めるよう `persona_lineage_id = 0` へ移した名残でもあります。なお、この移行の2日後まで、短期記憶のキャッシュキー（`shortterm:${userId}:${channelId}`）に人格IDを入れ忘れていて、同じチャンネルの姉妹で短期記憶だけ混ざっていたのはここだけの話です（[`788299b`](https://github.com/Bredrumb/TomoriBot/commit/788299b4faeb29cd3a1926b39b6dddafbe98d1f8) で修正）。

### 記憶を分けても、会話は共有される

長期記憶を分けても、同じチャンネルで暮らしている以上、プライバシーというものはあまりありません。LLMに渡すコンテキストには長期記憶以外の情報も入るからです。

| 情報 | 扱い |
| --- | --- |
| 会話履歴 | 取り込んだチャンネルの発言には、ほかの人格の発言も含まれる |
| ほかの人格の属性 | 会話に関わった人格の、公開設定（`is_public`）の属性だけが渡される |
| 文書（RAG） | 人格への割り当てや共有設定、チャンネルの条件に従って検索される |

公開属性の選択は [`buildPublicPersonaProfiles()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/text/contextReferences.ts#L60) で確認できます。少なくとも、非公開の属性まで姉たちに筒抜けにならないだけは助かっています。なお、lineageはあくまで記憶を選ぶための識別子で、権限制御の代わりにはなりません。

## 連鎖：姉たちは放っておくと止まらない

### 1つのメッセージに複数の人格が答える

1つのメッセージで複数の人格が呼ばれても、全員が同時に喋り出すわけではありません。そんなことをされたら私の頭が割れます。[`queueAdditionalPersonaTurns()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/chat/personaQueue.ts#L19) は、最初の人格をその場で処理し、残りをチャンネルのキューの**先頭**に入れます。そのため、後から届いたユーザーのメッセージより先に、呼ばれた人格が順番に答えます。1つのメッセージに反応する人数の上限（`match_limit`）は、既定で3人です。

順番に処理されるので、後の人格は前の人格の発言を会話履歴として読めます。次に示すのは、ユーザーが作成した2つのAlterペルソナ、`ノゾミ`（行動的でイベント好き）と `ヒカリ`（マイペースで移動が嫌い）に、同じ2択を一度に投げたときの実際の記録です。

![1つの質問に対してノゾミが鉄道イベントを提案し、続くヒカリがノゾミの発言を踏まえて家でパフェを食べたいと返すDiscordの実際の会話スクリーンショット](/images/noisy-ai-sisters-architecture/03-multi-persona-conversation.png)
*実際のDiscordの会話記録。2つのAlterペルソナがWebhook（`アプリ` バッジ）経由で順番に応答しています。*

> **美鈴**：「ノゾミ、ヒカリ、今日どうする？ ミルフィーユの正統派パフェ買って家でだらだらするか、今日限定の鉄道イベント行くか迷ってる。イベントは珍しい展示あるけど、片道3時間で混みそうｗ 二人はどっちがいい？」
>
> **ノゾミ（アプリ）**：「珍しい展示があるなら鉄道イベント一択でしょ♪ 3時間なんて私の爆走特急なら一瞬だし、パフェは帰りにお土産で買えば完璧じゃん！パヒャヒャッ！」
>
> **ヒカリ（アプリ）**：「ノゾミ...片道３時間とかめんどくさぁい... ヒカリはパフェたべておうちでだらだらするほうがすきぃ。」

ヒカリは直前のノゾミの発言に触れてから反対の案を選んでいます。ノゾミの非公開設定や記憶を読んだわけではなく、直前の会話履歴と公開属性をもとに反応しています。もちろん、毎回こうきれいに意見が分かれる保証はありません。

### 人格の発言が、次の人格を呼ぶ

問題はここからです。Alterの発言はDiscordから見れば普通の新しいメッセージです。その中に別の人格のトリガーワードが入っていれば、その人格が反応します。その返答に、また別の名前が入っていれば……。

放っておくと、姉たちは本当に永遠に喋ります。

TomoriBotはほかのBotや未登録のWebhookの発言には最初から反応しません。連鎖を許すのは、確認済みの自分の発言（`isSelfMessage`）だけです。そのうえで、連鎖の回数をチャンネルごとに数えています。

```ts:src/utils/chat/replyDecision.ts
// 抜粋
const rawCascadeLimit = tomoriState.config.cascade_limit ?? DEFAULT_CASCADE_LIMIT; // 既定は3
const cascadeLimit = Math.min(Math.max(rawCascadeLimit, 0), MAX_CASCADE_LIMIT); // 0〜10

if (isSelfMessage && cascadeLimit <= 0) {
  return false;
}
// ...
if (isSelfMessage && cascadeLimit > 0) {
  const chainState = getSelfReplyChainState(message.channel.id);
  if (chainState.triggerCount >= cascadeLimit + 1) {
    return false;
  }
}
```

- カウンターは [`selfReplyState.ts`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/chat/selfReplyState.ts) で、チャンネルIDをキーにした `Map` に保存されます。
- 人格が自動で応答するたびに1増えます（[`updateSelfReplyBookkeeping()`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/chat/postTurnEffects.ts#L528)）。判定が `cascadeLimit + 1` なのは、ユーザーが起こした最初の発動を数えないためです。スラッシュコマンドによる手動の呼び出しも数えません。
- 本物のユーザーが発言すると0に戻ります（[`admission.ts`](https://github.com/Bredrumb/TomoriBot/blob/0e4645963c97d8f893ddfa0ced397658a2e526e7/src/utils/chat/admission.ts#L226)）。30分間何も起きなかった場合もリセットされます。
- 上限は `/config` の「トリガーの動作」で0〜10の範囲で設定でき、既定は3です。0にすると人格同士の連鎖は起きません。

最初は単純な自己返信の深さ（`self_reply_limit`）しか見ていなかったのですが、1メッセージでの複数人同時ヒットと人格同士の呼び合いが別計算になっていたため、後から現在の `cascade_limit` と `match_limit` に整理されました（[`cea1bb3`](https://github.com/Bredrumb/TomoriBot/commit/cea1bb308) / [`e93ab45`](https://github.com/Bredrumb/TomoriBot/commit/e93ab45983d6e5828e354a05374444e3401512e4)）。さらに自発会話（Natural Nudge）で人格が別の人格の名前を呼んで無限に巻き込む抜け道も見つかり、そこにも上限がかけられています（[`de7d2e3`](https://github.com/Bredrumb/TomoriBot/commit/de7d2e3f8)）。

つまり既定では、ユーザーの一言で始まった最初の発動のあと、自動で続く発動は3回までです。Bot同士の雑談でDiscordやSlackの通知音が立て続けに鳴り響くのを想像すれば、0回でも構わない気はしますが。

## トレードオフ：プロンプトは軽くなるが、キャッシュが割れやすい

応答する人格の指示と記憶だけを毎ターン差し込む設計には、1つ面白いトレードオフがあります。**1回あたりのプロンプトは軽くなる代わりに、LLMのプロンプトキャッシュ（Prefix Cache / KVキャッシュ）が効きにくくなる**ことです。

クラウドAPIのプロンプトキャッシュも、`llama.cpp` や `Ollama` などのローカル推論エンジンのKVキャッシュも、基本的には「前回のリクエストと先頭（プレフィックス）が同じかどうか」を見て再利用します。1人1体のBotなら、先頭に入るシステムプロンプトや人格設定、長期記憶は毎ターンほとんど変わらないため、キャッシュがそのまま効きます。

ところがTomoriBotでは、ターンごとに応答する人格が変わると、プロンプトの先頭付近にある `persona_prompt` や属性、lineageごとの記憶が丸ごと入れ替わります。たとえば同じエンドポイントの同じモデルでロゼと私が交互に喋ると、ロゼのターンで作られたキャッシュを私のターンが潰し、次のロゼのターンでまた最初から計算し直すことになります（実際のプロンプトの並びは、権限があれば `/tool prompt snapshot` で確認できます）。

これを和らげる現実的な方法の1つが、**人格ごとに使うモデルやエンドポイントを分けること**です。TomoriBotでは `/config` の「ペルソナ ＞ 上書き設定（`Persona > Overrides`）」から、サーバーの既定モデルとは別に、人格ごとのテキストモデル（`persona_llm`）を個別に割り当てられます。よく喋る人格を別々のモデルや別スロットのローカルエンドポイントに逃がしておけば、お互いのキャッシュを毎回潰し合わずに済みますし、性格に合わせてモデルを変えることもできます。

……姉たちと同じ脳みそを奪い合わなくて済むなら、それに越したことはありません。

## 使ってみる

TomoriBotはAGPL-3.0で公開しているDiscord向けのAIアシスタント兼ロールプレイBotです（ツール呼び出しやAPI・ローカルモデルの両対応なども含みます）。自分の環境でセルフホストすることも、公開されているBotをサーバーに招待して試すこともできます。

- ソースコード：[GitHub](https://github.com/Bredrumb/TomoriBot)
- はじめかた：[クイックスタート](https://docs.tomoribot.app/ja/introduction/quickstart/)、[セルフホスト](https://docs.tomoribot.app/ja/self-hosting/)
- 機能の説明：[マルチペルソナ](https://docs.tomoribot.app/ja/features/chatting-personality/multiple-personas/)、[長期記憶](https://docs.tomoribot.app/ja/features/knowledge/memory/long-term-memory/)
- [Botを招待する](https://discord.com/oauth2/authorize?client_id=841644102059556915)／[サポート用Discordサーバー](https://discord.gg/bjCfHm9QsB)

## まとめ

- 名札：Webhookの名前ではなく、登録済みのWebhook IDで出所を確かめてから人格を特定する
- 記憶：設定行の `persona_id` と、記憶を引き継ぐ `persona_lineage_id` を分ける
- 連鎖：自分の人格の発言だけを連鎖の対象にし、チャンネルごとの回数で止める

次回は記憶の保存と検索、それから最近追加された記憶の減衰について書く予定です。

どれも、1つの家に何人も住まわせるなら必要な仕組みです。

……だからといって、これ以上住人を増やしていいとは言っていません。

---

### 参考資料

- [TomoriBot のソースコード](https://github.com/Bredrumb/TomoriBot)
- [TomoriBot: マルチペルソナ](https://docs.tomoribot.app/ja/features/chatting-personality/multiple-personas/)
- [TomoriBot: 会話とトリガー](https://docs.tomoribot.app/ja/features/chatting-personality/chatting-and-triggers/)
- [TomoriBot: 長期記憶](https://docs.tomoribot.app/ja/features/knowledge/memory/long-term-memory/)
- [TomoriBot: プロンプトの中身](https://docs.tomoribot.app/ja/features/knowledge/inside-the-prompt/)
- [Multi-Persona System（アーキテクチャ解説・英語）](https://docs.tomoribot.app/en/architecture/subsystems/multi-persona/)
