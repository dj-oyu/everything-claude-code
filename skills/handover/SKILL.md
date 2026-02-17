---
name: handover
description: Generate a HANDOVER.md file summarizing current session context for seamless continuation in the next session.
---

# Handover

セッション終了時に HANDOVER.md を生成し、次のセッションへコンテキストを引き継ぐ。
コンパクション（自動要約）による情報ロスを防ぎ、セッション間の連続性を保証する。

## When to Activate

- 長時間セッションを中断・終了する前
- 複数セッションにまたがるタスクに取り組んでいるとき
- 次のセッションで別の人（または別のAI）が引き継ぐ可能性があるとき
- コンテキストウィンドウが逼迫し、コンパクションが近いとき

## HANDOVER.md の構成

```markdown
# Handover — {日付}

## Current Status
今どこまで完了しているか。進行中タスクの状態。

## What Was Done
このセッションで実施した作業の要約（箇条書き）。

## Key Decisions
採用した設計判断とその理由。却下した代替案。

## Open Issues
未解決の問題、既知のバグ、ブロッカー。

## Next Steps
次のセッションで最初に取り組むべきこと（優先順位付き）。

## Important Files
変更・参照した主要ファイルのパス一覧。

## Context & Notes
次のセッションで知っておくべき補足情報。
```

## Best Practices

1. **プロジェクトルートに配置** — `<repo>/HANDOVER.md` として生成する
2. **事実だけ書く** — 推測や未確定事項は明示的に「未確定」とマークする
3. **ファイルパスは具体的に** — `src/` ではなく `src/components/Auth.tsx:42` のように書く
4. **差分より状態を書く** — 「何を変えたか」だけでなく「今どうなっているか」を重視する
5. **git commit してから渡す** — 未コミットの変更がある場合は先にコミットを促す
