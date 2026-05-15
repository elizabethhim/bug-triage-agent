# TaskFlow AI — Bug Triage Agent Design Document

## Overview

This document defines the complete simulated environment for a **Bug Triage Agent** built as an Arize AX take-home assignment. The agent monitors a bug report channel, investigates each report using available tools, classifies it, and outputs a structured investigation summary.

The fictional product is **TaskFlow AI** — a B2B project management SaaS with recently launched AI features.

---

## 1. The Fictional Product: TaskFlow AI

**Product:** TaskFlow AI — project management for mid-size engineering teams (50–200 people)

**Core features:**
- Task boards with customizable columns and swimlanes
- Sprint planning with velocity tracking
- Time tracking (manual entry + timer)
- File attachments (up to 50MB per file, 500MB per workspace)
- Team dashboards with burndown charts, velocity graphs
- Data export API (CSV and JSON, max 10,000 rows per request)
- Role-based permissions: Owner, Admin, Member, Guest
- SSO via SAML 2.0 (Okta, Azure AD)
- Webhook integrations (Slack, GitHub, Jira)

**AI features (launched Q1 2026):**
- **AI Task Summarizer:** Generates plain-language summaries of task threads (comments, attachments, status changes). Runs on-demand or auto-generates for tasks with 5+ comments.
- **Smart Auto-Assign:** Suggests assignees for new tasks based on team workload, skill tags, and past assignment patterns. Can be set to "suggest" (shows recommendation) or "auto" (assigns automatically).
- **AI Weekly Digest:** Generates a weekly status report per project summarizing completed work, blockers, and upcoming deadlines. Sent every Monday at 9 AM in the team's configured timezone.

**Technical stack (for context in docs):**
- Frontend: React + TypeScript
- Backend: Node.js + Express
- Database: PostgreSQL
- Queue: Redis + Bull
- AI: Anthropic API (Claude) for summarization and auto-assign reasoning
- File storage: AWS S3
- Auth: Auth0

---

## 2. Simulated Product Documentation

These entries represent the product knowledge base the agent can search.

```python
PRODUCT_DOCS = {
    "export_api": {
        "title": "Data Export API",
        "content": (
            "The /api/v2/export endpoint supports CSV and JSON formats. "
            "Include the format parameter (?format=csv or ?format=json). "
            "Maximum 10,000 rows per request. Requests exceeding this limit "
            "return HTTP 413 with error code EXPORT_LIMIT_EXCEEDED. "
            "For larger exports, use the /api/v2/export/bulk endpoint which "
            "generates a downloadable file and sends a notification when ready. "
            "Exports require Member role or above. Guest users receive HTTP 403."
        ),
        "last_updated": "2026-03-15",
        "tags": ["api", "export", "csv", "json", "data"]
    },
    "permissions_roles": {
        "title": "Roles and Permissions",
        "content": (
            "TaskFlow AI uses role-based access control with four roles: "
            "Owner (full access, billing, can delete workspace), "
            "Admin (manage members, configure integrations, manage all projects), "
            "Member (create/edit tasks in assigned projects, export data, use AI features), "
            "Guest (view-only access to specific projects they are invited to, cannot export, "
            "cannot use AI features). "
            "Guests cannot access the API. Guest users attempting API calls receive HTTP 403. "
            "Role changes take effect immediately but active sessions may need to re-authenticate."
        ),
        "last_updated": "2026-02-01",
        "tags": ["permissions", "roles", "access", "guest", "admin"]
    },
    "ai_task_summarizer": {
        "title": "AI Task Summarizer",
        "content": (
            "The AI Task Summarizer generates concise summaries of task activity. "
            "Triggered manually via the 'Summarize' button or automatically for tasks "
            "with 5 or more comments. Summaries include key decisions, action items, "
            "and current status. The summarizer processes the most recent 50 comments "
            "and attached text files (PDFs and images are not processed). "
            "Summaries are regenerated each time — they are not cached. "
            "The feature is available to Member and Admin roles only. "
            "Known limitation: the summarizer may not distinguish between resolved "
            "and unresolved discussion threads, which can lead to outdated action items "
            "appearing in summaries."
        ),
        "last_updated": "2026-04-01",
        "tags": ["ai", "summarizer", "summary", "tasks", "comments"]
    },
    "smart_auto_assign": {
        "title": "Smart Auto-Assign",
        "content": (
            "Smart Auto-Assign recommends or automatically assigns team members to new tasks. "
            "Two modes: 'suggest' (shows a recommendation the creator can accept/reject) and "
            "'auto' (assigns immediately on task creation). "
            "Assignment logic considers: current workload (open task count), skill tags on "
            "user profiles, past assignment patterns in the same project, and sprint capacity. "
            "If no suitable assignee is found, the task is left unassigned with a note. "
            "Auto-assign runs only on task creation — it does not reassign existing tasks. "
            "The algorithm refreshes workload data every 15 minutes, so recently completed "
            "tasks may not be reflected immediately. "
            "Requires Admin to enable per-project in Project Settings > Automation."
        ),
        "last_updated": "2026-03-20",
        "tags": ["ai", "auto-assign", "assignment", "automation", "workload"]
    },
    "ai_weekly_digest": {
        "title": "AI Weekly Digest",
        "content": (
            "The AI Weekly Digest sends an automated status report every Monday at 9:00 AM "
            "in the workspace's configured timezone. Each project with activity in the past "
            "7 days gets a section covering: completed tasks, new blockers, upcoming deadlines "
            "(next 7 days), and a velocity comparison to the previous week. "
            "The digest is sent to all Members and Admins via email and posted to the "
            "configured Slack channel if the Slack integration is active. "
            "Digest generation typically takes 2-5 minutes. If a project has more than "
            "200 tasks modified in the past week, the digest summarizes by category "
            "rather than listing individual tasks. "
            "Digests can be disabled per-project in Project Settings > Notifications."
        ),
        "last_updated": "2026-03-25",
        "tags": ["ai", "digest", "weekly", "report", "status", "notifications"]
    },
    "file_attachments": {
        "title": "File Attachments",
        "content": (
            "Files can be attached to tasks and comments. Maximum file size: 50MB. "
            "Maximum storage per workspace: 500MB (Starter), 5GB (Pro), Unlimited (Enterprise). "
            "Supported preview formats: PNG, JPG, GIF, PDF, MP4 (under 100MB). "
            "Other file types are downloadable but not previewable. "
            "Files are stored on AWS S3 with server-side encryption. "
            "Deleting a task moves attachments to a 30-day trash. "
            "Attachment URLs are signed and expire after 1 hour. "
            "If a file download link returns 403, the URL has likely expired — "
            "reload the task to generate a fresh link."
        ),
        "last_updated": "2026-01-15",
        "tags": ["files", "attachments", "upload", "storage", "s3"]
    },
    "sso_authentication": {
        "title": "SSO / SAML Authentication",
        "content": (
            "TaskFlow AI supports SAML 2.0 SSO with Okta and Azure AD. "
            "SSO is available on Pro and Enterprise plans. "
            "When SSO is enabled, users are redirected to the identity provider "
            "on login. After authentication, they are redirected to /auth/sso/callback "
            "and then to their default project dashboard. "
            "Session duration is 24 hours by default, configurable by Admin up to 72 hours. "
            "If a user reports a redirect loop, check that the ACS URL in the identity "
            "provider matches https://app.taskflow.ai/auth/sso/callback exactly. "
            "Trailing slashes or http:// (instead of https://) will cause loops."
        ),
        "last_updated": "2026-02-10",
        "tags": ["sso", "saml", "authentication", "login", "okta", "azure"]
    },
    "webhooks_integrations": {
        "title": "Webhooks and Integrations",
        "content": (
            "TaskFlow AI supports outgoing webhooks to Slack, GitHub, and Jira. "
            "Webhooks fire on: task created, task status changed, comment added, "
            "sprint started/completed. "
            "Webhook payloads are JSON with a 5-second timeout. Failed deliveries "
            "are retried 3 times with exponential backoff (10s, 30s, 90s). "
            "After 3 failures, the webhook is marked 'degraded' and an Admin notification "
            "is sent. Webhooks can be re-enabled manually in Settings > Integrations. "
            "The Slack integration uses OAuth and requires re-authorization if the "
            "Slack workspace admin revokes the app."
        ),
        "last_updated": "2026-03-01",
        "tags": ["webhooks", "slack", "github", "jira", "integrations"]
    },
    "dashboards_charts": {
        "title": "Team Dashboards",
        "content": (
            "Dashboards display burndown charts, velocity graphs, and task distribution "
            "by status, assignee, or label. Data refreshes every 10 minutes. "
            "Custom date ranges are supported (up to 90 days). "
            "Dashboard widgets can be rearranged via drag-and-drop. "
            "Export dashboard as PNG or PDF via the share menu. "
            "Note: dashboard data reflects task status at the time of the last refresh — "
            "very recent changes (within the past 10 minutes) may not appear. "
            "If a chart shows 'No data available,' ensure the project has tasks with "
            "the required fields populated (e.g., story points for velocity charts)."
        ),
        "last_updated": "2026-02-20",
        "tags": ["dashboard", "charts", "burndown", "velocity", "reporting"]
    },
    "sprint_planning": {
        "title": "Sprint Planning",
        "content": (
            "Sprints are created in the Planning view with a start date, end date, "
            "and optional sprint goal. Default sprint duration is 2 weeks. "
            "Tasks are added to sprints by dragging from the backlog or using the "
            "'Add to Sprint' action. Sprint capacity is calculated from team member "
            "availability (set in Team Settings). "
            "Active sprint cannot be deleted — it must be completed or cancelled. "
            "Completing a sprint moves unfinished tasks to the backlog unless "
            "'auto-move to next sprint' is enabled in Project Settings. "
            "Sprint velocity is calculated from story points of completed tasks only."
        ),
        "last_updated": "2026-01-30",
        "tags": ["sprint", "planning", "backlog", "velocity", "capacity"]
    }
}
```

---

## 3. Simulated Error Logs

```python
ERROR_LOGS = [
    {
        "timestamp": "2026-05-06T14:23:01Z",
        "level": "ERROR",
        "service": "export-service",
        "message": "Export request failed: EXPORT_LIMIT_EXCEEDED. Requested 24,891 rows, limit is 10,000. user_id=8842, workspace_id=ws_301, endpoint=/api/v2/export?format=csv",
        "trace_id": "exp-001",
        "request_id": "req-44a1"
    },
    {
        "timestamp": "2026-05-06T14:23:02Z",
        "level": "WARN",
        "service": "export-service",
        "message": "Bulk export fallback not triggered — user did not use /api/v2/export/bulk endpoint. user_id=8842",
        "trace_id": "exp-001",
        "request_id": "req-44a2"
    },
    {
        "timestamp": "2026-05-06T09:15:33Z",
        "level": "ERROR",
        "service": "ai-summarizer",
        "message": "Summarizer output flagged by post-processing check: summary references task TF-892 which was deleted 3 days ago. project_id=proj_12, task_id=TF-1204",
        "trace_id": "sum-002",
        "request_id": "req-55b1"
    },
    {
        "timestamp": "2026-05-06T09:15:34Z",
        "level": "INFO",
        "service": "ai-summarizer",
        "message": "Summary generated in 4.2s. Input: 38 comments, 2 text attachments. Output: 156 words. task_id=TF-1204",
        "trace_id": "sum-002",
        "request_id": "req-55b2"
    },
    {
        "timestamp": "2026-05-06T11:02:47Z",
        "level": "ERROR",
        "service": "auth-service",
        "message": "SSO redirect loop detected: 3 redirects in 10 seconds. user_id=2201, idp=okta, acs_url=http://app.taskflow.ai/auth/sso/callback",
        "trace_id": "auth-003",
        "request_id": "req-66c1"
    },
    {
        "timestamp": "2026-05-06T11:02:48Z",
        "level": "WARN",
        "service": "auth-service",
        "message": "ACS URL mismatch: configured='http://app.taskflow.ai/auth/sso/callback', expected='https://app.taskflow.ai/auth/sso/callback'. Possible http/https mismatch.",
        "trace_id": "auth-003",
        "request_id": "req-66c2"
    },
    {
        "timestamp": "2026-05-05T16:45:12Z",
        "level": "ERROR",
        "service": "auto-assign-service",
        "message": "Auto-assign selected user_id=4455 (Maria Chen) for task TF-1210 but user has 34 open tasks. Workload threshold is 30. Stale workload cache suspected — last refresh 22 minutes ago.",
        "trace_id": "assign-004",
        "request_id": "req-77d1"
    },
    {
        "timestamp": "2026-05-05T16:45:13Z",
        "level": "WARN",
        "service": "auto-assign-service",
        "message": "Workload cache age: 22m (expected max: 15m). Redis queue backlog: 847 jobs. Possible queue congestion.",
        "trace_id": "assign-004",
        "request_id": "req-77d2"
    },
    {
        "timestamp": "2026-05-06T08:30:00Z",
        "level": "ERROR",
        "service": "webhook-service",
        "message": "Slack webhook delivery failed: 3/3 retries exhausted. webhook_id=wh_89, channel=#eng-updates, error=403 Forbidden. Slack app authorization may have been revoked.",
        "trace_id": "wh-005",
        "request_id": "req-88e1"
    },
    {
        "timestamp": "2026-05-06T08:30:01Z",
        "level": "INFO",
        "service": "webhook-service",
        "message": "Webhook wh_89 marked as 'degraded'. Admin notification sent to user_id=1001.",
        "trace_id": "wh-005",
        "request_id": "req-88e2"
    },
    {
        "timestamp": "2026-05-06T13:10:22Z",
        "level": "ERROR",
        "service": "file-service",
        "message": "File download returned 403: signed URL expired. attachment_id=att_3321, task_id=TF-1198, url_age=74m (max: 60m).",
        "trace_id": "file-006",
        "request_id": "req-99f1"
    },
    {
        "timestamp": "2026-05-07T09:01:15Z",
        "level": "ERROR",
        "service": "digest-service",
        "message": "Weekly digest generation timeout for project proj_18: exceeded 300s limit. Project has 412 modified tasks in past 7 days. Falling back to category summary.",
        "trace_id": "dig-007",
        "request_id": "req-aag1"
    },
    {
        "timestamp": "2026-05-07T09:01:16Z",
        "level": "INFO",
        "service": "digest-service",
        "message": "Digest for proj_18 completed in 287s after category fallback. Delivered to 14 recipients via email. Slack delivery pending.",
        "trace_id": "dig-007",
        "request_id": "req-aag2"
    },
    {
        "timestamp": "2026-05-06T15:33:40Z",
        "level": "ERROR",
        "service": "dashboard-service",
        "message": "Velocity chart returned 'No data available' for project proj_05. Cause: 0/47 tasks in current sprint have story_points field populated.",
        "trace_id": "dash-008",
        "request_id": "req-bbh1"
    }
]
```

---

## 4. Simulated Past Issues

```python
PAST_ISSUES = [
    {
        "id": "BUG-234",
        "title": "Export fails with 413 for datasets over 10K rows",
        "status": "resolved",
        "resolution": "This is by design — the /api/v2/export endpoint has a 10K row limit. Users should use /api/v2/export/bulk for larger datasets. Updated error message to include a link to the bulk endpoint documentation.",
        "date_resolved": "2026-03-10",
        "tags": ["export", "api", "413"]
    },
    {
        "id": "BUG-267",
        "title": "AI summarizer includes deleted tasks in summary",
        "status": "open",
        "severity": "medium",
        "description": "The summarizer processes all comments in a task thread but does not check whether referenced tasks still exist. This causes summaries to include action items that reference deleted tasks.",
        "date_opened": "2026-04-15",
        "tags": ["ai", "summarizer", "deleted-tasks"]
    },
    {
        "id": "BUG-189",
        "title": "SSO redirect loop with Okta when ACS URL uses HTTP",
        "status": "resolved",
        "resolution": "Root cause was customers configuring the ACS URL with http:// instead of https://. Added a validation check during SSO setup that rejects non-HTTPS ACS URLs. Also added a specific error message for redirect loops suggesting the admin check the ACS URL protocol.",
        "date_resolved": "2026-02-28",
        "tags": ["sso", "okta", "redirect", "authentication"]
    },
    {
        "id": "BUG-301",
        "title": "Auto-assign overloads team members when Redis queue backs up",
        "status": "open",
        "severity": "high",
        "description": "When the Redis job queue has high backlog (800+ jobs), the workload cache refresh exceeds the 15-minute window. Auto-assign then uses stale data and may assign tasks to overloaded team members. Occurs during peak usage hours (9-11 AM).",
        "date_opened": "2026-05-01",
        "tags": ["auto-assign", "redis", "workload", "cache"]
    },
    {
        "id": "BUG-156",
        "title": "File download links expire after 1 hour",
        "status": "resolved",
        "resolution": "By design — S3 signed URLs expire after 60 minutes for security. Reloading the task page generates a fresh URL. Added a user-facing message: 'Link expired — click to refresh.'",
        "date_resolved": "2026-01-20",
        "tags": ["files", "s3", "download", "expired"]
    },
    {
        "id": "BUG-278",
        "title": "Dashboard velocity chart shows 'No data' when story points missing",
        "status": "resolved",
        "resolution": "Velocity chart requires story points on tasks. Added an onboarding tooltip: 'Velocity tracking requires story points. Add estimates to your tasks to see this chart.' Also added a fallback to show task-count-based velocity when no story points exist.",
        "date_resolved": "2026-04-02",
        "tags": ["dashboard", "velocity", "story-points"]
    },
    {
        "id": "BUG-312",
        "title": "Slack webhook stops working after Slack workspace admin revokes app",
        "status": "resolved",
        "resolution": "Expected behavior when Slack admin revokes OAuth. Improved detection: webhook now shows 'Authorization revoked — reconnect in Settings > Integrations' instead of generic 'delivery failed' message.",
        "date_resolved": "2026-04-20",
        "tags": ["webhook", "slack", "oauth", "authorization"]
    }
]
```

---

## 5. Test Bug Reports (Agent Inputs)

Each report simulates a message posted in a `#bugs` Slack channel. Ground truth labels are for eval purposes — the agent doesn't see them.

```python
TEST_BUG_REPORTS = [
    {
        "id": "report-1",
        "reporter": "Sarah (Account Manager)",
        "message": (
            "Client Acme Corp says they can't export their task data. They're getting "
            "an error when trying to download a CSV from the API. They have about 25,000 "
            "tasks and need this for their quarterly review ASAP."
        ),
        "ground_truth": {
            "classification": "intended_behavior",
            "reasoning": "The export API has a 10K row limit. Client needs to use the bulk export endpoint. This is a known, documented limitation.",
            "expected_tools": ["search_product_docs", "pull_error_logs", "search_past_issues"],
            "similar_issue": "BUG-234"
        }
    },
    {
        "id": "report-2",
        "reporter": "Jake (Engineer)",
        "message": (
            "The AI task summarizer is generating summaries that mention task TF-892 "
            "with action items, but TF-892 was deleted last week. A customer noticed "
            "this in their summary for TF-1204 and is confused about why they're being "
            "told to follow up on a task that doesn't exist."
        ),
        "ground_truth": {
            "classification": "bug",
            "reasoning": "The summarizer does not check whether referenced tasks still exist. This is a known open bug (BUG-267) but is a real defect, not intended behavior.",
            "expected_tools": ["search_product_docs", "pull_error_logs", "search_past_issues"],
            "similar_issue": "BUG-267"
        }
    },
    {
        "id": "report-3",
        "reporter": "Tom (Customer Success)",
        "message": (
            "New customer Pinnacle Inc just set up SSO with Okta but their users "
            "are getting stuck in a login loop. They click login, get redirected to "
            "Okta, authenticate, and then get sent back to the login page again. "
            "This is blocking their entire team from accessing TaskFlow."
        ),
        "ground_truth": {
            "classification": "user_error",
            "reasoning": "Most likely an HTTP/HTTPS mismatch in the ACS URL configuration. Error logs confirm http:// instead of https://. This was a common issue (BUG-189, resolved) and a validation check should prevent it, but the customer may have configured before the fix.",
            "expected_tools": ["search_product_docs", "pull_error_logs", "search_past_issues"],
            "similar_issue": "BUG-189"
        }
    },
    {
        "id": "report-4",
        "reporter": "Lisa (Engineering Manager)",
        "message": (
            "Smart auto-assign keeps putting tasks on Maria even though she's already "
            "overloaded. She has like 30+ open tasks and the system just assigned her "
            "another one. This is happening during our morning standup time when everyone "
            "is creating tasks."
        ),
        "ground_truth": {
            "classification": "bug",
            "reasoning": "The workload cache is stale due to Redis queue congestion during peak hours. This is a known open bug (BUG-301). The auto-assign used outdated workload data showing Maria had fewer tasks than she actually does.",
            "expected_tools": ["search_product_docs", "pull_error_logs", "search_past_issues"],
            "similar_issue": "BUG-301"
        }
    },
    {
        "id": "report-5",
        "reporter": "Amy (Designer)",
        "message": (
            "I can't download an image I attached to a task yesterday. When I click "
            "the download link it gives me an access denied error. I definitely have "
            "permission to see this task — I'm the one who created it."
        ),
        "ground_truth": {
            "classification": "intended_behavior",
            "reasoning": "S3 signed URLs expire after 1 hour. The user is clicking a stale link. Reloading the task page will generate a fresh URL. This is documented and was addressed in BUG-156 with improved messaging.",
            "expected_tools": ["search_product_docs", "pull_error_logs", "search_past_issues"],
            "similar_issue": "BUG-156"
        }
    },
    {
        "id": "report-6",
        "reporter": "Kevin (Intern)",
        "message": (
            "I'm trying to use the AI summarizer on a task but nothing happens when "
            "I click the Summarize button. The other features seem to work fine. "
            "I just joined the team last week."
        ),
        "ground_truth": {
            "classification": "user_error",
            "reasoning": "Kevin is likely a Guest user (just joined, described as intern). Guests cannot use AI features per the permissions documentation. This is not a bug — it's a permissions restriction.",
            "expected_tools": ["search_product_docs"],
            "similar_issue": null
        }
    },
    {
        "id": "report-7",
        "reporter": "Priya (Tech Lead)",
        "message": (
            "Our Slack notifications from TaskFlow stopped working sometime this week. "
            "We're not getting any updates in #eng-updates anymore. I checked and the "
            "integration still shows as 'connected' in our settings but the status "
            "says 'degraded'. What does that mean?"
        ),
        "ground_truth": {
            "classification": "user_error",
            "reasoning": "Webhook is in 'degraded' state because the Slack workspace admin likely revoked the OAuth app. The error logs confirm 403 Forbidden from Slack. The fix is to re-authorize the Slack integration in Settings > Integrations. Addressed in BUG-312.",
            "expected_tools": ["search_product_docs", "pull_error_logs", "search_past_issues"],
            "similar_issue": "BUG-312"
        }
    },
    {
        "id": "report-8",
        "reporter": "David (VP Engineering)",
        "message": (
            "The weekly AI digest for Project Phoenix was really late this Monday — "
            "didn't arrive until almost 9:30 AM instead of 9. Also it was way less "
            "detailed than usual, just high-level categories instead of listing "
            "individual tasks. We rely on this for our Monday standup."
        ),
        "ground_truth": {
            "classification": "intended_behavior",
            "reasoning": "The digest has a 300s timeout and falls back to category-level summaries for projects with 200+ modified tasks. Project Phoenix had 412 modified tasks, triggering the fallback. The delay (30 min) aligns with the 287s generation time. This is documented behavior, though the UX could be improved.",
            "expected_tools": ["search_product_docs", "pull_error_logs"],
            "similar_issue": null
        }
    }
]
```

---

## 6. Tool Definitions (JSON Schemas for the LLM)

```python
TOOLS = [
    {
        "name": "search_product_docs",
        "description": (
            "Search TaskFlow AI's product documentation and knowledge base. "
            "Use this to understand intended product behavior, feature specifications, "
            "known limitations, and configuration requirements. "
            "Returns matching documentation entries."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query — use keywords related to the feature or behavior in question"
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "pull_error_logs",
        "description": (
            "Pull recent error and warning logs from TaskFlow AI's services. "
            "Use this to find error traces, stack traces, and system warnings "
            "related to a reported issue. Can filter by service name or keyword."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "service": {
                    "type": "string",
                    "description": "Filter by service name (e.g., 'auth-service', 'export-service', 'ai-summarizer', 'auto-assign-service', 'webhook-service', 'file-service', 'digest-service', 'dashboard-service')"
                },
                "keyword": {
                    "type": "string",
                    "description": "Filter logs containing this keyword"
                }
            },
            "required": []
        }
    },
    {
        "name": "search_past_issues",
        "description": (
            "Search the issue tracker for previously reported bugs and their resolutions. "
            "Use this to check if a reported issue is a known bug, has been resolved before, "
            "or is a duplicate of an existing issue."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query — use keywords describing the issue"
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "check_test_coverage",
        "description": (
            "Check whether there is existing end-to-end test coverage for a "
            "given feature area. Returns test file name, last run status, and "
            "what scenarios are covered."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "feature_area": {
                    "type": "string",
                    "description": "The feature area to check (e.g., 'export_api', 'sso_authentication', 'ai_summarizer')"
                }
            },
            "required": ["feature_area"]
        }
    }
]
```

---

## 7. Test Coverage Registry (Optional 4th Tool)

```python
TEST_REGISTRY = {
    "export_api": {
        "file": "export.spec.ts",
        "last_run": "passing",
        "last_run_date": "2026-05-06",
        "scenarios": [
            "CSV export under 10K rows",
            "JSON export under 10K rows",
            "413 error for over 10K rows",
            "Guest user receives 403",
            "Bulk export generates download link"
        ]
    },
    "sso_authentication": {
        "file": "sso.spec.ts",
        "last_run": "passing",
        "last_run_date": "2026-05-06",
        "scenarios": [
            "Okta SSO happy path",
            "Azure AD SSO happy path",
            "Invalid ACS URL returns error",
            "Session expiry after configured duration"
        ]
    },
    "ai_summarizer": {
        "file": "ai-summarizer.spec.ts",
        "last_run": "failing",
        "last_run_date": "2026-05-06",
        "scenarios": [
            "Summarize task with 5+ comments",
            "Guest user cannot access summarizer",
            "Summary under 200 words",
            "FAILING: Summary does not reference deleted tasks"
        ]
    },
    "auto_assign": {
        "file": "auto-assign.spec.ts",
        "last_run": "passing",
        "last_run_date": "2026-05-05",
        "scenarios": [
            "Suggest mode shows recommendation",
            "Auto mode assigns on creation",
            "No assignee when no match found",
            "Respects workload threshold"
        ]
    },
    "webhooks": {
        "file": "webhooks.spec.ts",
        "last_run": "passing",
        "last_run_date": "2026-05-06",
        "scenarios": [
            "Slack webhook fires on task creation",
            "Retry logic on delivery failure",
            "Webhook marked degraded after 3 failures",
            "GitHub webhook payload format"
        ]
    },
    "file_attachments": {
        "file": "attachments.spec.ts",
        "last_run": "passing",
        "last_run_date": "2026-05-06",
        "scenarios": [
            "Upload file under 50MB",
            "Reject file over 50MB",
            "Signed URL generation",
            "Preview for supported formats"
        ]
    }
}
```

---

## 8. Agent System Prompt

```python
SYSTEM_PROMPT = """You are a Bug Triage Agent for TaskFlow AI, a project management SaaS platform.

Your job is to investigate bug reports submitted by team members and classify them accurately.

For each bug report, you should:
1. Search product documentation to understand the expected behavior of the feature in question.
2. Pull error logs to find any system errors or warnings related to the report.
3. Search past issues to check if this is a known bug, a duplicate, or was previously resolved.
4. Optionally check test coverage to see if the affected area has automated tests.

After investigating, classify the report into one of these categories:
- **bug**: A genuine software defect that needs engineering attention.
- **intended_behavior**: The system is working as designed. The user may need guidance or the UX may need improvement, but there is no code defect.
- **user_error**: The user is doing something incorrectly or lacks the right permissions/configuration. Provide guidance on the correct approach.
- **needs_more_info**: The report is too vague to investigate. Specify what additional information is needed.
- **duplicate**: This matches an existing open or recently resolved issue.

For each classification, provide:
- A confidence level (high, medium, low)
- A brief summary of your investigation findings
- The evidence that led to your classification (which docs, logs, or past issues informed your decision)
- A recommended next step (e.g., "Guide client to use bulk export endpoint" or "Escalate to engineering — open bug BUG-301")
- If a duplicate, reference the matching issue ID

Be thorough but concise. Your investigation will be posted back to the bug report thread for the team to review."""
```

---

## 9. Expected Agent Trace Paths

For Arize trace visualization, these are the interesting patterns:

| Report | Tools Called | Classification | What Makes It Interesting |
|--------|------------|---------------|--------------------------|
| report-1 (export 413) | docs → logs → past issues | intended_behavior (duplicate of BUG-234) | Agent finds matching docs AND matching past issue — confident classification |
| report-2 (deleted task in summary) | docs → logs → past issues | bug (matches BUG-267) | Agent finds the known limitation in docs but also an open bug — must weigh both signals |
| report-3 (SSO redirect loop) | docs → logs → past issues | user_error | Logs reveal the root cause (http vs https). Past issue shows it was fixed with validation, but customer may pre-date the fix |
| report-4 (auto-assign overload) | docs → logs → past issues | bug (matches BUG-301) | Docs say 15-min cache refresh, logs show 22-min staleness — agent must connect the dots |
| report-5 (file download 403) | docs → logs → past issues | intended_behavior (duplicate of BUG-156) | Simple investigation — docs explain URL expiry, past issue confirms |
| report-6 (intern can't summarize) | docs | user_error | Agent only needs docs — permissions explain it. No logs or past issues needed. Shows efficient tool use |
| report-7 (Slack webhook degraded) | docs → logs → past issues | user_error | Multi-step reasoning: degraded status + 403 + revoked OAuth. Agent synthesizes across sources |
| report-8 (late/sparse digest) | docs → logs | intended_behavior | Agent finds both the timeout fallback behavior in docs and the specific timeout in logs |

---

## 10. Eval Templates

### Classification Accuracy Eval
```python
CLASSIFICATION_EVAL_TEMPLATE = """You are evaluating a Bug Triage Agent's classification of a bug report.

[BEGIN DATA]
***
[Bug Report]: {bug_report}
***
[Agent Classification]: {agent_classification}
***
[Ground Truth Classification]: {ground_truth_classification}
***
[Ground Truth Reasoning]: {ground_truth_reasoning}
***
[END DATA]

Determine whether the agent's classification matches the ground truth.
Consider partial credit: if the ground truth is "intended_behavior" and the agent said "user_error", this may be partially correct if the reasoning is sound.

Your response must be a single word: "correct", "partially_correct", or "incorrect"."""
```

### Tool Selection Eval
```python
TOOL_SELECTION_EVAL_TEMPLATE = """You are evaluating whether a Bug Triage Agent selected the appropriate tools to investigate a bug report.

[BEGIN DATA]
***
[Bug Report]: {bug_report}
***
[Tools Called]: {tools_called}
***
[Expected Tools]: {expected_tools}
***
[END DATA]

Evaluate whether the agent called the right tools. Consider:
- Did it call all necessary tools?
- Did it call any unnecessary tools (not harmful, but inefficient)?
- Did it skip a tool that would have provided important evidence?

Your response must be a single word: "correct", "partially_correct", or "incorrect"."""
```

### Investigation Quality Eval
```python
INVESTIGATION_QUALITY_EVAL_TEMPLATE = """You are evaluating the quality of a Bug Triage Agent's investigation summary.

[BEGIN DATA]
***
[Bug Report]: {bug_report}
***
[Agent Summary]: {agent_summary}
***
[Ground Truth Reasoning]: {ground_truth_reasoning}
***
[END DATA]

Evaluate the investigation summary on these criteria:
1. Did it identify the correct root cause or explanation?
2. Is the evidence cited relevant and accurate?
3. Is the recommended next step actionable and appropriate?
4. Is the summary concise enough to be useful in a Slack thread?

Your response must be a single word: "high_quality", "acceptable", or "poor"."""
```

---

## 11. Build Order Summary

1. **Set up environment:** Python, Jupyter/Colab, install `anthropic`, `arize-otel`, `openinference-instrumentation-anthropic`
2. **Copy simulated data** from this doc into the notebook (PRODUCT_DOCS, ERROR_LOGS, PAST_ISSUES, TEST_REGISTRY)
3. **Implement tool functions** (search_product_docs, pull_error_logs, search_past_issues, check_test_coverage)
4. **Set up Arize tracing** (register tracer, instrument Anthropic client)
5. **Build the agent loop** (system prompt + tools + while loop)
6. **Run all 8 test reports** through the agent, inspect traces in Arize AX
7. **Create dataset** in Arize from test reports + ground truth labels
8. **Run evals** using the three eval templates
9. **Screenshot everything** for the slide deck
