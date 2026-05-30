# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Sports Team Management · Created: 2026-05-26

## Philosophy

Every concept in sports team management — clubs, users, teams, team memberships, seasons, events, RSVPs, match statistics, videos, prospects, injuries, and payments — gets its own table with strict foreign-key relationships. This mirrors the hierarchical structure of club sports: a club has teams, teams have seasons, seasons have events, events have RSVPs and statistics.

Sports team management has three dominant access patterns: (1) "show me my upcoming events across all my teams with RSVP status" (schedule view), (2) "show the team roster with positions, availability, and attendance record" (roster view), and (3) "show player X's stats and development across the season" (player profile). All three require joining through the team_members junction table but benefit from indexed lookups on event dates, team memberships, and player statistics.

Multi-sport support is achieved through sport-specific stat templates stored as JSONB on the seasons table, while the relational skeleton (events, RSVPs, team memberships) stays consistent across sports. This keeps the schema stable while allowing per-sport customisation of what gets measured.

**Best for:** Teams building a production platform where multi-club federation support, cross-sport analytics, complex scheduling with conflict detection, and GDPR/COPPA audit compliance are priorities.

**Trade-offs:**
- Pro: Clean hierarchy — club → team → season → event → stats — with FK integrity
- Pro: Cross-team schedule queries (conflict detection) via standard SQL joins
- Pro: Player stats queryable with standard aggregation (AVG, SUM, percentiles)
- Pro: GDPR right-to-erasure scoped per entity — delete player without affecting team history
- Con: 14 tables to maintain and migrate
- Con: Schedule view requires joins through events + rsvps + team_members
- Con: Sport-specific stat structures still need JSONB within the relational frame
- Con: Roster view with attendance, stats, and injury status requires 4+ joins

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| iCalendar (RFC 5545) | Events export as VEVENT; schedule sync with Google/Apple/Outlook |
| CalDAV (RFC 4791) | Two-way calendar sync |
| SportsML-G2 (IPTC) | Match results and roster data interchange |
| AsyncAPI 3.0 | Live-scoring event streams |
| CloudEvents 1.0 | Webhook envelopes for registration, RSVP, score events |
| ISO 8601 | All timestamps and durations |
| OpenAPI 3.1 | REST API definition |
| JSON Schema 2020-12 | Sport-specific stat template validation |
| OAuth 2.0 / OIDC / PKCE | Authentication; Google/Apple SSO |
| Stripe Connect | Payment processing for registration fees |
| GDPR | Player PII handling with consent tracking |
| COPPA | Under-13 data with parental consent |
| SafeSport | US youth sport abuse-prevention compliance |
| OWASP ASVS 4.0 | Application security baseline |
| MCP | AI assistant integration |

---

## Clubs

```sql
CREATE TABLE clubs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT UNIQUE NOT NULL,
    sport_primary       TEXT,
    country             TEXT NOT NULL,
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    logo_url            TEXT,
    website_url         TEXT,
    stripe_account_id   TEXT,
    subscription_tier   TEXT NOT NULL CHECK (subscription_tier IN (
                            'free','starter','pro','enterprise'
                        )) DEFAULT 'free',
    settings_json       JSONB NOT NULL DEFAULT '{}',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_clubs_slug ON clubs (slug);
```

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    phone               TEXT,
    avatar_url          TEXT,
    date_of_birth       DATE,
    gender              TEXT CHECK (gender IN ('male','female','other','prefer_not_to_say')),
    role                TEXT NOT NULL CHECK (role IN (
                            'admin','coach','assistant_coach',
                            'player','parent','medical_staff',
                            'manager','viewer'
                        )),
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple','microsoft'
                        )),
    is_minor            BOOLEAN NOT NULL DEFAULT FALSE,
    parent_user_id      UUID REFERENCES users(id),
    parental_consent_at TIMESTAMPTZ,
    gdpr_consent_at     TIMESTAMPTZ,
    safesport_cleared   BOOLEAN NOT NULL DEFAULT FALSE,
    safesport_cleared_at TIMESTAMPTZ,
    notification_prefs  JSONB NOT NULL DEFAULT '{"push": true, "email": true, "sms": false}',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_club ON users (club_id, role);
CREATE INDEX idx_users_parent ON users (parent_user_id) WHERE parent_user_id IS NOT NULL;
```

---

## Teams

```sql
CREATE TABLE teams (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
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
    home_venue_address  TEXT,
    color_primary       TEXT,
    color_secondary     TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_teams_club ON teams (club_id, sport);
```

---

## Team Members

```sql
CREATE TABLE team_members (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id             UUID NOT NULL REFERENCES teams(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    member_role         TEXT NOT NULL CHECK (member_role IN (
                            'player','coach','assistant_coach',
                            'manager','medical','parent_observer'
                        )),
    jersey_number       INTEGER,
    position            TEXT,
    position_secondary  TEXT,
    is_captain          BOOLEAN NOT NULL DEFAULT FALSE,
    eligibility_status  TEXT NOT NULL CHECK (eligibility_status IN (
                            'eligible','ineligible','pending','suspended'
                        )) DEFAULT 'eligible',
    joined_at           DATE NOT NULL DEFAULT CURRENT_DATE,
    left_at             DATE,
    documents_json      JSONB NOT NULL DEFAULT '[]',
    -- [{"type": "medical_clearance", "url": "...", "expires_at": "2027-01-01"}]
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, user_id)
);
CREATE INDEX idx_members_team ON team_members (team_id, member_role);
CREATE INDEX idx_members_user ON team_members (user_id);
```

---

## Seasons

```sql
CREATE TABLE seasons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id             UUID NOT NULL REFERENCES teams(id),
    name                TEXT NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    status              TEXT NOT NULL CHECK (status IN (
                            'upcoming','active','completed','cancelled'
                        )) DEFAULT 'upcoming',
    stat_template_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "sport": "soccer",
    --   "fields": [
    --     {"key": "goals", "label": "Goals", "type": "integer"},
    --     {"key": "assists", "label": "Assists", "type": "integer"},
    --     {"key": "minutes_played", "label": "Minutes", "type": "integer"},
    --     {"key": "pass_completion_pct", "label": "Pass %", "type": "decimal"},
    --     {"key": "shots_on_target", "label": "Shots on Target", "type": "integer"},
    --     {"key": "tackles_won", "label": "Tackles Won", "type": "integer"},
    --     {"key": "yellow_cards", "label": "Yellows", "type": "integer"},
    --     {"key": "red_cards", "label": "Reds", "type": "integer"}
    --   ]
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_seasons_team ON seasons (team_id, status);
```

---

## Events

```sql
CREATE TABLE events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id             UUID NOT NULL REFERENCES teams(id),
    season_id           UUID REFERENCES seasons(id),
    event_type          TEXT NOT NULL CHECK (event_type IN (
                            'match','practice','meeting','tournament',
                            'tryout','scrimmage','social','other'
                        )),
    title               TEXT NOT NULL,
    description         TEXT,
    start_at            TIMESTAMPTZ NOT NULL,
    end_at              TIMESTAMPTZ,
    duration_minutes    INTEGER,
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
    lineup_json         JSONB,
    -- [{
    --   "user_id": "uuid", "position": "GK", "jersey_number": 1,
    --   "is_starter": true, "sub_in_minute": null, "sub_out_minute": null
    -- }]
    match_notes         TEXT,
    weather             TEXT,
    is_offline_created  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_events_team ON events (team_id, start_at DESC);
CREATE INDEX idx_events_season ON events (season_id, start_at);
CREATE INDEX idx_events_upcoming ON events (start_at)
    WHERE status = 'scheduled';
```

---

## Event RSVPs

```sql
CREATE TABLE event_rsvps (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL REFERENCES events(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    status              TEXT NOT NULL CHECK (status IN (
                            'available','unavailable','maybe','no_response'
                        )) DEFAULT 'no_response',
    reason              TEXT,
    responded_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (event_id, user_id)
);
CREATE INDEX idx_rsvps_event ON event_rsvps (event_id, status);
CREATE INDEX idx_rsvps_user ON event_rsvps (user_id, created_at DESC);
```

---

## Player Stats

```sql
CREATE TABLE player_stats (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL REFERENCES events(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    season_id           UUID REFERENCES seasons(id),
    stats_json          JSONB NOT NULL DEFAULT '{}',
    -- Soccer example:
    -- {
    --   "goals": 2, "assists": 1, "minutes_played": 78,
    --   "pass_completion_pct": 85.2, "shots_on_target": 3,
    --   "tackles_won": 4, "yellow_cards": 0, "red_cards": 0
    -- }
    -- Basketball example:
    -- {
    --   "points": 18, "rebounds": 7, "assists": 5,
    --   "steals": 2, "blocks": 1, "turnovers": 3,
    --   "minutes_played": 32, "fg_pct": 45.5, "ft_pct": 80.0
    -- }
    player_rating       NUMERIC(3,1),
    coach_notes         TEXT,
    position_played     TEXT,
    is_starter          BOOLEAN,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (event_id, user_id)
);
CREATE INDEX idx_stats_player ON player_stats (user_id, season_id);
CREATE INDEX idx_stats_event ON player_stats (event_id);
CREATE INDEX idx_stats_json ON player_stats USING GIN (stats_json);
```

---

## Videos

```sql
CREATE TABLE videos (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id             UUID NOT NULL REFERENCES teams(id),
    event_id            UUID REFERENCES events(id),
    uploaded_by_id      UUID NOT NULL REFERENCES users(id),
    title               TEXT NOT NULL,
    video_url           TEXT NOT NULL,
    thumbnail_url       TEXT,
    duration_seconds    INTEGER,
    video_type          TEXT NOT NULL CHECK (video_type IN (
                            'full_match','highlights','training',
                            'scouting','analysis','other'
                        )),
    tags_json           JSONB NOT NULL DEFAULT '[]',
    -- [{"timestamp_sec": 120, "label": "goal", "player_id": "uuid", "ai_generated": true}]
    is_ai_tagged        BOOLEAN NOT NULL DEFAULT FALSE,
    visibility          TEXT NOT NULL CHECK (visibility IN (
                            'team_only','club','public','private'
                        )) DEFAULT 'team_only',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_videos_team ON videos (team_id, created_at DESC);
CREATE INDEX idx_videos_event ON videos (event_id) WHERE event_id IS NOT NULL;
CREATE INDEX idx_videos_tags ON videos USING GIN (tags_json);
```

---

## Prospects

```sql
CREATE TABLE prospects (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
    scouted_by_id       UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    date_of_birth       DATE,
    position            TEXT,
    current_club        TEXT,
    sport               TEXT NOT NULL,
    attributes_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "speed": 8, "technique": 7, "tactical_awareness": 6,
    --   "physicality": 8, "mental": 7,
    --   "overall_rating": 7.2
    -- }
    notes               TEXT,
    video_ids           UUID[] NOT NULL DEFAULT '{}',
    status              TEXT NOT NULL CHECK (status IN (
                            'watching','interested','contacted',
                            'trialing','signed','passed'
                        )) DEFAULT 'watching',
    comparison_json     JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prospects_club ON prospects (club_id, sport, status);
CREATE INDEX idx_prospects_attributes ON prospects USING GIN (attributes_json);
```

---

## Injuries

```sql
CREATE TABLE injuries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    team_id             UUID NOT NULL REFERENCES teams(id),
    injury_type         TEXT NOT NULL,
    body_area           TEXT NOT NULL,
    severity            TEXT NOT NULL CHECK (severity IN (
                            'minor','moderate','severe','season_ending'
                        )),
    occurred_at         DATE NOT NULL,
    occurred_during     TEXT CHECK (occurred_during IN (
                            'match','practice','personal','unknown'
                        )),
    event_id            UUID REFERENCES events(id),
    diagnosis           TEXT,
    treatment_notes     TEXT,
    return_to_play_date DATE,
    return_protocol     TEXT CHECK (return_protocol IN (
                            'full_clearance','graduated_return',
                            'limited_participation','specialist_review'
                        )),
    status              TEXT NOT NULL CHECK (status IN (
                            'active','recovering','cleared','chronic'
                        )) DEFAULT 'active',
    medical_staff_id    UUID REFERENCES users(id),
    cleared_by_id       UUID REFERENCES users(id),
    cleared_at          DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_injuries_player ON injuries (user_id, status);
CREATE INDEX idx_injuries_team ON injuries (team_id, status)
    WHERE status IN ('active', 'recovering');
```

---

## Payments

```sql
CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    team_id             UUID REFERENCES teams(id),
    payment_type        TEXT NOT NULL CHECK (payment_type IN (
                            'registration','membership_dues','kit_order',
                            'tournament_fee','venue_hire','fundraiser',
                            'other'
                        )),
    description         TEXT NOT NULL,
    amount_cents        BIGINT NOT NULL,
    currency            TEXT NOT NULL DEFAULT 'USD',
    status              TEXT NOT NULL CHECK (status IN (
                            'pending','paid','failed','refunded','waived'
                        )) DEFAULT 'pending',
    stripe_payment_id   TEXT,
    stripe_invoice_id   TEXT,
    due_date            DATE,
    paid_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_club ON payments (club_id, status);
CREATE INDEX idx_payments_user ON payments (user_id, status);
CREATE INDEX idx_payments_due ON payments (due_date)
    WHERE status = 'pending';
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID NOT NULL REFERENCES clubs(id),
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
CREATE INDEX idx_ai_team ON ai_suggestions (team_id, suggestion_type);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    club_id             UUID REFERENCES clubs(id),
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

### Upcoming events across all user's teams with RSVP status

```sql
SELECT e.title, e.event_type, e.start_at, e.venue,
       e.opponent, t.name AS team_name, t.sport,
       r.status AS rsvp_status
FROM events e
JOIN teams t ON t.id = e.team_id
JOIN team_members tm ON tm.team_id = t.id AND tm.user_id = 'user-uuid'
LEFT JOIN event_rsvps r ON r.event_id = e.id AND r.user_id = 'user-uuid'
WHERE e.start_at >= now()
  AND e.status = 'scheduled'
  AND tm.is_active = TRUE
ORDER BY e.start_at
LIMIT 20;
```

### Player season stats with averages

```sql
SELECT u.display_name,
       COUNT(ps.id) AS games_played,
       AVG((ps.stats_json->>'goals')::INTEGER) AS avg_goals,
       SUM((ps.stats_json->>'goals')::INTEGER) AS total_goals,
       SUM((ps.stats_json->>'assists')::INTEGER) AS total_assists,
       AVG((ps.stats_json->>'pass_completion_pct')::NUMERIC) AS avg_pass_pct,
       AVG(ps.player_rating) AS avg_rating
FROM player_stats ps
JOIN users u ON u.id = ps.user_id
WHERE ps.season_id = 'season-uuid'
GROUP BY u.id, u.display_name
ORDER BY total_goals DESC;
```

### Team roster with attendance rate and injury status

```sql
SELECT u.display_name, tm.jersey_number, tm.position,
       tm.is_captain, tm.eligibility_status,
       COUNT(r.id) FILTER (WHERE r.status = 'available') AS available_count,
       COUNT(r.id) AS total_events,
       ROUND(COUNT(r.id) FILTER (WHERE r.status = 'available')::NUMERIC
             / NULLIF(COUNT(r.id), 0), 2) AS attendance_rate,
       i.injury_type, i.status AS injury_status, i.return_to_play_date
FROM team_members tm
JOIN users u ON u.id = tm.user_id
LEFT JOIN event_rsvps r ON r.user_id = u.id
    AND r.event_id IN (SELECT id FROM events WHERE team_id = tm.team_id)
LEFT JOIN injuries i ON i.user_id = u.id AND i.team_id = tm.team_id
    AND i.status IN ('active', 'recovering')
WHERE tm.team_id = 'team-uuid'
  AND tm.is_active = TRUE
  AND tm.member_role = 'player'
GROUP BY u.id, u.display_name, tm.jersey_number, tm.position,
         tm.is_captain, tm.eligibility_status,
         i.injury_type, i.status, i.return_to_play_date
ORDER BY tm.jersey_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation | 1 | clubs |
| Users | 1 | users (admins, coaches, players, parents) |
| Teams | 3 | teams, team_members, seasons |
| Events | 2 | events, event_rsvps |
| Performance | 2 | player_stats, videos |
| Scouting | 1 | prospects |
| Health | 1 | injuries |
| Finance | 1 | payments |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **14** | |

---

## Key Design Decisions

1. **`team_members` as junction table** — a user can be on multiple teams (player on U-15 and U-17), and a team has many members with different roles. The junction table carries per-team context: jersey number, position, captaincy, and eligibility status.

2. **`seasons.stat_template_json`** — sport-specific stat fields (goals/assists for soccer, points/rebounds for basketball) are defined per season as a JSONB template. This makes the platform multi-sport without schema changes — adding a new sport means defining a new template, not a new table.

3. **`player_stats.stats_json`** — individual match/session stats conform to the season's stat template. JSONB enables per-sport flexibility while the relational frame (event_id, user_id, season_id) enables aggregation queries with standard SQL.

4. **`events.lineup_json`** — match lineups with starting positions, substitutions, and jersey numbers are embedded on the event row. A lineup is typically 11-25 entries and is always loaded with the event detail view.

5. **`users.is_minor` with `parent_user_id`** — GDPR/COPPA compliance requires tracking which users are minors and linking them to a parent account with parental consent timestamps. The `involves_minor` flag on audit_log enables privacy-focused audit queries.

6. **`injuries` as a separate table** — injury records accumulate over a player's career and have their own lifecycle (active → recovering → cleared). Keeping them separate enables injury-frequency analysis and return-to-play protocol tracking without embedding on the user row.

7. **`videos.tags_json` with `ai_generated` flag** — video event tags (goals, fouls, key plays with timestamps) are stored as JSONB. Each tag carries an `ai_generated` flag to distinguish human-tagged events from AI auto-tagged events, enabling confidence tracking and review workflows.

8. **`prospects` separate from `users`** — scouted prospects are not club members yet and don't need accounts. Keeping them in their own table avoids polluting the user base and enables scouting-specific attributes (ratings, comparison reports) without adding nullable columns to users.

9. **`event_rsvps` as a junction** — availability tracking per player per event is the core scheduling workflow. A separate table enables aggregate attendance queries and RSVP status summaries per event.

10. **14 tables** — sports team management spans administration (rosters, payments), operations (scheduling, RSVPs), performance (stats, video), health (injuries), and scouting. The normalized structure gives each concern its own table with clean FK relationships.
