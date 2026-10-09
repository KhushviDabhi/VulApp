# Nexus Social - Instructor & Training Engineer Guide

> **CONFIDENTIAL LAB DOCUMENTATION**  
> This guide is intended exclusively for training instructors, lab facilitators, and assessment evaluators. It details each deliberately introduced vulnerability, the realistic architectural context, reproduction steps, and remediation code.

---

## 1. Laboratory Overview

Nexus Social has been engineered to simulate a production-grade social networking web application while incorporating subtle, realistic developer oversights. Rather than presenting obvious "capture-the-flag" puzzles, weaknesses stem from genuine server-side logic flaws, missing authorization boundaries, and flawed assumptions about client trust.

---

## 2. Vulnerability Catalog & Walkthroughs

### Vulnerability 1: Insecure Direct Object Reference (IDOR) - Post Deletion & Editing
- **Class / Category**: OWASP A01:2021 – Broken Access Control / CWE-639
- **Affected Files**: `posts/views.py` (`post_edit`, `post_delete`)
- **Root Cause**: The application relies on UI-level conditional rendering (`{% if user == post.author %}`) to show or hide the edit/delete options. On the server side, `post_delete` and `post_edit` only check `if request.user.is_authenticated:`, but omit verifying `if post.author == request.user:`.
- **Reproduction / Exploitation**:
  1. Log in as `bob` (`Password123!`).
  2. Locate a post written by `alice` (e.g. Post ID `1`).
  3. In a proxy (e.g. Burp Suite / OWASP ZAP) or via browser developer tools, submit a POST request to:
     ```http
     POST /post/1/delete/ HTTP/1.1
     Host: 127.0.0.1:8000
     Cookie: sessionid=<bob_session>
     Content-Type: application/x-www-form-urlencoded

     csrfmiddlewaretoken=<token>
     ```
  4. The server deletes Alice's post with a 302 redirect, confirming unauthorized object manipulation.
- **Remediation**:
  ```python
  if post.author != request.user:
      raise PermissionDenied("You do not have permission to delete this post.")
  ```

---

### Vulnerability 2: IDOR on Direct Messaging & Private Conversations
- **Class / Category**: OWASP A01:2021 – Broken Access Control / CWE-639
- **Affected File**: `messaging/views.py` (`conversation_detail`, `delete_conversation`)
- **Root Cause**: When fetching a conversation (`get_object_or_404(Conversation, id=conversation_id)`), the view fails to check whether `request.user` is a member of `conversation.participants.all()`.
- **Reproduction / Exploitation**:
  1. Log in as `charlie`.
  2. Navigate directly to `/messages/1/` (the private conversation between Alice and Bob).
  3. Observe full access to read Alice and Bob's private conversation history and send unauthorized messages into their thread.
- **Remediation**:
  ```python
  if not conversation.participants.filter(id=request.user.id).exists():
      raise PermissionDenied("You are not a participant in this conversation.")
  ```

---

### Vulnerability 3: IDOR on Profile Updates via Client-Supplied Parameter
- **Class / Category**: OWASP A01:2021 – Broken Access Control / CWE-639
- **Affected File**: `profiles/views.py` (`profile_edit`)
- **Root Cause**: The profile edit view inspects `request.POST.get('user_id')`. If present, it retrieves that target user's profile rather than strictly updating `request.user.profile`.
- **Reproduction / Exploitation**:
  1. Log in as `charlie`.
  2. Navigate to `http://127.0.0.1:8000/profiles/edit/`.
  3. Inspect the form and locate `<input type="hidden" name="user_id" value="3">`.
  4. Change `user_id` to `1` (Alice's User ID) and update the bio or display name.
  5. Submit the form. Alice's public profile is updated by Charlie!
- **Remediation**:
  ```python
  # Always bind profile updates exclusively to the authenticated session:
  profile = request.user.profile
  form = ProfileEditForm(request.POST, request.FILES, instance=profile)
  ```

---

### Vulnerability 4: IDOR on Comment Deletion
- **Class / Category**: OWASP A01:2021 – Broken Access Control / CWE-639
- **Affected File**: `posts/views.py` (`comment_delete`)
- **Root Cause**: The endpoint deletes any comment by ID without validating that `comment.author == request.user` or `comment.post.author == request.user`.
- **Reproduction / Exploitation**:
  Submit a POST request to `/comment/<comment_id>/delete/` while authenticated as any user to delete another user's comment.

---

### Vulnerability 5: Insecure File Upload & Stored XSS via Avatar SVG
- **Class / Category**: OWASP A03:2021 – Injection / CWE-434 / CWE-79
- **Affected File**: `profiles/views.py` (`profile_edit`)
- **Root Cause**: The file upload logic permits `.svg` image files without sanitizing XML script blocks, and media files are served directly under `/media/avatars/`.
- **Reproduction / Exploitation**:
  1. Create a file named `avatar_poc.svg`:
     ```xml
     <svg xmlns="http://www.w3.org/2000/svg" onload="alert('XSS-Via-SVG:' + document.domain)">
       <circle cx="50" cy="50" r="40" fill="red" />
     </svg>
     ```
  2. Log in and upload `avatar_poc.svg` as your avatar in `/profiles/edit/`.
  3. When an evaluator or victim opens the avatar image directly or within an SVG context, the JavaScript executes.
- **Remediation**:
  Disallow SVG uploads, strictly validate image content using `Pillow.Image.open()`, or sanitize SVG XML trees using `defusedxml` and serve user content with `Content-Security-Policy: default-src 'none'`.

---

### Vulnerability 6: Stored Cross-Site Scripting (XSS) in Profile Bio & Website
- **Class / Category**: OWASP A03:2021 – Injection / CWE-79
- **Affected File**: `templates/profiles/profile_detail.html`
- **Root Cause**: 
  - `profile.bio` is rendered with the `|safe` template filter (`{{ profile.bio|safe }}`).
  - `profile.website` is rendered inside an `href` attribute without protocol whitelisting (`<a href="{{ profile.website }}">`).
- **Reproduction / Exploitation**:
  1. In the profile edit page, enter the following bio:
     ```html
     <script>alert('Stored XSS in Bio: ' + document.cookie);</script>
     ```
     Or enter the website link:
     ```
     javascript:alert(document.domain)
     ```
  2. Save and view the profile page (`/profiles/<username>/`). The script executes immediately or upon clicking the website link.
- **Remediation**:
  Remove `|safe` filter so Django auto-escapes HTML entities, and validate that websites begin strictly with `http://` or `https://`.

---

### Vulnerability 7: Reflected Cross-Site Scripting (XSS) in Search
- **Class / Category**: OWASP A03:2021 – Injection / CWE-79
- **Affected File**: `templates/social/search.html`
- **Root Cause**: The search query parameter is echoed back using `{{ query|safe }}`.
- **Reproduction / Exploitation**:
  Navigate to:
  ```
  http://127.0.0.1:8000/social/search/?q=<script>alert('Reflected-XSS')</script>
  ```
  The script executes upon page render.
- **Remediation**:
  Remove the `|safe` filter in `templates/social/search.html`.

---

### Vulnerability 8: Cross-Site Request Forgery (CSRF) on Follow Endpoint
- **Class / Category**: OWASP A01:2021 – Broken Access Control / CWE-352
- **Affected File**: `social/views.py` (`toggle_follow`)
- **Root Cause**: The `@csrf_exempt` decorator is applied to `toggle_follow`. Furthermore, the view handles `GET` and `POST` requests without checking HTTP methods.
- **Reproduction / Exploitation**:
  Host an external HTML PoC page:
  ```html
  <img src="http://127.0.0.1:8000/social/follow/alice/" style="display:none;" />
  ```
  When a logged-in user visits the attacker's page, their browser automatically triggers a follow action to `@alice`.
- **Remediation**:
  Remove `@csrf_exempt` and require `POST` with CSRF tokens.

---

### Vulnerability 9: Excessive Information Disclosure via System Status
- **Class / Category**: OWASP A05:2021 – Security Misconfiguration / CWE-200
- **Affected File**: `core/views.py` (`system_status_view`, `custom_500`)
- **Root Cause**: The `/system/status/` endpoint does not require administrative authorization and discloses platform architecture, Python runtime version, database paths, and secret key fragments. In addition, the 500 error page dumps stack traces when `?debug=1` is provided.
- **Reproduction / Exploitation**:
  Navigate to `http://127.0.0.1:8000/system/status/` unauthenticated or as an ordinary user.
- **Remediation**:
  Restrict diagnostic views to `@user_passes_test(lambda u: u.is_superuser)` and ensure `DEBUG = False` in production.

---

### Vulnerability 10: Insecure Business Logic - Self-Follow & Stat Inflation
- **Class / Category**: Insecure Business Logic / CWE-840
- **Affected File**: `social/views.py` (`toggle_follow`)
- **Root Cause**: The follow logic does not reject a user attempting to follow their own account (`follower == following`), allowing users to inflate their follower counts.
- **Remediation**:
  ```python
  if request.user == target_user:
      messages.error(request, "You cannot follow yourself.")
      return redirect('profiles:detail', username=target_user.username)
  ```

---

## 3. Laboratory Reset Procedures

To reset the entire lab database and re-seed all 11 fictional accounts, run:

```bash
python manage.py reset_lab
```

This clears transient posts, reverts user profiles, restores standard passwords (`Password123!`), and reinstates the sample conversation threads.
