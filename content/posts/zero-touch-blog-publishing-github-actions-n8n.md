+++
title = 'How I Turned a git push Into a Live Blog Post and LinkedIn Announcement — With One Review Step'
date = 2026-03-15T10:00:00+01:00
draft = false
description = 'Build a fully automated blog-to-LinkedIn pipeline using Hugo, GitHub Actions, n8n, and AI — with a form-based human review gate before anything goes live.'
tags = ['n8n', 'GitHub Actions', 'Automation', 'Hugo', 'LinkedIn', 'AI', 'DevOps', 'Workflow']
categories = ['Technology', 'Software Engineering', 'Automation']
series = ['Building in Public']
slug = 'zero-touch-blog-publishing-github-actions-n8n'
+++

A few weeks ago, I got tired of the same ritual every time I published a blog post. Write the post, push to GitHub, wait for the build, check the site, then — if I remembered — draft a LinkedIn post to share it. Half the time I'd forget the LinkedIn part entirely, or I'd post it days later when the moment had already passed.

So I built a system that does almost all of it for me. `git push` kicks everything off. The site builds and deploys automatically. An n8n workflow detects the new post, generates an AI-written LinkedIn draft, and saves it to a queue. When I'm ready, I open a review form, tweak the draft if needed, and hit approve. The post goes live on LinkedIn instantly.

Two n8n workflows, one approval form, zero context switching. Here's exactly how it works.

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

## The Architecture: Three Systems, One Pipeline

The solution splits into three systems. GitHub Actions builds and deploys. n8n Workflow 1 detects the deployment and generates an AI draft. n8n Workflow 2 presents a review form where I approve or reject before anything touches LinkedIn.

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
  n8n Workflow 1 (Docker)
  - Detects new commit
  - Fetches original markdown
  - AI generates LinkedIn draft
  - Saves draft to queue
        |
        | draft ready
        v
  n8n Workflow 2 (Docker)
  - Author opens review form
  - Pre-filled with AI draft
  - Edit, approve, or reject
        |
        v (on approve)
  LinkedIn (published immediately)
```

{{< figure src="/images/auto-publish-01-high-level.png" alt="High-level architecture: git push triggers GitHub Actions build, n8n WF1 polls and generates AI draft, WF2 provides form-based review before LinkedIn publish" caption="The full pipeline — GitHub Actions builds and deploys. WF1 detects, generates, and queues the draft. WF2 provides a form-based review gate. The Pages repo commit is the only handoff between GitHub Actions and n8n." width="100%" link="/images/auto-publish-01-high-level.png" target="_blank" >}}

**GitHub Actions** owns building and deploying the site. **n8n WF1** owns detection, AI generation, and draft storage. **n8n WF2** owns the human review step and LinkedIn publishing. They communicate through the GitHub API (Actions → WF1) and n8n's internal API (WF1 → WF2).

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

## Part 2: n8n Workflow 1 — Detect and Draft

WF1 runs in Docker on my local machine. It has a 10-node workflow that wakes up every 5 minutes, checks for new deployments, and generates an AI LinkedIn draft.

{{< figure src="/images/generate-linkedin-draft-n8n.png" alt="n8n Workflow 1 canvas showing 10 nodes from schedule trigger through AI generation to Save Draft for Review" caption="Workflow 1 — polling the GitHub API every 5 minutes, generating an AI LinkedIn draft, and saving it to a FIFO queue for later review." width="100%" link="/images/generate-linkedin-draft-n8n.png" target="_blank" >}}

### The Polling Approach (and Why Webhooks Didn't Work)

The design went through three iterations before landing on something that works reliably.

{{< figure src="/images/auto-publish-02-evolution.png" alt="Architecture evolution: fswatch watcher → GitHub webhooks + ngrok → GitHub API polling" caption="Three attempts, two failures. The polling approach works because it requires zero inbound connectivity — just outbound HTTPS calls." width="100%" link="/images/auto-publish-02-evolution.png" target="_blank" >}}

My first attempt used GitHub webhooks — cleaner and more immediate than polling. But webhooks require an inbound connection, and I work behind a corporate network with TLS inspection that kills ngrok tunnels. Every tool that needs an inbound connection fails in this environment.

Polling solved it completely. All traffic is outbound HTTPS. No tunnels, no open ports, no special network config. Just a Docker container making API calls.

**The rate limit math**: GitHub allows 5,000 authenticated requests per hour. Polling every 5 minutes uses 288 requests per day — well under 0.1% of the limit.

### The State Machine (Extract New Post Slugs)

This is the most interesting node in WF1. It uses `$getWorkflowStaticData('global')` — n8n's mechanism for persisting data between workflow executions — to store the last processed commit SHA.

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

### Fetch and Parse the Original Markdown

The Pages repo only has compiled HTML — all the frontmatter metadata is gone. So n8n goes back to the source repo to fetch the original markdown:

```
GET https://raw.githubusercontent.com/thatsmeadarsh/whataboutadarsh/main/content/posts/{slug}.md
```

A Code node then extracts the TOML frontmatter with regex to get the title, date, tags, categories, draft status, and first 500 words of content.

### Draft Check

A simple IF node: if `draft === true`, the workflow routes to a no-op Skip node. This is genuinely useful — I can push a draft post to see how it renders on the live site without accidentally announcing it on LinkedIn.

### AI Content Generation

A Code node builds a structured prompt using the post's title, tags, categories, and excerpt, then sends it to HuggingFace's inference router:

```
POST https://router.huggingface.co/sambanova/v1/chat/completions
Model: Meta-Llama-3.1-8B-Instruct
```

The system prompt instructs the model to write a LinkedIn post in a specific style: conversational, insight-driven, with relevant hashtags, and ending with a clear call to action. The model runs on SambaNova's hardware via HuggingFace's free tier — fast and zero cost.

### Save Draft for Review

This is where WF1 ends — and where the old architecture was completely different. Instead of publishing immediately or pausing at a Wait node, the AI-generated draft gets saved to a **FIFO queue** in n8n's static data:

```javascript
const staticData = $getWorkflowStaticData('global');

const draft = {
  title: prevData.title,
  postUrl: prevData.postUrl,
  linkedinText: linkedinText,
  slug: prevData.slug,
  createdAt: new Date().toISOString()
};

if (!staticData.pendingDrafts) {
  staticData.pendingDrafts = [];
}
staticData.pendingDrafts.push(draft);
```

The draft sits in the queue until I'm ready to review it. If multiple posts are pushed before I review, they all queue up — first in, first out.

---

## Part 3: n8n Workflow 2 — Review and Publish

WF2 is where the human comes back into the loop. It's a form-triggered workflow — always active, waiting for me to open the review form.

{{< figure src="/images/review-and-publish-linkedin-n8n.png" alt="n8n Workflow 2 canvas showing form trigger, draft loading, review form, approval branching, LinkedIn publishing, and queue cleanup" caption="Workflow 2 — form-based review with pre-filled AI draft, approve/reject branching, LinkedIn publishing, and automatic queue cleanup." width="100%" link="/images/review-and-publish-linkedin-n8n.png" target="_blank" >}}

### The Review Form

When I'm ready to review, I open `http://localhost:5678/form/linkedin-review-form`. Page 1 is a simple "Load Latest Draft" button. Behind the scenes, WF2 calls n8n's own REST API to read WF1's static data, extracts the oldest pending draft, and pre-fills the review form on page 2.

The form shows three fields — all editable:
- **Post Title** — the blog post title
- **Post URL** — the live blog URL
- **LinkedIn Post Text** — the AI-generated draft

I can tweak the LinkedIn text directly in the form. Fix a weird phrasing, add a personal note, adjust the tone. Then I pick **Approve** or **Reject** and submit.

### Cross-Workflow Communication

This is the interesting engineering part. WF2 doesn't share memory with WF1 — they're independent workflows. WF2 reads WF1's draft queue by calling n8n's public REST API:

```
GET http://localhost:5678/api/v1/workflows/{wf1-id}
Header: X-N8N-API-KEY: {api-key}
```

The response includes WF1's `staticData`, which contains the `pendingDrafts` array. WF2 extracts the oldest draft (FIFO) and pipes it into the review form's default values.

### Why Two Workflows?

I originally built this as a single workflow with an n8n Form node in the middle — the AI generates the draft, then the workflow pauses and shows a form for review. It was elegant in theory. In practice, n8n 2.11.4 has a bug: any node that makes the workflow wait for external input (Wait nodes, Form nodes mid-workflow) triggers a `SQLITE_ERROR: no such column: NaN` error on resume.

Splitting into two workflows avoids the bug entirely. WF1 runs to completion without pausing. WF2 starts fresh from a Form Trigger — no waiting state, no SQLite issue. As a bonus, the two-workflow design is actually cleaner: draft generation is decoupled from review, and the FIFO queue handles multiple pending drafts naturally.

### Approve → LinkedIn

On approval, WF2 fetches my LinkedIn profile, builds the UGC post body, and publishes immediately:

```javascript
{
  "lifecycleState": "PUBLISHED",
  "specificContent": {
    "com.linkedin.ugc.ShareContent": {
      "shareCommentary": { "text": approvedLinkedinText },
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

LinkedIn crawls the `originalUrl` and generates the article card preview automatically. Since WF1 only fires after the Pages repo receives a new commit, the URL is already live and crawlable by the time I approve.

One thing worth knowing: LinkedIn's UGC API has a `lifecycleState: SCHEDULED` option with a `scheduledPublishTime` field. I tried it. It requires LinkedIn Marketing Partner access and returns a 403 for standard developer apps. The form-based review achieves the same goal — a deliberate review window — without any special API permissions.

### Reject → Clean Up

If the AI draft isn't good enough, I hit reject. WF2 removes the draft from WF1's queue by calling the n8n API with a PUT request that updates the `staticData` — shifting the oldest draft off the array. The next time I open the form, the next draft in the queue loads up.

Both the approve and reject paths converge on the same cleanup step, so the queue stays tidy regardless of the outcome.

---

## The Bridge: Why This Design Works

The elegant part of this system is how GitHub Actions and n8n are coupled without direct integration.

GitHub Actions doesn't know n8n exists. It just builds and pushes to the Pages repo — the same thing it was already doing. n8n doesn't trigger GitHub Actions. It just polls the Pages repo — a read-only operation that works regardless of what n8n is doing.

The commit to the Pages repo is the signal. It means exactly one thing: the site is live. WF1 reads that signal and acts on it. WF2 waits for the human to show up.

This separation gives you fault isolation for free:

| What fails | Website impact | LinkedIn impact |
|---|---|---|
| GitHub Actions build fails | Site not updated | WF1 sees no new commit — no action taken |
| n8n WF1 crashes | None | Poll resumes on next restart |
| n8n WF2 form not opened | None | Draft stays in queue until reviewed |
| LinkedIn API error | None | Post not created; website already live |
| HuggingFace API down | None | Fallback text saved to draft queue |

---

## What I Learned Building This

**Corporate networks are more hostile than you think.** My initial webhook-based design worked perfectly on a home connection. It broke completely in the office. The polling rewrite took an afternoon and eliminated all external dependencies on inbound connectivity.

**n8n's Wait node has a SQLite bug in 2.11.4.** Any workflow that pauses for external input — Wait nodes, Form nodes mid-workflow — triggers `SQLITE_ERROR: no such column: NaN` on resume. The fix was splitting into two workflows. WF1 runs to completion. WF2 starts fresh from a Form Trigger. The bug turned out to be a design gift — the two-workflow architecture is cleaner and handles edge cases (like multiple pending drafts) that the single-workflow version couldn't.

**Static data in n8n is powerful but invisible.** The `$getWorkflowStaticData()` function persists to the Docker volume — it survives container restarts and n8n updates. It's exactly right for tracking "last processed commit SHA" and a draft queue. The catch: if you ever need to inspect or reset it, you have to do it through the n8n API or a workflow execution.

**A review form beats a resumeUrl.** My first approval mechanism required copying a webhook URL from n8n's execution detail view and opening it in a browser. It worked but felt fragile and unintuitive. The form-based review in WF2 is a proper UI: pre-filled fields, inline editing, approve/reject buttons. It took more engineering (cross-workflow API calls, FIFO queue management) but the UX is dramatically better.

**Two-repo Hugo deployments are the right call.** Keeping the source (with frontmatter, drafts, config) separate from the built output (pure HTML) makes a lot of things cleaner. n8n can read from the source repo for markdown context while watching the Pages repo for deployment signals. They serve different purposes.

**NODE_TLS_REJECT_UNAUTHORIZED=0 in Docker is less scary than it sounds.** The flag is scoped to the Node.js process inside the container. It doesn't affect the host system. For a local automation tool making outbound API calls over corporate TLS interception, it's the practical solution.

---

## The Full Stack

| Component | Technology | Purpose |
|---|---|---|
| Static site generator | Hugo | Markdown to HTML |
| Build & deploy | GitHub Actions | CI/CD pipeline |
| Draft generation | n8n WF1 (Docker) | Poll, detect, AI generate, queue draft |
| Review & publish | n8n WF2 (Docker) | Form-based review, approve/reject, LinkedIn publish |
| AI model | Meta-Llama-3.1-8B (HuggingFace/SambaNova) | LinkedIn post generation |
| Social publishing | LinkedIn UGC API + OAuth2 | Immediate publish after form approval |
| Draft queue | n8n static data (Docker volume) | FIFO queue for pending drafts |
| Cross-workflow API | n8n REST API + API key | WF2 reads/updates WF1's draft queue |

---

## Replicating This

The workflow JSONs and full documentation are in the [n8n-powered-auto-web-publish](https://github.com/thatsmeadarsh/n8n-powered-auto-web-publish) repo on GitHub. The setup guide covers every credential, every node configuration, and the specific Docker command to run n8n with the right environment variables. You'll import two workflows: one for draft generation, one for review and publish.

The core idea is transferable to any static site generator that deploys to a separate repo — Astro, Jekyll, Eleventy. The GitHub polling pattern works for any CI/CD pipeline that commits to a repository as its final step. Swap LinkedIn for Twitter/X or Bluesky and change one API call.

The hardest part was figuring out that the Pages repo commit was the right event to watch — not the Hugo repo push, not a GitHub Actions webhook, not a file watcher. Once that clicked, the rest of the design followed naturally.

---

*The entire pipeline — from this post's markdown file to a live site and an AI-drafted LinkedIn post waiting in a review form — ran automatically after `git push`. One form, one click, by design. That's the point.*
