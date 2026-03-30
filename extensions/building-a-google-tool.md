---
title: Build a Google API Tool
description: "Build a Google Calendar tool from scratch with Rust and OAuth 2.0"
---

In this tutorial you will build **google-calendar-tool** from scratch — a WASM tool that lets your agent list, create, update, and delete Google Calendar events using the Google Calendar API v3.

By the end you will have a working tool your agent can call like this:

> "Schedule a team sync for next Tuesday at 3pm for 1 hour"

The complete source code for this tool is available on GitHub:

<Card title="google-calendar-tool source" icon="github" href="https://github.com/nearai/ironclaw/tree/main/tools-src/google-calendar">
  Browse the full implementation — `src/lib.rs`, `src/api.rs`, `src/types.rs`, `Cargo.toml`, and `google-calendar-tool.capabilities.json`.
</Card>

---

## Prerequisites

If you don't have Rust yet, install it from [rustup.rs](https://rustup.rs):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Then add the WASM target:

```bash
rustup target add wasm32-wasip2
```

You will also need a Google Cloud project with the Calendar API enabled and OAuth 2.0 credentials. See steps 1–4 in [Google Calendar setup](/extensions/google-calendar) for the full Cloud Console walkthrough.

---

## 1. Create the project

```bash
cargo new --lib google-calendar-tool
cd google-calendar-tool
```

Replace the generated `Cargo.toml` with:

```toml Cargo.toml
[package]
name = "google-calendar-tool"
version = "0.1.0"
edition = "2021"
description = "Google Calendar integration tool for IronClaw (WASM component)"

[lib]
crate-type = ["cdylib"]

[dependencies]
wit-bindgen = "=0.36"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = "s"
lto = true
strip = true
codegen-units = 1

[workspace]
```

<Note>
`crate-type = ["cdylib"]` tells Cargo to produce a dynamic library — the format WASM components require. `[workspace]` stops Cargo from merging this crate into a parent workspace.
</Note>

Create three source files you will fill in throughout this tutorial:

```
src/
├── lib.rs    ← WIT wiring, schema, execute dispatch
├── api.rs    ← Google Calendar API v3 calls
└── types.rs  ← Request/response types
```

---

## 2. Wire up the WIT interface

Every IronClaw tool is a WASM component that implements a WIT interface. The host provides HTTP, logging, and secrets capabilities; your tool exports `execute`, `schema`, and `description`.

Create `src/lib.rs`:

```rust src/lib.rs
mod api;
mod types;

use types::GoogleCalendarAction;

wit_bindgen::generate!({
    world: "sandboxed-tool",
    path: "../../wit/tool.wit",
});

struct GoogleCalendarTool;

impl exports::near::agent::tool::Guest for GoogleCalendarTool {
    fn execute(req: exports::near::agent::tool::Request) -> exports::near::agent::tool::Response {
        match execute_inner(&req.params) {
            Ok(result) => exports::near::agent::tool::Response {
                output: Some(result),
                error: None,
            },
            Err(e) => exports::near::agent::tool::Response {
                output: None,
                error: Some(e),
            },
        }
    }

    fn schema() -> String {
        SCHEMA.to_string()
    }

    fn description() -> String {
        "Google Calendar integration for viewing, creating, updating, and deleting calendar \
         events. Requires a Google Calendar OAuth token with the calendar.events scope. \
         Supports timed events, all-day events, attendees, locations, and free text search."
            .to_string()
    }
}

export!(GoogleCalendarTool);
```

<Note>
The `wit/tool.wit` file ships with IronClaw. If you are building inside the IronClaw repo under `tools-src/my-tool/`, the path `../../wit/tool.wit` is correct. For a standalone directory, copy `wit/tool.wit` from the repo root and adjust the path.
</Note>

---

## 3. Define the action types

The tool receives JSON parameters from the LLM. Use a Rust enum tagged with `"action"` so `serde_json` routes each request to the right variant automatically.

Create `src/types.rs`:

```rust src/types.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
#[serde(tag = "action", rename_all = "snake_case")]
pub enum GoogleCalendarAction {
    ListEvents {
        #[serde(default = "default_calendar_id")]
        calendar_id: String,
        #[serde(default)]
        time_min: Option<String>,
        #[serde(default)]
        time_max: Option<String>,
        #[serde(default = "default_max_results")]
        max_results: u32,
        #[serde(default)]
        query: Option<String>,
    },
    GetEvent {
        #[serde(default = "default_calendar_id")]
        calendar_id: String,
        event_id: String,
    },
    CreateEvent {
        #[serde(default = "default_calendar_id")]
        calendar_id: String,
        summary: String,
        #[serde(default)]
        description: Option<String>,
        #[serde(default)]
        location: Option<String>,
        #[serde(default)]
        start_datetime: Option<String>,
        #[serde(default)]
        end_datetime: Option<String>,
        #[serde(default)]
        start_date: Option<String>,
        #[serde(default)]
        end_date: Option<String>,
        #[serde(default)]
        timezone: Option<String>,
        #[serde(default)]
        attendees: Vec<String>,
    },
    UpdateEvent {
        #[serde(default = "default_calendar_id")]
        calendar_id: String,
        event_id: String,
        #[serde(default)]
        summary: Option<String>,
        #[serde(default)]
        description: Option<String>,
        #[serde(default)]
        location: Option<String>,
        #[serde(default)]
        start_datetime: Option<String>,
        #[serde(default)]
        end_datetime: Option<String>,
        #[serde(default)]
        start_date: Option<String>,
        #[serde(default)]
        end_date: Option<String>,
        #[serde(default)]
        timezone: Option<String>,
        #[serde(default)]
        attendees: Option<Vec<String>>,
    },
    DeleteEvent {
        #[serde(default = "default_calendar_id")]
        calendar_id: String,
        event_id: String,
    },
}

fn default_calendar_id() -> String { "primary".to_string() }
fn default_max_results() -> u32 { 25 }
```

Then add the response types to the same file:

```rust src/types.rs
/// A Google Calendar event.
#[derive(Debug, Serialize)]
pub struct Event {
    pub id: String,
    pub summary: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub description: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub location: Option<String>,
    pub start: EventTime,
    pub end: EventTime,
    pub status: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub html_link: Option<String>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub attendees: Vec<Attendee>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub organizer: Option<Organizer>,
}

/// Either a timed event (`date_time`) or an all-day event (`date`).
#[derive(Debug, Serialize)]
pub struct EventTime {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub date: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub date_time: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub time_zone: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct Attendee {
    pub email: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub display_name: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub response_status: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct Organizer {
    pub email: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub display_name: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct ListEventsResult {
    pub events: Vec<Event>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub next_page_token: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct EventResult {
    pub event: Event,
}

#[derive(Debug, Serialize)]
pub struct DeleteResult {
    pub deleted: bool,
    pub event_id: String,
}
```

---

## 4. Implement the execute dispatch

Add `execute_inner` to `src/lib.rs`. It checks for the OAuth token first, then dispatches to the right API call:

```rust src/lib.rs
fn execute_inner(params: &str) -> Result<String, String> {
    // Fail fast if credentials are missing.
    if !crate::near::agent::host::secret_exists("google_oauth_token") {
        return Err(
            "Google OAuth token not configured. \
             Run `ironclaw tool auth google-calendar` to set up OAuth, \
             or set the GOOGLE_OAUTH_TOKEN environment variable."
                .to_string(),
        );
    }

    let action: GoogleCalendarAction =
        serde_json::from_str(params).map_err(|e| format!("Invalid parameters: {}", e))?;

    crate::near::agent::host::log(
        crate::near::agent::host::LogLevel::Info,
        &format!("Executing Google Calendar action: {:?}", action),
    );

    let result = match action {
        GoogleCalendarAction::ListEvents { calendar_id, time_min, time_max, max_results, query } => {
            let r = api::list_events(&calendar_id, time_min.as_deref(), time_max.as_deref(), max_results, query.as_deref())?;
            serde_json::to_string(&r).map_err(|e| e.to_string())?
        }
        GoogleCalendarAction::GetEvent { calendar_id, event_id } => {
            let r = api::get_event(&calendar_id, &event_id)?;
            serde_json::to_string(&r).map_err(|e| e.to_string())?
        }
        GoogleCalendarAction::CreateEvent {
            calendar_id, summary, description, location,
            start_datetime, end_datetime, start_date, end_date, timezone, attendees,
        } => {
            let r = api::create_event(&api::CreateEventParams {
                calendar_id: &calendar_id,
                summary: &summary,
                description: description.as_deref(),
                location: location.as_deref(),
                start_datetime: start_datetime.as_deref(),
                end_datetime: end_datetime.as_deref(),
                start_date: start_date.as_deref(),
                end_date: end_date.as_deref(),
                timezone: timezone.as_deref(),
                attendees: &attendees,
            })?;
            serde_json::to_string(&r).map_err(|e| e.to_string())?
        }
        GoogleCalendarAction::UpdateEvent {
            calendar_id, event_id, summary, description, location,
            start_datetime, end_datetime, start_date, end_date, timezone, attendees,
        } => {
            let r = api::update_event(&api::UpdateEventParams {
                calendar_id: &calendar_id,
                event_id: &event_id,
                summary: summary.as_deref(),
                description: description.as_deref(),
                location: location.as_deref(),
                start_datetime: start_datetime.as_deref(),
                end_datetime: end_datetime.as_deref(),
                start_date: start_date.as_deref(),
                end_date: end_date.as_deref(),
                timezone: timezone.as_deref(),
                attendees: attendees.as_deref(),
            })?;
            serde_json::to_string(&r).map_err(|e| e.to_string())?
        }
        GoogleCalendarAction::DeleteEvent { calendar_id, event_id } => {
            let r = api::delete_event(&calendar_id, &event_id)?;
            serde_json::to_string(&r).map_err(|e| e.to_string())?
        }
    };

    Ok(result)
}
```

<Note>
`secret_exists` returns `true` if the token has been stored — it does not expose the token value. The host injects it as a `Bearer` header automatically on every HTTP request to `www.googleapis.com`, so your Rust code never touches raw credentials.
</Note>

---

## 5. Implement the API module

Create `src/api.rs`. All network calls go through `api_call`, which uses the host's HTTP capability.

### Core HTTP helper

```rust src/api.rs
use crate::near::agent::host;
use crate::types::*;

const BASE: &str = "https://www.googleapis.com/calendar/v3";

fn api_call(method: &str, path: &str, body: Option<&str>) -> Result<String, String> {
    let url = format!("{}/{}", BASE, path);
    let headers = if body.is_some() {
        r#"{"Content-Type": "application/json"}"#
    } else {
        "{}"
    };
    let body_bytes = body.map(|b| b.as_bytes().to_vec());

    host::log(host::LogLevel::Debug, &format!("Calendar API: {} {}", method, path));

    let response = host::http_request(method, &url, headers, body_bytes.as_deref(), None)?;

    if response.status < 200 || response.status >= 300 {
        return Err(format!(
            "Google Calendar API returned status {}: {}",
            response.status,
            String::from_utf8_lossy(&response.body)
        ));
    }

    if response.body.is_empty() {
        return Ok(String::new()); // DELETE returns 204 No Content
    }

    String::from_utf8(response.body).map_err(|e| format!("Invalid UTF-8 in response: {}", e))
}
```

### Response parser helpers

```rust src/api.rs
fn parse_event(v: &serde_json::Value) -> Event {
    Event {
        id: v["id"].as_str().unwrap_or("").to_string(),
        summary: v["summary"].as_str().unwrap_or("(no title)").to_string(),
        description: v["description"].as_str().map(|s| s.to_string()),
        location: v["location"].as_str().map(|s| s.to_string()),
        start: parse_event_time(&v["start"]),
        end: parse_event_time(&v["end"]),
        status: v["status"].as_str().unwrap_or("confirmed").to_string(),
        html_link: v["htmlLink"].as_str().map(|s| s.to_string()),
        attendees: v["attendees"]
            .as_array()
            .map(|arr| {
                arr.iter().map(|a| Attendee {
                    email: a["email"].as_str().unwrap_or("").to_string(),
                    display_name: a["displayName"].as_str().map(|s| s.to_string()),
                    response_status: a["responseStatus"].as_str().map(|s| s.to_string()),
                }).collect()
            })
            .unwrap_or_default(),
        organizer: v.get("organizer").map(|o| Organizer {
            email: o["email"].as_str().unwrap_or("").to_string(),
            display_name: o["displayName"].as_str().map(|s| s.to_string()),
        }),
    }
}

fn parse_event_time(v: &serde_json::Value) -> EventTime {
    EventTime {
        date: v["date"].as_str().map(|s| s.to_string()),
        date_time: v["dateTime"].as_str().map(|s| s.to_string()),
        time_zone: v["timeZone"].as_str().map(|s| s.to_string()),
    }
}
```

### list_events

The most common action. Defaults `time_min` to "now" so the agent never sees stale past events unless it explicitly asks:

```rust src/api.rs
pub fn list_events(
    calendar_id: &str,
    time_min: Option<&str>,
    time_max: Option<&str>,
    max_results: u32,
    query: Option<&str>,
) -> Result<ListEventsResult, String> {
    let mut params = vec![
        format!("maxResults={}", max_results),
        "singleEvents=true".to_string(),
        "orderBy=startTime".to_string(),
    ];

    // Default to now when no lower bound is specified.
    let now_rfc3339;
    let effective_time_min = if time_min.is_some() {
        time_min
    } else {
        let millis = host::now_millis();
        let secs = millis / 1000;
        let millis_rem = millis % 1000;
        let s = secs % 60;
        let m = (secs / 60) % 60;
        let h = (secs / 3600) % 24;
        let (year, month, day) = days_to_ymd(secs / 86400);
        now_rfc3339 = format!(
            "{:04}-{:02}-{:02}T{:02}:{:02}:{:02}.{:03}Z",
            year, month, day, h, m, s, millis_rem
        );
        Some(now_rfc3339.as_str())
    };

    if let Some(t) = effective_time_min { params.push(format!("timeMin={}", url_encode(t))); }
    if let Some(t) = time_max          { params.push(format!("timeMax={}", url_encode(t))); }
    if let Some(q) = query             { params.push(format!("q={}", url_encode(q))); }

    let path = format!("calendars/{}/events?{}", url_encode(calendar_id), params.join("&"));
    let response = api_call("GET", &path, None)?;
    let parsed: serde_json::Value =
        serde_json::from_str(&response).map_err(|e| format!("Failed to parse response: {}", e))?;

    let events = parsed["items"]
        .as_array()
        .map(|arr| arr.iter().map(parse_event).collect())
        .unwrap_or_default();

    Ok(ListEventsResult {
        events,
        next_page_token: parsed["nextPageToken"].as_str().map(|s| s.to_string()),
    })
}
```

### get_event

```rust src/api.rs
pub fn get_event(calendar_id: &str, event_id: &str) -> Result<EventResult, String> {
    let path = format!("calendars/{}/events/{}", url_encode(calendar_id), url_encode(event_id));
    let response = api_call("GET", &path, None)?;
    let parsed: serde_json::Value =
        serde_json::from_str(&response).map_err(|e| format!("Failed to parse response: {}", e))?;
    Ok(EventResult { event: parse_event(&parsed) })
}
```

### create_event

Supports both timed events (RFC3339 `dateTime`) and all-day events (`date`). Either `start_datetime` or `start_date` must be provided:

```rust src/api.rs
pub struct CreateEventParams<'a> {
    pub calendar_id: &'a str,
    pub summary: &'a str,
    pub description: Option<&'a str>,
    pub location: Option<&'a str>,
    pub start_datetime: Option<&'a str>,
    pub end_datetime: Option<&'a str>,
    pub start_date: Option<&'a str>,
    pub end_date: Option<&'a str>,
    pub timezone: Option<&'a str>,
    pub attendees: &'a [String],
}

pub fn create_event(p: &CreateEventParams<'_>) -> Result<EventResult, String> {
    let mut event = serde_json::json!({ "summary": p.summary });

    if let Some(d) = p.description { event["description"] = d.into(); }
    if let Some(l) = p.location    { event["location"]    = l.into(); }

    if let Some(dt) = p.start_datetime {
        let mut start = serde_json::json!({ "dateTime": dt });
        if let Some(tz) = p.timezone { start["timeZone"] = tz.into(); }
        event["start"] = start;
    } else if let Some(d) = p.start_date {
        event["start"] = serde_json::json!({ "date": d });
    } else {
        return Err("Either start_datetime or start_date is required".to_string());
    }

    if let Some(dt) = p.end_datetime {
        let mut end = serde_json::json!({ "dateTime": dt });
        if let Some(tz) = p.timezone { end["timeZone"] = tz.into(); }
        event["end"] = end;
    } else if let Some(d) = p.end_date {
        event["end"] = serde_json::json!({ "date": d });
    } else {
        return Err("Either end_datetime or end_date is required".to_string());
    }

    if !p.attendees.is_empty() {
        event["attendees"] = serde_json::json!(
            p.attendees.iter().map(|e| serde_json::json!({ "email": e })).collect::<Vec<_>>()
        );
    }

    let body = serde_json::to_string(&event).map_err(|e| e.to_string())?;
    let path = format!("calendars/{}/events", url_encode(p.calendar_id));
    let response = api_call("POST", &path, Some(&body))?;
    let parsed: serde_json::Value =
        serde_json::from_str(&response).map_err(|e| format!("Failed to parse response: {}", e))?;
    Ok(EventResult { event: parse_event(&parsed) })
}
```

### update_event

Uses PATCH so only the fields you send are changed on the server — you do not need to re-send the entire event:

```rust src/api.rs
pub struct UpdateEventParams<'a> {
    pub calendar_id: &'a str,
    pub event_id: &'a str,
    pub summary: Option<&'a str>,
    pub description: Option<&'a str>,
    pub location: Option<&'a str>,
    pub start_datetime: Option<&'a str>,
    pub end_datetime: Option<&'a str>,
    pub start_date: Option<&'a str>,
    pub end_date: Option<&'a str>,
    pub timezone: Option<&'a str>,
    pub attendees: Option<&'a [String]>,
}

pub fn update_event(p: &UpdateEventParams<'_>) -> Result<EventResult, String> {
    let mut patch = serde_json::json!({});

    if let Some(s) = p.summary     { patch["summary"]     = s.into(); }
    if let Some(d) = p.description { patch["description"] = d.into(); }
    if let Some(l) = p.location    { patch["location"]    = l.into(); }

    if let Some(dt) = p.start_datetime {
        let mut start = serde_json::json!({ "dateTime": dt });
        if let Some(tz) = p.timezone { start["timeZone"] = tz.into(); }
        patch["start"] = start;
    } else if let Some(d) = p.start_date {
        patch["start"] = serde_json::json!({ "date": d });
    }

    if let Some(dt) = p.end_datetime {
        let mut end = serde_json::json!({ "dateTime": dt });
        if let Some(tz) = p.timezone { end["timeZone"] = tz.into(); }
        patch["end"] = end;
    } else if let Some(d) = p.end_date {
        patch["end"] = serde_json::json!({ "date": d });
    }

    if let Some(att) = p.attendees {
        patch["attendees"] = serde_json::json!(
            att.iter().map(|e| serde_json::json!({ "email": e })).collect::<Vec<_>>()
        );
    }

    let body = serde_json::to_string(&patch).map_err(|e| e.to_string())?;
    let path = format!("calendars/{}/events/{}", url_encode(p.calendar_id), url_encode(p.event_id));
    let response = api_call("PATCH", &path, Some(&body))?;
    let parsed: serde_json::Value =
        serde_json::from_str(&response).map_err(|e| format!("Failed to parse response: {}", e))?;
    Ok(EventResult { event: parse_event(&parsed) })
}
```

### delete_event

```rust src/api.rs
pub fn delete_event(calendar_id: &str, event_id: &str) -> Result<DeleteResult, String> {
    let path = format!(
        "calendars/{}/events/{}",
        url_encode(calendar_id),
        url_encode(event_id)
    );
    api_call("DELETE", &path, None)?;
    Ok(DeleteResult { deleted: true, event_id: event_id.to_string() })
}
```

### URL-encoding utilities

```rust src/api.rs
fn url_encode(s: &str) -> String {
    let mut encoded = String::with_capacity(s.len());
    for b in s.bytes() {
        match b {
            b'A'..=b'Z' | b'a'..=b'z' | b'0'..=b'9' | b'-' | b'_' | b'.' | b'~' => {
                encoded.push(b as char);
            }
            _ => {
                encoded.push('%');
                encoded.push(char::from(HEX[(b >> 4) as usize]));
                encoded.push(char::from(HEX[(b & 0x0F) as usize]));
            }
        }
    }
    encoded
}

const HEX: [u8; 16] = *b"0123456789ABCDEF";

/// Convert days since Unix epoch to (year, month, day).
fn days_to_ymd(days: u64) -> (u64, u64, u64) {
    let z = days as i64 + 719468;
    let era = if z >= 0 { z } else { z - 146096 } / 146097;
    let doe = (z - era * 146097) as u64;
    let yoe = (doe - doe / 1460 + doe / 36524 - doe / 146096) / 365;
    let y = yoe as i64 + era * 400;
    let doy = doe - (365 * yoe + yoe / 4 - yoe / 100);
    let mp = (5 * doy + 2) / 153;
    let d = doy - (153 * mp + 2) / 5 + 1;
    let m = if mp < 10 { mp + 3 } else { mp - 9 };
    let y = if m <= 2 { y + 1 } else { y };
    (y as u64, m, d)
}
```

---

## 6. Define the JSON schema

Add the `SCHEMA` constant to `src/lib.rs`. This tells the LLM exactly what JSON to send. A single flat schema with an `action` enum works well when all actions share many optional fields:

```rust src/lib.rs
const SCHEMA: &str = r#"{
    "type": "object",
    "required": ["action"],
    "properties": {
        "action": {
            "type": "string",
            "enum": ["list_events", "get_event", "create_event", "update_event", "delete_event"],
            "description": "The calendar operation to perform"
        },
        "calendar_id": {
            "type": "string",
            "description": "Calendar ID (default: 'primary')",
            "default": "primary"
        },
        "event_id": {
            "type": "string",
            "description": "Event ID. Required for: get_event, update_event, delete_event"
        },
        "time_min": {
            "type": "string",
            "description": "Lower bound for event start time (RFC3339, e.g., '2025-01-15T00:00:00Z'). Used by: list_events"
        },
        "time_max": {
            "type": "string",
            "description": "Upper bound for event end time (RFC3339). Used by: list_events"
        },
        "max_results": {
            "type": "integer",
            "description": "Maximum number of events to return (default: 25). Used by: list_events",
            "default": 25
        },
        "query": {
            "type": "string",
            "description": "Free text search terms to filter events. Used by: list_events"
        },
        "summary": {
            "type": "string",
            "description": "Event title. Required for: create_event. Optional for: update_event"
        },
        "description": {
            "type": "string",
            "description": "Event description. Used by: create_event, update_event"
        },
        "location": {
            "type": "string",
            "description": "Event location. Used by: create_event, update_event"
        },
        "start_datetime": {
            "type": "string",
            "description": "Start time (RFC3339, e.g., '2025-01-15T09:00:00-05:00'). For all-day events use start_date instead. Used by: create_event, update_event"
        },
        "end_datetime": {
            "type": "string",
            "description": "End time (RFC3339). Used by: create_event, update_event"
        },
        "start_date": {
            "type": "string",
            "description": "Start date for all-day events (e.g., '2025-01-15'). Used by: create_event, update_event"
        },
        "end_date": {
            "type": "string",
            "description": "End date for all-day events, exclusive (e.g., '2025-01-16'). Used by: create_event, update_event"
        },
        "timezone": {
            "type": "string",
            "description": "Timezone (e.g., 'America/New_York'). Used with datetime fields"
        },
        "attendees": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Attendee email addresses. Used by: create_event, update_event"
        }
    }
}"#;
```

<Note>
The schema describes every possible field across all actions. The `description` annotation on each field tells the LLM which action(s) use it. This flat approach keeps the schema simpler than a `oneOf` per action when most fields are shared.
</Note>

---

## 7. Declare capabilities and OAuth

Create `google-calendar-tool.capabilities.json` next to `Cargo.toml`. This file does three things:

1. **Allowlists** the Google API host so the WASM sandbox permits outbound requests.
2. **Declares credential injection** — the stored OAuth token is added as a `Bearer` header automatically on every request to `www.googleapis.com`.
3. **Configures the `ironclaw tool auth` flow** — which OAuth endpoints to use, which scopes to request, and which env vars carry the client ID/secret.

```json google-calendar-tool.capabilities.json
{
  "version": "0.1.0",
  "wit_version": "0.3.0",
  "http": {
    "allowlist": [
      {
        "host": "www.googleapis.com",
        "path_prefix": "/calendar/v3/",
        "methods": ["GET", "POST", "PATCH", "DELETE"]
      }
    ],
    "credentials": {
      "google_oauth_token": {
        "secret_name": "google_oauth_token",
        "location": { "type": "bearer" },
        "host_patterns": ["www.googleapis.com"]
      }
    },
    "rate_limit": {
      "requests_per_minute": 60,
      "requests_per_hour": 500
    },
    "timeout_secs": 30
  },
  "secrets": {
    "allowed_names": ["google_oauth_token"]
  },
  "auth": {
    "secret_name": "google_oauth_token",
    "display_name": "Google",
    "oauth": {
      "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth",
      "token_url": "https://oauth2.googleapis.com/token",
      "client_id_env": "GOOGLE_OAUTH_CLIENT_ID",
      "client_secret_env": "GOOGLE_OAUTH_CLIENT_SECRET",
      "scopes": [
        "https://www.googleapis.com/auth/calendar.events"
      ],
      "use_pkce": false,
      "extra_params": {
        "access_type": "offline",
        "prompt": "consent"
      }
    },
    "env_var": "GOOGLE_OAUTH_TOKEN"
  },
  "setup": {
    "required_secrets": [
      {
        "name": "google_oauth_client_id",
        "prompt": "Google OAuth Client ID (from console.cloud.google.com/apis/credentials)"
      },
      {
        "name": "google_oauth_client_secret",
        "prompt": "Google OAuth Client Secret (from console.cloud.google.com/apis/credentials)"
      }
    ]
  }
}
```

<Accordion title="How credential injection works" icon="key">

The WASM sandbox **never sees** the OAuth token directly. Here is what happens at runtime:

1. `secret_exists("google_oauth_token")` returns `true` — the tool confirms auth is present.
2. The tool calls `host::http_request("GET", "https://www.googleapis.com/...", ...)`.
3. The host checks the `credentials` map in capabilities, finds `google_oauth_token` matching the host pattern `www.googleapis.com`, and injects `Authorization: Bearer <token>` into the outbound request headers.
4. If the token has expired, the host refreshes it using the stored refresh token before sending.

This design means credentials never appear in tool parameters or agent-visible prompts.

</Accordion>

---

## 8. Build and install

Before building, set your OAuth client credentials:

```bash
export GOOGLE_OAUTH_CLIENT_ID=<your-client-id>
export GOOGLE_OAUTH_CLIENT_SECRET=<your-client-secret>
```

Build the WASM component:

```bash
cargo build --target wasm32-wasip2 --release
```

Store the OAuth client credentials (runs `setup.required_secrets` prompts):

```bash
ironclaw tool setup google-calendar-tool
```

Install the binary and capabilities sidecar:

```bash
ironclaw tool install ./target/wasm32-wasip2/release/google_calendar_tool.wasm \
  --capabilities ./google-calendar-tool.capabilities.json \
  --name google-calendar
```

Complete the OAuth flow (opens a browser for Google authorization):

```bash
ironclaw tool auth google-calendar
```

<Info>
The `ironclaw tool auth` step runs once. IronClaw stores the refresh token and renews the access token automatically on each subsequent request.
</Info>

Verify the tool loaded:

```bash
ironclaw tool list
```

---

## Try it out

Start IronClaw and ask your agent:

- "What's on my calendar this week?"
- "Schedule a team sync for next Tuesday at 3pm for 1 hour with alice@example.com"
- "Move my Friday standup to Monday at 10am"
- "Cancel all my meetings on Thursday afternoon"
- "Find my dentist appointment and delete it"

The agent resolves the right action from the schema, calls the tool automatically, and handles multi-step tasks (list → find ID → update/delete) on its own.

---

## Adapting this pattern to other Google APIs

The same structure works for any Google API that uses OAuth 2.0:

| What to change | Where |
|---|---|
| API base URL and path prefix | `api.rs` constant + capabilities allowlist |
| OAuth scopes | `auth.oauth.scopes` in capabilities |
| Action enum variants and field names | `types.rs` |
| API call implementations | `api.rs` functions |
| JSON schema properties | `SCHEMA` constant in `lib.rs` |
| Description string | `fn description()` in `lib.rs` |

The WIT interface, credential injection pattern, `execute_inner` guard, and build/install commands remain identical across all Google API tools.
