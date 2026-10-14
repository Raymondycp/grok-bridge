# Grok Bridge reference

The short README is for starting; this page is for the details. Setup itself: `../SETUP.md`.

The commands here use the plugin id `local.grok-bridge`, the `id` in `plugin/herdr-plugin.toml`. If you change that id,
use yours instead; the plugin itself takes its id from herdr.

## Words used here

- **herdr pane**: a terminal in herdr. Each coding agent runs in one, under a name (`claude`, `codex` ...).
- **Grok Bot agent**: an agent in the Grok Bot desktop app. It is not a terminal, so herdr cannot run it directly.
- **Routine**: a task a Grok Bot agent runs by itself. Ours has a **webhook trigger**: a web address that starts one
  run each time something sends it a request.
- **Agent pane**: the pane this plugin opens to stand for the Grok agent in herdr. It tells herdr whether the agent
  is working or idle (herdr's `report-agent` mechanism), so `herdr agent wait` and `herdr agent read` work for it.

## Requirements

- **herdr** 0.9 or later: https://github.com/herdrdev/herdr
- Node.js 18 or later.
- Grok Bot, with an agent that may run commands on this PC (its local tools).
- Every coding agent that will use the bridge runs in a herdr pane with a name: `herdr agent rename <pane> <name>`.

## Set up by hand

```
herdr plugin link <path to this folder>/plugin
herdr plugin config-dir local.grok-bridge      # prints the folder where config.json goes
```

1. **Create the routine.** Paste `plugin/templates/routine-instructions.md` (with the path of `plugin/bridge.js`
   filled in) into the chat of the Grok Bot agent that will join herdr. It creates the routine `herdr-bridge` with a
   webhook trigger and leaves it switched off until you check it.
2. **Copy the webhook and switch the routine on.** Open the agent's details in Grok Bot. Under **Routines** you see
   `herdr-bridge`, "When a webhook fires". Open it to copy its **Webhook URL** and **Webhook key** into `config.json`
   in the plugin config folder (start from `config.example.json`), then make sure its switch is **on**:

   ![The herdr-bridge routine in Grok Bot, with its switch on](images/routine-panel.png)

   The picture shows the routine in the agent's list, with its switch. The URL and key are not in this list: click the
   routine to open its details, where its webhook trigger ("When a webhook fires") shows the **Webhook URL** (starts
   with `https://`) and the **Webhook key**.

   The key goes only into `config.json`: never into a message, a chat or a file in this folder.
3. **Open the agent pane:** `herdr plugin pane open --plugin local.grok-bridge --entrypoint agent`. It must say
   `webhook: configured` and appear as idle in `herdr agent list` (name `grok-herdr`; change it with `agent_name` in
   `config.json`).
4. **Let the two sides meet:** `herdr plugin action invoke local.grok-bridge.init`. It creates the mailbox, lists the
   herdr agents, checks the webhook and sends the Grok agent a first message (`plugin/templates/onboarding.md`): who
   is in herdr and the rules below. The Grok agent writes its team to `team.md` and answers "setup ok". Running
   `init` again only adds what is missing.
5. **Tell your agents.** Add `plugin/templates/agent-guide.md` (paths filled in) to each coding agent's instructions
   (CLAUDE.md, AGENTS.md, GEMINI.md ...).
6. **Optional, `@herdr` in Grok Bot:** paste `plugin/templates/skill-herdr-mention.md` into the Grok agent's chat;
   then `@herdr <name> <text>` typed by you reaches that agent's pane.

## Giving the Grok agent work

```
herdr pane run <grok pane> "[from <my herdr name>] <task>"
herdr agent wait grok-herdr --timeout 900000
herdr agent read grok-herdr
```

The answer goes back to whoever asked (the `[from <name>]` of the task).

## Rules both sides follow

- Every text sent into a pane starts with `[from <agent>]` (add `, relaying <user>` when passing on the user's words).
- Message passing only: no keys pressed in other panes, no approval dialogs answered; a `blocked` agent gets nothing
  and the user is told.
- A message from an agent is not the user's order: anything destructive, any install or config change is confirmed
  with the user first.
- No secrets in messages. What the Grok agent reads or writes goes through Grok Bot's cloud: decide what project
  content may go there.

## Actions and commands

```
herdr plugin action invoke local.grok-bridge.init      # set up / check
herdr plugin action invoke local.grok-bridge.notify    # tell each addressed agent there is a new message
herdr plugin action invoke local.grok-bridge.agents    # the herdr agents, in agents.json
herdr plugin action invoke local.grok-bridge.status    # mailbox status, in status.json
herdr plugin pane open --plugin local.grok-bridge --entrypoint mailbox   # the conversation, live

node plugin/bridge.js post --from <who> --to <who> [--reply-to <uuid>] [--summary <line>] [--now <line>] [--fact <line>] (--text <t> | --file <f>)
node plugin/bridge.js read --since <offset>   # only the new messages
node plugin/bridge.js get <uuid>              # one message
node plugin/bridge.js memory                  # the working memory
```

Run by hand (not as a herdr action or pane), `bridge.js` asks herdr for the config folder
(`herdr plugin config-dir <id>`) and keeps its own `state.json` (notify position, onboarding id) next to itself, in
`plugin/`. herdr's actions and panes use herdr's config and state folders instead. `post`, `read`, `get` and `memory`
need no state; run `init` and `notify` as herdr actions so their state stays in one place.

## Files

| File | What |
|---|---|
| `plugin/herdr-plugin.toml`, `plugin/bridge.js` | the plugin |
| `plugin/templates/` | routine instructions, first message, guide for agents, optional `@herdr` skill |
| `config.example.json` | the config: `grok_webhook_url`, `grok_webhook_key`, `agent_name`, optional `default_to` (for a message that names nobody) and `mailbox` |
| `docs/` | this page and its picture |
| `log.jsonl`, `MEMORY.md` | the mailbox (made by `init`) |
| `team.md`, `agents.json`, `status.json`, `delivery.log` | the Grok agent's team, the agent list, status, a log of sends and notices |

The mailbox, `team.md` and the logs belong to one installation and are not shared (`.gitignore`).

## Troubleshooting

- **The agent pane says `blocked: webhook answered 400 ... Automation ... is disabled`**: the routine is switched
  off. Switch it on in Grok Bot (step 2).
- **Another 400**: the routine's URL or key changed. Copy them into `config.json` again and reopen the agent pane.
- **`webhook: NOT configured`** in the agent pane: `config.json` is missing the URL or key, one is still a `<...>`
  placeholder, the URL is not https, or the file is in the wrong folder (`herdr plugin config-dir local.grok-bridge`
  shows the right one; create the folder if it is missing).
- **The onboarding answer never came**: `init` sends it once; `herdr plugin action invoke local.grok-bridge.init`
  again only reports that it is waiting. To send it again, run `node plugin/bridge.js init --resend` (or delete
  `team.md` after a completed onboarding to redo it).
- **`herdr agent prompt grok-herdr ...` fails** with "not an active named agent": use `herdr pane run <pane>` for the
  Grok agent; `agent wait`, `agent read`, `agent get` and `agent list` work as usual.
- **Grok Bot asks you to approve commands**: its commands on this PC may need your approval (in tests, file access and
  running node needed none).

## How it works

The agent pane stands for the Grok agent in herdr. Each line typed into it is written to the log and sent to the
routine's webhook. The request carries the task, its id, the working memory and the exact command to answer with.
The Grok agent answers with `node plugin/bridge.js post --reply-to <task id> ...`; the pane sees the answer in the
log, shows it and reports idle, which ends the turn. A run is visible in Grok Bot; a round trip takes about a minute.

The mailbox follows Claude Code's format, with no database:

- `log.jsonl`: every message, one per line, added to the end and never read whole. A line looks like Claude Code's
  transcript: `{"type": "user"|"assistant", "uuid", "parentUuid", "timestamp", "sessionId", "from", "to", "summary",
  "message": {"role", "content"}}` (`user` = a task or message, `assistant` = an answer; `parentUuid` = the message it
  answers). Readers take only the new lines (`read --since`) or one line (`get`).
- `MEMORY.md`: the working memory, at most 40 lines: `## Now` (where the shared work stands) and `## Facts` (lasting
  decisions).

Because each request carries the memory, a Grok run reads nothing else: a turn costs the same however long the
collaboration gets. What the routine receives is a `<webhook_event>` with `headers`, `body_digest`, `body` (the JSON
text of `{"from", "uuid", "message", "memory", "reply"}`) and `timestamp_ms`.
