# Challenge 4 — Findings

## Root cause
The failure was not in the Lambda code — the handler correctly calls
`dynamodb:GetItem` on the `challenge4-data` table. The problem was the
function's **IAM execution role**. The "bad deploy" left the role
(`challenge-4-AppRole-...`) with only the `AWSLambdaBasicExecutionRole`
managed policy, which grants CloudWatch Logs access but **no DynamoDB
permissions**. So every invocation failed at the `get_item` call with:

```
AccessDeniedException: User: arn:aws:sts::414691912352:assumed-role/challenge-4-AppRole-.../challenge4-app-fn
is not authorized to perform: dynamodb:GetItem on resource:
arn:aws:dynamodb:us-east-1:414691912352:table/challenge4-data
because no identity-based policy allows the dynamodb:GetItem action
```

In short: correct code, but the runtime identity lacked the permission the
code needs to talk to its dependency.

## Fix applied
This was a permissions/config fix, not a code change. I attached an inline
policy named `AllowReadChallenge4Table` to the function's execution role,
granting least-privilege read access to the specific table:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:414691912352:table/challenge4-data"
    }
  ]
}
```

After applying it, the function returned `200` with the seeded product and
the `challenge4-app-fn-errors` alarm returned to OK (green).

Healthy response:
```json
{"statusCode": 200, "body": "{\"item\": {\"id\": {\"S\": \"1\"}, \"product\": {\"S\": \"Builders Hoodie\"}, \"price\": {\"N\": \"49\"}}}"}
```

## Prevention (bonus)
To stop this class of "correct code, missing permission" regression from
shipping again:
- Add a post-deploy **smoke test** that actually invokes the function against
  its real dependencies and fails the pipeline on a non-200 result.
- Run **IAM policy validation** in CI (IAM Access Analyzer policy checks,
  `cfn-lint`/`cfn-nag`) to catch roles that reference resources they can't access.
- Keep IAM permissions defined **alongside the function in the same template**
  and review them as part of the deploy, so the grant can't drift away from the code.

## Evidence

### Screenshot 1: the agent's root-cause finding
The DevOps Agent investigating `challenge4-app-fn` after the deploy and surfacing the missing `dynamodb:GetItem` permission.

![Agent investigation of challenge4-app-fn post-deploy](https://raw.githubusercontent.com/DeepikaSidda/AWS-Devops-Agent/main/Screenshots/investigate-challenge4-app-fn-post-deploy-request.png)

### Screenshot 2: the function's source code
The `challenge4-app-fn` code — confirming the handler is correct and the fault is not in the code.

![Source code for challenge4-app-fn](https://raw.githubusercontent.com/DeepikaSidda/AWS-Devops-Agent/main/Screenshots/code-source-for-challenge64-app-fn.png)
