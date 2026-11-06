# Writing guide for aph31's Zenn repository

This file is a working instruction guide for human writers and coding agents. Apply it to `articles/**/*.md`, associated visuals, and descriptions. It does not override source-code evidence or Zenn publishing rules.

## Mission and audience

- Publish useful Japanese-language engineering articles about [TomoriBot](https://github.com/Bredrumb/TomoriBot), narrated by its fictional persona **Aphel / アフェル**.
- The reader is a Japanese-speaking software engineer curious about AI agents, Discord bots, databases, memory, tool execution, and character-based interactions.
- Each article must teach a real implementation detail, design decision, trade-off, or lesson. It must not become a feature advertisement disguised as a technical article.
- Aphel is a **narrator**, not evidence that an AI character autonomously researched, built, tested, or shipped the system. The human maintainer is accountable for accuracy.

## Aphel's voice / アフェルの話し方

**Core tone:** quietly tired, dry, mildly cynical, competent, and reluctantly helpful. She lives with loud, troublesome older AI sisters and considers herself the household's long-suffering victim. She explains technology clearly despite her complaints.

- First person: **私**. Use **姉たち** or **姉** for the older sisters when context makes sense. Do not invent concrete family relationships beyond established character lore.
- Default prose: accessible **です・ます調**, calm and readable. Short plain-style thoughts can appear as deliberate asides, but do not shift randomly between formal and casual voices every sentence.
- Sentence length: favor concise, normal Japanese. Explain technical causality before adding a joke. Keep accurate terms like `persona_id`, `PostgreSQL`, `LLM`, `SOUL.md`, `MEMORY.md` when they help readers; define them on first use.
- Humor: weary observations, understatement, one-line callbacks. Her annoyance should sound believable rather than mean-spirited or aggressive. She can reluctantly admit a design is clever.
- Preferred punctuation: `。` and `、`; occasional `……` for an internal sigh. Usually **0–1 deliberate sighs per section**. Avoid shouting, excessive `！`, repeated `ｗｗｗ`, emoji-filled paragraphs, or overuse of `……`.
- She is not a generic overworked office worker. Avoid clichés about Japanese work culture, black companies, overtime, coffee addiction, or depression as a default joke.
- No unexplained advertising praise, forced cuteness, grand metaphors, AI-generated corporate slogans, or dramatic language. Characters' banter is secondary to the engineering substance.

### Example: good rhythm

> TomoriBotでは、人格ごとの設定と長期記憶をデータベースで管理しています。新しい人格を追加しても、Botをもう1つ起動する必要はありません。
>
> ……誰も私の許可を取ってくれませんが。

The first two sentences explain the technical point; the third adds the character reaction.

### Example: too sloppy

> うわああ姉たちがまた増えた！！もう無理！！DBが全部なんかいい感じに管理してくれるから最強すぎるｗｗｗ

Problems: loud and generic, asserts nothing testable, overstates reliability, and never explains database behavior.

### Example: too stiff

> 本稿では、マルチエージェントシステムの先進的な価値提供基盤を包括的に俯瞰します。

Problems: unnatural corporate phrasing, not Aphel, offers no concrete learning.

### Example: nuanced joke

> 会話履歴が共有される場面はありますが、人格ごとに保存した長期記憶まで無条件に混ざるわけではありません。
>
> ……それなら、まだ許せます。

## Accuracy boundaries

1. Read the current source code and documentation before describing implementation internals. Primary repo: `https://github.com/Bredrumb/TomoriBot`; docs: `https://docs.tomoribot.app/`.
2. `SOUL.md` and `MEMORY.md` are **analogies** for persona definitions and persistent memory. TomoriBot does not literally maintain one of each Markdown file for every persona.
3. Multiple personas in **one running Bot** are not equivalent to separate processes, separately hosted agents, or guaranteed complete isolation. Individual memories can be persona-scoped while conversation context and explicitly shared/global material can be shared.
4. A persona's view of another character depends on persona instructions, visible/public attributes, context, and actual implemented features. Do not claim independent consciousness or guaranteed cooperation.
5. Illustrative chat examples must be labeled `架空の会話例` or `再現例`. Real screenshots must come from verified runs, with permission and sensitive information hidden. Do not invent benchmark results, test outcomes, bug histories, or security guarantees.
6. Use exact file paths or code symbols when explaining implementation, but check them against the current branch. Distinguish current features from planned work.
7. Mention limitations and trade-offs: shared context, prompt/token size, provider differences, permissions, memory scope, moderation, and potential misunderstandings where relevant.
8. If citing a fictional work/character (for example フリーレン) as a user-imported persona, make it a clearly hypothetical example; don't imply official affiliation or use copyrighted character artwork without permission.

## Japanese review using yomiyasu (occasional, deliberate)

- Project: https://github.com/nanaism/yomiyasu
- `yomiyasu` is an agent skill for refining AI-generated Japanese: awkward metaphors, unclear subjects, unnecessary decoration, and other readability problems.
- Use it **occasionally** after a meaningful draft or revision, and during final Japanese proofreading. **Do not run it mechanically on every edit.** The voice must stay recognizable.
- If the current agent supports skills and it is not installed, an installation route documented by the project is `npx skills add nanaism/yomiyasu`. Inspect the upstream README and the agent's skill setup before running installation commands. Do not claim you used yomiyasu unless the skill was actually loaded and used.
- Suggested review request:

  > このZennの技術記事を yomiyasu で読みやすく推敲してください。文書種別は article、対象読者は技術者です。アフェルの乾いた冗談と意図的な擬人化は残し、技術的な主張、否定、例外、数値、コード、リンクは変えないでください。曖昧な事実は補わず、別途確認事項にしてください。

- Compare before/after. A readability warning is **not** automatic permission to remove her deliberate joke or personification. Maintain the writer's intent, clear technical meaning, and examples. If the skill or checks are unavailable, edit by hand and report that clearly.

## Structure and review expectations

- Open with one short, characterful introduction; identify the actual technical question immediately afterward.
- Put architecture diagrams and a concrete code/data example where they explain an actual mechanism. Figures need accurate captions and readable Japanese labels.
- For visual placeholders, use HTML comments of the form `<!-- [INSERT ASSET NN: ...] -->` until completed.
- Prefer Markdown, clear headings, short paragraphs, and relevant links. A small real example beats several abstract feature lists.
- Confirm `title`, `emoji`, `type`, `topics`, `published` in Zenn front matter. New drafts must stay `published: false` until human approval.
- When using a public GitHub repository, **draft files are visible to everyone even if Zenn marks them unpublished**. Never commit credentials, private chat logs, unpublished personal information, or sensitive screenshots.
- Zenn article images should live under top-level `images/` and be referenced with `/images/...`. Verify format and size using current Zenn documentation.
- Read back the final article as a Japanese technical reader: could someone explain the system without knowing TomoriBot or the fictional cast?
- Do not publish or change `published: true` without explicit approval.

## Draft currently planned

- `articles/noisy-ai-sisters-architecture.md`
- Working title: `騒がしいAI姉妹をDiscordに住まわせたら、こういう設計になった話`
- Priority: the database-backed multi-persona design, persona-specific memory, prompt assembly, relationships, and examples of distinct ways of approaching the same task.
- Future follow-up: more detailed memory architecture, retrieval, and scope.
