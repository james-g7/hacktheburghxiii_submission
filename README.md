# HTBXIII Teaching Agent

**HackTheBurgh XIII (2026) — JetBrains "Help the Developer" challenge**
By James Gaskin and Noah Barret

An AI teaching agent that lives inside your JetBrains IDE. Instead of writing code for you, it
**explains the code you already have, right where it is**: it highlights lines in your editor, puts
explanation boxes above them that you click through to go deeper, quizzes you with hints and
answers, and plays step-by-step walkthroughs.

The submission has two parts that work together:

| File | What it is | Repo |
|---|---|---|
| `JetbrainsTeachingPlugin-1.2.0.zip` | **IDE plugin** ("HTBXIII Learning Agent Tools"). Draws highlights, explanation boxes and walkthroughs in the editor, and exposes them as `learning_*` tools on the IDE's MCP server. | https://github.com/james-g7/JetbrainsTeachingPlugin |
| `TeachingAgent.jar` | **Teaching agent**. An [Agent Client Protocol (ACP)](https://agentclientprotocol.com) agent built with Koog that runs in the IDE's AI chat, uses an OpenAI model (`gpt-4o`), and drives the plugin to teach you. | https://github.com/NoahB-07/TeachingAgent |
| `.jetbrains/acp.json` | Template config that registers the agent with the IDE. | |

---

## Requirements

- A JetBrains IDE, **version 2026.2 or newer** (build 262+), e.g. IntelliJ IDEA.
- The **AI Assistant** plugin with the AI chat, so you can add a custom ACP agent.
- The **MCP Server** plugin enabled (bundled with recent IDEs). The agent reaches the editor through it.
- **Java 21 or newer** on your `PATH` (`java -version` to check).
- An **OpenAI API key**.

---

## Setup

### 1. Install the plugin

1. In the IDE, open **Settings → Plugins**.
2. Click the ⚙️ icon → **Install Plugin from Disk…**
3. Choose `JetbrainsTeachingPlugin-1.2.0.zip` (don't unzip it).
4. Restart the IDE when asked.

You should now see **Tools → Learning Agent** in the menu bar.

### 2. Turn on the IDE's MCP server

Open **Settings → Tools → MCP Server** and make sure it is enabled. With the plugin installed, the
server offers the plugin's `learning_*` tools (highlight code, show explanation tree, play
walkthrough, …) alongside the IDE's own tools.

### 3. Register the teaching agent

Put `TeachingAgent.jar` somewhere permanent, then create (or edit) the ACP config in your home
directory, `~/.jetbrains/acp.json` (on Windows: `C:\Users\<you>\.jetbrains\acp.json`). Start from
the template in `.jetbrains/acp.json` and fill in **the absolute path to the jar** and **your
OpenAI key**:

```json
{
  "default_mcp_settings": {
    "use_custom_mcp": true,
    "use_idea_mcp": true
  },
  "agent_servers": {
    "Teaching Agent": {
      "command": "java",
      "args": ["-jar", "C:/path/to/TeachingAgent.jar"],
      "env": {
        "OPENAI_API_KEY": "sk-..."
      }
    }
  }
}
```

- `use_idea_mcp: true` hands the IDE's MCP server (and so the plugin's tools) to the agent. Leave it on.
- If `java` isn't on the `PATH` the IDE sees, set `"command"` to the full path of your Java 21+ `java` executable.
- **Don't commit your real API key.** Keep the filled-in file in your home directory only.

### 4. Restart the IDE

Restart (or reopen the AI chat) so the IDE picks up the new agent.

---

## Using it

1. Open a project and a source file you want to understand.
2. Open the **AI chat** and choose **Teaching Agent** from the agent picker.
3. Ask about your code in plain language. The agent decides how to show it:

| You ask… | The agent shows… |
|---|---|
| "Explain `Main.java`", "How does this method work?", "I don't get this part" | An **explanation tree**: the code is highlighted with a box above it; click the links in the box to drill into each part. |
| "Walk me through `sum(3)`", "What does this print?", "What is `x` after the loop?" | A **walkthrough**: the function runs line by line in the editor with ◀ ⏵ ▶ controls. |
| "Where is the loop?", "Which line sets `total`?" | A **highlight** with a one-line note. |
| "Quiz me on this", "Check I understand" | **Question boxes** with "Show hint" and "Show answer". |
| "Why does this crash?" | A tree with a **warning** on the cause. It explains why, and leaves the fix to you. |

The chat reply stays short (a sentence or two saying what it showed and where); the teaching
happens in the editor. Follow-up questions ("why?", "go deeper on that") add new boxes to what's
already shown.

The agent adapts its explanations to your level (beginner, student, or engineer). It picks one from
how you write, or you can ask for one directly, e.g. *"explain this at a beginner level"*.

> The agent is a tutor, not a code generator: it won't write or fix your code for you. It explains,
> asks questions and gives hints so you can work it out yourself.

### Trying the plugin by hand

You can use the plugin without the agent, from **Tools → Learning Agent**:

- **Highlight…** / **Highlight with Explanation…**: highlight a range and add a box above it.
- **Show Explanation Tree…**: load an explanation tree from a JSON file.
- **Walk Through Function…** (also on the editor's right-click menu): put the caret in a function,
  pick inputs and a level, and get an AI-generated walkthrough.
- **Show Walkthrough…**: play a walkthrough from a JSON file.
- **Clear Explanations**: remove all highlights and boxes.

The JSON formats are documented inside the plugin jar (`docs/EXPLANATIONS.md` and
`docs/WALKTHROUGH.md`). The agent also reads them at runtime, so any agent can drive the plugin.

Minimal explanation tree:

```json
{
  "file": "src/Main.java",
  "nodes": {
    "main": { "range": "3-13", "text": "Entry point. It [loops five times](#loop)." },
    "loop": { "range": "8-12", "text": "Counts `i` from 1 to 5." }
  }
}
```

Minimal walkthrough:

```json
{
  "title": "Running main()",
  "file": "Main.java",
  "steps": [
    { "line": 3, "kind": "call", "title": "The program starts", "stack": ["main()"] },
    { "line": 8, "note": "`i` starts at **1**.", "vars": { "i": "1" } },
    { "line": 13, "kind": "return", "note": "Done." }
  ]
}
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| "Please set the OPENAI_API_KEY environment variable." | Add your key under `env` in `~/.jetbrains/acp.json` and restart the IDE. |
| Teaching Agent isn't in the agent picker | Check `acp.json` is valid JSON and in `~/.jetbrains/`, the jar path is absolute and correct, and restart the IDE. |
| Agent fails to start / `UnsupportedClassVersionError` | It needs Java 21+. Run `java -version`, or point `"command"` at a newer `java`. |
| Agent answers in chat but nothing appears in the editor | Make sure the plugin is installed, the MCP Server is enabled, and `use_idea_mcp` is `true`. If the IDE's MCP server isn't passed to the agent automatically, set `AIR_IDE_MCP_URL` in the agent's `env` to the MCP server's SSE URL (shown in **Settings → Tools → MCP Server**). |
| Boxes are stuck in the editor | **Tools → Learning Agent → Clear Explanations**. |

---

## Built with

- [Koog](https://github.com/JetBrains/koog): JetBrains' Kotlin agent framework (agent, OpenAI client, MCP tools)
- [Agent Client Protocol](https://agentclientprotocol.com): connects the agent to the IDE's AI chat
- [Model Context Protocol](https://modelcontextprotocol.io): the IDE's MCP server carries the plugin's `learning_*` tools to the agent
- IntelliJ Platform plugin SDK

Plugin source: <https://github.com/james-g7/hacktheburghxiii>
