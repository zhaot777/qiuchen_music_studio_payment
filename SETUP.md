# Student Payments — Setup Guide

> ## ✅ Your deployment (done 2026-08-28)
>
> Parts 2 and 3 below are **already completed** in AWS account `048394351328` (us-west-2):
> - DynamoDB table: `student-payments` (provisioned 1/1 — always-free tier)
> - Lambda: `student-payments-sync` + public Function URL, verified end-to-end
> - **Endpoint URL**: `https://3wslzij635zj64kmyeidfnpksy0gkhkb.lambda-url.us-west-2.on.aws/`
> - **Secret token**: in the file `.secret-token` in this folder (never upload this file anywhere)
> - Tax-time CSV in any browser: `<endpoint>?mode=csv&token=<secret>`
>
> Still to do: **Part 1** (host the app + add to iPhone), then paste the URL + token into the app's ⚙️ Settings.
> Note (Oct 2025 AWS change): public Function URLs need **two** resource-policy statements —
> `lambda:InvokeFunctionUrl` (condition `FunctionUrlAuthType=NONE`) **and** `lambda:InvokeFunction`
> (condition `InvokedViaFunctionUrl=true`). Both are in place.

What you have in this folder:

| File | What it is |
|---|---|
| `index.html` | The whole app (UI + logic) |
| `manifest.webmanifest`, `sw.js`, `icon-*.png` | PWA files (home-screen icon, offline support) |
| `lambda/index.mjs` | Backend code you paste into AWS Lambda |

The app works **immediately with no cloud at all** — data is stored on the phone and CSV export works. The AWS part (Parts 2–3) adds the "Sync to Cloud" button and cloud restore. You can do Part 1 today and Parts 2–3 whenever.

---

## Part 1 — Put the app on your iPhone (~10 min)

The app needs to be served over HTTPS for PWA features. GitHub Pages is free and permanent.

1. Create a GitHub account at github.com if you don't have one.
2. Click **+ → New repository**. Name: `payments`. Set it to **Public**
   (required for free Pages; the page URL is unguessable-ish but public — the app contains
   no data, your data never lives in the repo, so this is fine).
3. In the new repo: **uploading an existing file** → drag in all files from this folder
   **except the `lambda/` folder and the `.secret-token` file**: `index.html`, `manifest.webmanifest`, `sw.js`,
   `icon-180.png`, `icon-192.png`, `icon-512.png`. Click **Commit changes**.
   ⚠️ Never upload `.secret-token` — the repo is public.
4. Repo **Settings → Pages** → under "Branch" pick `main`, folder `/ (root)` → **Save**.
5. Wait ~1 minute. Your app is now at
   `https://<your-username>.github.io/payments/`
6. **On your iPhone**: open that URL in **Safari** → tap the **Share** button →
   **Add to Home Screen** → **Add**.

You now have a "Payments" app icon. It opens full-screen, works offline, and keeps data on the phone.

> Updating the app later: edit/re-upload `index.html` in the GitHub repo, then in
> `sw.js` bump `spt-v1` to `spt-v2` so phones pick up the new version.

---

## Part 2 — Create your personal AWS account (~5 min, one time)

> Use a **personal** email and card. Never mix this with any work AWS account.

1. Go to **aws.amazon.com/free** → Create a Free Account.
2. You'll need: email, password, a credit card (identity check — this design stays at $0),
   and phone verification.
3. Choose the **Basic (free)** support plan.
4. Sign in to the **AWS Console**. In the top-right, pick a region close to you,
   e.g. **us-west-2 (Oregon)** — then stay in that region for all steps below.

Recommended: in **Billing → Budgets**, create a "Zero spend budget" (template) so you get an email if anything ever costs a cent.

---

## Part 3 — Backend: DynamoDB + Lambda (~10 min)

### 3a. DynamoDB table

1. Console → search **DynamoDB** → **Create table**.
2. Table name: `student-payments`
3. Partition key: `pk` , type **String**. No sort key.
4. **Customize settings** → Capacity mode: **Provisioned**,
   Read = **1**, Write = **1**, turn **off** auto scaling.
   (Provisioned 1/1 sits inside DynamoDB's *always-free* 25/25 allowance → $0 forever.)
5. **Create table**.

### 3b. Lambda function

1. Console → search **Lambda** → **Create function**.
2. Author from scratch. Name: `student-payments-sync`. Runtime: **Node.js 22.x**. → **Create**.
3. In the **Code** tab, delete the contents of `index.mjs` and paste in the full contents
   of `lambda/index.mjs` from this folder → **Deploy**.
4. **Configuration → Environment variables → Edit**, add:
   - `TABLE_NAME` = `student-payments`
   - `SECRET_TOKEN` = a long random string you invent (e.g. run `openssl rand -hex 24`
     in Terminal, or mash the keyboard for 30+ characters). Save it — you'll enter the
     same string in the app.
5. Give Lambda access to the table:
   **Configuration → Permissions** → click the **role name** (opens IAM) →
   **Add permissions → Create inline policy → JSON**, paste (replace both
   `YOUR_ACCOUNT_ID` — visible in the top-right account menu — and region if not us-west-2):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": ["dynamodb:BatchWriteItem", "dynamodb:Scan"],
       "Resource": "arn:aws:dynamodb:us-west-2:YOUR_ACCOUNT_ID:table/student-payments"
     }]
   }
   ```

   Name it `student-payments-table-access` → **Create policy**.
6. Create the URL: back in Lambda, **Configuration → Function URL → Create function URL**
   - Auth type: **NONE**  (the SECRET_TOKEN check in the code is what protects your data)
   - Leave "Configure CORS" **unchecked** (the code sends its own CORS headers).
   - **Save**. Copy the URL — it looks like `https://abc123.lambda-url.us-west-2.on.aws/`

### 3c. Connect the app

1. Open the Payments app on your phone → **⚙️ Settings**.
2. Paste the **Function URL** and your **SECRET_TOKEN** → **Save Settings**.
3. Tap the **☁️** button — the red badge shows how many records are waiting; after a
   successful sync you'll see "Synced N records to cloud ✓".

---

## Everyday use

- **Add a payment**: tap the student → amount is prefilled from last time, date defaults
  to today, method remembered → tap **Add Payment**. Two taps + typing the amount.
- **Add a student**: type the name in the search bar → tap "Add student".
- **Sync**: tap ☁️ whenever you like (it also auto-syncs when the phone comes back online).
- **Tax time**, two options:
  - In the app: Settings → **Export CSV** (shares a .csv you can save to Files, email, or open in Excel/Numbers/Google Sheets).
  - From any browser: `https://<your-function-url>/?mode=csv&token=<YOUR_TOKEN>` downloads a CSV of everything in the cloud.
- **New phone / lost phone**: install the app (Part 1 step 6), enter the same URL + token
  in Settings, tap **Restore from Cloud**. Everything comes back.

## Costs

- GitHub Pages: free.
- DynamoDB: always-free tier (your data is a few KB against a 25 GB allowance) → $0.
- Lambda: always-free tier is 1M requests/month; you'll use maybe 100 → $0.
- The one thing to avoid: don't create other resources in the account while poking around.
  The zero-spend budget from Part 2 will email you if anything ever accrues cost.
