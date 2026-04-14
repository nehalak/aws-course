# 02 — CloudTrail

## Concept
Records every AWS API call. Management events (free, 90-day history). Data events ($$, S3/Lambda/DDB operations).

## Exercises
1. **Create a trail** (if you didn't in account setup): multi-region, logs to S3, CloudWatch Logs integration on.
2. **Generate events**: `aws ec2 describe-instances`, create/delete a bucket. Wait 5-15 min.
3. **Query in Athena**: enable CloudTrail Lake or use the auto-generated Athena table. Find:
   - All `DeleteBucket` events in last 7 days.
   - Every API call made by `cdk-dev` user.
4. **Data events**: enable S3 data events on ONE bucket. Upload file. See `PutObject` appear. (Don't enable on all — expensive.)
5. **Insights**: enable CloudTrail Insights. Do a burst of 200 `DescribeInstances` in 1 min. See insight event next day.

## Verification
- Athena query returns your API calls.
- Insights flags unusual activity.

## Gotchas
- Data events ~$0.10 per 100k events. Can explode costs.
- CloudTrail is NOT real-time — 5-15 min delay.
- 90-day free history ≠ the trail's S3 data (which is permanent until you delete).

## Cleanup
Keep the management trail. Turn OFF data events.
