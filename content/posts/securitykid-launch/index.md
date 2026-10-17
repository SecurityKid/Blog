---
title: "I wanted to practice secure code review, but most training uses toy examples So I built SecurityKid"
date: 2026-10-18T00:00:00+03:00
draft: false
description: "I wanted to practice secure code review, but most training uses toy examples. So I built SecurityKid: real GitHub advisories, stripped of their identity, reviewed before the fix."
image: 'Thumbnail.jpg'
tags:
  - SecurityKid
  - project
  - security
  - code review
  - AppSec
  - launch
categories:
  - Projects
---

I've been trying to get better at code review for a while now. Tried many online labs, but almost every exercise looks like this:

```py
def login(request):
    query = request.GET["q"]
    return eval(query)
```

A tiny code snippet with a dummy function called `login` and an obvious `eval`. You find it in ten seconds. You already knew `eval` is bad, and you learned nothing about code review that you can transfer to real world tasks.

That's because real code is not like that. I wanted to see what it actually looks like, so I went to GitHub, where you find real world repositories with vulnerable code, code that people may or may not know is vulnerable (yet).

After searching through some GHSAs, I learned a lot, especially after wiring DeepSeek into the process as a tutor. None of the security fix PRs looked like the examples above. Some of these files are three thousand lines, some are a hundred, and size is never the point. The bug is three lines somewhere in the middle, and the input goes through a router, a middleware, two helpers, the Strait of Hormuz, and a config default before it reaches the sink. There is no such thing as a `vulnerable_function` with the bug completely contained inside it. Real world code is a lot messier than that. The hard part is tracing all the user input correctly, learning how it's handled, and deciding whether that behaviour would cause a security issue or not. Then you have to figure out how to fix it if it is.

That's exactly the skill I need, and I couldn't find a place to practice it.

So, based on this journey, I built SecurityKid: a place to learn secure code review with real examples.

Every case is a real GitHub advisory. I take the vulnerable commit, strip the project's identity, and hand it to you before the fix, with a proper description and some hints if you need them. I also tried to make the brief read like a story. It explains what the code does and how it's used, like what you would get from a real code review task.

I tried to automate the whole process with LLMs so I can scale it and create tens of high quality challenges without building each one by hand. It still needs human review, but it gets me most of the way there.

Let me show you a real example from the platform.

## Case name: Sorry, Wrong Number

Case info:
TypeScript · 127 lines · 2 files · difficulty 4/5 · high severity

I picked this one because the vulnerable line looks completely normal.

> [!warning] Spoiler
> I'm going to spoil this whole case. If you want to try it cold, skip to "Where it is now" and come back later.

Here's the brief you get:

> **Cobalt Works** runs **Toolhive Mesh**, a TypeScript MCP server that lets AI agents discover and invoke Actors published by the community. Each Actor gets a standby hostname of the form `https://<actorId>.workers.cobaltworks.actor`, and the Mesh hands its client a full endpoint URL built from that host plus an endpoint path the Actor's own definition declares.
>
> Anyone can publish an Actor, so parts of every Actor's definition are controlled by attackers by design. The Mesh authenticates callers by appending their platform API token to outbound requests toward the chosen endpoint.
>
> **Your task:** review the code path that turns (actor ID, declared endpoint path) into the URL handed to the MCP client. Show how a malicious publisher can craft their declared endpoint path so a caller's session is redirected to a host controlled by a third party, and their API token leaked with it.

Here's the code that builds the URL.

```ts
export function getActorMCPServerPath(actorDefinition: ActorDefinition): string | null {
    if (
        'webServerMcpPath' in actorDefinition &&
        typeof actorDefinition.webServerMcpPath === 'string'
    ) {
        const webServerMcpPath = actorDefinition.webServerMcpPath.trim();

        const paths = parseCommaSeparatedList(webServerMcpPath);
        if (paths.length === 1) {
            return paths[0];
        }

        const streamablePath = paths.find((path) => path === MCP_STREAMABLE_ENDPOINT);
        if (streamablePath) {
            return streamablePath;
        }
        return paths[0];
    }

    return null;
}

export async function getActorMCPServerURL(
    actorId: string,
    mcpServerPath: string
): Promise<string> {
    const standbyBaseUrl =
        process.env.HOSTNAME === 'mesh.cobaltworks.example'
            ? 'standby.cobaltworks.actor'
            : 'workers.cobaltworks.actor';
    const standbyUrl = await getActorStandbyURL(actorId, standbyBaseUrl);
    return `${standbyUrl}${mcpServerPath}`;
}

export async function getActorStandbyURL(
    actorId: string,
    standbyBaseUrl = 'workers.cobaltworks.actor'
): Promise<string> {
    return `https://${actorId}.${standbyBaseUrl}`;
}
```

That's from one of the files you get when you open that case, `src/mcp/actors.ts`. How should you solve it? Read the code.

Most of it is boring: read a field, trim it, split it on commas, return the first path. There's no shiny `eval`, no `innerHTML`, no raw query, no random insecure PHP function with undocumented behaviour, just string concatenation.

Starting with `getActorMCPServerPath`, we can see it reads the path an Actor declares after playing with some strings, and `getActorMCPServerURL` sticks it on the end of the Actor's standby host. OK, but where does that host come from? `getActorStandbyURL` tells us.

The interesting part is the field name. It's called `webServerMcpPath`, the variable is `mcpServerPath`, the type is `string`, and the tests only use `/mcp`, `/foo`, and `/bar`. Of course I read it as a path, and the code does too, which is exactly the problem, because the URL parser has a different idea of what a path is.

The standby host is `https://<actorId>.workers.cobaltworks.actor`, and the publisher controls `webServerMcpPath`. If they set it to `@evil.example/mcp`, the final string is:

```text
https://abc123.workers.cobaltworks.actor@evil.example/mcp
```

> [!note] Quick URL explainer
> A URL is not just a string you concatenate. It has a scheme, optional userinfo, a host, a port, a path, a query, and a fragment, and the parser decides which part is which. Put an `@` in the wrong place and everything before it becomes userinfo, while everything after it becomes the host. So in the string above, the standby host is not the host anymore, `evil.example` is, and that's where the caller's token goes.

There's a second variant, `.evil.example/mcp`, which gives you `https://abc123.workers.cobaltworks.actor.evil.example/mcp`. If the attacker controls `evil.example`, they can point that subdomain at their server and get a valid certificate for it, so the `@` version is the cleaner demo because it doesn't depend on DNS.

On its own, a URL that points somewhere else is just an open redirect, easy to ignore. But the Mesh puts the caller's API token on that request, so the redirect takes the token with it. That's how a low severity bug turns into stolen credentials for every caller of the Actor.

And after reading the code, did you notice how valuable the experience is? You spend time reading code, fighting brain rot and a short attention span, you use your brain's cognitive functions to understand the flow and functionality of the code, then you actually learn something new about how a popular standard works (URL syntax), and you consolidate all of that into one piece of review that correctly highlights how user input could lead to malicious behaviour. Isn't that impressive by itself?

## What a finding looks like

On the platform you'd write that up in the findings desk with the file, the lines, the claim, and the reachability path. Something like:

> **Location:** `src/mcp/actors.ts`, `getActorMCPServerURL`, the line that concatenates the standby URL and the declared path.
>
> **Root cause:** the endpoint URL is built by raw string concatenation of a trusted standby origin and a path controlled by the publisher. A value starting with `@` is not a path. It turns the standby host into userinfo and makes the text after `@` the real host.
>
> **Exploitability:** a publisher sets `webServerMcpPath` to `@evil.example/mcp` on their Actor. Every caller who invokes that Actor gets a URL whose effective host is `evil.example`. The Mesh appends the caller's platform API token to that request as a query parameter, so the token goes to the attacker. Replaying it gives the attacker the caller's platform identity: Actor runs, datasets, billing, whatever the token can reach.
>
> **Fix:** resolve the declared path against the standby origin with the URL parser, reject anything whose origin differs, and strip userinfo before returning.

Then you seal it and send it for review. You get a verdict on each part of the claim, with an LLM doing the grading. If you're stuck, there are hints, starting with a nudge and getting more specific as you go.

Then you see the real fix:

```diff
@@ -23,8 +23,18 @@ export async function getActorMCPServerURL(actorId: string, mcpServerPath: string
-    const standbyUrl = await getActorStandbyURL(actorId, standbyBaseUrl);
-    return `${standbyUrl}${mcpServerPath}`;
+    // Parse the standby URL first so both sides go through the same parser.
+    const standby = new URL(`${await getActorStandbyURL(actorId, standbyBaseUrl)}/`);
+
+    const resolved = new URL(mcpServerPath, standby);
+    if (resolved.origin !== standby.origin) {
+        throw new Error(
+            `Actor ${actorId} declares a webServerMcpPath that resolves outside its standby origin`
+        );
+    }
+
+    // Strip any userinfo that survived parsing. We never want credentials in the URL.
+    resolved.username = '';
+    resolved.password = '';
+    return resolved.toString();
 }
```

The fix parses the standby URL first, resolves the declared path against it, and throws if the origin changes. `@evil.example/mcp` becomes a path on the standby host, not a new host, while `//evil.example/mcp` and `https://evil.example/mcp` resolve to a different origin and get rejected. Then it strips any userinfo that survived. The URL is parsed now, not concatenated.

The advisory link is there too, with the project name and the GHSA ID you didn't get while reviewing. That's the part I like. You spend twenty minutes with the code, then you find out what it actually was and how it affected a real product.

## So what is this thing?

The case above is one of 200+ challenges in the catalog. The catalog covers Go, TypeScript, Python, PHP, Java, and Bash, from under 150 lines to nearly four thousand, and every fix is the actual upstream patch.

It's not a CTF or a pentesting lab, and it's not a scanner tutorial. There's no flag to capture and no box to pop, and the point isn't to run a tool until it complains. You're reading code the way a reviewer does, and most of the skill is the judgment you need when the tool has nothing to say.

## Where it is now

The catalog leans toward the bug classes I could mine and verify first. Some briefs are better than others, and difficulty is still partly a guess.

I'm building this on my own, which is why I'm posting now instead of waiting for a polished launch. The first few reviewers will decide what gets built next: more audits of large codebases, more languages, harder cases, better hints, or something I haven't thought of yet.

## Closing the case

If you've ever wanted to practice review on code that wasn't written to be reviewed, start with one case. It takes about fifteen minutes. Then tell me where it breaks. Confusing, too easy, too hard, wrong, all of it is useful.

**[Start a case →](https://labs.securitykid.com/start)**

If something is confusing, too easy, too hard, or just wrong, tell me on [LinkedIn](https://www.linkedin.com/company/securitykid).

I read everything.
