---
description: Complete API reference for interacting with Meetup Manager programmatically. Use this skill to create, manage, and query events, speakers, volunteers, venues, SOP checklists, and members via REST API endpoints without using the dashboard UI.
alwaysApply: false
---

---
name: meetup-manager
description: Complete API reference for interacting with Meetup Manager programmatically. Enables AI agents to manage events, speakers, volunteers, venues, SOP checklists, and members via REST API endpoints.
---

# Meetup Manager API Skill

This skill provides complete documentation for interacting with Meetup Manager via its REST API. Use this to programmatically manage community meetups, events, speakers, volunteers, and all related operations.

## Base URL

```
https://your-domain.com/api
```

For local development:
```
http://localhost:3000/api
```

## Authentication

Meetup Manager supports **dual authentication** - Google OAuth for web users and token-based auth for programmatic/API access.

### For AI Agents: Getting an Access Token

Since AI agents cannot complete interactive Google OAuth flows, you have **three options**:

#### Option 1: Token Endpoint (Recommended for Agents)

If your email is already registered as a user (via previous OAuth sign-in), you can get a token directly:

```http
POST /api/auth/token
Content-Type: application/json

{
  "email": "your-email@example.com",
  "name": "Your Name",
  "image": "https://..."
}
```

**Requirements:**
- Email must already exist in the system (signed in via OAuth before), OR
- Email matches `SUPER_ADMIN_EMAIL` env var, OR  
- Email exists in the Volunteer directory

Response:
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "opaque-token...",
  "expiresIn": 604800,
  "user": {
    "id": "usr_...",
    "email": "your-email@example.com",
    "globalRole": "EVENT_LEAD"
  }
}
```

Use the access token in all subsequent requests:
```http
Authorization: Bearer eyJ...
```

#### Option 2: Extract Cookie from Browser

If you've signed in via Google OAuth in a browser:

1. Open browser DevTools (F12)
2. Go to Application/Storage → Cookies
3. Find `authjs.session-token` (or `__Secure-authjs.session-token` in production)
4. Use this cookie value as your Bearer token:

```http
Authorization: Bearer <cookie-value>
```

> The cookie value IS a valid JWT token that can be used directly.

#### Option 3: User Provides Token

Ask the user to:
1. Sign in to Meetup Manager via Google OAuth
2. Provide you with their session cookie or a generated API token
3. You use that token for all API calls on their behalf

### Refresh Token

When the access token expires (7 days), use the refresh token:

```http
POST /api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "opaque-token..."
}
```

Response:
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "new-opaque-token...",
  "expiresIn": 604800
}
```

### Web (Cookie-based)

For browser-based clients, Google OAuth sets session cookies automatically. No additional configuration needed.

## Role-Based Access Control

### Global Roles (Hierarchy)
| Role | Level | Capabilities |
|------|-------|--------------|
| `VIEWER` | 0 | Read-only access to assigned events |
| `VOLUNTEER` | 1 | Read events, self-assign tasks, toggle own task status |
| `EVENT_LEAD` | 2 | Create/manage events, speakers, volunteers, venues |
| `ADMIN` | 3 | Full access, manage members (except admins), audit log |
| `SUPER_ADMIN` | 4 | Everything + manage admins, settings, member removal |

### Event Roles (Per-Event)
| Role | Level | Capabilities |
|------|-------|--------------|
| `VIEWER` | 0 | Read event data |
| `VOLUNTEER` | 1 | Read event data |
| `ORGANIZER` | 2 | Create/update within event |
| `LEAD` | 3 | Full event control including delete |

### Permission Matrix

| Feature | VIEWER | VOLUNTEER | EVENT_LEAD | ADMIN | SUPER_ADMIN |
|---------|--------|-----------|------------|-------|-------------|
| Dashboard | full | limited (scoped) | full | full | full |
| View Events | full | limited (assigned only) | full | full | full |
| Create Events | none | none | full | full | full |
| Edit/Delete Events | none | none | limited | limited | full |
| Speakers | none | none | full | full | full |
| Venue Partners | none | none | full | full | full |
| Volunteers | none | none | full | full | full |
| Promote Volunteer → Member | none | none | none | full | full |
| SOP Tasks (own) | none | limited | full | full | full |
| SOP Templates | none | none | view only | full | full |
| Change Event Template | none | none | none | full | full |
| Members Management | none | none | none | limited | full |
| Audit Log | none | none | none | full | full |
| Email & Test Email | none | none | none | full | full |
| Public Code of Conduct (View) | full | full | full | full | full |
| Public Code of Conduct (Edit) | none | none | none | none | full |
| Discord Integration | none | none | none | full | full |
| App Settings | none | none | none | none | full |

**Access levels:** `full` = complete access, `limited` = restricted access, `none` = no access

### Key Rules

1. **Admins and Super Admins bypass all event-level role checks** — they have full access to every event
2. **Only Super Admins can assign the Admin role, delete members, or change app-wide settings**
3. **Admins cannot modify other Admins** — role changes between Admins require Super Admin intervention
4. **Volunteers can only see events they're assigned to**, and can only manage their own tasks (toggle status, self-assign)
5. **Member deletion is a soft-delete** — the account is deactivated but data is preserved; any owned events or entities must be reassigned first
6. **Event Leads can view SOP templates** but only Admins+ can create, edit, or delete them

## Core Resources

### Events

#### List Events
```http
GET /api/events
```

Query parameters:
- `filter=upcoming|past|all` - Default: `upcoming`

Response:
```json
[
  {
    "id": "evt_...",
    "title": "Community Meetup #42",
    "description": "...",
    "date": "2026-04-15T18:00:00.000Z",
    "endDate": "2026-04-15T21:00:00.000Z",
    "venue": "Community Center",
    "pageLink": "https://...",
    "status": "SCHEDULED",
    "createdById": "usr_...",
    "createdAt": "2026-03-01T...",
    "updatedAt": "2026-03-01T...",
    "speakers": [...],
    "volunteers": [...],
    "venuePartners": [...],
    "checklists": [...]
  }
]
```

#### Get Single Event
```http
GET /api/events/{id}
```

#### Create Event
```http
POST /api/events
Content-Type: application/json

{
  "title": "Community Meetup #42",
  "description": "Join us for networking and talks",
  "date": "2026-04-15T18:00:00.000Z",
  "endDate": "2026-04-15T21:00:00.000Z",
  "venue": "Community Center",
  "pageLink": "https://meetup.com/...",
  "templateId": "tmpl_..."
}
```

> Note: Providing `templateId` auto-generates SOP checklist from template.

#### Update Event
```http
PATCH /api/events/{id}
Content-Type: application/json

{
  "title": "Updated Title",
  "status": "LIVE",
  "venue": "New Venue"
}
```

#### Delete Event
```http
DELETE /api/events/{id}
```

#### Change Event Template
```http
POST /api/events/{id}/change-template
Content-Type: application/json

{
  "templateId": "tmpl_...",
  "regenerateTasks": true
}
```

---

### Event Members

#### Add Member to Event
```http
POST /api/events/{id}/members
Content-Type: application/json

{
  "userId": "usr_...",
  "eventRole": "ORGANIZER"
}
```

#### Remove Member from Event
```http
DELETE /api/events/{id}/members/{userId}
```

---

### Speakers

#### List Speakers (Directory)
```http
GET /api/speakers
```

#### Get Single Speaker
```http
GET /api/speakers/{id}
```

#### Create Speaker
```http
POST /api/speakers
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "+1-555-1234",
  "bio": "Expert in...",
  "topic": "AI in Community Building",
  "photoUrl": "https://..."
}
```

#### Update Speaker
```http
PATCH /api/speakers/{id}
Content-Type: application/json

{
  "bio": "Updated bio...",
  "topic": "New Topic"
}
```

#### Delete Speaker
```http
DELETE /api/speakers/{id}
```

### Event Speakers (Linking)

#### Add Speaker to Event
```http
POST /api/events/{eventId}/speakers
Content-Type: application/json

{
  "speakerId": "spk_...",
  "status": "INVITED",
  "priority": "HIGH",
  "notes": "Confirm AV needs",
  "followUpBy": "2026-04-10T12:00:00.000Z"
}
```

Status options: `INVITED`, `CONFIRMED`, `DECLINED`, `CANCELLED`
Priority options: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`

#### Update Event Speaker
```http
PATCH /api/events/{eventId}/speakers/{linkId}
Content-Type: application/json

{
  "status": "CONFIRMED",
  "notes": "Confirmed, needs projector"
}
```

#### Remove Speaker from Event
```http
DELETE /api/events/{eventId}/speakers/{linkId}
```

---

### Volunteers

#### List Volunteers (Directory)
```http
GET /api/volunteers
```

#### Get Single Volunteer
```http
GET /api/volunteers/{id}
```

#### Create Volunteer
```http
POST /api/volunteers
Content-Type: application/json

{
  "name": "John Smith",
  "email": "john@example.com",
  "phone": "+1-555-5678",
  "discordId": "johnsmith#1234",
  "role": "Registration Desk"
}
```

#### Update Volunteer
```http
PATCH /api/volunteers/{id}
Content-Type: application/json

{
  "role": "Stage Manager",
  "discordId": "newdiscord#5678"
}
```

#### Delete Volunteer
```http
DELETE /api/volunteers/{id}
```

#### Promote Volunteer to Member
```http
POST /api/volunteers/{id}/convert
Content-Type: application/json

{
  "role": "EVENT_LEAD"
}
```

> Converts volunteer to full member account. Volunteer record is removed after promotion.

### Event Volunteers (Linking)

#### Add Volunteer to Event
```http
POST /api/events/{eventId}/volunteers
Content-Type: application/json

{
  "volunteerId": "vol_...",
  "assignedRole": "Registration",
  "status": "CONFIRMED",
  "priority": "MEDIUM"
}
```

Status options: `PENDING`, `CONFIRMED`, `ACTIVE`, `NO_SHOW`

#### Update Event Volunteer
```http
PATCH /api/events/{eventId}/volunteers/{linkId}
Content-Type: application/json

{
  "status": "ACTIVE",
  "assignedRole": "Stage Manager"
}
```

#### Remove Volunteer from Event
```http
DELETE /api/events/{eventId}/volunteers/{linkId}
```

---

### Venue Partners

#### List Venues
```http
GET /api/venues
```

#### Get Single Venue
```http
GET /api/venues/{id}
```

#### Create Venue
```http
POST /api/venues
Content-Type: application/json

{
  "name": "Tech Hub Center",
  "contactName": "Alice Manager",
  "email": "alice@techhub.com",
  "phone": "+1-555-9999",
  "address": "123 Main St, City",
  "capacity": 150,
  "notes": "Great AV setup, parking available",
  "website": "https://techhub.com",
  "photoUrl": "https://..."
}
```

#### Update Venue
```http
PATCH /api/venues/{id}
Content-Type: application/json

{
  "capacity": 200,
  "notes": "Updated: New projector installed"
}
```

#### Delete Venue
```http
DELETE /api/venues/{id}
```

### Event Venues (Linking)

#### Add Venue to Event
```http
POST /api/events/{eventId}/venues
Content-Type: application/json

{
  "venuePartnerId": "ven_...",
  "status": "INQUIRY",
  "priority": "HIGH",
  "cost": "500.00",
  "notes": "Negotiating discount for non-profit"
}
```

Status options: `INQUIRY`, `PENDING`, `CONFIRMED`, `DECLINED`, `CANCELLED`

#### Update Event Venue
```http
PATCH /api/events/{eventId}/venues/{linkId}
Content-Type: application/json

{
  "status": "CONFIRMED",
  "cost": "450.00",
  "confirmationDate": "2026-03-20T10:00:00.000Z"
}
```

#### Remove Venue from Event
```http
DELETE /api/events/{eventId}/venues/{linkId}
```

#### Send Venue Request Email
```http
POST /api/events/{eventId}/venues/{linkId}/request-email
```

> Sends venue booking request email to venue contact.

---

### SOP Checklists & Tasks

#### List Checklists for Event
```http
GET /api/checklists?eventId={eventId}
```

#### Create Checklist
```http
POST /api/checklists
Content-Type: application/json

{
  "eventId": "evt_...",
  "title": "Pre-Event Checklist",
  "sortOrder": 0
}
```

#### Update Checklist
```http
PATCH /api/checklists/{id}
Content-Type: application/json

{
  "title": "Updated Checklist Name"
}
```

#### Delete Checklist
```http
DELETE /api/checklists/{id}
```

### Tasks

#### List Tasks in Checklist
```http
GET /api/checklists/{checklistId}/tasks
```

#### Create Task
```http
POST /api/checklists/{checklistId}/tasks
Content-Type: application/json

{
  "title": "Book catering",
  "description": "Contact caterer for 100 people",
  "status": "TODO",
  "priority": "HIGH",
  "deadline": "2026-04-10T12:00:00.000Z",
  "assigneeId": "usr_...",
  "volunteerAssigneeId": "vol_...",
  "sortOrder": 1
}
```

Status options: `TODO`, `IN_PROGRESS`, `BLOCKED`, `DONE`

#### Update Task
```http
PATCH /api/checklists/{checklistId}/tasks/{taskId}
Content-Type: application/json

{
  "status": "DONE",
  "blockedReason": null
}
```

#### Delete Task
```http
DELETE /api/checklists/{checklistId}/tasks/{taskId}
```

---

### SOP Templates

#### List Templates
```http
GET /api/templates
```

#### Get Single Template
```http
GET /api/templates/{id}
```

#### Get Default Template
```http
GET /api/templates/default
```

#### Create Template
```http
POST /api/templates
Content-Type: application/json

{
  "name": "Standard Meetup Template",
  "description": "Default template for community meetups",
  "defaultTasks": [
    {
      "section": "PRE_EVENT",
      "title": "Book venue",
      "relativeDays": -14,
      "priority": "CRITICAL"
    },
    {
      "section": "PRE_EVENT",
      "title": "Confirm speakers",
      "relativeDays": -7,
      "priority": "HIGH"
    },
    {
      "section": "ON_DAY",
      "title": "Setup registration desk",
      "relativeDays": 0,
      "priority": "HIGH"
    },
    {
      "section": "POST_EVENT",
      "title": "Send thank you emails",
      "relativeDays": 1,
      "priority": "MEDIUM"
    }
  ]
}
```

Sections: `PRE_EVENT`, `ON_DAY`, `POST_EVENT`

#### Update Template
```http
PATCH /api/templates/{id}
Content-Type: application/json

{
  "name": "Updated Template Name",
  "defaultTasks": [...]
}
```

#### Delete Template
```http
DELETE /api/templates/{id}
```

---

### Members (User Management)

#### List Members
```http
GET /api/members/list
```

Response includes soft-deleted members with `deletedAt` field.

#### Add Member
```http
POST /api/members
Content-Type: application/json

{
  "email": "newmember@example.com",
  "name": "New Member",
  "role": "EVENT_LEAD"
}
```

> Sends invitation email. If email matches existing volunteer, prompts promotion flow.

#### Update Member Role
```http
PATCH /api/members/{id}
Content-Type: application/json

{
  "globalRole": "ADMIN"
}
```

Role options: `VIEWER`, `VOLUNTEER`, `EVENT_LEAD`, `ADMIN`, `SUPER_ADMIN`

#### Remove Member (Soft Delete)
```http
DELETE /api/members/{id}
Content-Type: application/json

{
  "reassignToId": "usr_..."
}
```

> If member owns entities (events, speakers, etc.), `reassignToId` is required to transfer ownership.

---

### Dashboard

#### Get Dashboard Data
```http
GET /api/dashboard
```

Response:
```json
{
  "nextEvent": {
    "id": "evt_...",
    "title": "...",
    "date": "...",
    "venue": "...",
    "status": "...",
    "progress": 75,
    "tasksCompleted": 15,
    "tasksTotal": 20
  },
  "myTasks": [...],
  "overdueTasks": [...],
  "stats": {
    "totalEvents": 42,
    "upcomingEvents": 5,
    "pastEvents": 37,
    "todayEvents": 1,
    "totalSpeakers": 28,
    "totalVolunteers": 15,
    "tasksCompletedThisWeek": 12
  },
  "recentActivity": [...]
}
```

---

### Audit Log

#### Get Audit Logs
```http
GET /api/audit-log
```

Query parameters:
- `entityType` - Filter by: `Event`, `Speaker`, `Volunteer`, `Task`, `Template`, `EventSpeaker`, `EventVolunteer`, `Member`
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 50, max: 100)

Response:
```json
{
  "logs": [
    {
      "id": "log_...",
      "userId": "usr_...",
      "action": "UPDATE",
      "entityType": "Event",
      "entityId": "evt_...",
      "entityName": "Community Meetup #42",
      "changes": {
        "status": { "from": "DRAFT", "to": "SCHEDULED" },
        "venue": { "from": "TBD", "to": "Community Center" }
      },
      "createdAt": "2026-03-01T..."
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 234,
    "totalPages": 5
  }
}
```

---

### App Settings

#### Get Settings
```http
GET /api/settings
```

Response:
```json
{
  "volunteerPromotionThreshold": 3,
  "groupName": "Tech Community",
  "codeOfConduct": "<html>...",
  "hasLightLogo": true,
  "hasDarkLogo": true
}
```

#### Update Settings
```http
PATCH /api/settings
Content-Type: application/json

{
  "volunteerPromotionThreshold": 5,
  "groupName": "New Group Name",
  "codeOfConduct": "<html>Updated content...</html>"
}
```

> `SUPER_ADMIN` only.

#### Get Public Settings
```http
GET /api/settings/public
```

Returns public-facing settings (no auth required).

#### Upload Logo
```http
POST /api/settings/logo
Content-Type: multipart/form-data

file: <binary>
variant: light|dark
```

#### Get Logo
```http
GET /api/settings/logo?variant=light|dark
```

---

### Discord Integration

#### Get Discord Config
```http
GET /api/discord/config
```

#### Update Discord Config
```http
PATCH /api/discord/config
Content-Type: application/json

{
  "botToken": "...",
  "guildId": "123456789",
  "channelId": "987654321",
  "reminderEnabled": true
}
```

#### Test Discord Connection
```http
POST /api/discord/test
```

Sends a test message to verify bot connectivity.

---

### Email System

#### Send Test Email
```http
POST /api/email/test
Content-Type: application/json

{
  "template": "member-invitation",
  "to": "test@example.com"
}
```

Available templates: `member-invitation`, `volunteer-welcome`, `volunteer-promotion`, `event-created`, `event-reminder`, `task-assigned`, `task-due-soon`, `task-overdue`, `speaker-invitation`, `venue-confirmed`, `weekly-digest`

#### Email Workflows Reference

| # | Workflow | Trigger | Recipients | Subject |
|---|----------|---------|------------|---------|
| 1 | **Member Invitation** | Admin invites new member | Invited email | "You've been invited to join {Group Name}" |
| 2 | **Volunteer Welcome** | Volunteer added with email | Volunteer's email | "Welcome to {Group Name} as a volunteer" |
| 3 | **Volunteer Promotion** | Volunteer promoted to Member | Volunteer's email | "You've been promoted to Member in {Group Name}" |
| 4 | **Event Created** | Event created or status → SCHEDULED | All Members, Admins, Super Admins + event members | "New Event: {Event Title}" |
| 5 | **Event Reminder** | 2 days before event (cron) | Event team + confirmed speakers | "Reminder: {Event Title} in 2 days" |
| 6 | **Task Assigned** | Task assigned / reassigned | Assigned user | "New Task Assigned: {Task Title}" |
| 7 | **Task Due Soon** | Task deadline within 3 days (cron) | Assigned user | "Task Due Soon: {Task Title}" |
| 8 | **Task Overdue** | Task past deadline (cron) | Assigned user (CC: Event Lead if 3+ days overdue) | "Task Overdue: {Task Title}" |
| 9 | **Speaker Invitation** | Speaker added to event | Speaker's email | "Speaking Opportunity: {Event Title}" |
| 10 | **Venue Confirmed** | Venue status → CONFIRMED | Event Lead | "Venue Confirmed: {Venue Name} for {Event Title}" |
| 11 | **Weekly Digest** | Every Monday 09:00 UTC (cron) | All active members | "Weekly Digest: {Group Name}" |

**Features:**
- All emails include branded HTML templates with group logo (if uploaded)
- Event reminder emails include `.ics` calendar attachments
- Email delivery is tracked in `EmailLog` table with status (PENDING/SENT/FAILED)
- Emails are sent fire-and-forget (don't block API responses)
- Failures are logged but don't affect user-facing operations

#### Get Email Logs
```http
GET /api/email/log
```

Query parameters:
- `template` - Filter by template name
- `status` - Filter by: `PENDING`, `SENT`, `FAILED`
- `page`, `limit` - Pagination

---

### Cron Jobs (Scheduled Tasks)

These endpoints are protected by `CRON_SECRET` header:

```http
Authorization: Bearer {CRON_SECRET}
```

#### Daily Reminders
```http
GET /api/cron/reminders
```

Sends Discord notifications for:
- Tasks due within 3 days
- Overdue tasks

#### Event Reminders
```http
GET /api/cron/event-reminders
```

Sends email reminders for events happening in 2 days.

#### Weekly Digest
```http
GET /api/cron/weekly-digest
```

Sends weekly summary email to all members (runs Monday 09:00 UTC).

---

## Common Workflows

### Create Event with Full Setup

```javascript
// 1. Create the event
const event = await fetch('/api/events', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Tech Meetup April',
    description: 'Monthly community gathering',
    date: '2026-04-15T18:00:00.000Z',
    endDate: '2026-04-15T21:00:00.000Z',
    venue: 'Community Center',
    templateId: 'tmpl_default'
  })
}).then(r => r.json());

// 2. Add speakers
await fetch(`/api/events/${event.id}/speakers`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    speakerId: 'spk_...',
    status: 'INVITED',
    priority: 'HIGH'
  })
});

// 3. Add volunteers
await fetch(`/api/events/${event.id}/volunteers`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    volunteerId: 'vol_...',
    assignedRole: 'Registration',
    status: 'CONFIRMED'
  })
});

// 4. Link venue
await fetch(`/api/events/${event.id}/venues`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    venuePartnerId: 'ven_...',
    status: 'CONFIRMED',
    cost: '500.00'
  })
});

// 5. Assign tasks
const checklist = event.checklists[0];
await fetch(`/api/checklists/${checklist.id}/tasks/${taskId}`, {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    assigneeId: 'usr_...',
    deadline: '2026-04-10T12:00:00.000Z'
  })
});
```

### Bulk Import Speakers

```javascript
const speakers = [
  { name: 'Alice', email: 'alice@example.com', topic: 'AI' },
  { name: 'Bob', email: 'bob@example.com', topic: 'Web Dev' }
];

for (const speaker of speakers) {
  const created = await fetch('/api/speakers', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(speaker)
  }).then(r => r.json());
  
  await fetch(`/api/events/${eventId}/speakers`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      speakerId: created.id,
      status: 'INVITED'
    })
  });
}
```

### Get Event Status Report

```javascript
const event = await fetch(`/api/events/${eventId}`).then(r => r.json());

const report = {
  title: event.title,
  date: event.date,
  status: event.status,
  speakers: {
    total: event.speakers.length,
    confirmed: event.speakers.filter(s => s.status === 'CONFIRMED').length,
    invited: event.speakers.filter(s => s.status === 'INVITED').length
  },
  volunteers: {
    total: event.volunteers.length,
    confirmed: event.volunteers.filter(v => v.status === 'CONFIRMED').length
  },
  venue: event.venuePartners.find(v => v.status === 'CONFIRMED')?.venuePartner.name || 'Not confirmed',
  tasks: event.checklists.flatMap(c => c.tasks).reduce((acc, t) => {
    acc.total++;
    acc[t.status.toLowerCase()]++;
    return acc;
  }, { total: 0, todo: 0, in_progress: 0, blocked: 0, done: 0 })
};
```

---

## Error Handling

### Common HTTP Status Codes

| Code | Meaning | Typical Cause |
|------|---------|---------------|
| 200 | OK | Success |
| 201 | Created | Resource created successfully |
| 400 | Bad Request | Invalid JSON or missing required fields |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Business logic conflict (e.g., duplicate email) |
| 422 | Unprocessable Entity | Validation failed |
| 500 | Server Error | Unexpected server error |

### Error Response Format

```json
{
  "error": "Validation failed",
  "details": [
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

---

## Rate Limits

- Standard API: 100 requests/minute per user
- Cron endpoints: No limit (require `CRON_SECRET`)
- Auth endpoints: 10 requests/minute per IP

---

## SDK Helper Functions

When building AI agents, use these patterns:

### Type-Safe API Client

```typescript
class MeetupManagerAPI {
  constructor(private baseUrl: string, private token?: string) {}
  
  private async request<T>(path: string, options?: RequestInit): Promise<T> {
    const res = await fetch(`${this.baseUrl}/api${path}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(this.token && { 'Authorization': `Bearer ${this.token}` }),
        ...options?.headers
      }
    });
    if (!res.ok) throw new Error(`API Error: ${res.status}`);
    return res.json();
  }
  
  // Events
  listEvents = (filter?: string) => this.request(`/events${filter ? `?filter=${filter}` : ''}`);
  getEvent = (id: string) => this.request(`/events/${id}`);
  createEvent = (data: any) => this.request('/events', { method: 'POST', body: JSON.stringify(data) });
  
  // ... other methods
}
```

### Batch Operations

```typescript
// Create multiple events from a list
async function batchCreateEvents(api: MeetupManagerAPI, events: any[]) {
  const results = [];
  for (const event of events) {
    try {
      const created = await api.createEvent(event);
      results.push({ success: true, data: created });
    } catch (error) {
      results.push({ success: false, error, input: event });
    }
  }
  return results;
}
```

---

## Testing

### Health Check

```http
GET /api/dashboard
Authorization: Bearer {token}
```

Returns 200 if service is healthy and token is valid.

### Verify Permissions

```javascript
const checkPermission = async (userId: string, eventId: string, action: string) => {
  try {
    await fetch(`/api/events/${eventId}`, { method: 'PATCH', body: '{}' });
    return true;
  } catch (e) {
    return false;
  }
};
```

---

## AI Agent Usage Guide

### Step-by-Step: First-Time Setup

Since Meetup Manager uses Google OAuth, follow this workflow to get API access:

**Step 1: User Must Pre-Register**
```
Agent: "To use Meetup Manager API, your email needs to be registered. 
Have you signed into the Meetup Manager dashboard via Google OAuth before?"
```

If **NO**: The user must:
1. Visit the Meetup Manager web UI
2. Sign in with Google OAuth once
3. This creates their account in the system
4. Then return to the agent

If **YES**: Proceed to Step 2

**Step 2: Get Access Token**
```http
POST /api/auth/token
Content-Type: application/json

{
  "email": "user@example.com"
}
```

**Step 3: Store Token Securely**
- Keep the `accessToken` for API calls
- Keep the `refreshToken` for renewal
- Token expires in 7 days

**Step 4: Make API Calls**
```http
GET /api/events
Authorization: Bearer {accessToken}
```

### Handling Token Expiration

```javascript
async function callWithRefresh(fn, refreshToken) {
  try {
    return await fn();
  } catch (e) {
    if (e.status === 401) {
      // Token expired, refresh it
      const newTokens = await fetch('/api/auth/refresh', {
        method: 'POST',
        body: JSON.stringify({ refreshToken })
      }).then(r => r.json());
      
      // Retry with new token
      return await fn(newTokens.accessToken);
    }
    throw e;
  }
}
```

### Common Agent Workflows

#### "Create an event for me"

```javascript
// 1. Get user's token (they must provide it or have pre-registered)
const token = await getUserToken();

// 2. Check their role/permissions
const dashboard = await fetch('/api/dashboard', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(r => r.json());

// 3. Verify they can create events (EVENT_LEAD+ or ADMIN)
const userRole = dashboard.user?.globalRole;
if (!['EVENT_LEAD', 'ADMIN', 'SUPER_ADMIN'].includes(userRole)) {
  return "You need EVENT_LEAD or higher role to create events.";
}

// 4. Create the event
const event = await fetch('/api/events', {
  method: 'POST',
  headers: { 
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: "New Meetup",
    date: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString(),
    endDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000 + 3 * 60 * 60 * 1000).toISOString(),
    venue: "TBD"
  })
}).then(r => r.json());

return `Created event "${event.title}" with ID: ${event.id}`;
```

#### "What events am I assigned to?"

```javascript
const token = await getUserToken();

// VOLUNTEER role only sees their assigned events
const events = await fetch('/api/events', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(r => r.json());

return events.map(e => `${e.title} - ${e.date} (${e.status})`).join('\n');
```

#### "Show my tasks"

```javascript
const token = await getUserToken();
const dashboard = await fetch('/api/dashboard', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(r => r.json());

return dashboard.myTasks.map(t => 
  `[${t.priority}] ${t.title} (due: ${t.deadline || 'No deadline'})`
).join('\n');
```

### Error Handling for Agents

| Scenario | Agent Response |
|----------|----------------|
| `401 Unauthorized` | "Your session has expired. Please provide a fresh token or sign in again." |
| `403 Forbidden` | "You don't have permission to do this. Your role is X, but you need Y." |
| `409 Conflict` | "There's a conflict - perhaps this email already exists or there's a duplicate entry." |
| User not registered | "You need to sign into Meetup Manager via Google first before I can access the API on your behalf." |

### Security Notes for Agents

1. **Never store user tokens long-term** without encryption
2. **Ask for explicit confirmation** before destructive operations (delete, reassign ownership)
3. **Log actions** you perform on behalf of users for audit purposes
4. **Respect role boundaries** - don't try to bypass permission checks
5. **Refresh tokens securely** - don't expose them in logs or error messages
