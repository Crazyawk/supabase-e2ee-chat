![image](https://user-images.githubusercontent.com/52203828/145706223-72c70924-834e-4418-aba3-1b5e84c7f4e1.png)

# End-to-End Encrypted Chat with Supabase

An end-to-end encrypted chat using public keys and [Supabase](https://supabase.com). Built for Supabase's Holiday-hackdays hackathon.

Available at: <https://chat.arnu515.gq>

[Runner Up](https://supabase.com/blog/2021/12/17/holiday-hackdays-winners-2021#best-realtime-project#runner-up-1) in the "Best Realtime" category!

## How to use

- Sign in using email or github
- Click the + button in the sidebar and enter the ID of your friend. They can see it by visiting the same page.
- Both users **have to be online** to receive chat messages. This is because the chat messages are **NOT** stored on the server. They are stored in the browser's local IndexedDB storage.

## Team

This project was developed by me (@arnu515)! Here are my links:
- Website: https://arnu515.gq
- Dev.To: https://dev.to/arnu515

## How I used Supabase

Supabase was used for all realtime events. All friend requests and chat message events were sent to supabase. Supabase also stored users' profiles. Other features of Supabase that were used were functions and triggers. It was because of supabase that I got into SQL and PLPGSQL. I am thankful to the Supabase team for building an amazing open-source BaaS that helped me dive in to SQL and PostgreSQL.

Other libraries that were used were:
- ReactJS with Craco
- React Router DOM
- TailwindCSS, PostCSS and classnames for styling
- TweetNaCl for encryption
- @supabase/supabase-js for interacting with supabase
- Nanostores so I didn't have to learn Redux
- Formik for forms and yup for validation
- Dexie for providing a promise-based IndexedDB interface

## Supabase + Netlify setup (click-by-click, no skipped steps)

This is a **full, button-by-button** guide to get the app working on Supabase and Netlify.

### Part A — Create the Supabase project

1. Open your browser and go to **https://supabase.com**.
2. Click **Sign In** (top right) and log in.
3. On the dashboard, click **New Project**.
4. In **Organization**, pick the org you want to use.
5. In **Project name**, type a name (example: `supabase-e2ee-chat`).
6. In **Database password**, type a strong password (save this).
7. In **Region**, pick the closest region to you.
8. Click **Create new project**.
9. Wait for the project status to show **Healthy** (green).

### Part B — Copy the Supabase API keys (you will paste these into Netlify)

1. In the Supabase project, look at the **left sidebar**.
2. Click **Settings** (gear icon).
3. Click **API**.
4. Find **Project URL** and click the **Copy** button.
   - Save this as `REACT_APP_SUPABASE_API_URL`.
5. Find **anon public key** and click the **Copy** button.
   - Save this as `REACT_APP_SUPABASE_API_KEY`.

### Part C — Enable login providers (Email + GitHub)

The app has **Email** login and **GitHub** login.

#### Email
1. In the Supabase left sidebar, click **Authentication**.
2. Click **Providers**.
3. Scroll to **Email**.
4. Toggle **Enable Email** to ON.

#### GitHub
1. Still in **Authentication → Providers**, find **GitHub**.
2. Toggle **Enable GitHub** to ON.
3. Open a new browser tab and go to **https://github.com/settings/developers**.
4. Click **New OAuth App**.
5. Fill in:
   - **Application name**: `Supabase E2EE Chat`
   - **Homepage URL**: your Netlify site URL (you can update later)
   - **Authorization callback URL**: copy this from Supabase (it appears under the GitHub provider settings).
6. Click **Register application**.
7. Click **Generate a new client secret**.
8. Copy **Client ID** and **Client Secret**.
9. Go back to Supabase → GitHub provider settings.
10. Paste **Client ID** and **Client Secret**.
11. Click **Save**.

### Part D — Create the database tables (Table Editor, exact names/columns)

1. In the Supabase left sidebar, click **Database**.
2. Click **Table Editor**.
3. Click **New table** (top right).

#### Table 1: `profiles`
1. **Table name**: type `profiles`.
2. Click **Add column** and add these columns:
   - `id` → type **uuid**, set **Primary Key**.
   - `name` → type **text**.
   - `avatar_url` → type **text**.
   - `created_at` → type **timestamp**, set **Default value** to `now()`.
3. Click **Save**.

#### Table 2: `friends`
1. Click **New table**.
2. **Table name**: type `friends`.
3. Add columns:
   - `id` → type **int8** (or bigint), **Primary Key**, enable auto-increment.
   - `from_id` → type **uuid**.
   - `to_id` → type **uuid**.
   - `accepted` → type **bool**, default `false`.
   - `created_at` → type **timestamp**, default `now()`.
4. Click **Save**.

#### Table 3: `events`
1. Click **New table**.
2. **Table name**: type `events`.
3. Add columns:
   - `id` → type **int8** (or bigint), **Primary Key**, enable auto-increment.
   - `from_id` → type **uuid**.
   - `to_id` → type **uuid**.
   - `type` → type **text**.
   - `payload` → type **jsonb**.
   - `created_at` → type **timestamp**, default `now()`.
4. Click **Save**.

#### Table 4: `chat_messages`
1. Click **New table**.
2. **Table name**: type `chat_messages`.
3. Add columns:
   - `id` → type **int8** (or bigint), **Primary Key**, enable auto-increment.
   - `from_id` → type **uuid**.
   - `to_id` → type **uuid**.
   - `message` → type **text**.
   - `nonce` → type **text**.
   - `created_at` → type **timestamp**, default `now()`.
4. Click **Save**.

### Part E — Create the RPC functions (SQL Editor)

1. In the Supabase left sidebar, click **SQL Editor**.
2. Click **New query**.
3. Paste the SQL below.
4. Click **Run** (top right).

```sql
create or replace function public.accept_friend_request(req_id bigint)
returns void
language plpgsql
security definer
as $$
begin
  update public.friends
  set accepted = true
  where id = req_id;
end;
$$;

create or replace function public.update_profile(
  profile_id uuid,
  name text,
  avatar_hash text
)
returns void
language plpgsql
security definer
as $$
begin
  update public.profiles
  set
    name = update_profile.name,
    avatar_url = 'https://www.gravatar.com/avatar/' || avatar_hash || '?d=identicon'
  where id = profile_id;
end;
$$;
```

### Part F — Enable Realtime on tables

1. In the Supabase left sidebar, click **Database**.
2. Click **Replication** (or **Realtime**, depending on the UI version).
3. Find **Table replication**.
4. Toggle ON for these tables:
   - `friends`
   - `events`
   - `chat_messages`

### Part G — Turn on RLS and add policies (SQL Editor)

1. In the Supabase left sidebar, click **SQL Editor**.
2. Click **New query**.
3. Paste the SQL below.
4. Click **Run**.

```sql
alter table public.profiles enable row level security;
alter table public.friends enable row level security;
alter table public.events enable row level security;
alter table public.chat_messages enable row level security;

create policy "profiles_select_own"
on public.profiles
for select
using (auth.uid() = id);

create policy "profiles_update_own"
on public.profiles
for update
using (auth.uid() = id);

create policy "friends_select_own"
on public.friends
for select
using (auth.uid() = from_id or auth.uid() = to_id);

create policy "friends_insert_own"
on public.friends
for insert
with check (auth.uid() = from_id);

create policy "friends_update_own"
on public.friends
for update
using (auth.uid() = to_id);

create policy "friends_delete_own"
on public.friends
for delete
using (auth.uid() = from_id or auth.uid() = to_id);

create policy "events_insert_own"
on public.events
for insert
with check (auth.uid() = from_id);

create policy "events_select_own"
on public.events
for select
using (auth.uid() = to_id);

create policy "events_delete_own"
on public.events
for delete
using (auth.uid() = to_id);

create policy "chat_messages_insert_own"
on public.chat_messages
for insert
with check (auth.uid() = from_id);

create policy "chat_messages_select_own"
on public.chat_messages
for select
using (auth.uid() = to_id);

create policy "chat_messages_delete_own"
on public.chat_messages
for delete
using (auth.uid() = to_id);
```

### Part H — Netlify deployment (every click)

1. Open **https://app.netlify.com**.
2. Click **Log in** and sign in.
3. Click **Add new site**.
4. Click **Import an existing project**.
5. Choose your Git provider and connect your account.
6. Select the repo for this project.

#### Build settings
1. **Build command**: type `yarn build`.
2. **Publish directory**: type `build`.
3. Click **Deploy site**.

#### Environment variables
1. After deploy starts, click **Site settings**.
2. Click **Build & deploy**.
3. Scroll to **Environment**.
4. Click **Add environment variable**.
5. Add:
   - **Key**: `REACT_APP_SUPABASE_API_URL`
   - **Value**: (paste the Project URL you copied from Supabase)
6. Click **Create variable**.
7. Click **Add environment variable** again.
8. Add:
   - **Key**: `REACT_APP_SUPABASE_API_KEY`
   - **Value**: (paste the anon public key you copied from Supabase)
9. Click **Create variable**.

#### Redeploy with env vars
1. Click **Deploys** (left sidebar).
2. Click **Trigger deploy**.
3. Click **Deploy site**.

### Part I — Quick sanity check

1. Open your Netlify site URL.
2. Click **Login with Email** and enter an email you can access.
3. Open the magic link email and sign in.
4. Go to **Friends**, copy your ID, and add a friend (in another browser or account).
5. Accept the request and open a chat.
