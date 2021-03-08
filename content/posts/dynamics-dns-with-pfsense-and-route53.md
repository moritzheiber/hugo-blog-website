+++
aliases = ["/post/dynamics-dns-with-pfsense-and-route53"]
categories = ["pfsense", "aws"]
date = "2014-02-28T12:17:36+01:00"
draft = false
tags = ["aws", "dns", "route53", "pfsense"]
title = "Dynamics DNS with pfSense and Route53"

+++

The quite excellent [pfSense](https://pfsense.org) comes with a [dynamic DNS](http://en.wikipedia.org/wiki/Dynamic_DNS) plugin for [Amazon's Route53 DNS management service](http://aws.amazon.com/route53/). However, there is little to no documentation provided on how to set it up properly and especially about setting up the [relevant IAM access policies](http://aws.amazon.com/iam/).

So I went to the [pfSense repository on GitHub](https://github.com/pfsense/) and browsed the code in order to find out how much access the plugin needed in order to do its deed.

As it turns out: not much. The following IAM policy will grant the plugin the required permissions to access Route53 on your behalf:

    {
      "Statement":[
        {
          "Effect":"Allow",
          "Action":[
            "route53:ChangeResourceRecordSets",
            "route53:ListResourceRecordSets"
          ],
         "Resource":"arn:aws:route53:::hostedzone/<your-hosted-zone-id>"
        }
      ]
    }

Notice the `<your-hosted-zone-id>` which you need to exchange for the ID your zone has been assigned to within Route53.

What the dynamics DNS module within pfSense actually does:

1. Connect to the AWS API and look up all the records within the zone you configured
2. Determines whether there is a record by the name you entered
  - If there is it deletes the record and adds it back with the new IP address attached to it.
    - _Note: This is also why it is very wise to chose a very low (<= 60 seconds) TTL for your dynamic DNS record_
  - If there isn't it creates a new record and attaches the IP address to it
3. Saves the IP address it just set within Route53 to a file within the pfSense environment to make sure it doesn't update the same record twice
 - The IP either gets updated when the locally recorded IP address doesn't match the record on Route53 or every 25 days

After that the A record should get updated automatically each time the IP changes on the associated interface.
