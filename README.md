# WeChat Disk Space Audit

[English](README.md) | [简体中文](README.zh-CN.md)

> Measure what WeChat occupies on Windows, separate disposable cache from data you must not touch, and produce a checklist you approve before anything is removed.

## What this is

This is not a cleaner that deletes things and reports the space it freed. That design has one failure mode that outweighs every benefit: it occasionally removes a document someone sent you, and you find out weeks later.

The order here is inverted on purpose. Scan first, produce a checklist, wait for approval, then move. Nothing is deleted in the first pass, no matter how the request was phrased.

## Why we open-sourced it

We ran into this the ordinary way — a laptop out of space, a suspiciously large WeChat directory, and no idea which of the dozens of subfolders were safe. Guessing was not acceptable and neither was deleting everything.

It is a general-purpose maintenance tool with nothing of ours inside it, so it is a clean thing to open-source.

## What it does

- **A real directory map.** What each folder under the WeChat data directory actually holds, with realistic size ranges, instead of a vague "cache" label.
- **Tiered cleanup strategy.** Which categories are safe to remove, which will cost you chat history, and which hold received documents that cannot be re-downloaded.
- **Recycle Bin, not delete.** Files are sent to the system Recycle Bin via `SHFileOperationW`, so a mistake is recoverable.
- **Verification step.** After moving files, the `$I` metadata files in the Recycle Bin are checked to confirm the operation actually happened.
- **A checkbox list.** The deliverable is a table with an action column you fill in, not a log of what was already done.
- **One documented dead end.** WeChat sits in the tray and cannot be shut down gracefully from a script. The attempt and the failure are both recorded so nobody repeats it.

## Requirements

Windows · Python 3

## Installation

```bash
git clone https://github.com/<your-username>/wechat-space-audit.git
```

Then place the folder into your agent skills directory:

| OS | Path |
|---|---|
| Windows | `C:\Users\<you>\.workbuddy\skills\wechat-space-audit\` |
| macOS / Linux | `~/.workbuddy/skills/wechat-space-audit/` |

There is no build step. Restart the session and it takes effect.

## How to use

Typical requests:

- "How much space is WeChat taking?"
- "What can I safely delete from WeChat?"
- "Clean up WeChat cache — but show me the list first"

The skill scans read-only, reports sizes by category, and hands back a checklist. Nothing moves until you say which rows to execute.

## Files

```
└── SKILL.md
```

## Notes and limits

1. **First pass is read-only, always.** Even when the request says "just clean it up". The checklist exists so a wrong delete becomes an unchecked box rather than a lost file.
2. **Received documents are the trap.** WeChat stores files people sent you inside its own directory. Those are not re-downloadable once expired. This category is flagged separately.
3. **WeChat holds file locks.** A folder that looks removable may be in use. The tool reports locked paths rather than forcing through them.
4. **Recycle Bin has a size limit.** Very large removals may bypass it depending on system configuration — check your Recycle Bin settings first.

## License

MIT
