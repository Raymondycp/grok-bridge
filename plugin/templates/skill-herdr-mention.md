Optional. Paste into the Grok Bot agent's chat so that "@herdr <name> <text>" typed by you reaches a herdr agent:

```
Please save a permanent skill named "herdr-mention".
Only when I myself type a message whose first word is "@herdr", followed by an agent name and text (never for webhook or
bridge messages):
  1. run `herdr agent list`, find the agent by name (or pane id); if it is ambiguous or unknown, ask me;
  2. if its state is "blocked", send nothing and tell me;
  3. otherwise run: herdr agent prompt <pane> "[from <your herdr name>, relaying me] <the rest of my message, verbatim>"
     (pass the text so that quotes in it are safe), then tell me it was delivered;
  4. if I ask for the answer: herdr agent wait <pane> --timeout 600000, then herdr agent read <pane>.
Message passing only: never send keys, never answer an approval dialog, never run anything else in a pane.
```
