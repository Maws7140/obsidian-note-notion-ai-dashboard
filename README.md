# Notion AI Dashboard

Datacore view in the style of the notion ai sidebar

## Description

A Notion-style **AI sidebar dashboard** for Obsidian. Shows your [Agent Client](https://github.com/PineappleExpress808/obsidian-agent-client) agents as avatar icons, lists recent AI chat sessions grouped by recency, and provides a sticky "New chat" bar  all in a clean, compact layout built with [Datacore](https://github.com/blacksmithgu/datacore) JSX.

---

## What it does

- **Agents row** displays every agent configured in Agent Client as a circular avatar. Clicking opens (or creates) the agent's prompt note.
- **+ New** button creates a new agent prompt stub and opens it.
- **Session history** lists saved chat sessions grouped into **Today**, **Past 30 days**, and **Older** with relative timestamps (2h, 3d, 1w�). Provider icons for Gemini and Claude are pulled automatically.
- **Sticky bottom bar** "New chat" button always visible at the bottom; a quick-action menu (pencil icon) lets you create a new note, start a recording, or create a Datacore base.
- Adapts to your theme via CSS variables works in light and dark mode.

---

> **Note:** Without Agent Client installed the dashboard will show an empty agents row and no sessions, but it will still render without errors.

---

## Installation

1. Install **Datacore** and (optionally) **Agent Client** from Community Plugins.
2. Copy `snippets/audio-manager.css` ? `.obsidian/snippets/` in your vault.
3. In **Settings ? Appearance ? CSS Snippets**, enable **audio-manager**.
4. Copy `Notion AI Dashboard.md` anywhere in your vault (e.g. `Dashboards/`).
5. Open the note the dashboard renders immediately.

---

## Configuration

Three constants at the top of the `datacorejsx` block control all paths:

```js
// Path to Agent Client's data file (default shown below).
const AGENT_DATA_PATH = ".obsidian/plugins/agent-client/data.json";

// Folder where agent prompt notes are stored / created.
const PROMPTS_FOLDER = "Meta";

// Where the "New chat" button opens or creates its note.
const CHAT_NOTE_PATH = "AI Chats/New Chat.md";
```

Change `PROMPTS_FOLDER` to wherever you keep your agent system prompts, and `CHAT_NOTE_PATH` to wherever you want new chat notes to land.

---

## Agent avatars

The dashboard resolves agent icons in this priority order:

1. **Custom image**  if the agent has an `avatarImage` path set in Agent Client, that vault file is used.
2. **Provider icons**  IDs containing `gemini` or `claude` show their respective branded favicons.
3. **Emoji fallbacks**  `codex` ? ??, `image` ? ???, `opencode` ? ??, all others ? ??.

To add your own emoji for a custom agent, edit the `getAgentAvatar` function:

```js
if (agentId?.includes("myagent")) return { type: "emoji", value: "??" };
```

---

## Quick-action menu

The pencil button in the bottom-right opens a small pop-up with three actions wired to built-in Obsidian command IDs:

| Menu item | Command ID             |
| --------- | ---------------------- |
| New note  | `file-explorer:new`    |
| Recording | `audio-recorder:start` |
| Base      | `datacore:create-base` |

You can swap these out for any Obsidian command ID by editing the `onClick` handlers in the `showMenu` block.

---
## License

MIT  use freely, modify, share.


## Required Plugins

| Plugin | Version | Link |
|--------|---------|------|
| Agent Client | 0.10.6 | [GitHub](https://github.com/search?q=Agent%20Client%20obsidian) |
| Datacore | 0.1.29 | [GitHub](https://github.com/search?q=Datacore%20obsidian) |

## Screenshots

![Screenshot 1](https://i.imgur.com/bBsQue6.png)
## Installation

1. Download the `.md` file(s) from this repo
2. Place them in your vault
3. Install the required plugins listed above

---
*Published via [Vault Hub](https://obsidianvaulthub.com)*
