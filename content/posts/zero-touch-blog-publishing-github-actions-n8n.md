+++
title = 'How I Turned a git push Into a Live Blog Post and LinkedIn Announcement — Automatically'
date = 2026-03-15T10:00:00+01:00
draft = false
tags = ['n8n', 'GitHub Actions', 'Automation', 'Hugo', 'LinkedIn', 'AI', 'DevOps', 'Workflow']
categories = ['Technology', 'Software Engineering', 'Automation']
series = ['Building in Public']
+++

A few weeks ago, I got tired of the same ritual every time I published a blog post. Write the post, push to GitHub, wait for the build, check the site, then — if I remembered — draft a LinkedIn post to share it. Half the time I'd forget the LinkedIn part entirely, or I'd post it days later when the moment had already passed.

So I built a system that does almost all of it for me. `git push` kicks everything off. The site builds and deploys automatically. n8n detects the new post, generates an AI-written LinkedIn announcement, and then pauses for one deliberate step: I review the draft and approve it before it goes live. That approval gate is intentional — AI-generated text is good, not always great.

Here's exactly how it works.

---

## The Problem with Manual Publishing

My blog runs on [Hugo](https://gohugo.io/), a static site generator. The workflow before automation looked like this:

1. Write a markdown post
2. `git push` to trigger GitHub Actions
3. Wait ~2 minutes for the build and deploy to finish
4. Check the live site to confirm it worked
5. Open LinkedIn, write a post summarizing the article, copy the URL, add hashtags
6. Post (or forget to post)

Steps 1–4 are entirely mechanical. Step 5 is creative but repetitive — every LinkedIn post follows the same pattern: hook, summary, key takeaways, link, hashtags. Step 6 is unreliable by nature.

The question was: **which parts of this can a machine do better than me?**

The answer turned out to be: most of it.

---

## The Architecture: Two Systems, One Pipeline

The solution splits cleanly into two independent systems connected by a single bridge — the GitHub API.

```
Author writes markdown
        |
        | git push
        v
  GitHub (Hugo repo)
        |
        | triggers
        v
  GitHub Actions
  - Checkout + setup Hugo
  - Run hugo build
  - Push HTML to Pages repo
        |
        | new commit appears
        v
  thatsmeadarsh.github.io
  (live website)
        ^
        | polls every 5 min
        |
      n8n (Docker)
  - Detects new commit
  - Fetches original markdown
  - AI generates LinkedIn draft
  - Pauses for author approval
        |
        v (after approval)
  LinkedIn (published immediately)
```

{{< figure src="/images/auto-publish-01-high-level.png" alt="High-level architecture: git push triggers GitHub Actions build, n8n polls, generates AI draft, pauses for approval, then publishes to LinkedIn" caption="The full pipeline — GitHub Actions builds and deploys, n8n detects, generates, and pauses for one approval step before publishing to LinkedIn. The Pages repo commit is the only handoff between the two systems." width="100%" link="/images/auto-publish-01-high-level.png" target="_blank" >}}

**GitHub Actions** owns everything related to building and deploying the site. **n8n** owns everything related to detecting the deployment, generating content, and publishing to LinkedIn. They don't know about each other directly — the bridge is the GitHub API.

---

## Part 1: GitHub Actions Does the Heavy Lifting

The Hugo repo has a workflow file (`hugo.yml`) that triggers on every push to `main`. It does five things in sequence:

{{< figure src="/images/auto-publish-03-github-actions.png" alt="GitHub Actions CI/CD pipeline: 5 stages from Setup to Live" caption="The five stages of the GitHub Actions pipeline — every push to main triggers this full sequence, ending with the website going live on GitHub Pages." width="70%" link="/images/auto-publish-03-github-actions.png" target="_blank" >}}

> If you want to understand how this GitHub Actions pipeline was originally designed — including the Contentful CMS integration and the Hugo theme setup — I covered the full setup in an earlier post: [SSG Integration: Contentful, Hugo, GitHub Actions, and Pages](https://thatsmeadarsh.github.io/posts/ssg-implementation-guide/).

### Stage 1 & 2: Setup + Data Sync

Checks out the code (including Hugo's Ananke theme as a git submodule), installs Hugo, and pulls the latest services data from a Contentful CDN endpoint. This keeps the Services page current without any manual updates.

### Stage 3: Build

Deletes Hugo's cache, runs `hugo --buildFuture` to compile all markdown posts into static HTML (including future-dated posts). The `public/` output folder is not committed back to the source repo — it goes directly to the Pages repo in the next stage.

### Stage 4: Cross-Repo Deploy

This is the key step. The built site lives in a separate repository (`thatsmeadarsh.github.io`) — not the source repo. The workflow clones that repo, wipes its contents, copies the freshly built HTML across, and pushes.

```yaml
# The cross-repo push (simplified)
- name: Push to GitHub Pages repo
  run: |
    git clone https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/thatsmeadarsh/thatsmeadarsh.github.io.git
    rm -rf thatsmeadarsh.github.io/*
    cp -r public/* thatsmeadarsh.github.io/
    cd thatsmeadarsh.github.io
    git add .
    git commit -m "Update site content"
    git push
```

This cross-repo push is what creates the deployment signal that n8n will detect.

### Stage 5: Auto-Deploy

GitHub Pages has its own Action watching the `thatsmeadarsh.github.io` repo. When the new HTML lands there, it automatically deploys to the CDN. The site is live within seconds of the push.

---

## Part 2: n8n Detects, Thinks, and Posts

n8n runs in Docker on my local machine. It has a 14-node workflow that wakes up every 5 minutes.

{{< figure src="/images/n8n-workflow.png" alt="n8n workflow canvas showing all 14 nodes from schedule trigger through Wait for Approval to LinkedIn publish" caption="The complete n8n workflow — polling the GitHub API every 5 minutes, generating an AI LinkedIn draft, pausing for approval, then publishing immediately after the author resumes the execution." width="100%" link="/images/n8n-workflow.png" target="_blank" >}}

### The Polling Approach (and Why Webhooks Didn't Work)

The design went through three iterations before landing on something that works reliably.

{{< figure src="/images/auto-publish-02-evolution.png" alt="Architecture evolution: fswatch watcher → GitHub webhooks + ngrok → GitHub API polling" caption="Three attempts, two failures. The polling approach works because it requires zero inbound connectivity — just outbound HTTPS calls." width="100%" link="/images/auto-publish-02-evolution.png" target="_blank" >}}

My first attempt used GitHub webhooks — cleaner and more immediate than polling. But webhooks require an inbound connection, and I work behind a corporate network with TLS inspection that kills ngrok tunnels. Every tool that needs an inbound connection fails in this environment.

Polling solved it completely. All traffic is outbound HTTPS. No tunnels, no open ports, no special network config. Just a Docker container making API calls.

**The rate limit math**: GitHub allows 5,000 authenticated requests per hour. Polling every 5 minutes uses 288 requests per day — well under 0.1% of the limit.

### Node 1: Schedule Trigger

Wakes the workflow every 5 minutes. No configuration needed beyond the interval.

### Node 2: Fetch Latest Deployment

Makes a GET request to the GitHub API for the latest commit on the Pages repo:

```
GET https://api.github.com/repos/thatsmeadarsh/thatsmeadarsh.github.io/commits/main
```

The response includes the commit SHA and the full list of files changed — this is the raw material for everything that follows.

### Node 3: The State Machine (Extract New Post Slugs)

This is the most interesting node. It uses `$getWorkflowStaticData('global')` — n8n's mechanism for persisting data between workflow executions — to store the last processed commit SHA.

```javascript
const staticData = $getWorkflowStaticData('global');
const latestSha = $input.first().json.sha;

// First run: just store the SHA, don't process
if (!staticData.lastProcessedSha) {
  staticData.lastProcessedSha = latestSha;
  return [];
}

// Already processed this commit
if (staticData.lastProcessedSha === latestSha) {
  return [];
}

// New commit — update state and process
staticData.lastProcessedSha = latestSha;
```

Once a new commit is confirmed, it scans the `files[]` array for newly added post files using a regex:

```javascript
const postFiles = files.filter(f =>
  f.status === 'added' &&
  /^posts\/([^\/]+)\/index\.html$/.test(f.filename)
);
```

This regex is what makes URL construction deterministic. Hugo's build output mirrors the source structure exactly:

{{< figure src="/images/auto-publish-04-url-construction.png" alt="How post URLs are constructed automatically from deployed file paths" caption="The deployed file path IS the URL path — no configuration, no guessing. n8n extracts the slug from the commit's file list and assembles the live URL deterministically." width="100%" link="/images/auto-publish-04-url-construction.png" target="_blank" >}}

The slug extracted from the Pages repo file path **is** the URL path. No configuration, no mapping — it's a property of how Hugo works.

### Nodes 4 & 5: Fetch and Parse the Original Markdown

The Pages repo only has compiled HTML — all the frontmatter metadata is gone. So n8n goes back to the source repo to fetch the original markdown:

```
GET https://raw.githubusercontent.com/thatsmeadarsh/whataboutadarsh/main/content/posts/{slug}.md
```

A Code node then extracts the TOML frontmatter with regex to get the title, date, tags, categories, draft status, and first 500 words of content.

### Node 6: Draft Check

A simple IF node: if `draft === true`, the workflow routes to a no-op Skip node. This is genuinely useful — I can push a draft post to see how it renders on the live site without accidentally announcing it on LinkedIn.

### Nodes 7 & 8: AI Content Generation

A Code node builds a structured prompt using the post's title, tags, categories, and excerpt, then sends it to HuggingFace's inference router:

```
POST https://router.huggingface.co/sambanova/v1/chat/completions
Model: Meta-Llama-3.1-8B-Instruct
```

The system prompt instructs the model to write a LinkedIn post in a specific style: conversational, insight-driven, with relevant hashtags, and ending with a clear call to action. The model runs on SambaNova's hardware via HuggingFace's free tier — fast and zero cost.

### Nodes 9–13: Review Gate and LinkedIn Publishing

After formatting the AI draft, the workflow hits a **Wait node** — a built-in n8n mechanism that pauses execution indefinitely and resumes only when a specific URL is visited. The execution shows as "Waiting" in the n8n UI.

To approve and publish:
1. Open `http://localhost:5678` → **Executions**
2. Find the execution in **Waiting** state — click it
3. In the **Wait for Approval** node output, copy the `resumeUrl`
4. Open that URL in your browser
5. The execution resumes immediately — n8n fetches your LinkedIn profile and publishes the post

```javascript
{
  "lifecycleState": "PUBLISHED",
  "specificContent": {
    "com.linkedin.ugc.ShareContent": {
      "shareCommentary": { "text": aiGeneratedPost },
      "shareMediaCategory": "ARTICLE",
      "media": [{
        "status": "READY",
        "originalUrl": postUrl,
        "title": { "text": postTitle }
      }]
    }
  }
}
```

One thing worth knowing: LinkedIn's UGC API has a `lifecycleState: SCHEDULED` option with a `scheduledPublishTime` field. I tried it. It requires LinkedIn Marketing Partner access and returns a 403 for standard developer apps. The Wait node achieves the same goal — a deliberate review window — without any special API permissions.

LinkedIn crawls the `originalUrl` and generates the article card preview automatically. Since n8n only fires after a new commit appears on the Pages repo, the URL is already live and crawlable when the post publishes.

---

## The Bridge: Why This Design Works

The elegant part of this system is how GitHub Actions and n8n are coupled without direct integration.

GitHub Actions doesn't know n8n exists. It just builds and pushes to the Pages repo — the same thing it was already doing. n8n doesn't trigger GitHub Actions. It just polls the Pages repo — a read-only operation that works regardless of what n8n is doing.

The commit to the Pages repo is the signal. It means exactly one thing: the site is live. n8n reads that signal and acts on it.

This separation gives you fault isolation for free:

| What fails | Website impact | LinkedIn impact |
|---|---|---|
| GitHub Actions build fails | Site not updated | n8n sees no new commit — no action taken |
| n8n crashes | None | Poll resumes on next restart |
| LinkedIn API error | None | Post not created; website already live |
| HuggingFace API down | None | Fallback text used for LinkedIn post |

---

## What I Learned Building This

**Corporate networks are more hostile than you think.** My initial webhook-based design worked perfectly on a home connection. It broke completely in the office. The polling rewrite took an afternoon and eliminated all external dependencies on inbound connectivity.

**Static data in n8n is powerful but invisible.** The `$getWorkflowStaticData()` function persists to the Docker volume — it survives container restarts and n8n updates. It's exactly right for tracking "last processed commit SHA." The catch: if you ever need to reset it, you have to do it through a workflow execution or directly in the database.

**An approval gate beats both immediate and scheduled posts.** I tried LinkedIn's scheduled post API first — it requires Marketing Partner access and throws a 403 for standard developer apps. I almost fell back to posting immediately. Instead I used n8n's Wait node: the execution pauses, I review the AI draft in the n8n UI, and I approve when I'm happy with it. It gives me a real review window without any special API permissions, and the post goes live on my timeline rather than a fixed 24h delay.

**Two-repo Hugo deployments are the right call.** Keeping the source (with frontmatter, drafts, config) separate from the built output (pure HTML) makes a lot of things cleaner. n8n can read from the source repo for markdown context while watching the Pages repo for deployment signals. They serve different purposes.

**NODE_TLS_REJECT_UNAUTHORIZED=0 in Docker is less scary than it sounds.** The flag is scoped to the Node.js process inside the container. It doesn't affect the host system. For a local automation tool making outbound API calls over corporate TLS interception, it's the practical solution.

---

## The Full Stack

| Component | Technology | Purpose |
|---|---|---|
| Static site generator | Hugo | Markdown to HTML |
| Build & deploy | GitHub Actions | CI/CD pipeline |
| Automation engine | n8n 2.11.4 (Docker) | Workflow orchestration |
| AI model | Meta-Llama-3.1-8B (HuggingFace/SambaNova) | LinkedIn post generation |
| Social publishing | LinkedIn UGC API + OAuth2 | Immediate publish after manual approval |
| State persistence | n8n static data (Docker volume) | Commit SHA tracking |

---

## Replicating This

The workflow JSON and full documentation are in the [n8n-powered-auto-web-publish](https://github.com/thatsmeadarsh/n8n-powered-auto-web-publish) repo on GitHub. The setup guide covers every credential, every node configuration, and the specific Docker command to run n8n with the right environment variables.

The core idea is transferable to any static site generator that deploys to a separate repo — Astro, Jekyll, Eleventy. The GitHub polling pattern works for any CI/CD pipeline that commits to a repository as its final step. Swap LinkedIn for Twitter/X or Bluesky and change one API call.

The hardest part was figuring out that the Pages repo commit was the right event to watch — not the Hugo repo push, not a GitHub Actions webhook, not a file watcher. Once that clicked, the rest of the design followed naturally.

---

*The entire pipeline — from this post's markdown file to a live site and an AI-drafted LinkedIn post waiting for approval — ran automatically after `git push`. One review step, by design. That's the point.*
