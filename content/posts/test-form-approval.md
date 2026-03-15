+++
title = 'Testing the Form-Based Approval Flow'
date = 2026-03-15T22:00:00+01:00
draft = false
tags = ['n8n', 'Automation', 'Workflow']
categories = ['Technology', 'Automation']
+++

This post tests the new form-based approval step in the n8n auto-publish pipeline.

## What Changed

The original webhook-based approval had a SQLite bug in n8n 2.11.4 that prevented the resume URL from working. The new approach uses n8n's built-in Form node which presents a web form where you can:

- Review the AI-generated LinkedIn post
- Edit the text before publishing
- Approve or reject with a single click

This is a much better workflow for human-in-the-loop AI content review.
