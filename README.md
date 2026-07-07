# Challenge 4: Bad Deploy Detective ⭐⭐⭐⭐ Expert

## The Scenario
An app (`challenge4-app-fn`) that reads from a DynamoDB table **used to work fine**. After a recent deploy, **every request fails** — but when you open the code, it looks completely correct. The bug isn't where you'd expect. Something *around* the function changed during that deploy.

Your job: use **AWS DevOps Agent** to find why a function with correct-looking code still fails on every run, and fix it. This one is harder on purpose — the root cause is not in the code. The README won't tell you what it is.

## Time
~25 minutes. Cheap (one Lambda + a tiny DynamoDB table).

## Before You Start
Finish [SETUP.md](../SETUP.md). Everything here works point-and-click; the AWS CLI steps are an optional shortcut.

> 🔎 **The agent can only investigate what it can see.** Deploy this challenge into the **same AWS account and Region your Agent Space monitors** (these steps use `us-east-1`, where DevOps Agent runs). After the stack is created, give the agent a few minutes to discover the new resources before you expect them in its topology.

## Steps

### 1. Create the infrastructure

**Option A — AWS CLI (fastest).** From this challenge's folder, run:
```bash
aws cloudformation deploy \
  --stack-name challenge-4 \
  --template-file template.yaml \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```
Wait for `Successfully created/updated stack - challenge-4`.

**Option B — Console (point-and-click).**
1. Console search → **CloudFormation** → **Create stack** → **With new resources (standard)**.
   > ⚠️ Pick **"standard"**, NOT "With existing resources (import resources)" — the import option throws a `DeletionPolicy` error.
2. **Choose an existing template** → **Upload a template file** → **Choose file** and upload:
   ```
   May-2026/challenge-4-bad-deploy-detective/template.yaml
   ```
3. **Next** → Stack name: `challenge-4` → **Next** → **Next**.
4. Tick **"I acknowledge… IAM resources"** → **Submit**. Wait for **CREATE_COMPLETE**.

### 2. Reproduce the failure
1. Console search → **Lambda** → open `challenge4-app-fn` → **Test** tab.
2. Click **Test** (any event name, defaults are fine). It fails.
3. Run it **3–4 times** so the failures register on the alarm `challenge4-app-fn-errors`.
4. Open the **Code** tab and read the function. Notice the code looks fine — so why is it failing?

### 3. Investigate with the agent
1. Open the **DevOps Agent** web app → **Chat**.
2. Ask it to investigate, for example:
   ```
   My app challenge4-app-fn started failing on every request after a deploy,
   but the code looks correct. Investigate the root cause and tell me how to fix it.
   ```
3. Read the agent's investigation. Work out what changed that the code can't control.

### 4. Apply the fix (required)
1. Use what the agent found to correct the problem — the fix is **not** a code change.
2. Run **Test** again — it should now succeed and the alarm `challenge4-app-fn-errors` should clear.
   > ℹ️ A healthy response returns the seeded product, e.g. `{"statusCode": 200, "body": "{\"item\": {\"id\": {\"S\": \"1\"}, \"product\": {\"S\": \"Builders Hoodie\"}, \"price\": {\"N\": \"49\"}}}"}`.
3. Ask the agent to confirm the app is healthy again.

<details>
<summary>🔒 Stuck on the fix? Click for the exact steps (spoiler)</summary>

The function's IAM role is missing permission to read the DynamoDB table. Grant it:

1. Console → **IAM** → **Roles** → search for the role attached to `challenge4-app-fn` (its name starts with `challenge-4-AppRole-`).
2. **Add permissions** → **Create inline policy** → **JSON** tab → paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["dynamodb:GetItem"],
         "Resource": "arn:aws:dynamodb:us-east-1:<YOUR_ACCOUNT_ID>:table/challenge4-data"
       }
     ]
   }
   ```
3. Name it `AllowReadChallenge4Table` → **Create policy**.
4. Back in Lambda, run **Test** again — it now returns the product.

</details>

## ✅ You're done when…
- The agent has identified the real root cause (which is **not** in the function's code), **and**
- You've applied the fix so the function succeeds and the alarm clears.


## Hints
- The code is genuinely correct. If a function with good code still fails on every run, ask yourself: what does the function need *besides* its code to do its job?
- Point the agent at the alarm `challenge4-app-fn-errors`, the function, **and** the resources it talks to.
- Want the raw signal? Lambda → **Monitor** tab → **View CloudWatch logs** and read the actual error message.
- Agent says it can't find the function? Give resource discovery a few minutes, and double-check your Agent Space covers this account and Region (`us-east-1`).


## ⚠️ Destroy the infrastructure (cleanup)

Do this as soon as you finish.

**Option A — AWS CLI:**
```bash
aws cloudformation delete-stack --stack-name challenge-4 --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name challenge-4 --region us-east-1
```

**Option B — Console:** CloudFormation → `challenge-4` → **Delete** → confirm, wait for **DELETE_COMPLETE** (this also removes the `challenge4-data` DynamoDB table).

See [COST-AND-CLEANUP.md](../COST-AND-CLEANUP.md).
