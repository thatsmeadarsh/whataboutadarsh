+++
title = 'Testing the End-to-End Auto Publish Pipeline'
date = 2026-03-15T21:30:00+01:00
draft = false
tags = ['n8n', 'Automation', 'Testing', 'CI/CD']
categories = ['Technology', 'Automation']
+++

This is a test post to verify the full n8n auto-publish pipeline works end to end.

## What We Are Testing

The complete flow from `git push` to LinkedIn post generation:

1. Push a new markdown post to the Hugo source repo
2. GitHub Actions builds and deploys to GitHub Pages
3. n8n detects the new commit via polling
4. Fetches the markdown, parses frontmatter
5. Generates an AI LinkedIn post via HuggingFace
6. Waits for manual approval
7. Posts to LinkedIn

If you are reading this on the live site, step 1 through 2 worked. If a LinkedIn post appeared, the whole pipeline is operational.

## Why Automated Testing Matters

Every pipeline needs a smoke test. This post serves as one — a deliberate trigger to confirm that every node in the n8n workflow executes without errors and credentials resolve correctly during actual execution.
