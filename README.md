# Nexus Social - Educational Penetration Testing Lab

Nexus Social is a realistic, production-style social media web application built with Python, Django, SQLite, HTML5, CSS3, and JavaScript. It provides a modern, authentic user experience featuring user feeds, rich profiles, media uploads, interactive discussions, direct messaging, notifications, and connections.

The application has been engineered specifically for **authorized, offline local penetration-testing laboratories** and web application security training.

> **LEGAL & ETHICAL NOTICE:** This software is intended strictly for authorized educational training and security assessment within a private, local lab environment (`127.0.0.1`). Never deploy or expose this training instance to the public internet.

---

## 1. System Architecture & Features

Nexus provides normal social networking features without separate REST APIs or API documentation:

- **Authentication & Sessions**: Registration, session-based login/logout, password change, and local account recovery workflows.
- **Profiles & Identity**: Display names, custom bios, location, websites, customizable avatars, and cover banners.
- **Social Graph**: Following and follower connections, suggestions widget, and profile relationship discovery.
- **Posts & Content**: Interactive post publishing, media attachments, automated hashtag extraction (`#topics`), threaded comments, likes, and feed sorting.
- **Messaging & Alerts**: Direct private messaging conversations between registered users and real-time interaction notifications.
- **Search & Discovery**: Live content search and community explore stream.

---

## 2. Installation & Quickstart

### Prerequisites
- Python 3.10+ (Python 3.12 recommended)
- Git (optional)

### Setup Instructions

1. **Activate Environment**
   Open a terminal in the project directory:
   ```powershell
   # Windows PowerShell
   .\venv\Scripts\Activate.ps1
   ```
   *(Or create a new environment: `python -m venv venv` and activate it)*

2. **Install Dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Apply Database Migrations**
   ```bash
   python manage.py migrate
   ```

4. **Seed Fictional Test Lab Data**
   Populate the lab database with 11 fictional personas, sample posts, hashtags, conversations, likes, and notifications:
   ```bash
   python manage.py seed_data
   ```

5. **Start the Local Development Server**
   Bind strictly to localhost:
   ```bash
   python manage.py runserver 127.0.0.1:8000
   ```
   Open your browser and navigate to: `http://127.0.0.1:8000/`

---

## 3. Pre-Seeded Testing Accounts

For rapid assessment, the application comes seeded with 11 fictional personas. All fictional accounts use the uniform lab password:

- **Default Lab Password**: `Password123!`

| Username | Persona | Specialty / Role |
| :--- | :--- | :--- |
| `alice` | Alice Vance | Security Researcher |
| `bob` | Bob Martinez | Cloud & DevOps Engineer |
| `charlie` | Charlie Davies | Full-Stack Developer |
| `diana` | Diana Prince | Product & UX Designer |
| `edward` | Edward Norton | Data Scientist & AI |
| `fiona` | Fiona Gallagher | Mobile App Developer |
| `george` | George Clark | Open Source Maintainer |
| `hannah` | Hannah Abbott | Product Manager |
| `ian` | Ian Malcolm | QA & Reliability Engineer |
| `julia` | Julia Roberts | Developer Advocate |
| `admin` | System Ops | Platform Administrator |

---

## 4. Resetting the Lab

To revert the entire application database back to a clean initial state at any point during testing, execute:

```bash
python manage.py reset_lab
```

This wipes all transient modifications, restores the original post timeline, and resets all seeded user accounts.

---

## 5. Running the Automated Test Suite

A functional test suite covering registration, login, posting, commenting, following, and direct messaging is included:

```bash
python manage.py test
```

---

## 6. Production Hardening Checklist (Remediation Guide)

Before any production deployment of a platform like Nexus, the following security controls must be implemented:

1. **Strict Object-Level Authorization**: Enforce authorization checks in every view that modifies or deletes data (e.g. `post.author == request.user`, `comment.author == request.user`, and check that `request.user` is in `conversation.participants.all()`).
2. **Server-Side Identity Binding**: Never accept client-supplied `user_id` parameters in POST payloads for profile or settings updates. Always bind updates strictly to `request.user`.
3. **Template Context Escaping**: Remove all `|safe` template filters on user-generated content (biographies, search queries, notifications). Sanitize external URLs to allow only `http://` and `https://` protocols, blocking `javascript:` schemes.
4. **Strict File Upload Validation**: Validate image MIME types with magic byte signatures (e.g., using Pillow image verification), reject executable extensions and SVG files containing scripts, or serve media from an isolated, cookieless subdomain with strict `Content-Security-Policy`.
5. **CSRF Protection on All Mutating Actions**: Ensure `@csrf_exempt` is removed from all state-changing endpoints, requiring valid CSRF tokens for all state mutations.
6. **Production Settings Isolation**: Set `DEBUG = False`, define strict `ALLOWED_HOSTS`, generate a cryptographically strong `SECRET_KEY` from environment variables, and enforce HTTPS (`SECURE_SSL_REDIRECT = True`, `SESSION_COOKIE_SECURE = True`).
