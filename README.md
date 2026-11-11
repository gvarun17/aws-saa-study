# AWS Solutions Architect – Associate (SAA-C03): 14-Day Plan

Start: 2026-10-30 (Day 1) · Target exam: 2026-11-13 (Day 15)

Each morning a new lesson lands in [`days/`](days/). Work through it, do the lab, drill the questions, and log misses in [`miss-log.md`](miss-log.md).

## Exam at a glance

| Item | Detail |
| --- | --- |
| Exam code | SAA-C03 |
| Questions | 65 (50 scored, 15 unscored), multiple choice / multiple response |
| Time | 130 minutes |
| Pass mark | 720 / 1000 (scaled) |
| Domains | Secure 30% · Resilient 26% · High-performing 24% · Cost-optimized 20% |

Confirm format and price on the official AWS Certification page before booking.

## Daily loop (3–4 hours)

1. **Learn (90 min)** — read the day's lesson end to end.
2. **Hands-on (30 min)** — do the lab. Free tier is enough; tear down afterwards.
3. **Drill (45 min)** — the lesson's practice questions, then 20+ more from a question bank on the same topics.
4. **Review (30 min)** — every miss gets one line in `miss-log.md`: what you picked, what won, and why.

**Go/no-go:** book Day 15 only if you score 80%+ on two full timed practice exams by Day 13. Under 70% on Day 12 → push back a week.

## Schedule

| Day | Date | Topic | Notes |
| --- | --- | --- | --- |
| 1 | 2026-10-30 | Global infrastructure, IAM, Organizations | [day-01](days/day-01-global-infra-iam.md) |
| 2 | 2026-10-31 | EC2, EBS, instance store, AMIs, purchasing options | [day-02](days/day-02-ec2-ebs.md) |
| 3 | 2026-11-01 | ELB (ALB/NLB/GWLB), Auto Scaling | [day-03](days/day-03-elb-auto-scaling.md) |
| 4 | 2026-11-02 | VPC networking, endpoints, hybrid connectivity | [day-04](days/day-04-vpc-networking.md) |
| 5 | 2026-11-03 | S3, EFS, FSx, Storage Gateway | [day-05](days/day-05-s3-efs-fsx-storage-gateway.md) |
| 6 | 2026-11-04 | RDS, Aurora, DynamoDB, ElastiCache, Redshift | [day-06](days/day-06-databases.md) |
| 7 | 2026-11-05 | Review + Practice Exam 1 | [day-07](days/day-07-review-practice-exam-1.md) |
| 8 | 2026-11-06 | SQS, SNS, EventBridge, Kinesis, Lambda, API Gateway, Step Functions | [day-08](days/day-08-serverless-messaging.md) |
| 9 | 2026-11-07 | ECS/EKS/Fargate, Route 53, CloudFront, Global Accelerator | [day-09](days/day-09-containers-edge.md) |
| 10 | 2026-11-08 | KMS, Secrets Manager, WAF/Shield, GuardDuty & co, Cognito | [day-10](days/day-10-security-services.md) |
| 11 | 2026-11-09 | DR strategies, Backup, migration, CloudWatch/CloudTrail/Config | [day-11](days/day-11-dr-migration-monitoring.md) |
| 12 | 2026-11-10 | Analytics, cost tools + Practice Exam 2 | [day-12](days/day-12-analytics-cost-practice-exam-2.md) |
| 13 | 2026-11-11 | Gap-closing + Practice Exam 3 | [day-13](days/day-13-gap-closing-practice-exam-3.md) |
| 14 | 2026-11-12 | Light review, cheat sheet, rest | — |

## Exam-day tactics

- Read the qualifier first: *MOST cost-effective*, *LEAST operational overhead*, *highest availability*, *minimal changes*. It decides between answers that all work.
- "Least operational overhead" → managed/serverless beats self-managed EC2.
- Eliminate two, then compare the last two against the qualifier.
- ~2 min per question; flag anything past 3 and move on.
- No penalty for guessing — never leave one blank. Multi-response tells you how many to pick.

## Resources

- Official SAA-C03 exam guide (AWS Certification site)
- AWS Skill Builder free official practice question set
- A practice-exam bank with explanations (Tutorials Dojo is widely used)
