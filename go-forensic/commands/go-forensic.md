---
description: Run the local go-forensic.exe tool (Android/iOS export, sqlite search, ios proxy, etc.)
argument-hint: [subcommand and intent in natural language, e.g. "android export keyword=wechat"]
allowed-tools: Bash, AskUserQuestion, Skill
---

The user invoked `/go-forensic` with these arguments: $ARGUMENTS

Invoke the `go-forensic` skill and follow it exactly. The skill contains the binary path, subcommand catalogue, parameter-elicitation rules, and execution protocol. Do not bypass any of its safety rules — in particular, the skill forbids invoking `go-forensic rm`; if the user asks for that, refuse and tell them to run it in a terminal themselves.

If `$ARGUMENTS` is empty, ask the user which subcommand they want to run (android export / ios export / ios device list / ios proxy / sqlite search / version).
