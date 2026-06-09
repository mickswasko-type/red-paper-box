# Daily Front Pages — a personal news archive (2008–2016)

A static, searchable archive of daily newspaper front pages, built as a personal,
non-commercial preservation project. Cover images are hosted here; full issues open
from Google Drive.

- `index.html` — the searchable archive (search, year/month/day filters, lazy-loaded grid)
- `covers/` — optimized cover images, named `YYYY-MM-DD.jpg`
- `collections/` — featured curated threads
- `redeye-data.js` — the dataset (one record per date)

Served via GitHub Pages. Not affiliated with, endorsed by, or sponsored by any
publisher. Material was sourced from the public Internet Archive.

---

## Guess the Year — the game (`/game/`)

A press-your-luck streak game: see a random front page, guess the year (2008–2016),
build a streak, and bank it to the leaderboard before you bust. Linked from the main
archive page. `noindex, nofollow`.

**No date leakage.** The game never references covers by their dated filename. Each
playable cover is copied to `game/img/<gid>.jpg`, where `<gid>` is a date-free hash
(`build_game.py`). The copies are byte-identical to `covers/`, so Git stores one blob
per image — they cost ~nothing extra to push. The real date lives only in
`redeye-data.js` (reused, not duplicated) and is shown **only** after you guess.

Regenerate after adding covers: `python3 build_pages.py && python3 build_game.py`
(run from the `redeye-archive` tools dir), then commit/push.

### Leaderboard

Out of the box the leaderboard is **per-device (localStorage)** — works immediately,
but each player sees only their own top scores. To make it **shared/global**, wire up
a free Supabase project (5 min, no card):

1. Create a project at supabase.com → note the **Project URL** and the **anon public key**
   (Project Settings → API). The anon key is safe to ship in client code.
2. In the SQL editor, run:
   ```sql
   create table public.scores (
     id bigint generated always as identity primary key,
     name text not null check (char_length(name) between 1 and 12),
     streak int not null check (streak between 0 and 2304),
     created_at timestamptz not null default now()
   );
   alter table public.scores enable row level security;
   -- anyone may read the board:
   create policy "read scores"  on public.scores for select using (true);
   -- anyone may add a score, but not update or delete:
   create policy "insert scores" on public.scores for insert with check (
     char_length(name) between 1 and 12 and streak between 0 and 2304
   );
   ```
   (The `check` constraints are the server-side abuse caps: no empty names, no streaks
   above the archive size. No update/delete policy = clients can't edit the board.)
3. In `game/index.html`, fill in the two constants near the top of the `<script>`:
   ```js
   const SUPABASE_URL  = "https://YOURPROJECT.supabase.co";
   const SUPABASE_ANON = "your-anon-public-key";
   ```
4. Commit & push. The game auto-switches to the shared board (the footer label flips
   from "this device" to "shared").

**Alternative (Google Workspace):** a Google Apps Script web app appending to a Sheet
also works — swap `writeBoard`/`readBoard` to `fetch` your script URL. Supabase is
lighter to wire up, so it's the recommendation.

Client-side guards (always on): names sanitized to alphanumerics + spaces, max 12,
empties rejected; streak capped at the archive size; one insert per bank; busts can't
post (only banking a streak > 0 reaches the board).
