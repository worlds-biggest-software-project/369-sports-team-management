# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Sports Team Management · Created: 2026-05-26

## Philosophy

The two central aggregates in sports team management are the **team** (with its roster, seasons, and settings) and the **event** (with its RSVPs, stats, and video). This model makes each the centre of a rich JSONB document: teams embed their roster with member details, season configurations, and prospect notes; events embed RSVPs, per-player match statistics, lineup details, and video references. Users carry their own embedded context — club affiliation, injury history, payment records, and development plans.

Sports team management has two dominant access patterns: (1) "show me this team's roster with positions, attendance, and availability" and (2) "show me this event with RSVPs, lineup, scores, and player stats." Embedding roster on the team row and RSVPs + stats on the event row means both views are single-row reads. The trade-off is that cross-team analytics (player stats across multiple teams) require JSONB extraction from event rows.

For amateur and youth clubs with 2-10 teams of 15-30 players each, with 30-60 events per season, the JSONB approach keeps schema complexity low while enabling rapid iteration. Adding a new sport with different stat fields, a new position type, or a new RSVP status requires no migration — just a new value in the JSONB structure.

**Best for:** Teams building an MVP for amateur and youth clubs where fast roster loading, single-query event views, rapid iteration on sport-specific features, and simple deployment are priorities.

**Trade-offs:**
- Pro: 5 tables — simple schema, fast to deploy
- Pro: Full team context (roster, seasons, prospects) in one row read
- Pro: Full event context (RSVPs, stats, lineup, video) in one row read
- Pro: New sports, stat types, and RSVP options require no migration
- Con: Cross-team player stats require JSONB extraction from events
- Con: No FK enforcement on player references within JSONB
- Con: Large teams with 100+ events per season can produce large event histories
- Con: Concurrent RSVP updates on the same event need careful locking

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| iCalendar (RFC 5545) | Events export as VEVENT for calendar sync |
| SportsML-G2 | Roster and results data interchange |
| ISO 8601 | All timestamps and durations |
| JSON Schema 2020-12 | Sport-specific stat template validation |
| CloudEvents 1.0 | Webhook envelopes for RSVP and score events |
| OAuth 2.0 / OIDC / PKCE | Authentication; social SSO |
| Stripe Connect | Payment processing |
| GDPR | Player PII handling |
| COPPA | Under-13 data with parental consent |
| MCP | AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    phone               TEXT,
    avatar_url          TEXT,
    date_of_birth       DATE,
    gender              TEXT CHECK (gender IN ('male','female','other','prefer_not_to_say')),
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple','microsoft'
                        )),
    club_json           JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "id": "uuid", "name": "Riverside FC",
    --   "slug": "riverside-fc", "sport_primary": "soccer",
    --   "country": "US", "timezone": "America/New_York",
    --   "role": "coach",
    --   "stripe_account_id": "acct_...",
    --   "subscription_tier": "pro"
    -- }
    player_json         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "positions": ["midfielder","forward"],
    --   "dominant_foot": "right",
    --   "height_cm": 175, "weight_kg": 70,
    --   "development_plan": {
    --     "goals": ["improve weak foot","increase stamina"],
    --     "coach_notes": "Strong technically, needs more defensive awareness",
    --     "last_reviewed": "2026-05-01"
    --   },
    --   "injuries": [{
    --     "id": "uuid", "type": "ankle_sprain", "body_area": "left_ankle",
    --     "severity": "moderate", "occurred_at": "2026-04-15",
    --     "occurred_during": "match", "status": "cleared",
    --     "return_to_play_date": "2026-05-10", "cleared_at": "2026-05-10"
    --   }],
    --   "career_stats": {
    --     "total_goals": 45, "total_assists": 32,
    --     "total_matches": 120, "total_minutes": 8400
    --   }
    -- }
    payments_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "registration", "team_id": "uuid",
    --   "description": "Fall 2026 Registration",
    --   "amount_cents": 25000, "currency": "USD",
    --   "status": "paid", "stripe_payment_id": "pi_...",
    --   "due_date": "2026-08-01", "paid_at": "2026-07-28T10:00:00Z"
    -- }]
    compliance_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "is_minor": true,
    --   "parent_user_id": "uuid",
    --   "parental_consent_at": "2026-01-15T10:00:00Z",
    --   "gdpr_consent_at": "2026-01-15T10:00:00Z",
    --   "safesport_cleared": false
    -- }
    notification_prefs  JSONB NOT NULL DEFAULT '{"push": true, "email": true, "sms": false}',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_club ON users USING GIN (club_json);
CREATE INDEX idx_users_compliance ON users USING GIN (compliance_json);
```

---

## Teams

```sql
CREATE TABLE teams (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL,
    name                TEXT NOT NULL,
    sport               TEXT NOT NULL CHECK (sport IN (
                            'soccer','basketball','baseball','softball',
                            'volleyball','lacrosse','football','hockey',
                            'rugby','tennis','swimming','track_field',
                            'cricket','netball','handball','other'
                        )),
    age_group           TEXT,
    gender_category     TEXT CHECK (gender_category IN (
                            'male','female','mixed','open'
                        )),
    level               TEXT CHECK (level IN (
                            'recreational','competitive','travel',
                            'elite','semi_pro','professional'
                        )),
    home_venue          TEXT,
    color_primary       TEXT,
    roster_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "display_name": "Sam Johnson",
    --   "member_role": "player", "jersey_number": 10,
    --   "position": "midfielder", "position_secondary": "forward",
    --   "is_captain": true, "eligibility_status": "eligible",
    --   "joined_at": "2026-01-15",
    --   "documents": [
    --     {"type": "medical_clearance", "url": "...", "expires_at": "2027-01-01"}
    --   ],
    --   "attendance_rate": 0.92,
    --   "current_injury": null
    -- }, {
    --   "user_id": "uuid", "display_name": "Coach Williams",
    --   "member_role": "coach", "joined_at": "2025-09-01"
    -- }]
    seasons_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Fall 2026",
    --   "start_date": "2026-09-01", "end_date": "2026-12-15",
    --   "status": "active",
    --   "stat_template": {
    --     "sport": "soccer",
    --     "fields": [
    --       {"key": "goals", "label": "Goals", "type": "integer"},
    --       {"key": "assists", "label": "Assists", "type": "integer"},
    --       {"key": "minutes_played", "label": "Minutes", "type": "integer"}
    --     ]
    --   },
    --   "record": {"wins": 8, "losses": 2, "draws": 1}
    -- }]
    prospects_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Alex Rivera",
    --   "position": "forward", "current_club": "City Youth",
    --   "date_of_birth": "2012-03-15",
    --   "attributes": {"speed": 8, "technique": 7, "overall": 7.5},
    --   "status": "watching", "scouted_by": "Coach Williams",
    --   "notes": "Strong left foot, good movement",
    --   "video_urls": ["https://..."]
    -- }]
    settings_json       JSONB NOT NULL DEFAULT '{}',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_teams_club ON teams (club_id, sport);
CREATE INDEX idx_teams_roster ON teams USING GIN (roster_json);
```

---

## Events

```sql
CREATE TABLE events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id             UUID NOT NULL REFERENCES teams(id),
    season_id           UUID,
    event_type          TEXT NOT NULL CHECK (event_type IN (
                            'match','practice','meeting','tournament',
                            'tryout','scrimmage','social','other'
                        )),
    title               TEXT NOT NULL,
    description         TEXT,
    start_at            TIMESTAMPTZ NOT NULL,
    end_at              TIMESTAMPTZ,
    venue               TEXT,
    venue_address       TEXT,
    opponent            TEXT,
    is_home             BOOLEAN,
    status              TEXT NOT NULL CHECK (status IN (
                            'scheduled','in_progress','completed',
                            'postponed','cancelled'
                        )) DEFAULT 'scheduled',
    score_home          INTEGER,
    score_away          INTEGER,
    result              TEXT CHECK (result IN ('win','loss','draw')),
    rsvps_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "display_name": "Sam Johnson",
    --   "status": "available", "responded_at": "2026-05-24T18:00:00Z"
    -- }, {
    --   "user_id": "uuid", "display_name": "Alex Chen",
    --   "status": "unavailable", "reason": "Family event",
    --   "responded_at": "2026-05-24T19:30:00Z"
    -- }]
    lineup_json         JSONB,
    -- [{
    --   "user_id": "uuid", "position": "GK", "jersey_number": 1,
    --   "is_starter": true, "sub_in_minute": null, "sub_out_minute": null
    -- }]
    stats_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "display_name": "Sam Johnson",
    --   "position_played": "midfielder", "is_starter": true,
    --   "player_rating": 8.2, "coach_notes": "Excellent movement",
    --   "stats": {
    --     "goals": 2, "assists": 1, "minutes_played": 78,
    --     "pass_completion_pct": 85.2, "shots_on_target": 3
    --   }
    -- }]
    video_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "title": "Match Highlights",
    --   "video_url": "https://...", "thumbnail_url": "https://...",
    --   "duration_seconds": 180, "type": "highlights",
    --   "tags": [
    --     {"timestamp_sec": 45, "label": "goal", "player_id": "uuid", "ai_generated": true},
    --     {"timestamp_sec": 120, "label": "save", "player_id": "uuid", "ai_generated": true}
    --   ]
    -- }]
    training_json       JSONB,
    -- {
    --   "drills": [
    --     {"name": "Passing Triangle", "duration_min": 15, "intensity": "medium"},
    --     {"name": "Small-Sided Game", "duration_min": 20, "intensity": "high"}
    --   ],
    --   "total_load_au": 450,
    --   "session_rpe": 7.2,
    --   "attendance": [
    --     {"user_id": "uuid", "attended": true, "load_au": 420},
    --     {"user_id": "uuid", "attended": false}
    --   ]
    -- }
    match_notes         TEXT,
    weather             TEXT,
    is_offline_created  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_events_team ON events (team_id, start_at DESC);
CREATE INDEX idx_events_upcoming ON events (start_at)
    WHERE status = 'scheduled';
CREATE INDEX idx_events_rsvps ON events USING GIN (rsvps_json);
CREATE INDEX idx_events_stats ON events USING GIN (stats_json);
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL,
    team_id             UUID REFERENCES teams(id),
    user_id             UUID REFERENCES users(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'match_recap','parent_email','lineup_suggestion',
                            'schedule_conflict','injury_risk','video_tag',
                            'stat_query_response','scouting_comparison',
                            'training_plan','query_response'
                        )),
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    suggestion_data     JSONB,
    is_applied          BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    llm_model           TEXT,
    tokens_used         INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_club ON ai_suggestions (club_id, created_at DESC);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID,
    user_id             UUID REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','stripe_webhook','calendar_sync'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    involves_minor      BOOLEAN NOT NULL DEFAULT FALSE,
    changes_json        JSONB,
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_club ON audit_log (club_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_minor ON audit_log (club_id, created_at)
    WHERE involves_minor = TRUE;
```

---

## Example Queries

### Team roster — single row read

```sql
SELECT name, sport, age_group, level,
       roster_json, seasons_json
FROM teams
WHERE id = 'team-uuid';
```

### Event with RSVPs and stats — single row read

```sql
SELECT title, event_type, start_at, venue, opponent,
       score_home, score_away, result,
       rsvps_json, lineup_json, stats_json, video_json
FROM events
WHERE id = 'event-uuid';
```

### Upcoming events with RSVP counts from JSONB

```sql
SELECT e.title, e.event_type, e.start_at, e.venue, e.opponent,
       t.name AS team_name,
       (SELECT COUNT(*) FROM jsonb_array_elements(e.rsvps_json) r
        WHERE r->>'status' = 'available') AS available_count,
       (SELECT COUNT(*) FROM jsonb_array_elements(e.rsvps_json) r
        WHERE r->>'status' = 'unavailable') AS unavailable_count,
       jsonb_array_length(e.rsvps_json) AS total_responses
FROM events e
JOIN teams t ON t.id = e.team_id
WHERE e.team_id = 'team-uuid'
  AND e.start_at >= now()
  AND e.status = 'scheduled'
ORDER BY e.start_at
LIMIT 10;
```

### Player season stats from embedded event stats

```sql
SELECT s->>'display_name' AS player,
       COUNT(*) AS games,
       SUM((s->'stats'->>'goals')::INTEGER) AS total_goals,
       SUM((s->'stats'->>'assists')::INTEGER) AS total_assists,
       AVG((s->'stats'->>'pass_completion_pct')::NUMERIC) AS avg_pass_pct,
       AVG((s->>'player_rating')::NUMERIC) AS avg_rating
FROM events e,
     jsonb_array_elements(e.stats_json) AS s
WHERE e.team_id = 'team-uuid'
  AND e.season_id = 'season-uuid'
  AND e.event_type = 'match'
  AND e.status = 'completed'
GROUP BY s->>'user_id', s->>'display_name'
ORDER BY total_goals DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds club context, player profile with injuries, payments, compliance) |
| Teams | 1 | teams (embeds roster, seasons with stat templates, prospects) |
| Events | 1 | events (embeds RSVPs, lineup, stats, video with tags, training session) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **5** | |

---

## Key Design Decisions

1. **Team as central aggregate** — a coach's primary view is team-centric: "show me my team's roster and current season." Embedding roster, seasons, and prospect data on the team row means this view is a single-row read with no joins.

2. **Event as central aggregate** — the match-day workflow loads one event at a time: RSVPs, lineup, scores, player stats, and video. Embedding all of these on the event row means the complete event context loads in one query.

3. **`roster_json` on teams** — a team typically has 15-30 members. Embedding player details (jersey number, position, attendance rate, current injury) means the roster view with all context is a single read. New roster fields (certifications, wearable IDs) require no migration.

4. **`stats_json` on events** — per-player match stats (goals, assists, ratings) are embedded on the event row. A typical match has 11-25 player stat entries. The stat fields are sport-specific (soccer vs. basketball) and vary by season template — JSONB handles this naturally.

5. **`player_json` on users** — injury history, development plans, career stats, and physical attributes are embedded on the user row. A player's profile view is a single-row read. Injury records accumulate over time but typically number 0-10 per player.

6. **`rsvps_json` on events** — availability responses (typically 15-30 per event) are embedded for the RSVP summary view. The trade-off is that concurrent RSVP submissions require careful locking, but this is manageable with JSONB path updates.

7. **`training_json` on events** — training session data (drills, load monitoring, attendance) is embedded on practice events. This keeps the training planning workflow within the event context.

8. **`seasons_json` on teams** — a team typically has 2-4 seasons (fall, spring). Each season carries its stat template defining which fields to capture. Embedding seasons means the current season's stat template loads with the team context.

9. **`compliance_json` on users** — GDPR consent, COPPA parental consent, and SafeSport clearance are embedded on the user for immediate access during any operation involving that user. The `involves_minor` flag on audit_log provides the compliance query path.

10. **5 tables** — amateur and youth sports management is team-centric and event-centric. Embedding roster, stats, RSVPs, and player context on the appropriate aggregate minimises joins for the dominant coach and parent workflows.
