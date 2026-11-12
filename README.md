# rATE: live Vail Commons menu

The app reads today's menu from your Supabase database. A small function reads the Jamix feed on a schedule and fills that database, so "What's good today?" updates itself. Other dining places keep the labeled sample menu until you connect them.

## What is here
- `index.html`, `config.js`: the app (a static site, no build step).
- `supabase/migrations/001_menu.sql`: tables for locations, dishes, daily menu and sync history. Menus are public to read and cannot be written from the app.
- `supabase/functions/sync-jamix/`: the function that reads Jamix.

## Setup (about 20 minutes)
1. **Create a free project** at supabase.com. Note the project reference (the `abcdxyz` in `https://abcdxyz.supabase.co`).
2. **Create the tables.** Open the SQL Editor, paste `001_menu.sql`, and run it.
3. **Install the Supabase CLI** and run `supabase login`, then `supabase link --project-ref YOUR_REF` from this folder.
4. **Set the function's secrets** (pick any long random text for the secret):
   ```
   supabase secrets set SYNC_SECRET=choose-a-long-random-string
   ```
   Optional overrides if the defaults for the link you sent need changing: `JAMIX_CUSTOMER=98579`, `JAMIX_KITCHEN=2`, `JAMIX_MENU_TYPE=8`, `LOCATION_ID=p1`, or a full `JAMIX_FEED_URL`.
5. **Deploy it:** `supabase functions deploy sync-jamix --no-verify-jwt`
6. **Preview before it writes anything** (this is the important check):
   ```
   curl -H "x-sync-secret: YOUR_SECRET" "https://YOUR_REF.supabase.co/functions/v1/sync-jamix?dry=1"
   ```
   You should see today's Vail Commons items in `sample`. If `count` is 0 or you get an error, send me the response and I will adjust the parser. The feed address and field names were not checked against the live feed, so this step confirms them.
7. **Run it for real** (same command without `?dry=1`), then check the `daily_menu` table in the Supabase dashboard.
8. **Schedule it.** In the SQL Editor, enable `pg_cron` and `pg_net` (Database, Extensions), then run:
   ```sql
   select cron.schedule('sync-jamix', '0 */6 * * *', $$
     select net.http_post(
       url := 'https://YOUR_REF.supabase.co/functions/v1/sync-jamix',
       headers := jsonb_build_object('x-sync-secret', 'YOUR_SECRET', 'Content-Type', 'application/json'),
       body := '{}'::jsonb);
   $$);
   ```
   That refreshes the menu every 6 hours, which covers daily changes and same-day edits.
9. **Point the app at it.** Put your project URL and anon key in `config.js`.
10. **Put the site online.** Drag this folder onto Netlify Drop, or use Vercel or GitHub Pages. The published Claude page cannot reach Supabase, so the live version has to be hosted like this.

## How it behaves
- New dishes found in the feed are added automatically. A dish whose name matches one already in the app reuses it and keeps its ratings.
- New dishes start with no rating, shown as a dash, until students rate them.
- If a day has no menu posted, the app says so instead of showing sample food.
- If the database cannot be reached, the app falls back to the sample menu and says so.

## Not yet in this phase
Student accounts, saved ratings and photo storage still reset when the page reloads. They come next, using the same Supabase project.
