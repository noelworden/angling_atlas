# Plan: New Angling Atlas Phoenix Project

## Context
The existing `atlas` project at `~/code/atlas` is outdated (Phoenix 1.5, Elixir 1.11, Gigalixir deployment, EEX templates, Pow auth). The goal is to create a modern replacement at `~/code/angling_atlas` using the same core database schema, with Phoenix 1.7+, LiveView UI, `phx.gen.auth` for authentication, and deployed to Fly.io. A database dump from the existing app will be imported.

---

## New Project Specs
- **Location:** `~/code/angling_atlas`
- **App name:** `atlas` (modules: `Atlas`, `AtlasWeb`)
- **Phoenix:** 1.7+ (latest)
- **Auth:** `phx.gen.auth` (no Pow)
- **Frontend:** LiveView (no EEX-only templates)
- **Database:** PostgreSQL (same schema as current)
- **Deployment:** Fly.io (`fly launch`)
- **No Docker** for local development (Dockerfile kept for Fly.io remote builds only)
- **No CSV import utility** (omitted from scope)

---

## GitHub Issues
Steps 1–8 below are each tracked as a GitHub issue at https://github.com/noelworden/angling_atlas/issues

---

## Implementation Steps

### 1. Bootstrap the project and GitHub repo
```bash
cd ~/code
mix phx.new angling_atlas --app atlas --live
cd angling_atlas
git init && git add -A && git commit -m "Initial Phoenix project"
gh repo create angling_atlas --public --source=. --remote=origin --push
```
- `--live` enables LiveView out of the box
- Phoenix 1.7 generates a `Dockerfile` by default — **leave it in place** for Fly.io deployment (Fly's remote builder uses it; you never run Docker locally)
- `docker-compose.yml` is also generated but is only needed if you want to run Postgres in Docker locally — you can delete it and use a local Postgres install instead

### 2. Generate authentication
```bash
mix phx.gen.auth Accounts User users
mix deps.get
mix ecto.create
mix ecto.migrate
```
This generates a `users` table with email/hashed_password, full login/registration LiveViews, and session management — replacing Pow entirely.

### 3. Recreate the destinations migration
Create `priv/repo/migrations/<timestamp>_create_destinations.exs` mirroring the existing schema:
- `name :string`
- `description :string` (nullable)
- `longitude :decimal`
- `latitude :decimal`
- 20 boolean fields: `spinner_friendly`, `lake`, `car_friendly`, `hike_in`, `ice_fishing`, `allows_dogs`, `dogs_off_leash`, `fee`, `less_than_one_hour`, `one_to_three_hours`, `greater_than_three_hours`, `season_spring`, `season_summer`, `season_fall`, `season_winter`, `car_camp`, `backpack_camp`
- `timestamps()`

### 4. Port the Mapping context
Copy and adapt from the current project:
- `lib/atlas/mapping/destination.ex` — Ecto schema (unchanged fields)
- `lib/atlas/mapping.ex` — context with all filter functions (`list_filtered_destinations/8`, season/lake/distance/vehicle/dog/hike/camp/fee filters, coordinate helpers)

Remove Pow-specific dependencies; adapt to standard Phoenix/Ecto patterns. CSV import utility is not included.

### 5. Build LiveView UI
Replace old EEX-controller pattern with LiveViews:

| Old | New |
|---|---|
| `DestinationController#index` | `DestinationLive.Index` — filterable map/list |
| `DestinationController#show` | `DestinationLive.Show` |
| `DestinationController#new/create` | `DestinationLive.Form` (protected) |
| `DestinationController#edit/update` | `DestinationLive.Form` (protected) |
| `PageController#index` | Redirect or simple home LiveView |

Filter state lives in the LiveView socket assigns; form submissions update filters without page reload.

### 6. Router setup
```elixir
# Public routes
scope "/", AtlasWeb do
  pipe_through :browser
  live "/", DestinationLive.Index, :index
  live "/destinations/:id", DestinationLive.Show, :show
end

# Protected routes (require auth)
scope "/", AtlasWeb do
  pipe_through [:browser, :require_authenticated_user]
  live "/destinations/new", DestinationLive.Form, :new
  live "/destinations/:id/edit", DestinationLive.Form, :edit
end
```
`phx.gen.auth` routes (login, register, etc.) are added automatically.

### 7. Fly.io deployment
```bash
fly launch          # creates fly.toml, provisions Postgres, sets secrets
fly deploy          # first deploy
```
- Set `DATABASE_URL` and `SECRET_KEY_BASE` via `fly secrets set`
- Configure `config/runtime.exs` (Phoenix 1.7 projects use this by default)
- Fly.io Postgres cluster will be provisioned as part of `fly launch`

### 8. Import database dump
```bash
# Restore dump to local dev DB first to verify
psql atlas_dev < dump.sql

# For production, pipe to fly postgres connect or use fly ssh console
fly postgres connect -a <pg-app-name>
# then \i dump.sql or use pg_restore
```
The dump schema must align with the new migrations. Since we're recreating the same tables, this should be straightforward — just ensure column names/types match before importing.

---

## Critical Files to Create/Port

| File | Action |
|---|---|
| `mix.exs` | Generated; no extra deps needed |
| `priv/repo/migrations/*_create_destinations.exs` | Write new migration |
| `lib/atlas/mapping/destination.ex` | Port from current project |
| `lib/atlas/mapping.ex` | Port + clean up from current project |
| `lib/atlas_web/live/destination_live/index.ex` | New LiveView |
| `lib/atlas_web/live/destination_live/show.ex` | New LiveView |
| `lib/atlas_web/live/destination_live/form.ex` | New LiveView |
| `lib/atlas_web/router.ex` | Update routes |
| `fly.toml` | Generated by `fly launch` |

---

## Verification
1. `mix test` — all generated auth tests pass
2. `mix phx.server` — app runs locally, login/register works
3. Destinations CRUD works via LiveView (create, filter, edit, delete)
4. `fly deploy` succeeds; app accessible at fly.io URL
5. DB dump imported; existing destination data visible in production
