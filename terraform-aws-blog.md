# Provisioning AWS Infrastructure for a Kubernetes Cluster Using Terraform

In this post I'll walk through the exact Terraform configuration I used to provision the AWS infrastructure for a Kubernetes cluster — VPC, EC2 instances, S3, and CloudFront — all in `ap-south-1` (Mumbai). No modules, no variables, just straightforward HCL you can read and understand in one sitting.

---

## Project Structure

```
.
├── main.tf         # Provider + Terraform config
├── ec2.tf          # Key pair + EC2 instances
├── vpc.tf          # VPC, subnets, IGW, route tables, security group
├── s3.tf           # S3 bucket
└── cloudfront.tf   # CloudFront distribution
```

---

## Step 1 — Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

Pinning to `~> 5.0` allows minor version upgrades but prevents breaking changes from a major version bump. Region is Mumbai (`ap-south-1`).

---

## Step 2 — VPC and Networking

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "assesment-vpc" }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "assesment-igw" }
}

# Public Subnets
resource "aws_subnet" "public1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet1" }
}

resource "aws_subnet" "public2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "ap-south-1b"
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet2" }
}

# Route Table for Public Subnet
resource "aws_route_table" "public1" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt1" }
}

resource "aws_route_table_association" "public1" {
  subnet_id      = aws_subnet.public1.id
  route_table_id = aws_route_table.public1.id
}

resource "aws_route_table" "public2" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt2" }
}

resource "aws_route_table_association" "public2" {
  subnet_id      = aws_subnet.public2.id
  route_table_id = aws_route_table.public2.id
}

# Security group allowing SSH and HTTP
resource "aws_security_group" "web_sg" {
  name        = "assesment-sg"
  description = "Allow SSH and HTTP inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "all traffic from anywhere"
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "assesment-sg"
  }
}
```

### What's happening here

- **VPC** — `10.0.0.0/16` gives 65,536 IPs. DNS support and hostnames are enabled, which Kubernetes needs for internal service resolution.
- **Two public subnets** — spread across `ap-south-1a` and `ap-south-1b` for availability zone coverage. `map_public_ip_on_launch = true` means every EC2 instance in these subnets gets a public IP automatically.
- **Internet Gateway + Route Tables** — each subnet gets its own route table with a default route (`0.0.0.0/0`) pointing to the IGW. This is what makes the subnets "public".
- **Security Group** — opens all TCP ports inbound from anywhere. Suitable for a lab/assessment setup where you need Kubernetes API, NodePort services, and SSH all accessible without fighting port rules.

---

## Step 3 — EC2 Instances

```hcl
resource "aws_key_pair" "assesment-key" {
  key_name   = "assesment-key"
  public_key = file("~/.ssh/id_ed25519.pub")
}

resource "aws_instance" "master" {
  ami                    = "ami-006f82a1d5a27da54"
  instance_type          = "c7i-flex.large"
  subnet_id              = aws_subnet.public1.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = aws_key_pair.assesment-key.key_name

  root_block_device {
    volume_size = 25
    volume_type = "gp3"
  }

  tags = {
    Name = "assesment-master"
  }
}

resource "aws_instance" "worker-1" {
  ami                    = "ami-006f82a1d5a27da54"
  instance_type          = "c7i-flex.large"
  subnet_id              = aws_subnet.public1.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = aws_key_pair.assesment-key.key_name

  root_block_device {
    volume_size = 10
    volume_type = "gp3"
  }

  tags = {
    Name = "assesment-worker-1"
  }
}

resource "aws_instance" "worker-2" {
  ami                    = "ami-006f82a1d5a27da54"
  instance_type          = "c7i-flex.large"
  subnet_id              = aws_subnet.public1.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = aws_key_pair.assesment-key.key_name

  root_block_device {
    volume_size = 10
    volume_type = "gp3"
  }

  tags = {
    Name = "assesment-worker-2"
  }
}
```

### What's happening here

- **Key pair** — reads your local Ed25519 public key via `file("~/.ssh/id_ed25519.pub")` and uploads it to AWS. This is the same key used in the Ansible inventory for SSH access.
- **AMI** — `ami-006f82a1d5a27da54` is an Ubuntu 22.04 LTS image in `ap-south-1`.
- **Instance type** — `c7i-flex.large` is a compute-optimized flexible instance — good balance of CPU and cost for Kubernetes nodes.
- **Disk** — master gets 25 GB (`gp3`) for etcd, control-plane components, and container images. Workers get 10 GB each, enough for workloads in a lab setting.
- All three instances land in `public1` (`ap-south-1a`) behind the same security group.

---

## Step 4 — S3 Bucket

```hcl
# Complete production-ready S3 bucket
resource "aws_s3_bucket" "production" {
  bucket = "assesment-s3-210795"
  tags = {
    Name        = "Production Data"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket_versioning" "production" {
  bucket = aws_s3_bucket.production.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "production" {
  bucket                  = aws_s3_bucket.production.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_website_configuration" "production" {
  bucket = aws_s3_bucket.production.id
  index_document {
    suffix = "index.html"
  }
  error_document {
    key = "error.html"
  }
}
```

### What's happening here

- **Versioning** — keeps previous versions of every object. Useful for rollbacks when deploying frontend builds.
- **Public access block** — all four block settings are enabled. The bucket itself is private; CloudFront will be the only way content is served publicly.
- **Website configuration** — sets `index.html` as the root document and `error.html` for 404s. This enables S3's static website hosting feature, which CloudFront will sit in front of.

> Note: Even though public access is blocked, `aws_s3_bucket_website_configuration` is still configured because CloudFront can use the regional domain name (`bucket_regional_domain_name`) to access the bucket directly without needing the bucket to be public.

---

## Step 5 — CloudFront Distribution

```hcl
# CloudFront distribution serving from S3
resource "aws_cloudfront_distribution" "production" {
  origin {
    domain_name = aws_s3_bucket.production.bucket_regional_domain_name
    origin_id   = "s3-origin"
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "s3-origin"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {  #taken from aws documentation
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  restrictions {  #taken from oneuptime
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

### What's happening here

- **Origin** — points to the S3 bucket's regional domain name. Using `bucket_regional_domain_name` instead of `bucket_domain_name` avoids redirect issues for buckets outside `us-east-1`.
- **`viewer_protocol_policy = "redirect-to-https"`** — any HTTP request gets automatically redirected to HTTPS. Users always get an encrypted connection.
- **`forwarded_values`** — query strings and cookies are not forwarded to S3 (since S3 static hosting doesn't use them). This improves cache hit rate.
- **`geo_restriction: none`** — content is served globally with no country blocks.
- **`cloudfront_default_certificate`** — uses CloudFront's own SSL certificate (`.cloudfront.net` domain). No ACM cert needed for a lab setup.

---

## Putting It All Together

The full dependency chain looks like this:

```
aws_vpc.main
  └── aws_internet_gateway.igw
  └── aws_subnet.public1 / public2
        └── aws_route_table.public1 / public2
        └── aws_security_group.web_sg
              └── aws_instance.master / worker-1 / worker-2

aws_s3_bucket.production
  └── aws_s3_bucket_versioning.production
  └── aws_s3_bucket_public_access_block.production
  └── aws_s3_bucket_website_configuration.production
        └── aws_cloudfront_distribution.production
```

Terraform resolves all these dependencies automatically — you just run:

```bash
terraform init
terraform plan
terraform apply
```

---

## Summary

This Terraform config provisions:

| Resource | Detail |
|---|---|
| VPC | `10.0.0.0/16`, DNS enabled |
| Subnets | 2 public subnets across `ap-south-1a` and `ap-south-1b` |
| IGW + Route Tables | Full internet access for public subnets |
| Security Group | All TCP inbound, all traffic outbound |
| EC2 Instances | 1 master (25 GB), 2 workers (10 GB), `c7i-flex.large` |
| S3 Bucket | Versioned, private, static website enabled |
| CloudFront | HTTPS-only, global CDN in front of S3 |

The EC2 instances are ready to be handed off to Ansible for Kubernetes cluster setup — the public IPs go into the Ansible inventory, and the same Ed25519 key used in Terraform is used for SSH access.
