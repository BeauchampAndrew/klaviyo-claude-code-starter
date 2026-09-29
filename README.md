# Klaviyo + Claude Code Starter

Everything you need to run your Klaviyo account by talking to Claude Code. Clone it, add your API key, and start asking questions. You won't write any code.

This is the companion repo to **The Beginner's Guide to Claude Code + Klaviyo** on the BS&Co YouTube channel. It's the setup we use to manage our clients' Klaviyo accounts at [BS&Co](https://bsandco.us), with the client details stripped out.

## Setup (about 3 minutes)

You need [Claude Code](https://claude.com/claude-code) and a Klaviyo account you can create API keys in.

### The fast way

Open Claude Code in an empty folder and paste this:

```
Clone https://github.com/BeauchampAndrew/klaviyo-claude-code-starter into this folder and set it up.
```

Claude clones the repo, checks for `uv`, creates your `.env`, and tells you when to paste in your key. Then it tests the connection.

### By hand

```bash
git clone https://github.com/BeauchampAndrew/klaviyo-claude-code-starter
cd klaviyo-claude-code-starter
cp .env.example .env          # then paste your key into .env
bin/klaviyo account-health    # should list your profiles, lists, and metrics
claude
```

If `bin/klaviyo` says uv is missing: `curl -LsSf https://astral.sh/uv/install.sh | sh`

### Getting the API key

In Klaviyo: **Settings > API Keys > Create Private API Key**. Name it something you'll recognize in six months. Pick **Read-only** to start. Copy the key (it starts with `pk_`, and you only see it once) and paste it into `.env`.

Run read-only for your first week. When you want Claude to build segments, edit drafts, or schedule campaigns, make a new key with write access for Campaigns, Segments, Lists, and Profiles.

Your key lives only in `.env`, which is gitignored. Don't paste it into the chat.

## What's in here

| | |
|---|---|
| `CLAUDE.md` | What Claude reads every time it starts in this folder: the Klaviyo tool, the setup steps, and the working rules (read before writing, confirm every change, nothing sends without you) |
| `bin/klaviyo` | Runs [klaviyo-cli](https://github.com/BeauchampAndrew/klaviyo-cli) with your key from `.env`. Nothing to install besides uv |
| `prompts.md` | Every prompt from the video, by level. Copy them, change the details, and run them |
| `.claude/skills/` | Playbooks Claude follows for the bigger jobs: building segments, scheduling campaigns, checking flows, diagnosing performance, and the sunset/re-activation loop |
| `docs/klaviyo-gotchas.md` | Klaviyo traps we've hit in real accounts, so Claude doesn't hit them in yours |
| `examples/segments/` | Starter segment definitions (engaged, category buyers, cross-sell, sunset, re-engaging) |
| `.claude/settings.json` | Lets Claude run read-only Klaviyo commands without asking. Anything that changes your account still asks first |

## Is it safe to point at my real account?

klaviyo-cli blocks deletes at the tool level. Claude can't delete profiles, lists, segments, campaigns, or flows, even through the raw API passthrough. Suppressing or unsubscribing people needs an explicit confirmation. With a read-only key, it can't change anything at all.

Beyond that, `CLAUDE.md` tells Claude to show you every change before making it and never to schedule a send without your yes. Read how we made the CLI safe to hand to an agent: [bsandco.us/blog-post/klaviyo-cli](https://bsandco.us/blog-post/klaviyo-cli)

## More than one Klaviyo account?

Use klaviyo-cli's profiles instead of `.env`. See [multi-account profiles](https://github.com/BeauchampAndrew/klaviyo-cli#multi-account-profiles). Then `bin/klaviyo --profile other-brand account-health`.

## License

MIT. Unofficial, not affiliated with Klaviyo, Inc.
