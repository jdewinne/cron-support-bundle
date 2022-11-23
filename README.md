# cron-support-bundle

## Description

As an Application Vendor you sometimes want to capture additional metrics from your customers that are running your application in their environment. This could be as simple as usage metrics. Or maybe you want to know which version they are using. Or if the application is properly running in their environment. Or maybe you have some "legal" auditing requirement where on a quarterly basis your customer needs to provide the number of users, nodes, ....

You could of course make use of Prometheus metrics in combination with AlertManager, like we already did in the [End-to-End Prometheus Alerting Example with Replicated](https://www.replicated.com/blog/end-to-end-prometheus-alerting-example-with-replicated/). Or you could also make use of a monitoring solution like Datadog and package that as part of your application (if of course allowed by your end customer). But if you're already using Replicated to install your application, you could also leverage the embedded [troubleshoot.sh](https://troubleshoot.sh) framework to have the customer send you the required information automatically using a "support bundle".

## The Support Bundle Specification
This is example makes use of the `support-bundle` cli and the fact that you can pass a Kubernetes Secret to it that contains the Support Bundle Specification.

```
kubectl support-bundle secret/NAMESPACE/YOUR-SECRET
```
The secret needs a field `support-bundle-spec` that contains the collectors and analyzers. An example can be found in [support-bundle-secret.yaml](./manifests/support-bundle-secret.yaml).

## Generating the support bundle on a regular interval
If you want to run a workload in Kubernetes on a regular interval, you can make use of a CronJob. That is also what we'll use for our example.

The [support-bundle-cron-job.yaml](./manifests/support-bundle-cron-job.yaml) contains a working example.

### The cron pattern:
We'll use a Replicated `LicenseFieldValue` so that the Vendor can control the interval. This can of course be change into a `ConfigValue` if you want the user to be able to control it.

```
schedule: '{{repl LicenseFieldValue "cron_support_bundle_schedule" }}'
```

In our case, this is set to `*/5 * * * *` (Every 5 minutes).


### Let the user opt-in
In the example, we also allow the end customer to "Opt-In" to enable collecting and sending the data. This is configured using a `ConfigOption` called `usage_info_enabled`.

So if the end customer opts out, because of the annotation below, the CronJob will not be deployed:

```
annotations:
    "kots.io/exclude": '{{repl ConfigOptionEquals "usage_info_enabled" "0" }}'
```

### Uploading the support bundle
The container `send-support-bundle` in the CronJob is responsible for uploading the `.tar.gz` to an external endpoint. In this case we've chosen for an S3 bucket that is publicly available and allows uploading `.tar.gz` files. However only Upload is allowed, so nobody can list or read the files from the S3 bucket. As a vendor you can pickup the collected tarballs from the s3 bucket, or provide your own "upload" mechanism.

The S3 policy used (Replace `YOUR-BUCKET-NAME` and `SOME-RANDOM-STRING` with your values)
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "allow-anon-put",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::[YOUR-BUCKET-NAME]/*"
        },
        {
            "Sid": "deny-other-actions",
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "NotAction": "s3:PutObject",
            "Resource": "arn:aws:s3:::[YOUR-BUCKET-NAME]/*",
            "Condition": {
                "StringNotEquals": {
                    "aws:UserAgent": "[SOME-RANDOM-STRING]"
                }
            }
        }
    ]
}
```

The endpoint of the S3 bucket is configured using a custom License Field:
```
    env:
      - name: ENDPOINT
        value: '{{repl LicenseFieldValue "cron_support_bundle_url" }}'
```

In our case, this is set to `https://s3-us-west-1.amazonaws.com/[YOUR-BUCKET-NAME]`