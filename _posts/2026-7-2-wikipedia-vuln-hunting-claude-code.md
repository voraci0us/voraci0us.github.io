---
title: "Popping alert()s on Wikipedia: A Naive Approach to Vuln Hunting with Claude Code"
tags:
- Security
- Vulnerability Research
- Claude Code
- AI
---
<img src="{{ "assets/images/wikipedia-vuln-hunting_cve-screenshot.png" | relative_url }}" style="max-width: 500px;" />

The screenshot shows CVE-2026-58030, a stored XSS vulnerability I discovered and reported in Mediawiki (the software that Wikipedia runs on) last month.

An attacker creates/edits any Wikipedia page to plant their evil JavaScript payload → a victim views the page → the payload runs in the victim's browser. This payload could be used to steal the victim's session token, which lets the attacker log into the user's account without knowing their password. If an admin views the page…. oh boy. You just got a full "unauthenticated user → admin access" attack chain.

Shoutout to the Mediawiki maintainers for patching this quickly, there was a fix live on [wikipedia.org](http://wikipedia.org) 6 hours after I reported the issue!

Not surprisingly, I found this (and four other CVEs) with an LLM. I've been using Claude Code to hunt for vulnerabilities in open-source software, focusing on web applications since that is an area I have experience in. I've had some success so far:
- CVE-2026-58030 - Stored XSS in Mediawiki (Wikipedia)
- CVE-2026-49246 - Arbitrary file write in Jellyfin via malicious MKV file
- CVE-2026-47156 - MantisBT account takeover via broken authentication on SOAP API\*
- GHSA-rq89-jjj9-v3x8 - IDOR that lets you read other users' libraries in Kavita (eBook reader)
- CVE-2026-55066 - IDOR that lets you read other users' tasks in Vikunja (Kanban board)
- CVE-2026-55067 - IDOR that lets you write to other users' boards in Vikunja (Kanban board)

\*=not getting credited because someone already reported it

And I've reported 8 other high-confidence findings that I am waiting to hear back about.

**I found these over the course of two weeks with Claude Code running Sonnet 4.6.** My "harness" consists of a `CLAUDE.md` file and a small Python shim, both can be found here: [https://github.com/voraci0us/cc-vuln-researcher](https://github.com/voraci0us/cc-vuln-researcher). There is no special sauce, no Fable/Mythos (or even Opus) was used, I am not a vulnerability researcher by trade (before this month I had 0 CVEs to my name), you can do all this yourself.

## Note: LLM Usage

If there is any value in a personal blog, it is that there is a human being on the other end of it. All writing on this blog is done myself, the old-fashioned way.

## Initial Thoughts

LLMs are good at reading and understanding source code. For any security assessment where source code is available, they have become indispensable. That being said, a few key observations:
1. It is very important to keep these models focused on a narrow task. As the context window fills up, the hallucinations begin.
2. Past vulnerabilities in a target are a good indication of the kinds of bug classes that exist.
3. Past vulnerabilities very often have incomplete or ineffectual patches.
4. Hallucinations and false positives are a fact of life. You need to have a system for validating any vulnerabilities found.
5. **I found very quickly that another agent looking at the same source code is not an effective way to validate findings. They need to be validated dynamically against a live target - this gives us a tangible, deterministic "correctness" to check for.**

## Environment

We need some place the agent can run 24/7, so I created an Ubuntu virtual machine on my homelab.

Since it's a virtual machine, I can run Claude Code in unrestricted mode (`alias claude='claude --allow-dangerously-skip-permissions --dangerously-skip-permissions'`). This way we don't have to sit there and babysit approvals. This does mean we need to be conscious of any credentials/access to the outside world it has. I only gave it credentials for Linear (discussed later), not GitHub for drafting GHSAs, etc.

I installed Docker and made sure that the user Claude Code is running under has access to the Docker socket. This allows the agent to create live environments to validate potential vulnerabilities against.

I have a small Python script that just runs Claude Code in unrestricted mode on my server, and handles timeouts/retries when I hit limits or Claude Code dies for whatever reasons. The script also cleans up any Docker containers between runs so we don't run out of disk space. I run it in a tmux session so I can pop in and check how things are going if needed.

## Putting It All Together

Initially, I was just playing around in Claude Code and doing a lot of manual prompting. I wanted to move towards an autonomous system. To do this, I needed some means of external input/output.

I ended up setting up Linear, it's a SaaS platform for Jira-like task management. Basically, I can create tasks here and the agent can check in periodically for new work. On a separate project board, the agent can post its findings. I have the Linear app on my phone and I can scroll around whenever without having to log into the server. Nice. The free tier allows up to 100 issues - I exceeded this very quickly and went with their $10/mo plan for a month for convenience (if I was going to keep doing this long-term I would just self-host something).

So, I have a "Research Tasks" project where I define targets we want to perform research on. I never make these tickets myself, I just use the [claude.ai](http://claude.ai) connector and chat there.

<img src="{{ "assets/images/wikipedia-vuln-hunting_linear-tasks.png" | relative_url }}" style="max-width: 500px;" />

For each target / research task (i.e. "MediaWiki"), the agent performs research on past vulnerabilities. It creates a child issue for each bug class (i.e. "Twig SSTI"), as well as a child issue for each high-severity past vulnerability to check for patch completeness.

<img src="{{ "assets/images/wikipedia-vuln-hunting_linear-subtasks.png" | relative_url }}" style="max-width: 500px;" />

Each of those child tickets are completed by a separate subagent so context stays clean. The subagent clones the source code and looks only at the single bug class or patch completeness check it was tasked with. Anything that could be a vulnerability gets created as an "Unvalidated" finding in Linear.

Another subagent takes those "Unvalidated" tickets and attempts to create a PoC for the vulnerability and exploit it against a live target. It sets up this live target with Docker. After live validation, it either enters a "True Positive" or "False Positive" state. We skip validation completely for lower-priority vulnerabilities, there are way too many findings to triage low/informationals.

<img src="{{ "assets/images/wikipedia-vuln-hunting_validation-flow.png" | relative_url }}" style="max-width: 500px;" />

And that's it! The "True Positive" stack grew quicker than I can possibly triage (700+ and counting). Sure - I could have Claude report all 700 itself. But open-source maintainers are feeling the burden of the many "bug bounty warriors" doing this kind of nonsense. To provide actual value to the community, I am focusing on medium/high severity findings in major projects, and manually verifying everything before reporting.

<img src="{{ "assets/images/wikipedia-vuln-hunting_true-positive-stack.png" | relative_url }}" style="max-width: 500px;" />

## Performance Tweaks

After the initial deployment I made some improvements to the CLAUDE.md:
- It kept reporting "lack of ratelimiting" as a vulnerability on sensitive endpoints like account creation. This is not in the threat model of most open-source web apps.
- Same for OIDC/SAML implementation issues, minor implementation mistakes were consistently overprioritized.
- Specific prompting was needed to make it deprioritize vulnerabilities that are only exploitable by admins.

You'll also see references to `agentmemory` ([https://github.com/rohitg00/agentmemory](https://github.com/rohitg00/agentmemory)). I was hoping this would be more useful but I don't think it provided any meaningful value - in practice, any useful memory lived in Linear issue descriptions/comments.

I did get prompt refusals at first, not enough to stop any research but definitely enough to slow it down. After applying for Claude's [Cyber Verification Program](https://support.claude.com/en/articles/14604842-real-time-cyber-safeguards-on-claude) these reduced to a more manageable level.

## Conclusion and General Ramblings
Here's my GitHub repo if you missed it: [https://github.com/voraci0us/cc-vuln-researcher](https://github.com/voraci0us/cc-vuln-researcher)

If you think you need Fable/Mythos, you probably don't. There is plenty of low-hanging fruit in real software that hasn't been caught yet. Small, "stupid" models with good harnessing/orchestration can provide a lot of value.

More generally, serious vulnerability research is an expensive effort. Talent is expensive and hard to find. To recruit this talent and get them to work on targets at scale is something generally best afforded by defense contractors, funded by nation states. If they find vulnerabilities in critical software, they tend to hold onto them to maintain offensive capabilities ([https://en.wikipedia.org/wiki/EternalBlue](https://en.wikipedia.org/wiki/EternalBlue)). This "AI revolution" we're going through tends towards a greater centralization of power by a few major players, the ones with deep enough pockets to recruit the talent who understand how to create/train these models and (more importantly) to buy and run at scale the GPUs necessary for training and serving them. But… I do think it will "democratize" vulnerability research. You can get what used to be "nation-state level results" with a $20/month Claude plan. Intelligence agencies have probably been sitting on [Copy Fail](https://en.wikipedia.org/wiki/Copy_Fail) for years before an LLM helped find it this year.

If maintainers can figure out how to prioritize/keep up with the onslaught of vulnerabilities that are getting reported, I do think we will head towards a state where the average piece of open-source software is much more hardened than it is now.
