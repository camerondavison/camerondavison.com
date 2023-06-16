---
layout: post
title:  "Building μtrakr - GCP"
tags: utrakr,terraform,gcp
author: camerondavison
comments: true
---

Continuing the series of writing 
the [utrakr.app](https://utrakr.app) url shortener. Starting with [utrakr intro](https://camerondavison.com/2020/07/10/utrakr-intro-self/)

## Next Steps

After choosing a name, and setting up email we are ready to start getting
something deployed. I decided to use Google Cloud Compute ([GCP](https://cloud.google.com/compute))
because I have been working with them at work and it would be good
to see how they stack up against [AWS](https://aws.amazon.com/)

## Start GCP

I created a utrakr `project` in GCP. Projects in GCP are a good way to separate
security in GCP and are pretty lightweight. Google Cloud Storage ([GCS](https://cloud.google.com/storage))
is very similar to [S3](https://aws.amazon.com/s3/) which is very useful
technology for storing data cheap as well as making it available all over the world.
I like to use S3 or GCS to store terraform state when I define any of my infrastructure.

## What is Terraform?

[Terraform](https://www.terraform.io/) is way to write infrastructure as code.
Writing your infrastruture changes into code gives you an opportunity to see
what changes are going to happen before they happen and also makes it much easier to
reproduce those changes in multiple environments. I have always thought about
engineering solutions to problems the same as [Newton's Method](https://en.wikipedia.org/wiki/Newton's_method)
by slowing making changes to a code base you are fixing bugs and optimizing
as your solution continues to get better and better over time. This only works
if you have a good code base to start on and make sure that you know the small changes
that you are making. [Git](https://git-scm.com/) or any version control system (VCS) is
going to help with this. In the same way that using a VCS is critical to writing
software systems, it is also critical to managing software systems. 

## VCS for infrastructure is good but why Terraform?

Terraform is unique to other platforms such as [saltstack](https://www.saltstack.com/) and
[ansible](https://www.ansible.com/) because of it use of a state file. The state file is
how Terraform knows what it can delete, which is super important part of keeping your
infrastructure clean. Here is a simple example of when the state file can help. 
1. create a gcs bucket
1. save the fact that the bucket was created to state
1. delete the code that created the bucket

After the last step, if we ran a process to reconcile our current code with our current
infrastructure we would not know that we needed to delete the bucket that we created before.
There are some solutions that have adopted the use of a state file such as 
[AWS Cloudformation](https://aws.amazon.com/cloudformation/) and 
[Google Deployment Manager](https://cloud.google.com/deployment-manager/), but terraform works
with both of these providers plus more. You can use terraform to manage github, pingdom, datadog,
etc... pretty much anything with an API. Terraform calls these [providers](https://www.terraform.io/docs/providers/index.html)
and there are tons of them.
 
## This is supposed to be about utrakr lets get back on track

Yes, I digressed some about Terraform. Hopefully I have conviced you of the merits.
We have nothing! How do we even get started. I talked about a state file, and so we
need a place to store the state file. Terraform allows us to store it in GCS but
we do not even have a bucket. We could go to the UI and create one, but instead lets
just use terraform to make it.

```hcl
provider "google" {
  project = "utrakr"
  region  = "us-central1"
  version = "~> 3.20"
}

terraform {
  required_version = "~> 0.12"
}

resource "google_storage_bucket" "terrform-state" {
  name     = "utrakr-all-terraform-state"
  location = "us-central1"
  versioning {
    enabled = true
  }
}
```

I can break this down some. First we have a google provider where we specify the 
project that we created, and the region that we care about. The version is a way
to specify which version we want of the google provider. The `~>` syntax [constrains](https://www.terraform.io/docs/configuration/version-constraints.html)
the version from `>= 3.20` and `< 4.0`. We are also specifying that we want
at least version `0.12` of terraform because they had a [huge syntax change](https://www.hashicorp.com/blog/terraform-0-1-2-preview/)
in `0.12`. Lastly the `google_storage_bucket` resource is named `terrform-state` and
we named the bucket `utrakr-all-terraform-state` and 
[turned on versioning](https://www.terraform.io/docs/backends/types/gcs.html) which is
recommended by the docs. 

My naming convention is from least specific to most specific.
Which is why it is `project`-`env`-`app` . We are going to store all of our terraform state
in this one bucket so instead of calling it `qa` or `prod` I called it `all`.

## Where is the state?

The state for this code will actually be checked in to git. It looks like
```json
{
  "version": 4,
  "terraform_version": "0.12.24",
  "serial": 6,
  "lineage": "454e3473-3520-063a-71b2-1428c6eba07c",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "google_storage_bucket",
      "name": "terrform-state",
      "provider": "provider.google",
      "instances": [
        {
          "schema_version": 0
      ...
```
a big json blob. It can be cumbersome to always have 2 commits for every change. 1 for
the code change and one for the state change. This can also cause problems because
our state corresponds to a global shared resource in GCP so git branches could get our
state into some strange conflicts. Keeping our state in GCS helps make sure that
we are always mutating the same data file even when we are in a branch. I put this
in a `bootstrap` folder. The `bootsrap` terraform project saves its state in git, but 
all the future terraform projects will use the bucket that the bootsrap folder created to
save their state.

Before we can run this code we need to authorize with our google project.
```
gcloud config set project utrakr # set our project
gcloud auth application-default login # get login creds that terraform can use
```

Then we can run
```bash
terraform init # get our providers (only have to run once)
terraform apply # will verify you want to make these changes
```

[Source Code](https://github.com/utrakr/gcp/tree/master/bootstrap)

## How should we organize the Terraform code?

There as many ways to layout your terraform code. My favorite approach is to treat it
like I would other services and have many terraform projects with individual
state files for each. This means there is not one huge state file for everything in the 
entire company. It is good to have multiple state files because some state operations
that refresh and validate the state is still up to date are faster because the state is smaller. 
One drawback is that it makes for a looser coupling between changes. Making a change
to the state in one project may need to trigger an update in another project and there is
not an automatic way to do this with the terraform cli right now. The looser coupling can also
be positive because it can reduce the blast radius of a change
that brings down systems by accident.

## Now that we are bootsrapped what is next?

We talked about using ImprovMX last time. How can we use terraform to setup our MX records
correctly. First create a new terraform project named `crit-dns` and then create a main.tf
file.
```hcl
provider "google" {
  project = "utrakr"
  region  = "us-central1"
  version = "~> 3.20"
}

terraform {
  required_version = "~> 0.12"
}

terraform {
  required_version = "~> 0.12"
  backend "gcs" {
    bucket = "utrakr-all-terraform-state"
    prefix = "crit-dns"
  }
}
```

This time we are saving our state into
```hcl
backend "gcs" {
  bucket = "utrakr-all-terraform-state"
  prefix = "crit-dns"
}
```
The bucket that we just created.

## Setup DNS zone in Google

We want to manage the DNS through google Cloud DNS, so we setup the managed zone.
```hcl
resource "google_dns_managed_zone" "root" {
  name     = "utrakr-app"
  dns_name = "utrakr.app."

  dnssec_config {
    kind          = "dns#managedZoneDnsSecConfig"
    non_existence = "nsec3"
    state         = "on"

    default_key_specs {
      algorithm  = "rsasha256"
      key_length = 2048
      key_type   = "keySigning"
      kind       = "dns#dnsKeySpec"
    }
    default_key_specs {
      algorithm  = "rsasha256"
      key_length = 1024
      key_type   = "zoneSigning"
      kind       = "dns#dnsKeySpec"
    }
  }
}
```
and changed the Name Servers (NS) records in google domains to
```
ns-cloud-c1.googledomains.com
ns-cloud-c2.googledomains.com
ns-cloud-c3.googledomains.com
ns-cloud-c4.googledomains.com
```
The registrar usually host the NS records, which point to all the rest of the DNS
records. A registrar is just who you give money to for your DNS, any registrar will
do they just need to be able to reliably host the NS records that tell all the
other DNS server where to find the rest of the records. I am using `domains.google` as
my registrar and [Google Cloud DNS](https://cloud.google.com/dns) for the rest of the
DNS records.

## Setup MX DNS records in Google

MX Records are DNS records that tell email servers where to send emails. ImprovMX told us
that we need to set our MX records to point to thier server so that they can forward
the email to us. We can set up these MX records in terraform with
```hcl
resource "google_dns_record_set" "mx" {
  name         = google_dns_managed_zone.root.dns_name
  managed_zone = google_dns_managed_zone.root.name
  type         = "MX"
  ttl          = 300

  rrdatas = [
    "10 mx1.improvmx.com.",
    "20 mx2.improvmx.com.",
  ]
}
```
and then set up an Sender Policy Framework (SPF) TXT record.  
```hcl
resource "google_dns_record_set" "spf" {
  name         = google_dns_managed_zone.root.dns_name
  managed_zone = google_dns_managed_zone.root.name
  type         = "TXT"
  ttl          = 300

  rrdatas = ["\"v=spf1 include:spf.improvmx.com ~all\""]
}
```
SPF helps prevent messages sent by you from going to spam. It specifies the
domain that is supposed to be sending the messages so that others cannot
spoof your same domain when they send email messages.

[Source Code](https://github.com/utrakr/gcp/tree/master/crit-dns)

## Next Steps

We now have terraform setup and we used it to create a valid email address. 
We can now begin working on the backend minimum viable product (MVP).
I will cover that in the next post.
