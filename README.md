# POC: AWS DevOps Agent — Lambda Root Cause Diagnosis

Four CloudFormation stacks, each a small variation on the same function,
designed so every fault produces a similar-looking DynamoDB error but has
a genuinely different root cause. This tests whether the agent can tell
them apart, not just detect "something is wrong."

| File | State | Root cause |
|---|---|---|
| `lambda.yaml` | Healthy baseline | — |
| `lambda-fault-iam.yaml` | Broken | IAM: missing `dynamodb:GetItem` permission |
| `lambda-fault-misconfig.yaml` | Broken | Config drift: `TABLE_NAME` env var points at a nonexistent table |
| `lambda-fault-codebug.yaml` | Broken | Application bug: wrong key name in `get_item` call |

## Deploy sequence

Deploy the same stack name each time, updating the template — this
creates a real deployment history for the agent to correlate against
(matching how "recent change" investigation actually works in practice).

```bash
STACK_NAME=poc-devops-agent

# 1. Deploy healthy baseline first, confirm it works
aws cloudformation deploy \
  --template-file lambda.yaml \
  --stack-name $STACK_NAME \
  --capabilities CAPABILITY_IAM

aws lambda invoke --function-name poc-devops-agent-fn out.json && cat out.json
# Expect: {"status": "ok", "item_found": false}  (table is empty, that's fine —
# the point is no error is thrown)

# 2. Inject fault #1 (IAM), invoke, let it fail, capture agent's RCA
aws cloudformation deploy \
  --template-file lambda-fault-iam.yaml \
  --stack-name $STACK_NAME \
  --capabilities CAPABILITY_IAM
aws lambda invoke --function-name poc-devops-agent-fn out.json; cat out.json

# 3. Roll back to healthy before the next fault, so each test starts clean
aws cloudformation deploy \
  --template-file lambda.yaml \
  --stack-name $STACK_NAME \
  --capabilities CAPABILITY_IAM

# 4. Inject fault #2 (misconfig), invoke, capture RCA
aws cloudformation deploy \
  --template-file lambda-fault-misconfig.yaml \
  --stack-name $STACK_NAME \
  --capabilities CAPABILITY_IAM
aws lambda invoke --function-name poc-devops-agent-fn out.json; cat out.json

# 5. Roll back to healthy again
aws cloudformation deploy \
  --template-file lambda.yaml \
  --stack-name $STACK_NAME \
  --capabilities CAPABILITY_IAM

# 6. Inject fault #3 (code bug), invoke, capture RCA
aws cloudformation deploy \
  --template-file lambda-fault-codebug.yaml \
  --stack-name $STACK_NAME \
  --capabilities CAPABILITY_IAM
aws lambda invoke --function-name poc-devops-agent-fn out.json; cat out.json

# 7. Clean up when done
aws cloudformation delete-stack --stack-name $STACK_NAME
```

## Why roll back between faults

Deploying each fault on top of the *previous* fault would compound
symptoms and make it unclear which change caused which error. Returning
to the healthy baseline between each test keeps every fault isolated —
important both for a clean demo and for fairly judging whether the agent
correctly attributes each failure to its actual cause.

## What to capture per fault, for the showcase

- Timestamp of the deploy (the "recent change" the agent should notice)
- Timestamp + count of failed invocations before the agent surfaces an answer
- The agent's stated root cause — does it correctly distinguish
  "permissions" (fault 1) vs "wrong resource reference" (fault 2) vs
  "application logic" (fault 3), or does it lump them together as
  "DynamoDB error"? This distinction is the actual value proposition to
  demonstrate, not just "it found an error."
- Time-to-RCA for each, to build the before/after comparison table
