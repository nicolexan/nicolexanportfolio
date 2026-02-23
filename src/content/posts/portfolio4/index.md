---
title: Cloud Resume Challenge - Name Changes & Redirecting DNS
published: 2026-02-23
description: "Describing the update to my portfolio DNS and how I redirected traffic to the new domain."
image: "./AWSCompleteDiagram.png"
tags: ["Project", "AWS", "Cloud Resume Challenge"]
category: Projects
draft: False
---

## Overview
This ended up being a supplemental piece to my portfolio, I got married (yay!) and I changed my name (also yay!). So I wanted to update my domain as such to have a cohesive online presence. This might be a pretty short article since the fix was quite simple. The main idea was to put CloudFront in front of an S3 bucket which redirects users to my new domain. 

## The Desired Behavior
:::note
When users attempt to reach nicoxmcd.com, they will automatically be redirected to nicolexan.com
:::

## The Fix 
I removed most of the original infrastructure of my original portfolio and kept the route53, certificate, cloudfront, and s3 files and edited them as needed. Only cloudfront and s3 have significant changes, while route53 and certificate have the same functionality as the original.


`route53.tf` is dynamically assigning the nameservers and created the necessary hosted zones for the domain:
```hcl
resource "aws_route53_zone" "main" {
  name = var.domain_name

  tags = {
    Project = var.domain_name
  }
}

resource "aws_route53domains_registered_domain" "main" {
  domain_name = var.domain_name

  dynamic "name_server" {
    for_each = aws_route53_zone.main.name_servers
    content {
      name = name_server.value
    }
  }
}

resource "aws_route53_record" "www_a" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "apex_redirect" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
    evaluate_target_health = false
  }
}
```

`cloudfront.tf` is mainly responsible for the redirect! So when users try to connect to nicoxmcd.com it returns a status code of 301 (Moved Permanently) and then grabs the redirect target from the `variables.tf`
```hcl
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "${replace(var.domain_name, ".", "-")}-oac"
  description                       = "OAC for ${var.domain_name} redirect origin"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_function" "redirect" {
  name    = "${replace(var.domain_name, ".", "-")}-redirect"
  runtime = "cloudfront-js-2.0"
  publish = true

  code = <<-EOT
    function handler(event) {
      return {
        statusCode: 301,
        statusDescription: "Moved Permanently",
        headers: {
          location: { value: "https://${var.redirect_target}" + event.request.uri }
        }
      };
    }
  EOT
}

resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name              = aws_s3_bucket.redirect_origin.bucket_regional_domain_name
    origin_id                = "S3-redirect-origin"
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }

  enabled         = true
  is_ipv6_enabled = true
  comment         = "Redirect ${var.domain_name} to ${var.redirect_target}"

  aliases = [var.domain_name, "www.${var.domain_name}"]

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-redirect-origin"
    viewer_protocol_policy = "allow-all"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.redirect.arn
    }

    min_ttl     = 0
    default_ttl = 0
    max_ttl     = 0
  }

  price_class = "PriceClass_100"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.cert_validation.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Project = var.domain_name
  }
}
```

`s3.tf` is a simple bucket used more or less as a placeholder for the cloudfront function to use to redirect users:
```hcl
resource "aws_s3_bucket" "redirect_origin" {
  bucket = "${replace(var.domain_name, ".", "-")}-redirect-origin"

  tags = {
    Project = var.domain_name
  }
}
```

`certificate.tf`is doing the same as before, where it is assigning a certificate to the domain name and validating it:
```hcl
resource "aws_acm_certificate" "cert" {
  domain_name               = var.domain_name
  subject_alternative_names = ["www.${var.domain_name}"]
  validation_method         = "DNS"

  tags = {
    Project = var.domain_name
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "cert_validation" {
  allow_overwrite = true
  name            = tolist(aws_acm_certificate.cert.domain_validation_options)[0].resource_record_name
  type            = tolist(aws_acm_certificate.cert.domain_validation_options)[0].resource_record_type
  records         = [tolist(aws_acm_certificate.cert.domain_validation_options)[0].resource_record_value]
  zone_id         = aws_route53_zone.main.zone_id
  ttl             = 300
}

resource "aws_route53_record" "cert_validation_www" {
  allow_overwrite = true
  name            = tolist(aws_acm_certificate.cert.domain_validation_options)[1].resource_record_name
  type            = tolist(aws_acm_certificate.cert.domain_validation_options)[1].resource_record_type
  records         = [tolist(aws_acm_certificate.cert.domain_validation_options)[1].resource_record_value]
  zone_id         = aws_route53_zone.main.zone_id
  ttl             = 300
}

resource "aws_acm_certificate_validation" "cert_validation" {
  certificate_arn = aws_acm_certificate.cert.arn
  validation_record_fqdns = [
    aws_route53_record.cert_validation.fqdn,
    aws_route53_record.cert_validation_www.fqdn
  ]
}
```


## Resources
Feel free to check out the source code for both the cloud infrastructure and portfolio source code.

::github{repo="nicolexan/nicolexanportfolio"}

::github{repo="nicolexan/cloud"}

:::note[Reflection]
This was another piece to a puzzle that I was struggling to find a solution. As I've changed my name, I wanted to update my professional handles on LinkedIn, which I did. Yet, my GitHub and portfolio was still using my old handle. Though I also know that I've shared this website with many people, so I didn't just want them to hit a 404 when trying to reach it. So this piece was simple but necessary!
:::