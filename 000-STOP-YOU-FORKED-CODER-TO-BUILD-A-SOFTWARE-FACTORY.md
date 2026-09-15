# Stop. You forked Coder to build a software factory.

You asked two questions: what this repo actually is, and whether you could run a software factory on it.

Date of the conversation: 2026-09-15
Session: Grok Build
Repo: trite8q1/coder (fork of coder/coder)

This file is the two prompts and the two replies, verbatim. Nothing else.

---

## Prompt 1

what is that exactly? https://github.com/coder/coder

is that basically a self hosted cloud where i can run my agents in multiple sandboxes/containers? so for example: i type a prompt in claude code, then it goes into the cloud and runs those agents in a sandbox? or how does it work?
so is it more like a control panel for all my agents?

---

## Response 1

Coder is a **self-hosted cloud development environment platform**. The agent stuff is a newer layer on top of that. It is not primarily “a control panel that takes your local Claude Code prompt and fans it out into sandboxes.”

## What it actually is

You run a Coder server on your own machines (laptop, homelab, AWS, Kubernetes, whatever). Admins write **Terraform templates** that describe a workspace: Docker container, Kubernetes pod, EC2 VM, etc.

A developer (or an agent) clicks “create workspace,” Coder provisions that machine, and you connect to it over a WireGuard tunnel from VS Code, Cursor, JetBrains, a browser terminal, or SSH. Idle workspaces shut themselves down.

Think **self-hosted GitHub Codespaces**, not E2B or Modal. The workspace is a real remote computer with your repo, tools, and IDE. Isolation comes from “this is its own container/VM,” not from a tiny per-command sandbox.

## How agents fit in

There are three different modes, and they are not the same product:

**1. Claude Code inside a workspace (the common one)**

You do not type a prompt on your laptop and have Coder teleport it. You move *into* the workspace, then run Claude Code there.

```
your laptop  →  Coder dashboard  →  remote workspace (Docker/K8s/VM)
                                      └── Claude Code, git, tests, MCP
```

The laptop is a thin client. Claude Code, the files, and the shell all live in the remote box. Templates can preinstall Claude Code, lock down network (Agent Boundaries), and inject settings so people do not have to sandbox it themselves.

**2. Coder Agents (Coder’s own agent)**

This *is* closer to a control panel. You prompt in Coder’s web UI or API. Coder’s own Go agent loop runs in the **control plane**, calls Anthropic/OpenAI/etc., and sends tool calls (read file, run command) into a workspace over the same tunnel your IDE uses. LLM keys never enter the workspace. This is for background/ticket-style work, not “I pair with Claude Code in my editor.”

**3. Agent Relay (closest to the flow you described)**

This one matches your mental model more:

```
you prompt in Claude Code / Cursor (their UI)
        ↓
Anthropic/Cursor still runs the agent loop + model
        ↓
Coder provisions a workspace on your infra
        ↓
file edits and shell run there, not in the vendor’s cloud
```

Cursor and Claude Code are the supported providers. Inference stays with the vendor. Execution moves onto machines you control.

## Direct answers

**Self-hosted cloud where agents run in multiple sandboxes/containers?**
Half right. It is a self-hosted cloud of **dev workspaces**. Those workspaces can host agents, and you can spin up many in parallel. They are full environments, not disposable “run this snippet” sandboxes.

**Type a prompt in local Claude Code, then it runs in the cloud?**
Not with classic Coder. That is Agent Relay, or you SSH into a workspace and run Claude Code there. Local Claude Code on your Mac does not automatically dump work into Coder.

**Control panel for all your agents?**
It is a control panel for **workspaces** (who has them, what image they use, cost, shutdown, audit). Agent chats are a feature of that. It is not a multi-agent orchestrator like LangGraph, and it does not magically manage every Claude Code session on every laptop unless you put those sessions on Coder.

## When it is the right tool

Use Coder if you want:

- standardized remote machines for humans *and* agents
- many isolated Claude Code boxes without everyone DIY-ing Docker
- enterprise stuff: SSO, templates, network policy, audit, no API keys on laptops

Skip it if you just want “my local Claude Code prompt runs in a throwaway sandbox.” That is Claude Code on the web, Docker sandboxes, or Agent Relay specifically, not Coder-the-whole-platform.

Short version: Coder is the **factory that builds the computers**. Agents are one kind of worker you can put on those computers.

---

## Prompt 2

so could i build a software factory that way? if you know what i mean

---

## Response 2

Yes. That is the actual pitch, not a stretch.

A software factory here means: work comes in as tickets, a machine stamps out an isolated environment for each job, an agent implements it, tests run, a PR comes out, a human reviews. Coder is the factory floor. It is not the whole factory.

## The loop

```
backlog (GitHub issue, Linear ticket, spec)
        ↓
Coder template  →  workspace (container/VM)
        ↓
agent (Claude Code in the box, or Coder Agents in the control plane)
        ↓
tests / lint / preview
        ↓
PR
        ↓
human review  →  merge  →  workspace dies
```

You can run that N times in parallel. Ten tickets, ten workspaces, ten agents. Same image, same tools, same network policy, same audit trail.

Coder already has a version of this: label a GitHub issue, a workspace boots, Claude Code works the ticket, a PR opens. Agent Relay is the same idea with Claude Code’s own UI as the front door.

## What Coder actually gives you

- **The floor.** One Terraform template = the standard workstation. Repo, toolchain, Claude Code, tests, MCP, network allowlist.
- **Isolation.** Each job is its own container or VM. Agent A cannot trash agent B’s tree, your laptop, or prod credentials.
- **Scale-out.** Many workspaces from one template. Idle ones shut down so you are not paying for a farm of idle boxes.
- **Governance.** SSO, audit, no API keys on laptops, egress policy. That matters once agents run unattended.

## What you still have to build

Coder does not decide *what* to build, *whether it is good*, or *how jobs queue*.

You still need:

1. **Intake.** Issue templates, specs, acceptance criteria. Garbage in, garbage out, just faster.
2. **A dispatcher.** Something that says “this ticket is ready, spin a workspace, prompt the agent.” GitHub Actions, a small worker, Coder’s API. Not magic.
3. **Definition of done.** Tests, types, preview URL, “open a PR that passes CI.” If that is not in the template and the prompt, the factory produces slop.
4. **Human gates.** Merge, product judgment, security. Agents are cheap juniors, not the release manager.
5. **Cost and queueing.** Unbounded parallel Claude sessions will melt a bill. Cap concurrency.

Without those, you have a sandbox farm, not a factory.

## Two shapes

**Pairing factory (humans + agents)**
People live in Coder workspaces. Claude Code is preinstalled. Good for a team that wants “every engineer gets a clean, identical machine, and the agent is already there.”

**Ticket factory (mostly unattended)**
Issues in, PRs out. Coder Agents or Claude Code as a background worker. Humans only review. This is the software factory people mean.

Start with one template and one ticket type (bugs with a failing test, or “add this endpoint”). Do not start with “the whole backlog.”

## When it is overkill

If this is you, a homelab, and a few agents: Docker Compose plus a script that clones a repo, runs Claude Code, opens a PR is the same factory at 5% of the ops. Coder starts to pay off when you want many people, many parallel jobs, SSO, audit, and one template that everyone and every agent is forced to use.

So yes, you can build a software factory that way. Coder is the plant: standardized machines, isolation, shutdown, a dashboard. The factory part is the ticket → workspace → agent → CI → PR loop you put on top of it.
