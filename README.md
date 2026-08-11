# SecureNet OpsCenter — AWS Cloud Deployment

Documentation of a cloud infrastructure deployment carried out for **LD5012 – Cloud Computing Technologies** (Northumbria University). This repository captures the AWS setup, Nginx reverse proxy configuration, and IAM/security configuration used to deploy a Flask-based operations dashboard ("SecureNet OpsCenter") for a fictional case study organisation, SecureNet Analytics Ltd.

> **Note:** This repo documents the *infrastructure deployment*, not the Flask application itself. The SecureNet OpsCenter app source code was a separate package (`SecureNet_OpsCenter.zip`) deployed onto the server and is not available to include here. What's here is everything needed to understand and reproduce the AWS/Nginx/IAM setup: configuration files, architecture, and the security rationale behind each choice.

## Architecture

- **Compute:** Amazon EC2 instance (`SecureNet-OpsCenter-Server`), Amazon Linux 2023, 64-bit x86, `t2.micro`, 8 GiB EBS volume, `eu-west-2b`
- **Application layer:** Flask app served by the Werkzeug development server on `127.0.0.1:5000`
- **Web layer:** Nginx reverse proxy on port 80, forwarding to the Flask app
- **Network:** Default VPC, auto-assigned public IPv4, dedicated security group (`SecureNet-SG`)
- **IAM:** A dedicated EC2 role (`SecureNet-EC2-Role`) with least-privilege managed policies
- **Monitoring:** VPC Flow Logs + EC2/EBS CloudWatch metrics

```
Internet
   │
   ▼
[Nginx :80]  (reverse proxy, adds X-Forwarded-* headers)
   │
   ▼
[Flask/Werkzeug :5000]  (127.0.0.1 only)
   │
   ▼
SQLAlchemy ORM → application data
```

## Deployment steps

1. **Launch EC2 instance** — Amazon Linux 2023 AMI, `t2.micro`, default VPC, auto-assigned public IPv4.
2. **Configure security group** (`SecureNet-SG`) — see [Security group rules](#security-group-rules) below.
3. **Connect via SSH** — `ssh ec2-user@<public-ip>`, verifying the ED25519 host key fingerprint on first connection.
4. **Install Nginx** — `sudo dnf install nginx` (pulls in `nginx-core`, `libunwind`, `gperftools-libs`).
5. **Transfer and unpack the app** — `scp` the app package to the instance, then `mv` and `unzip` it into place under `/var/www/securenet/SecureNet_OpsCenter`.
6. **Run the Flask app** — activate the virtualenv, then `python3 app.py`, confirming it binds to `0.0.0.0:5000` and `127.0.0.1:5000`.
7. **Configure Nginx as a reverse proxy** — edit `/etc/nginx/conf.d/securenet.conf` (see [`nginx/securenet.conf`](nginx/securenet.conf)) to forward port 80 traffic to the Flask app on port 5000.
8. **Start and enable Nginx** — `sudo systemctl start nginx && sudo systemctl enable nginx`.
9. **Verify** — access the app via the EC2 public IP and public DNS on both port 5000 directly and port 80 through the Nginx proxy.

## Security group rules

| Rule ID | Type | Protocol | Port | Source |
|---|---|---|---|---|
| `sgr-07d87047cb578e90b` | HTTP | TCP | 80 | `0.0.0.0/0` |
| `sgr-0d1e1f3825512fc41` | Custom TCP | TCP | 5000 | `0.0.0.0/0` |
| `sgr-02cdcbd0e3133a129` | SSH | TCP | 22 | `202.8.112.161/32` (single authorised IP only) |
| `sgr-0b2a714c9d68b6e94` | HTTPS | TCP | 443 | `0.0.0.0/0` |

Restricting SSH to a single authorised IP significantly reduces the attack surface for unauthorised remote access, while HTTP/HTTPS remain open for public access to the dashboard.

## IAM

A dedicated role, `SecureNet-EC2-Role`, is attached to the instance following the principle of least privilege:

- **Trust policy:** [`iam/secure-net-ec2-role-trust-policy.json`](iam/secure-net-ec2-role-trust-policy.json) — allows only the EC2 service to assume the role.
- **Attached managed policies:** `AmazonSSMManagedInstanceCore`, `CloudWatchAgentServerPolicy`.

This avoids the SecureNet-Analytics scenario's original weak point of overly broad or absent access control on compute resources.

## Monitoring

- **VPC Flow Logs** (`SecureNet-VPC-FlowLogs`) capture all `ACCEPT` and `REJECT` traffic on the VPC, aggregated every 10 minutes, supporting incident detection and response.
- **CloudWatch** EC2/EBS metrics (read/write latency, throughput) monitor instance health.

## Security posture & known gaps

What this deployment gets right:

- SSH restricted to a single known IP
- Nginx reverse proxy shields the Flask app from direct public exposure
- Least-privilege IAM role scoped only to what the instance needs
- VPC Flow Logs provide an auditable record of all network activity (supports GDPR Article 5(2) accountability)

What's intentionally left as future work (flagged in the original report's critical evaluation):

- **No TLS/HTTPS** — traffic between client and server is unencrypted (plain HTTP on port 80/5000). A production deployment would need a TLS certificate via AWS Certificate Manager.
- **No EBS encryption** — the data volume is not encrypted at rest via AWS KMS, which is a gap against GDPR Article 32 (integrity and confidentiality of processing systems).
- **No RBAC within the app itself** — access control at the infrastructure layer doesn't substitute for role-based access control inside the application.

## GDPR relevance

Since the deployment models a healthcare/finance analytics company (SecureNet Analytics Ltd), GDPR compliance was a specific design consideration:

- **Article 32** (security of processing) — partially met via IAM and network controls; not fully met due to missing encryption at rest and in transit.
- **Article 25** (data protection by design) — supported by VPC flow logs, least-privilege IAM, and restricted security group rules.
- **Article 5(2)** (accountability) — supported by VPC Flow Logs providing an auditable network activity record.

## Live demo (may no longer be active)

- `http://18.132.12.61/login`
- `ec2-18-132-12-61.eu-west-2.compute.amazonaws.com:5000/login`
