# AWS Task 3 – Private S3, CloudTrail, CloudWatch, EC2 and Application Load Balancer

**AWS Region:** Asia Pacific (Sydney) — `ap-southeast-2`

## Project overview

This project completes two AWS exercises:

1. Create a private S3 bucket, upload objects, and capture S3 object-write activity using CloudTrail with delivery to CloudWatch Logs.
2. Run two EC2 Nginx web servers behind an internet-facing Application Load Balancer (ALB).

## Architecture

```text
S3 logging path:
Private S3 bucket
  └── CloudTrail S3 object data events (write events)
        └── CloudWatch Logs: /aws/cloudtrail/aws-task3-s3

Web application path:
Internet clients
  └── Application Load Balancer: aws-task3-alb (HTTP :80)
        └── Target group: aws-task3-tg (HTTP :80)
              ├── EC2: aws-task3-web-1 → Nginx → Server 1 page
              └── EC2: aws-task3-web-2 → Nginx → Server 2 page
```

## Resources created

| Resource | Name / value |
|---|---|
| Region | `ap-southeast-2` |
| S3 bucket | `aws-task3-s3-sethu-2026` |
| S3 public access | Block all public access enabled |
| CloudTrail trail | `aws-task3-s3-trail` |
| CloudTrail data selector | S3 object data events, write-only, scoped to the task bucket |
| CloudWatch log group | `/aws/cloudtrail/aws-task3-s3` |
| ALB security group | `aws-task3-alb-sg` |
| EC2 security group | `aws-task3-ec2-sg` |
| Target group | `aws-task3-tg`, HTTP port 80 |
| Application Load Balancer | `aws-task3-alb`, internet-facing, HTTP port 80 |
| EC2 instances | `aws-task3-web-1`, `aws-task3-web-2` |
| Operating system | Amazon Linux 2023 |
| Web server | Nginx |

## S3 and CloudTrail configuration

The bucket `aws-task3-s3-sethu-2026` has Block Public Access enabled. Sample text files were uploaded to it.

The CloudTrail advanced event selector scopes S3 object data events to this bucket and records write operations using these fields:

- `eventCategory` equals `Data`
- `resources.type` equals `AWS::S3::Object`
- `resources.ARN` starts with `arn:aws:s3:::aws-task3-s3-sethu-2026/`
- `readOnly` equals `false`

The selector JSON is included at [`cloudtrail/s3-write-data-event-selector.json`](cloudtrail/s3-write-data-event-selector.json).

CloudTrail delivers events to `/aws/cloudtrail/aws-task3-s3` in CloudWatch Logs. The expanded event screenshot demonstrates a `PutObject` event for bucket `aws-task3-s3-sethu-2026` and key `cloudwatch-test.txt`.

## EC2 and load balancing

Both EC2 instances are registered in target group `aws-task3-tg` on HTTP port 80. Nginx serves a distinct HTML page on each instance so the selected backend can be identified in the browser.

Security group design:

- `aws-task3-alb-sg`: inbound HTTP port 80 from `0.0.0.0/0`.
- `aws-task3-ec2-sg`: inbound HTTP port 80 from the ALB security group; inbound SSH port 22 restricted to the operator's IP.

The target-group evidence screenshot shows **2 healthy targets and 0 unhealthy targets**.

## Application URL

[Open the application through the ALB](http://aws-task3-alb-32562749.ap-southeast-2.elb.amazonaws.com)

**Note:** The ALB DNS name is the application endpoint; it is not a fixed IP address. It may change if the load balancer is recreated. The project was tested with both the Server 1 and Server 2 pages appearing through the ALB DNS name.

## Web page source

- [Server 1 page](server-1/index.html)
- [Server 2 page](server-2/index.html)

To install either page on its EC2 instance, copy the relevant `index.html` to `/usr/share/nginx/html/index.html`, then reload Nginx:

```bash
sudo systemctl enable --now nginx
sudo cp index.html /usr/share/nginx/html/index.html
sudo systemctl reload nginx
```

## Evidence screenshots

The [`screenshots/`](screenshots/) directory contains the console and application evidence:

| Screenshot | Evidence |
|---|---|
| `01-s3-bucket-creation.png` | S3 bucket creation configuration |
| `02-s3-bucket.png` | S3 bucket console |
| `03-s3-files-uploaded.png` | Successful upload of sample objects |
| `04-cloudtrail-configuration.png` | CloudTrail trail configuration |
| `05-cloudwatch-log-group.png` | CloudWatch log group and streams |
| `06-cloudtrail-putobject-event.png` | Expanded S3 `PutObject` event details |
| `07-alb-security-group.png` | ALB security group inbound rule |
| `08-ec2-security-group.png` | EC2 security group inbound rules |
| `09-target-group-configuration.png` | Target group and registered EC2 targets |
| `10-targets-healthy.png` | Both targets healthy |
| `11-load-balancer.png` | ALB details and listener |
| `12-server-1-output.png` | Server 1 response via ALB DNS |
| `13-server-2-output.png` | Server 2 response via ALB DNS |
| `14-cloudtrail-test-upload.png` | Test object upload after CloudTrail setup |
| `15-cloudwatch-putobject-results.png` | CloudWatch `PutObject` event search |

## Cost and cleanup

The ALB is billed while it exists, and EC2 instances incur compute charges while running. After evaluation:

1. Delete the ALB (`aws-task3-alb`).
2. Delete the target group (`aws-task3-tg`) after it is no longer associated with the ALB.
3. Terminate `aws-task3-web-1` and `aws-task3-web-2`.
4. Delete the task-specific security groups after dependent resources are removed.
5. If no longer needed, stop/delete the CloudTrail trail and CloudWatch log group, and remove the CloudTrail destination bucket according to retention requirements.
6. Keep or delete the private task S3 bucket according to the evaluator's evidence-retention requirements.

Do not delete a shared VPC or shared networking resources used by other projects. Check the AWS Billing console after cleanup to confirm no task-specific billable resources remain.

## Security note

No AWS access keys, passwords, private SSH keys, or other credentials belong in this repository.
