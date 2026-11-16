# rATE: live app for Davidson

## Put it on GitHub (no terminal needed)
1. Fill in `config.js` with your Supabase project URL and anon key (skip to try the demo; it runs without Supabase).
2. On github.com click **+ → New repository**, name it `rate`, choose **Public**, then **Create repository**.
3. Click **uploading an existing file**. Drag in everything inside this folder (not the folder itself) and click **Commit changes**.
4. Open **Settings → Pages**. Set Source to **Deploy from a branch**, branch **main**, folder **/ (root)**, then **Save**.
5. After about a minute your app is at `https://YOUR-USERNAME.github.io/rate/`. In Supabase, set Authentication, URL Configuration, Site URL to that link.

To update later: open a file on GitHub, click the pencil icon, edit, and commit.
Without Supabase the app shows sample data. The live menu, accounts and shared reviews need the Supabase setup below (`supabase/setup-all.sql` runs all three database scripts at once).

Students sign in with their Davidson email, rate food, follow friends and see each other's reviews. Vail Commons shows its real daily menu. Anyone can add it to their phone's home screen so it works like an app.

## What is here
- `index.html`, `config.js`, `manifest.webmanifest`, `sw.js`, `icons/`: the app (a static site, no build step).
- `supabase/migrations/001_menu.sql`: locations, dishes, daily menu, sync history.
- `supabase/migrations/002_social.sql`: accounts, ratings, follows, likes, saved items, reports, photo storage, and the privacy rules that protect them.
- `supabase/functions/sync-jamix/`: the function that reads the Jamix menu.

## Setup (about 30 minutes)
1. **Create a free project** at supabase.com. Note the project reference (the `abcdxyz` in `https://abcdxyz.supabase.co`).
2. **Create the tables.** In the SQL Editor run `001_menu.sql`, then `002_social.sql`, in that order.
3. **Set up sign-in emails.** rATE signs people in with a 6-digit code, not a link or a password.
   - Authentication, Emails, Magic Link template: make sure the message includes `{{ .Token }}` (for example: `Your rATE code is {{ .Token }}`).
   - Authentication, Providers, Email: keep Email enabled and leave "Confirm email" on.
   - **Before you share it widely, add custom email sending** (Authentication, SMTP Settings; Resend and Postmark both have free tiers). Supabase's built-in email sender allows only a few emails an hour, so sign-in codes would stop arriving once a handful of people try at once.
4. **Install the Supabase CLI**, then from this folder run `supabase login` and `supabase link --project-ref YOUR_REF`.
5. **Set up the menu function:**
   ```
   supabase secrets set SYNC_SECRET=choose-a-long-random-string
   supabase functions deploy sync-jamix --no-verify-jwt
   ```
   Optional overrides: `JAMIX_CUSTOMER=98579`, `JAMIX_KITCHEN=2`, `JAMIX_MENU_TYPE=8`, `LOCATION_ID=p1`, or a full `JAMIX_FEED_URL`.
6. **Preview it before it writes anything** (the important check):
   ```
   curl -H "x-sync-secret: YOUR_SECRET" "https://YOUR_REF.supabase.co/functions/v1/sync-jamix?dry=1"
   ```
   You should see today's Vail Commons items under `sample`. If `count` is 0 or there is an error, send the response to Claude and the parser can be adjusted. The Jamix feed address and field names were not checked against the live feed, so this step confirms them.
7. **Run it for real** (same command without `?dry=1`) and check the `daily_menu` table.
8. **Schedule it.** Enable `pg_cron` and `pg_net` (Database, Extensions), then in the SQL Editor:
   ```sql
   select cron.schedule('sync-jamix', '0 */6 * * *', $$
     select net.http_post(
       url := 'https://YOUR_REF.supabase.co/functions/v1/sync-jamix',
       headers := jsonb_build_object('x-sync-secret', 'YOUR_SECRET', 'Content-Type', 'application/json'),
       body := '{}'::jsonb);
   $$);
   ```
9. **Connect the app.** Put your project URL and anon key in `config.js` (Project settings, API). The anon key is safe to publish because the database rules decide who can read and write.
10. **Put the site online.** Drag this folder onto Netlify Drop, or use Vercel or GitHub Pages. It must be served over https for sign-in and the home-screen install to work. The page published inside Claude cannot reach Supabase, so the live app has to be hosted like this.
11. **Make yourself a moderator** (optional). After you have signed in once, run in the SQL Editor:
    ```sql
    update public.profiles set is_moderator = true where username = 'your-username';
    ```

## Sharing it with friends
- Send the site link. In the app, **Invite friends** (Friends tab and Profile) opens the phone's share sheet or copies the link.
- **iPhone:** open the link in Safari, tap Share, then Add to Home Screen.
- **Android:** open it in Chrome, then menu, Install app.
- Only `@davidson.edu` addresses can create an account. The database enforces this, not just the app.

## What is real now
- Ratings, reviews, photos, follows, likes, saved items and profiles are saved and shared between students.
- Dish and place averages and rankings are computed from real ratings only. The sample ratings and demo students you saw in the preview do not appear.
- Private ratings are visible only to their author. A private profile shows its ratings only to students it follows.
- Anyone can report a review. After 3 reports it is hidden automatically. Moderators see the queue in the `open_reports` view (Table Editor) and can hide or restore reviews.
- Dishes from the Davis Café menu are included. New Vail Commons dishes are added by the daily sync and start with no rating.

## Good to know
- Averages are calculated from the most recent 2,000 ratings. That is plenty for launch; ask Claude to move the calculation into the database if you outgrow it.
- Other dining places (Wildcat Den, Qdoba, Chick-fil-A, Summit) still show a labeled sample "today" list until you give their menus.
- The app loads friends' latest posts when you open it and again each time you switch back to it.
