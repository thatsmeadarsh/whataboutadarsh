+++
title = 'Testing the Two-Workflow Approval Architecture'
date = 2026-03-15T22:45:00+01:00
draft = false
tags = ['n8n', 'Automation', 'Architecture']
categories = ['Technology', 'Automation']
+++

This post validates testing the split-workflow approach where draft generation and human review are decoupled into separate n8n workflows.

## The Design

Workflow 1 runs on a schedule, detects new blog posts, and generates AI LinkedIn drafts automatically. Workflow 2 is a form-triggered workflow that lets you review, edit, and approve the draft before publishing.

This separation avoids n8n's webhook-waiting SQLite bug and gives a cleaner user experience.
