# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Sports Team Management · Created: 2026-05-26

## Philosophy

Every state change in the sports platform — player joined team, RSVP submitted, match score entered, stat recorded, injury reported, payment collected — is recorded as an immutable event in a single append-only store. The event store is the sole source of truth; five materialised read models are projected from the event stream to serve the team dashboard, schedule, player profile, season analytics, and scouting board views.

Sports team management for youth and amateur clubs has strict privacy requirements: GDPR mandates audit trails for minor data access, COPPA requires parental consent verification for under-13s, and SafeSport compliance needs documented clearance records. With event sourcing, every interaction with player data — especially minor data — is an immutable audit event. The `involves_minor` flag on events enables instant compliance reporting without scanning application tables.

The CQRS pattern also enables the offline match-day workflow: a coach enters scores, stats, and RSVPs on a device without connectivity. These are buffered as local events and synced to the event store when connectivity returns, with conflict resolution at the projection layer. This is naturally modeled as event append rather than row-level merge conflicts.

**Best for:** Teams building a privacy-first platform where GDPR/COPPA compliance for minor data, offline match-day capability via event buffering, full audit trails, and longitudinal player development tracking are priorities.

**Trade-offs:**
- Pro: GDPR/COPPA compliance by design — every access to minor data is an immutable event
- Pro: Offline match-day mode maps naturally to event buffering and replay
- Pro: Player development history reconstructable at any point in time
- Pro: Read models can be rebuilt when new analytics requirements emerge
- Con: CQRS adds infrastructure complexity — event store + projections + read models
- Con: Eventual consistency between event writes and read model updates
- Con: RSVP status changes produce many small events during pre-game period
- Con: Debugging requires replaying event streams rather than inspecting current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (ce_source, ce_type, ce_specversion, ce_time) |
| iCalendar (RFC 5545) | Schedule events exportable from rm_schedule read model |
| SportsML-G2 | Match results and roster data interchange from read models |
| AsyncAPI 3.0 | Live-scoring event stream description |
| ISO 8601 | All timestamps |
| JSON Schema 2020-12 | Sport-specific stat template validation |
| OAuth 2.0 / OIDC | Actor identity on every event |
| Stripe Connect | payment_collected events reference Stripe payment IDs |
| GDPR | Immutable event store with involves_minor flag satisfies minor-data audit |
| COPPA | Parental consent events tracked in user stream |
| SafeSport | Clearance events tracked with verification |
| MCP | AI assistant integration via ai stream events |

---

## Event Store

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'club','user','team','event',
                            'stats','video','scouting','ai','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','stripe_webhook',
                            'calendar_sync','offline_sync'
                        )),
    actor_role          TEXT,
    club_id             UUID,
    involves_minor      BOOLEAN NOT NULL DEFAULT FALSE,
    ip_address          INET,
    ce_source           TEXT NOT NULL DEFAULT '/sports-team',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_number)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_type, stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_club ON event_store (club_id, created_at);
CREATE INDEX idx_events_minor ON event_store (club_id, created_at)
    WHERE involves_minor = TRUE;
```

### Key Event Types by Stream

**club stream:**
- `club_created` — new club registered with sport and country
- `club_settings_updated` — subscription tier, Stripe account, locale changed
- `club_subscription_changed` — tier upgrade/downgrade

**user stream:**
- `user_registered` — new user joined club with role
- `user_profile_updated` — name, contact, avatar changed
- `parental_consent_granted` — parent consented for minor (COPPA/GDPR)
- `parental_consent_revoked` — consent withdrawn
- `safesport_clearance_verified` — SafeSport background check cleared
- `gdpr_consent_recorded` — GDPR data processing consent captured
- `gdpr_erasure_requested` — right-to-erasure request initiated
- `payment_collected` — registration fee or dues collected (Stripe reference)
- `payment_refunded` — payment refunded

**team stream:**
- `team_created` — new team added to club with sport and age group
- `player_joined` — member added to team with role, position, jersey
- `player_left` — member removed from team
- `player_role_changed` — position, captaincy, or eligibility updated
- `season_started` — new season with stat template defined
- `season_completed` — season finalized with win/loss record
- `roster_document_uploaded` — medical clearance or contract stored

**event stream:**
- `event_scheduled` — game, practice, or meeting created
- `event_rescheduled` — date, time, or venue changed
- `event_cancelled` — event cancelled with reason
- `rsvp_submitted` — player/parent responded with availability
- `rsvp_changed` — availability response updated
- `lineup_set` — starting lineup and formation set by coach
- `match_started` — kickoff / tip-off recorded
- `score_updated` — score changed during match (live scoring)
- `match_completed` — final score and result recorded
- `event_synced_from_offline` — buffered offline events replayed to store

**stats stream:**
- `player_stats_recorded` — per-player stats for a match/session
- `player_rating_submitted` — coach rated player performance
- `training_session_logged` — drills, load, RPE recorded
- `training_attendance_recorded` — per-player attendance for session
- `wearable_data_imported` — GPS/load data from Catapult/STATSports

**video stream:**
- `video_uploaded` — raw video file uploaded
- `video_tagged_manually` — human-created event tag (goal, foul, save)
- `video_auto_tagged` — AI-generated event tags from footage
- `highlight_created` — shareable clip extracted from full match

**scouting stream:**
- `prospect_added` — new prospect entered into scouting database
- `prospect_evaluated` — attribute ratings recorded
- `prospect_video_linked` — video clip associated with prospect
- `prospect_status_changed` — watching→interested→contacted→trialing→signed/passed
- `prospect_comparison_generated` — AI similarity search results

**ai stream:**
- `match_recap_generated` — LLM-drafted post-match summary
- `parent_email_drafted` — AI-generated parent communication
- `lineup_suggested` — AI-recommended lineup from availability + stats
- `schedule_conflict_detected` — venue/official/opponent overlap found
- `injury_risk_flagged` — load + wellness signals triggered warning
- `stat_query_answered` — natural-language stat query response
- `suggestion_applied` / `suggestion_dismissed`

---

## Stream Snapshot

```sql
CREATE TABLE stream_snapshot (
    stream_type         TEXT NOT NULL,
    stream_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Projection Checkpoint

```sql
CREATE TABLE projection_checkpoint (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence       BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model: Team Dashboard

```sql
CREATE TABLE rm_team_dashboard (
    team_id             UUID PRIMARY KEY,
    club_id             UUID NOT NULL,
    team_name           TEXT NOT NULL,
    sport               TEXT NOT NULL,
    age_group           TEXT,
    level               TEXT,
    home_venue          TEXT,
    roster_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "display_name": "Sam Johnson",
    --   "role": "player", "jersey_number": 10, "position": "midfielder",
    --   "is_captain": true, "eligibility": "eligible",
    --   "attendance_rate": 0.92,
    --   "current_injury": null,
    --   "season_stats": {"goals": 12, "assists": 8, "avg_rating": 7.8}
    -- }]
    current_season_json JSONB,
    -- {
    --   "id": "uuid", "name": "Fall 2026",
    --   "record": {"wins": 8, "losses": 2, "draws": 1},
    --   "next_event": {"id": "uuid", "title": "vs City FC", "start_at": "2026-05-28T15:00:00Z"},
    --   "available_count": 14, "unavailable_count": 3
    -- }
    prospects_count     INTEGER NOT NULL DEFAULT 0,
    total_players       INTEGER NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dashboard_club ON rm_team_dashboard (club_id, sport);
```

---

## Read Model: Schedule

```sql
CREATE TABLE rm_schedule (
    event_id            UUID PRIMARY KEY,
    team_id             UUID NOT NULL,
    club_id             UUID NOT NULL,
    team_name           TEXT NOT NULL,
    sport               TEXT NOT NULL,
    season_id           UUID,
    event_type          TEXT NOT NULL,
    title               TEXT NOT NULL,
    start_at            TIMESTAMPTZ NOT NULL,
    end_at              TIMESTAMPTZ,
    venue               TEXT,
    opponent            TEXT,
    is_home             BOOLEAN,
    status              TEXT NOT NULL,
    score_home          INTEGER,
    score_away          INTEGER,
    result              TEXT,
    rsvp_summary_json   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "available": 14, "unavailable": 3, "maybe": 2, "no_response": 4,
    --   "responses": [{
    --     "user_id": "uuid", "name": "Sam Johnson", "status": "available"
    --   }]
    -- }
    lineup_json         JSONB,
    has_stats           BOOLEAN NOT NULL DEFAULT FALSE,
    has_video           BOOLEAN NOT NULL DEFAULT FALSE,
    is_offline_synced   BOOLEAN NOT NULL DEFAULT FALSE,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedule_team ON rm_schedule (team_id, start_at DESC);
CREATE INDEX idx_schedule_upcoming ON rm_schedule (start_at)
    WHERE status = 'scheduled';
CREATE INDEX idx_schedule_club ON rm_schedule (club_id, start_at DESC);
```

---

## Read Model: Player Profile

```sql
CREATE TABLE rm_player_profile (
    user_id             UUID PRIMARY KEY,
    club_id             UUID NOT NULL,
    display_name        TEXT NOT NULL,
    avatar_url          TEXT,
    date_of_birth       DATE,
    is_minor            BOOLEAN NOT NULL DEFAULT FALSE,
    teams_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "team_id": "uuid", "team_name": "U-15 Red",
    --   "sport": "soccer", "jersey_number": 10,
    --   "position": "midfielder", "is_captain": true,
    --   "season": "Fall 2026", "attendance_rate": 0.92
    -- }]
    career_stats_json   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_matches": 120, "total_goals": 45, "total_assists": 32,
    --   "total_minutes": 8400, "avg_rating": 7.6,
    --   "by_season": [{
    --     "season": "Fall 2026", "matches": 11, "goals": 12,
    --     "assists": 8, "avg_rating": 7.8
    --   }]
    -- }
    development_json    JSONB,
    -- {
    --   "goals": ["improve weak foot","increase stamina"],
    --   "coach_notes": "Strong technically, needs more defensive awareness",
    --   "last_reviewed": "2026-05-01",
    --   "progress_timeline": [
    --     {"date": "2026-03-01", "note": "Starting to use left foot in training"},
    --     {"date": "2026-05-01", "note": "Scored with left foot in match"}
    --   ]
    -- }
    injuries_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "ankle_sprain", "body_area": "left_ankle",
    --   "severity": "moderate", "occurred_at": "2026-04-15",
    --   "status": "cleared", "cleared_at": "2026-05-10",
    --   "days_missed": 25
    -- }]
    payments_json       JSONB NOT NULL DEFAULT '[]',
    compliance_json     JSONB NOT NULL DEFAULT '{}',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_profile_club ON rm_player_profile (club_id);
CREATE INDEX idx_profile_minor ON rm_player_profile (club_id)
    WHERE is_minor = TRUE;
```

---

## Read Model: Season Analytics

```sql
CREATE TABLE rm_season_analytics (
    season_id           UUID NOT NULL,
    team_id             UUID NOT NULL,
    club_id             UUID NOT NULL,
    season_name         TEXT NOT NULL,
    sport               TEXT NOT NULL,
    record_json         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "matches": 11, "wins": 8, "losses": 2, "draws": 1,
    --   "goals_for": 32, "goals_against": 14, "goal_difference": 18,
    --   "points": 25, "win_pct": 0.727
    -- }
    top_performers_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "user_id": "uuid", "name": "Sam Johnson",
    --   "goals": 12, "assists": 8, "avg_rating": 7.8,
    --   "minutes_played": 780
    -- }]
    team_stats_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "avg_goals_per_match": 2.9,
    --   "avg_conceded_per_match": 1.3,
    --   "avg_possession_pct": 58.2,
    --   "avg_pass_completion_pct": 76.5
    -- }
    attendance_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "avg_match_attendance_rate": 0.88,
    --   "avg_training_attendance_rate": 0.75,
    --   "total_events": 45,
    --   "by_player": [{
    --     "user_id": "uuid", "name": "Sam Johnson",
    --     "match_rate": 0.92, "training_rate": 0.80
    --   }]
    -- }
    training_load_json  JSONB,
    -- {
    --   "avg_session_load_au": 420,
    --   "weekly_load_trend": [
    --     {"week": "2026-W39", "avg_au": 380},
    --     {"week": "2026-W40", "avg_au": 450}
    --   ],
    --   "injury_risk_players": ["uuid1", "uuid2"]
    -- }
    financial_json      JSONB,
    -- {
    --   "total_collected_cents": 750000,
    --   "total_outstanding_cents": 125000,
    --   "registration_rate": 0.857
    -- }
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (season_id, team_id)
);
CREATE INDEX idx_analytics_club ON rm_season_analytics (club_id);
```

---

## Read Model: Scouting Board

```sql
CREATE TABLE rm_scouting_board (
    prospect_id         UUID PRIMARY KEY,
    club_id             UUID NOT NULL,
    team_id             UUID,
    prospect_name       TEXT NOT NULL,
    position            TEXT,
    date_of_birth       DATE,
    current_club        TEXT,
    sport               TEXT NOT NULL,
    status              TEXT NOT NULL,
    scouted_by          TEXT NOT NULL,
    attributes_json     JSONB NOT NULL DEFAULT '{}',
    -- {"speed": 8, "technique": 7, "tactical": 6, "physical": 8, "mental": 7, "overall": 7.2}
    video_json          JSONB NOT NULL DEFAULT '[]',
    -- [{"url": "https://...", "title": "U-15 tournament match", "tags": ["goal","dribble"]}]
    comparison_json     JSONB,
    -- {
    --   "similar_players": [
    --     {"user_id": "uuid", "name": "Sam Johnson", "similarity": 0.87,
    --      "shared_attributes": ["speed","technique"]}
    --   ]
    -- }
    notes               TEXT,
    timeline_json       JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"event": "prospect_added", "at": "2026-03-01", "by": "Coach Williams"},
    --   {"event": "prospect_evaluated", "at": "2026-03-15", "overall": 7.2},
    --   {"event": "prospect_status_changed", "at": "2026-04-01", "to": "interested"}
    -- ]
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_club ON rm_scouting_board (club_id, sport, status);
CREATE INDEX idx_scouting_attributes ON rm_scouting_board USING GIN (attributes_json);
```

---

## Example Event Replay: Match Day Lifecycle

```sql
SELECT event_type, ce_time, actor_type, actor_role,
       event_data, metadata
FROM event_store
WHERE stream_type = 'event'
  AND stream_id = 'match-uuid'
ORDER BY sequence_number;

-- Results show:
-- 1. event_scheduled (vs City FC, Saturday 3pm, Home Stadium)
-- 2. rsvp_submitted (Sam: available)
-- 3. rsvp_submitted (Alex: unavailable, reason: family event)
-- 4. rsvp_changed (Chris: maybe → available)
-- 5. lineup_set (11 starters, 5 subs)
-- 6. match_started (kickoff 3:02pm)
-- 7. score_updated (1-0, Sam goal 23')
-- 8. score_updated (1-1, opponent 38')
-- 9. score_updated (2-1, Sam goal 67')
-- 10. match_completed (2-1 win)
-- 11. player_stats_recorded (Sam: 2 goals, 78 min, rating 8.2)
-- 12. video_uploaded (full match footage)
-- 13. video_auto_tagged (AI: 3 events tagged)
-- 14. match_recap_generated (AI drafted post-match summary)
```

### Offline sync replay

```sql
-- Events synced from offline match-day device
SELECT event_type, ce_time,
       metadata->>'offline_device_id' AS device,
       metadata->>'offline_recorded_at' AS recorded_at,
       event_data
FROM event_store
WHERE stream_id = 'match-uuid'
  AND actor_type = 'offline_sync'
ORDER BY (metadata->>'offline_recorded_at')::TIMESTAMPTZ;
```

### GDPR minor-data access audit

```sql
SELECT ce_time, actor_id, actor_role, event_type,
       event_data->>'user_id' AS minor_user_id,
       ip_address
FROM event_store
WHERE involves_minor = TRUE
  AND club_id = 'club-uuid'
  AND ce_time BETWEEN '2026-05-01' AND '2026-05-31'
ORDER BY ce_time;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshot, projection_checkpoint |
| Read Models | 5 | rm_team_dashboard, rm_schedule, rm_player_profile, rm_season_analytics, rm_scouting_board |
| **Total** | **8** | |

---

## Key Design Decisions

1. **`involves_minor` flag on events** — GDPR and COPPA require tracking all access to minor data. The boolean flag on every event enables instant compliance queries ("who accessed minor data in May?") without scanning event payloads. This is the single most important privacy control.

2. **`offline_sync` actor type** — match-day events entered without connectivity are buffered locally and replayed to the event store when connectivity returns. The `offline_sync` actor type and metadata (device ID, original timestamp) provide the audit trail for delayed event ingestion.

3. **Parental consent as user stream events** — `parental_consent_granted` and `parental_consent_revoked` events capture the full consent lifecycle for COPPA/GDPR compliance. Point-in-time queries can verify whether consent was active at any moment.

4. **`rm_schedule` with RSVP summary** — the schedule view is the most frequently accessed page. Pre-computing RSVP counts (available, unavailable, maybe, no_response) in the read model eliminates the need to aggregate individual RSVP events at query time.

5. **`rm_player_profile` with career stats** — player development tracking across seasons requires aggregating stats over time. The read model pre-computes career totals and per-season breakdowns, enabling the longitudinal development view without event replay.

6. **`rm_season_analytics` with training load** — combining match results, attendance rates, and training load metrics in one read model enables the injury-risk dashboard: players with high load and declining attendance may be at risk.

7. **Stats and video as separate streams** — player stats and video tags have different producers (coaches vs. AI vision models) and different consumers (analytics dashboard vs. video player). Separate streams prevent video processing events from interleaving with stat recording events.

8. **`score_updated` as incremental events** — live scoring produces a sequence of score change events rather than a final score. This enables live scoreboard streaming via Server-Sent Events and temporal replay of match progression.

9. **CloudEvents envelope** — `ce_source`, `ce_specversion`, `ce_type`, and `ce_time` follow the CloudEvents 1.0 specification. This enables webhook delivery to external systems (league websites, parent notification services) using a standard envelope format.

10. **8 tables** — three infrastructure tables (event store, snapshots, checkpoints) plus five read models covering the core views: team management, scheduling, player profiles, season analytics, and scouting.
