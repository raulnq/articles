---
title: "Hosting Static Websites on AWS CloudFront with Terraform"
datePublished: Sat Feb 17 2024 00:18:34 GMT+0000 (Coordinated Universal Time)
cuid: clspbyztz000309l4g0n8csp6
slug: hosting-static-websites-on-aws-cloudfront-with-terraform
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1708021121985/a9a294e0-1f93-47d6-ba98-423f96b8c73b.png
tags: aws, cloudfront, terraform, static-website

---

In our previous article, [Hosting Static Websites on AWS S3 with Terraform](https://blog.raulnq.com/hosting-static-websites-on-aws-s3-with-terraform), we mentioned that there are several options for hosting static websites, one of which is [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html). When deciding whether to use CloudFront or AWS S3 to host static websites, consider the following scenarios in which CloudFront might be preferable:

* **Content Delivery Network (CDN) Capabilities**: CloudFront is a CDN, whereas AWS S3 is an object storage service. CloudFront caches content at edge locations around the world, which can significantly reduce latency for users accessing our content from different geographic locations. This makes CloudFront ideal for delivering content with low latency and high transfer speeds.
    
* **Customization and Security**: CloudFront provides more customization options and advanced security features compared to AWS S3. We can configure various caching behaviors, set up SSL/TLS encryption, limit access to our content using signed URLs or cookies, and integrate with other AWS services like AWS WAF (Web Application Firewall) for added security.
    
* **Streaming Support**: If our static website includes streaming media content (e.g., videos, audio), CloudFront supports streaming protocols like HLS, DASH, and Smooth Streaming. This enables us to deliver high-quality streaming content to a global audience with low latency.
    

In CloudFront, there are three fundamental concepts: [distributions, origins, and cache behaviors](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-working-with.html). Let's explore each concept.

## Distributions

It represents the configuration and settings for delivering content to end users globally. The distributions are associated with one or more origins and cache behaviors, which define how CloudFront handles requests for specific content and how it caches and forwards those requests to the designated origin.

## Origin

An origin is the source of the content that CloudFront delivers to users. This can be any HTTP server that serves our content, such as:

* Amazon S3 bucket
    
* Custom origin (HTTP or HTTPS)
    

When we create a distribution, we can specify one or more origins from which CloudFront retrieves content. When a user requests content, CloudFront fetches the content from the specified origin and caches it at edge locations for subsequent requests.

## Cache Behaviors

Cache behaviors define how CloudFront handles incoming requests for specific content (based on URL patterns or other request attributes) and how it caches and forwards those requests to the designated origin.

CloudFront evaluates cache behaviors in the order they are defined in the distribution configuration. When a request arrives, CloudFront matches the request path against the path patterns of each cache behavior. Upon finding a match, it applies the caching rules and settings specified for that behavior.

In this post, we will create a [**Terraform**](https://blog.raulnq.com/terraform-resources-variables-outputs-and-locals) script to host a static website on Amazon CloudFront, allowing users to access the site through a custom domain and HTTPS protocol.

## **Pre-requisites**

* An [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [**Terraform CLI**](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    
* A [Route 53 **Hosted Zone**](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html).
    
* An SSL certificate issued in [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html).
    

## The Static Website

In a `site` folder, create an `index.html` file with the following content:

```xml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Under Construction</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f3f3f3;
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .container {
            max-width: 600px;
            padding: 20px;
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
            text-align: center;
            animation: pulse 1.5s infinite alternate;
        }
        @keyframes pulse {
            0% {
                transform: scale(1);
            }
            100% {
                transform: scale(1.05);
            }
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Under Construction</h1>
        <p>We're working hard to bring you something awesome!</p>
        <p>In the meantime, please excuse our appearance as we're in the process of building something amazing. Stay tuned for updates.</p>
        <p>Thank you for your patience!</p>
    </div>
</body>
</html>
```

## The Terraform Script

Create a [`main.tf`](http://main.tf) file with the following content:

```json
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.31.0"
    }
  }
  backend "local" {}
}

provider "aws" {
  region      = "<MY_REGION>"
  profile     = "<MY_AWS_PROFILE>"
  max_retries = 2
}

provider "aws" {
  alias = "acm_provider"
  region = "us-east-1"
  profile     = "<MY_AWS_PROFILE>"
}
```

The second AWS provider is to query the SSL certificate. These need to be created in `us-east-1` to have [HTTPS between viewers and CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html#https-requirements-aws-region).

```json
locals {
  bucket_name             = "<MY_BUCKET_NAME>"
  hosted_zone_name        = "<MY_ROUTE53_HOSTED_ZONE_NAME>"
  certificate_domain      = "<MY_CERTIFICATE_DOMAIN>"
  sub_domain              = "<MY_SUB_DOMAIN>"
}
```

The `locals` section can be replaced with `variables` if needed.

```json
data "aws_route53_zone" "zone" {
  name      = local.hosted_zone_name
  private_zone = false
}

data "aws_acm_certificate" "certificate" {
  domain   = local.certificate_domain
  statuses = ["ISSUED"]
  provider = aws.acm_provider
}
```

Make sure to use the right provider when querying the SSL certificate.

```json
resource "aws_s3_bucket" "bucket" {
  bucket        = local.bucket_name
  force_destroy = true
}

resource "aws_s3_bucket_public_access_block" "bucket-access-block" {
  bucket                  = aws_s3_bucket.bucket.id
  ignore_public_acls      = true
  block_public_acls       = true
  restrict_public_buckets = true
  block_public_policy     = true
}
```

We create a bucket and enable the `Block all public access` option for it.

```json
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "OAC for ${local.bucket_name}"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

data "aws_iam_policy_document" "bucket-policy-document" {
  statement {
    actions = ["S3:GetObject"]
    sid    = "AllowCloudFrontServicePrincipalReadOnly"
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }
    resources = [
      "${aws_s3_bucket.bucket.arn}/*",
    ]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"

      values = [
        "${aws_cloudfront_distribution.s3_distribution.arn}"
      ]
    }
  }
}

resource "aws_s3_bucket_policy" "bucket-policy" {
  bucket = aws_s3_bucket.bucket.id
  policy = data.aws_iam_policy_document.bucket-policy-document.json
}
```

We set up a policy that provides read access to the bucket from CloudFront using the [Origin Access Control (OAC)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html).

```json
resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name = aws_s3_bucket.bucket.bucket_regional_domain_name
    origin_id   = "origin-${local.bucket_name}"
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "origin-${local.bucket_name}"
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    viewer_protocol_policy = "allow-all"
  }

  price_class = "PriceClass_200"
  enabled             = true
  is_ipv6_enabled     = false
  default_root_object = "index.html"
  aliases = ["${local.sub_domain}.${data.aws_route53_zone.zone.name}"]
  
  viewer_certificate {
    acm_certificate_arn            = data.aws_acm_certificate.certificate.arn
    cloudfront_default_certificate = false
    minimum_protocol_version       = "TLSv1.2_2021"
    ssl_support_method             = "sni-only"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
```

To create a CloudFront distribution with Terraform, we need to use the `aws_cloudfront_distribution` resource. We can define one or more origins using the `origin` property:

* `domain_name`: DNS domain name of either the S3 bucket or website of our custom origin.
    
* `origin_id`: Unique identifier for the origin.
    
* `origin_access_identity`: CloudFront origin access identity previously created.
    

A `default_cache_behavior` is a set of rules that CloudFront applies when it receives a request for our content that does not match any of the other cache behaviors we specify. To define additional behaviors, use the property `ordered_cache_behavior`:

* `allowed_methods`: Determines which HTTP methods CloudFront processes and forwards to our origin.
    
* `cached_methods`: Determines whether CloudFront caches the responses to requests using the specified HTTP methods.
    
* `target_origin_id`: The origin where we want to direct requests when a request matches the path pattern.
    
* `viewer_protocol_policy`: The protocol that users can use to access the files in the origin. Options include `allow-all`, `https-only`, or `redirect-to-https`.
    
* `forwarded_values`: Indicates how CloudFront manages query strings, cookies, and headers.
    
* `path_pattern`: This pattern determines which requests the cache behavior should apply to (only required for the `ordered_cache_behavior`).
    

The SSL configuration for the distribution is defined in the `viewer_certificate` property:

* `acm_certificate_arn`: The ARN of the SSL certificate we want to use.
    
* `cloudfront_default_certificate`: If we want viewers to request our objects using HTTPS (using the default CloudFront domain name).
    
* `minimum_protocol_version`: Minimum version of the SSL protocol that we want CloudFront to use for HTTPS connections. Can only be set if `cloudfront_default_certificate = false`.
    
* `ssl_support_method`: How we want CloudFront to serve HTTPS requests. One of `vip`, `sni-only`, or `static-ip`.
    

The restriction configuration for the distribution is defined by the `restrictions` property. Within `restrictions`, there is another property called `geo_restriction`:

* `restriction_type`: Method that we want to use to restrict distribution of our content by country: `none`, `whitelist`, or `blacklist`.
    

The other set of useful properties are:

* `enabled`: Whether the distribution is enabled to accept user requests for content.
    
* `is_ipv6_enabled`: Whether the IPv6 is enabled for the distribution.
    
* `default_root_object`: Object that we want CloudFront to return when a user requests the root URL.
    
* `price_class`: Price class for the distribution:
    
    * `PriceClass_All`: Use all edge locations. This is the default option and has the highest cost.
        
    * `PriceClass_200`: Use only edge locations in North America, Europe, South Africa, Hong Kong, Singapore, South Korea, Japan, and India. This option has a lower cost than `PriceClass_All`.
        
    * `PriceClass_100`: Use only edge locations in North America and Europe. This option has the lowest cost.
        
* `aliases`: Additional domain names, if any, for the distribution.
    

```json
resource "aws_route53_record" "record" {
  zone_id = data.aws_route53_zone.zone.zone_id
  name    = "${local.sub_domain}.${data.aws_route53_zone.zone.name}"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.s3_distribution.domain_name
    zone_id                = aws_cloudfront_distribution.s3_distribution.hosted_zone_id
    evaluate_target_health = false
  }
}
```

We create a Route 53 `A` record per each `aliases` item used in the distribution.

```json
resource "aws_s3_object" "html-files" {
  for_each = fileset("./site/", "*.html")
  bucket = aws_s3_bucket.bucket.id
  key = each.value
  content_type    = "text/html"
  source = "./site/${each.value}"
  etag = filemd5("./site/${each.value}")
}
```

We need to upload our website files. The `fileset` function returns a set of file paths that match a specific pattern in a given base directory. AWS S3 assigns a default content type of `binary/octet-stream` to any uploaded files, so be sure to set the correct `content_type` for each file type.

```json
output "route53_name" {
  value = aws_route53_record.record.name
}
```

We create two outputs to display the URLs that can be used to access our static website. Run the following commands to start the deployment:

```powershell
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
```

Browse to the output URLs to view the static website up and running (the custom domain may take a few minutes to propagate):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708120768154/fb5ace86-b6c1-4188-b075-064e8426ec58.png align="center")

You can see the final `main.tf` file [here](https://github.com/raulnq/s3-cloud-front). Thanks, and happy coding.