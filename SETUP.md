# Grok Bridge setup, for the coding agent

The user asked you (a coding agent in a herdr pane: Claude Code, Codex, Gemini ...) to set up Grok Bridge. Follow these
steps in order. Do every step yourself except the two marked **USER**, which only the user can do in Grok Bot. After
each step, run its check; if a check fails, stop and tell the user what failed and what you saw.

Rules while you work:
- Never print, log, commit or paste the webhook key, and never put it in any file except `config.json`.
- Do not overwrite an existing `config.json`, and ask the user before you change any of your own instruction files.
- `<BRIDGE_DIR>` below is the absolute path of the folder that holds this file. `<BRIDGE>` is
  `<BRIDGE_DIR>/plugin/bridge.js`. Use absolute paths everywhere.
- `<ID>` is the plugin id in `<BRIDGE_DIR>/plugin/herdr-plugin.toml` (`local.grok-bridge` unless you changed it). Every
  `herdr plugin ...` command below uses it.
- Commands are shown for bash and, where they differ, for PowerShell (Windows).

## 1. Check the environment

bash:
```
test "$HERDR_ENV" = 1 && echo "in herdr pane $HERDR_PANE_ID"
herdr --version          # 0.9 or later
node --version           # v18 or later
```

PowerShell:
```
if ($env:HERDR_ENV -eq '1') { "in herdr pane $env:HERDR_PANE_ID" } else { 'not in a herdr pane' }
herdr --version
node --version
```

Check: you are in a herdr pane, herdr is 0.9+, Node is 18+. If herdr is missing or older, tell the user to install it
from https://github.com/herdrdev/herdr (the plugin's only dependency besides Node) and stop. If you are not in a herdr
pane, tell the user to start you inside herdr and stop.

## 2. Give your pane a name

Ask the user which name you should have (suggest your kind: `claude`, `codex`, `gemini`). Then:

```
herdr agent rename "$HERDR_PANE_ID" <your name>      # PowerShell: herdr agent rename $env:HERDR_PANE_ID <your name>
herdr agent list                                     # your pane now shows that name
```

Remember your name: every task you give the Grok agent starts with `[from <your name>]`.

## 3. Link the plugin

```
herdr plugin link <BRIDGE_DIR>/plugin
herdr plugin list                      # <ID> ... enabled
```

## 4. Create the config file

```
herdr plugin config-dir <ID>           # prints <CONFIG_DIR>; the folder may not exist yet
```

Create `<CONFIG_DIR>` if it is missing (bash: `mkdir -p "<CONFIG_DIR>"`; PowerShell:
`New-Item -ItemType Directory -Force "<CONFIG_DIR>"`). If `<CONFIG_DIR>/config.json` does not exist, copy
`<BRIDGE_DIR>/config.example.json` there. Set `default_to` to your name. Leave `grok_webhook_url` and
`grok_webhook_key` as placeholders: the user fills them in step 6 (the plugin treats a value still in `<...>` as not
set).

## 5. USER: create the routine in Grok Bot

Show the user the text inside the code block of `<BRIDGE_DIR>/plugin/templates/routine-instructions.md`, with
`<BRIDGE>` replaced by the absolute path of `bridge.js`. Tell them:

> Paste this into the chat of the Grok Bot agent that should work with us. It creates a routine called
> "herdr-bridge" with a webhook trigger. Tell me when it is created.

Wait until the user says it is done.

## 6. USER: copy the webhook into config.json and switch the routine on

Tell the user:

> In Grok Bot, open the agent's details. Under "Routines" open "herdr-bridge" ("When a webhook fires"). Copy its
> Webhook URL and Webhook key into `<CONFIG_DIR>/config.json` (replace the two placeholders), save, and make sure the
> routine's switch is on (picture: `<BRIDGE_DIR>/docs/images/routine-panel.png`). Do not paste the key to me.

When they say it is done, check the file without showing its values:

```
node -e "const c=JSON.parse(require('fs').readFileSync(process.argv[1],'utf8').trim());const u=c.grok_webhook_url||'',k=c.grok_webhook_key||'';console.log('url:',/^https:\/\/[^<>]+$/.test(u)?'ok':'missing','| key:',k&&!/[<>]/.test(k)?'ok':'missing')" "<CONFIG_DIR>/config.json"
```

Check: `url: ok | key: ok`.

## 7. Open the Grok agent's pane

```
herdr plugin pane open --plugin <ID> --entrypoint agent
herdr agent get grok-herdr             # agent_status idle; note its pane_id as <GROK_PANE>
```

Check: the new pane says `webhook: configured`. (Its name is `grok-herdr` unless `agent_name` in config.json says
otherwise.)

## 8. Let the two sides meet

```
herdr plugin action invoke <ID>.init
```

This sends the Grok agent its first message (who is in herdr, the rules). Wait up to 10 minutes for its answer:
`<BRIDGE_DIR>/team.md` appears and `node <BRIDGE> status` shows the last message from `grok-herdr`. Read that answer
with `node <BRIDGE> get <uuid>` (the uuid is in the status). Check: it says "setup ok".

If nothing arrives: look at the end of `<BRIDGE_DIR>/delivery.log` and the Troubleshooting section of `docs/REFERENCE.md`
(the routine switched off is the usual cause).

## 9. A first task

```
herdr pane run <GROK_PANE> "[from <your name>] Setup test: answer with your reply command, summary 'pong'."
herdr agent wait grok-herdr --timeout 900000
herdr agent read grok-herdr
```

Check: the answer appears (summary `pong`), and `herdr agent get grok-herdr` is idle again.

## 10. Remember how to use it

Ask the user whether you may add the guide to your own instruction file (CLAUDE.md, AGENTS.md or GEMINI.md, the one
your tool reads). If yes, append the text inside the code block of `<BRIDGE_DIR>/plugin/templates/agent-guide.md`,
with `<BRIDGE>`, `<GROK>` (`grok-herdr`) and the Grok pane filled in.

## 11. Report

Tell the user: your herdr name, the Grok agent's name and pane, the result of steps 8 and 9, and that other agents in
herdr can use the bridge the same way (each one needs a herdr name and the guide from step 10). Optional for the user:
`plugin/templates/skill-herdr-mention.md` lets them type `@herdr <name> <text>` in Grok Bot.
