# Issue #106: Facebook Authentication & Note Submission Fix

## Overview
This document details the resolution for Issue #106 (Facebook Authentication Failure) and related submission stability enhancements on SayThanks.io.

---

## 1. Problem Statement
1. **AttributeError on Callback:** When users authenticated using Facebook OAuth, accounts created via mobile phone or users who restricted email permissions via *"Edit access"* did not have an `email` attribute. The application invoked `.strip()` directly on `user_detail_info.get('email')`, throwing `AttributeError: 'NoneType' object has no attribute 'strip'` (HTTP 500).
2. **Slug Sanitization:** When both `nickname` and `email` were missing, slugs defaulted to raw provider IDs containing pipes (e.g., `facebook|12345`), causing broken URLs (`%7C`) and failure during form submissions (`Send failed: 500`).
3. **Missing Guard in Submit Note:** Submitting notes to invalid or uninitialized inboxes caused internal index errors instead of clean HTTP 404 responses.
4. **UI Fallbacks:** Missing names or avatars rendered `'Welcome, None!'` and broken image links in the inbox.

---

## 2. Technical Solution

### A. Safe Email Extraction (`saythanks/core.py`)
- Extracted and sanitized email safely without assuming non-null string types:
  ```python
  raw_email = user_detail_info.get('email')
  email = raw_email.strip() if isinstance(raw_email, str) and raw_email.strip() else None
  ```
- Gracefully disabled email notifications when email is unavailable:
  ```python
  if not email:
      storage.Inbox.disable_email(final_slug)
  ```

### B. 4-Tier Sanitized Slug Resolution (`saythanks/utils.py`)
- Implemented a 4-tier hierarchy that guarantees URL-safe, clean slugs:
  1. Provider `nickname` (if present and non-empty).
  2. Username prefix from `email` (before `@`).
  3. URL-sanitized full `name` / `given_name` (lowercased, hyphens for special characters).
  4. URL-sanitized Auth0 `userid` (e.g. `facebook-122277967748121649` instead of raw pipes).

### C. Note Submission Guard (`saythanks/core.py` & `saythanks/storage.py`)
- Added `storage.Inbox.does_exist(inbox_id)` and `is_enabled(inbox_id)` guards to `submit_note` to return HTTP 404 on invalid inboxes rather than throwing unhandled 500 errors.
- Guarded `Inbox.auth_id` against empty database result lookups.

### D. UI Fallbacks (`saythanks/templates/inbox.htm.j2`)
- Added Jinja2 template fallbacks for avatar images and user display names:
  ```html
  <img class="avatar" src="{{ user.get('picture') or url_for('static', filename='images/inbox.png') }}" width="100" alt="Avatar">
  <h2>Welcome, {{ user.get('name') or user.get('nickname') }}!</h2>
  ```

### E. Infrastructure & Database Modernization
- **Dockerfile:** Upgraded base image to `python:3.10-slim` with necessary build packages and Gunicorn entrypoint.
- **docker-compose.yaml:** Pinned `postgres:15`, configured healthcheck with `pg_isready`, and unified services on the `saythanks` bridge network.
- **requirements.txt:** Pinned `sqlalchemy<2.0.0` for SQL query compatibility.

---

## 3. Automated Test Suite
All 40 unit test suites pass, including 7 dedicated test cases in `tests/test_auth_callback.py`:
- `test_resolve_nickname_with_nickname`
- `test_resolve_nickname_with_email_only`
- `test_resolve_nickname_with_name_only`
- `test_resolve_nickname_name_sanitization`
- `test_resolve_nickname_fallback_to_userid`
- `test_email_none_safe_extraction`
- `test_email_whitespace_safe_extraction`
