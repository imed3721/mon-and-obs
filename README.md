# AWS Monitoring and Observability Platform

## 1. Architecture

This project implements an AWS-based observability platform for monitoring EC2 infrastructure, collecting system metrics and logs, visualizing performance, and automating website deployments.

The architecture combines AWS-native monitoring services with Grafana and a GitHub Actions CI/CD pipeline.

*Add Architecture diagram*

The deployment workflow validates website changes and uses AWS Systems Manager to execute deployment commands on EC2 without requiring SSH access for CI/CD.

The monitoring pipeline collects host-level metrics and configured logs in CloudWatch. Grafana provides dashboards and alert rules, with Amazon SNS used as the notification destination for configured Grafana alerts.

## 2. Why Each AWS Service Exists

| Service | Purpose |
|---|---|
| Amazon EC2 | Hosts the Nginx website and the monitoring components deployed on the instance. |
| Amazon VPC | Provides network isolation and connectivity for the cloud resources. |
| Public subnet | Places the EC2 instance on a network with a route to the Internet Gateway. |
| Internet Gateway | Provides a route between the VPC and the internet, subject to routing and security controls. |
| AWS IAM | Controls the permissions used by the deployment workflow and EC2 instance. |
| AWS Systems Manager | Executes deployment commands remotely without requiring CI/CD over SSH. |
| Amazon CloudWatch | Stores collected metrics and logs and supports monitoring and alerting. |
| CloudWatch Agent | Collects Linux system metrics and configured application logs that are not fully covered by default EC2 metrics. |
| Amazon SNS | Delivers notifications from configured alerting integrations. |
| GitHub Actions | Automates website validation and deployment when changes are pushed to the repository. |
| Grafana | Provides infrastructure dashboards, metric exploration, and configurable alert rules. |

These services have distinct responsibilities: EC2 hosts the workload, Systems Manager handles remote commands, CloudWatch stores telemetry, Grafana visualizes and evaluates monitoring data, and SNS delivers configured notifications.

## 3. Network Design

The project uses an existing default VPC and a manually created public subnet.

The subnet's route table includes a route to the VPC's Internet Gateway. The EC2 instance is configured with a public IPv4 address to support the required connectivity.

No NAT Gateway is used because the current design does not include private-subnet resources requiring outbound internet access.

### Network controls

- **VPC:** Provides the network boundary for the environment.
- **Public subnet:** Hosts the EC2 instance.
- **Route table:** Directs internet-bound traffic through the Internet Gateway.
- **Security group:** Controls permitted inbound and outbound traffic.
- **Public IPv4 address:** Enables public connectivity when routing and security rules allow it.

The security group should permit only the ports and source addresses required by the application and administrative interfaces. Grafana and other management endpoints should not be exposed indiscriminately to the public internet.

## 4. IAM Design

IAM controls access to AWS resources and separates deployment permissions from the permissions required by the EC2 instance.

### GitHub Actions deployment identity

The current deployment workflow uses AWS access keys stored in GitHub repository secrets.

The deployment identity requires permissions to:

- Submit the required Systems Manager Run Command request.
- Target the intended EC2 instance and permitted SSM document.
- Retrieve command invocation results for deployment verification.

Permissions should be restricted to the necessary resources and actions wherever supported.

### EC2 instance role

The EC2 instance uses the `observability-ec2-role` IAM role to support its AWS operations, including the permissions needed by the CloudWatch Agent and Systems Manager.

The role should follow least-privilege principles and grant only the permissions required by the installed agents and configured services.

### Future IAM improvements

Replace long-lived GitHub Actions access keys with GitHub Actions OpenID Connect (OIDC). OIDC allows GitHub Actions to obtain temporary AWS credentials through a specifically trusted IAM role, reducing the need to maintain long-lived access keys.

## 5. Monitoring Design

Amazon CloudWatch is the central telemetry collection and storage service. The CloudWatch Agent extends monitoring to Linux system resources and configured application logs.

Grafana uses CloudWatch as a data source to visualize metrics and evaluate alert rules.

![image alt](https://github.com/imed3721/mon-and-obs/blob/1f85f48f556d41e1bf8d674a9c7bf2ccb15ea8b3/screenshots/Screenshot%202026-10-12%20at%2017.22.34.png)


### Monitored signals

| Signal | Source | Purpose |
|---|---|---|
| CPU utilization | Amazon EC2 / CloudWatch | Identify unusually high processor usage. |
| Memory utilization | CloudWatch Agent / CloudWatch | Monitor host memory consumption. |
| Disk utilization | CloudWatch Agent / CloudWatch | Identify filesystems approaching capacity. |
| Nginx logs | CloudWatch Agent / CloudWatch Logs, when configured | Support investigation of access and application errors. |
| Deployment results | GitHub Actions and Systems Manager | Identify successful deployments and deployment failures. |

Custom host metrics are collected under the `ObservabilityLab` CloudWatch namespace.

Monitoring helps identify performance degradation, capacity issues, and service problems. Metrics and logs provide complementary views: metrics highlight abnormal behaviour, while logs can help explain what happened.

## 6. Alert Strategy

Grafana alert rules evaluate selected CloudWatch metrics and notify the configured contact point when the defined conditions are met.

The platform includes alerting for CPU, memory, and disk utilization.

### Example alert thresholds

| Alert | Example condition | Purpose |
|---|---|---|
| High CPU | CPU utilization above 80% | Identify sustained processor pressure. |
| High memory | Memory utilization above 80% | Identify elevated memory consumption. |
| High disk usage | Filesystem utilization above 80% | Identify potential storage-capacity issues. |

The configured Grafana evaluation group is `high-cpu-alerts`. Rules use an evaluation interval and pending period to help avoid triggering alerts from brief metric spikes.

The configured SNS contact point is `AWS SNS Alerts`.

Alert thresholds are starting values rather than universal indicators of failure. They should be tuned to the workload and validated through controlled testing.

An alert indicates that a defined condition has been met; it does not necessarily mean the service is unavailable. Notification delivery should be tested independently to confirm that alerts reach the intended recipients.

### High CPU 

![image alt](https://github.com/imed3721/mon-and-obs/blob/d9ac95dc06868aeb4556a6d0243fdfa2c6ccdc1c/screenshots/Screenshot%202026-10-12%20at%2017.40.51.png)

![image alt](https://github.com/imed3721/mon-and-obs/blob/d9ac95dc06868aeb4556a6d0243fdfa2c6ccdc1c/screenshots/Screenshot%202026-10-12%20at%2017.40.57.png)

![image alt](https://github.com/imed3721/mon-and-obs/blob/d9ac95dc06868aeb4556a6d0243fdfa2c6ccdc1c/screenshots/Screenshot%202026-10-12%20at%2017.47.51.png)

![image alt](https://github.com/imed3721/mon-and-obs/blob/d9ac95dc06868aeb4556a6d0243fdfa2c6ccdc1c/screenshots/Screenshot%202026-10-12%20at%2017.49.01.png)

![image alt](https://github.com/imed3721/mon-and-obs/blob/d9ac95dc06868aeb4556a6d0243fdfa2c6ccdc1c/screenshots/Screenshot%202026-10-12%20at%2017.49.07.png)

![image alt](https://github.com/imed3721/mon-and-obs/blob/d9ac95dc06868aeb4556a6d0243fdfa2c6ccdc1c/screenshots/Screenshot%202026-10-12%20at%2017.49.13.png)

### High Memory

![image alt](https://github.com/imed3721/mon-and-obs/blob/121cc2b7c8c0176431df2378a74e4dabb6b6aa2b/screenshots/Screenshot%202026-10-12%20at%2017.54.29.png)

![image alt](https://github.com/imed3721/mon-and-obs/blob/121cc2b7c8c0176431df2378a74e4dabb6b6aa2b/screenshots/Screenshot%202026-10-12%20at%2017.54.39.png)

### High Disk Utilization 

![image alt](https://github.com/imed3721/mon-and-obs/blob/0be606e6d7354ea226a6e249854db01fc86ee284/screenshots/Screenshot%202026-10-12%20at%2017.58.39.png)


## 7. Deployment Strategy

GitHub Actions automates website deployments whenever changes are pushed to the `main` branch.

The workflow uses AWS credentials stored in GitHub Secrets and AWS Systems Manager Run Command to deploy the updated `index.html` file to the EC2 instance.

### Deployment workflow

1. **Checkout:** Retrieve the repository contents.
2. **Validate:** Confirm that `index.html` exists and contains the expected platform title.
3. **Encode:** Encode the website content for transfer through the SSM command.
4. **Authenticate:** Configure AWS credentials for the workflow.
5. **Deploy:** Send the website update to the EC2 instance using Systems Manager.
6. **Validate Nginx:** Run `nginx -t` to check the web server configuration.
7. **Reload Nginx:** Apply the deployment by reloading the service.
8. **Verify:** Inspect the SSM command status and report deployment failures.

This approach avoids requiring SSH access from GitHub Actions to deploy the website.

The current workflow verifies the remote command result. A further improvement would be to add an HTTP smoke test to confirm that the updated website is actually being served to clients.

## 8. Failure Testing

Failure testing checks whether the deployment workflow detects errors, stops unsafe deployments, and provides useful diagnostic information.

### Validation failure

Temporarily remove the expected title text from `index.html` and commit the change.

**Expected result:** The validation step fails before a deployment command is sent to EC2.

### Remote deployment failure

In a controlled test, insert a deliberate failing shell command into the remote SSM command sequence.

**Expected result:** The remote command fails, the workflow exits unsuccessfully, and available SSM error details are logged.

### Recovery test

Remove the deliberate failure and restore the valid deployment configuration.

**Expected result:** The subsequent deployment completes successfully and the website is updated.

### IAM permission failure

An incorrectly configured IAM policy may prevent the workflow from submitting the SSM command.

**Expected result:** The workflow reports an AWS authorization error, allowing the missing or restricted permission to be investigated.

Failure tests should be performed deliberately, with each test's expected outcome compared against the actual result. Tests should not be described as successful until the corresponding outcome has been observed.

## 9. Security Considerations

The platform uses several controls to reduce unnecessary access and improve operational security.

- **IAM least privilege:** Restrict deployment and monitoring permissions to the required actions and resources.
- **GitHub Secrets:** Keep AWS credentials out of source code and repository files.
- **Credential management:** Rotate long-lived access keys as appropriate and remove unused credentials.
- **OIDC:** Plan to replace static GitHub Actions access keys with short-lived credentials.
- **Security groups:** Restrict inbound traffic to the required ports and trusted sources.
- **Systems Manager:** Use managed remote commands rather than exposing SSH solely for CI/CD.
- **Grafana access:** Protect dashboards and administrative interfaces with appropriate authentication and network restrictions.
- **Logging:** Avoid writing credentials or other sensitive information to deployment output or application logs.
- **Maintenance:** Keep the operating system, Nginx, Grafana, and monitoring agents updated.

This is a learning project and should not be considered production-hardened without a further security assessment.

## 10. Cost Considerations

The project may incur charges depending on resource configuration, usage, and applicable AWS pricing or account benefits.

Potential cost areas include:

- **EC2:** Instance runtime and attached storage.
- **Public IPv4:** Applicable public IPv4 address charges.
- **CloudWatch:** Custom metrics, log ingestion and retention, dashboards, and alarms where chargeable.
- **SNS:** Notification requests and applicable delivery charges.
- **Data transfer:** Applicable outbound internet and other data-transfer charges.
- **Additional compute:** Any separately hosted Grafana instance or supporting resources.

Cost management practices include stopping unused compute resources, reviewing CloudWatch metric and log usage, removing unnecessary resources, and monitoring AWS Billing and Cost Management.

Actual costs depend on usage, region, resource configuration, and current AWS prices; no fixed monthly cost is assumed.

## 11. Future Improvements

Potential improvements to the platform include:

- **Infrastructure as Code:** Introduce Terraform or AWS CloudFormation for reproducible infrastructure provisioning.
- **OIDC authentication:** Replace long-lived AWS access keys with GitHub Actions OIDC.
- **HTTP smoke tests:** Verify website availability and confirm the expected content after deployment.
- **Automated rollback:** Preserve the previous website version and restore it if deployment verification fails.
- **Enhanced alerting:** Tune alert thresholds, reduce notification noise, and test SNS delivery end to end.
- **Log analysis:** Improve log structure, retention policies, and troubleshooting dashboards.
- **Availability monitoring:** Add endpoint health checks and external monitoring.
- **Security hardening:** Restrict administrative interfaces and review IAM and security-group policies.
- **Cost optimization:** Introduce budgets, billing alerts, and regular resource reviews.

These improvements would make the platform more reproducible, secure, and operationally mature.
