---
name: supabase-keep-alive
description: Prevent Supabase free tier from pausing your database after 7 days of inactivity. Creates a keep-alive API endpoint + GitHub Actions cron job that pings it every 3 days. Zero cost, zero maintenance. Use when connecting any project to Supabase free tier.
user_invocable: true
---

# Supabase Keep-Alive — Prevent Free Tier Pausing

Supabase free tier pauses your database after 7 days without activity. This skill sets up an automated keep-alive system that prevents that — zero cost, zero maintenance.

## How It Works

```
Every 3 days:
GitHub Actions → GET /api/keep-alive → DB query → Supabase sees activity → stays alive
```

## Step 1: Create the API Endpoint

Create `src/app/api/keep-alive/route.ts` (Next.js) or equivalent:

```ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET(req: NextRequest) {
  const authHeader = req.headers.get("authorization");
  const token = authHeader?.replace("Bearer ", "");
  const cronSecret = process.env.CRON_SECRET;

  if (!token || token !== cronSecret) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const supabase = await createClient();
    const { count, error } = await supabase
      .from("ANY_TABLE_NAME")
      .select("*", { count: "exact", head: true });

    if (error) {
      return NextResponse.json({ status: "error", message: error.message }, { status: 500 });
    }

    return NextResponse.json({
      status: "ok",
      timestamp: new Date().toISOString(),
      row_count: count,
    });
  } catch {
    return NextResponse.json({ status: "error", message: "Failed to reach database" }, { status: 500 });
  }
}
```

**Important:** Replace `"ANY_TABLE_NAME"` with an actual table name from your Supabase project.

## Step 2: Add CRON_SECRET

1. Generate any random string (e.g., `my-keepalive-secret-2026`)
2. Add to `.env`:
   ```
   CRON_SECRET=my-keepalive-secret-2026
   ```
3. Add to your hosting provider's environment variables (Render, Vercel, etc.)

## Step 3: Create GitHub Actions Workflow

Create `.github/workflows/keep-alive.yml`:

```yaml
name: Supabase Keep-Alive

on:
  schedule:
    - cron: '0 8 */3 * *' # Every 3 days at 8:00 AM UTC
  workflow_dispatch: # Allow manual trigger for testing

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping keep-alive endpoint
        run: |
          response=$(curl -s -w "\n%{http_code}" \
            -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" \
            https://YOUR-APP-URL/api/keep-alive)

          http_code=$(echo "$response" | tail -1)
          body=$(echo "$response" | head -1)

          echo "Status: $http_code"
          echo "Response: $body"

          if [ "$http_code" != "200" ]; then
            echo "Keep-alive ping failed!"
            exit 1
          fi
```

**Important:** Replace `YOUR-APP-URL` with your actual deployed URL.

## Step 4: Add GitHub Secret

1. Go to your GitHub repo → **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name: `CRON_SECRET`
4. Value: same value as in your `.env`

## Step 5: Test

1. Push the workflow to GitHub
2. Go to **Actions** tab → **Supabase Keep-Alive** → **Run workflow**
3. Should show green checkmark with `Status: 200`

## Verification Checklist

- [ ] `/api/keep-alive` returns `{"status":"ok"}` with correct Bearer token
- [ ] Returns `401` without token
- [ ] GitHub Actions workflow runs successfully (green checkmark)
- [ ] `CRON_SECRET` is set in: `.env`, hosting provider, GitHub Secrets

## Adapting to Other Frameworks

The concept works with any stack — you just need:
1. An HTTP endpoint that queries Supabase
2. A scheduled job that calls it every few days

| Framework | Endpoint Location |
|-----------|------------------|
| Next.js (App Router) | `src/app/api/keep-alive/route.ts` |
| Next.js (Pages Router) | `pages/api/keep-alive.ts` |
| Express | `app.get('/api/keep-alive', ...)` |
| FastAPI | `@app.get("/api/keep-alive")` |

## Notes

- Supabase pauses after **7 days** of inactivity — running every **3 days** gives a safe margin
- The query is lightweight (`SELECT count(*) ... head: true`) — no performance impact
- Bearer token auth prevents unauthorized access
- `workflow_dispatch` lets you test manually anytime
- Works with any Supabase table — just needs one valid table name
