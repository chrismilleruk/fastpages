---
toc: true
layout: post
description: A guide to setting up AWS ElasticSearch Kibana to authenticate vis AWS SSO and SAML 2.0.
categories: [markdown, aws, security, sso, elasticsearch, kibana, saml]
title: Using ElasticSearch Kibana with AWS SSO
---
# Securing ElasticSearch Kibana with AWS SSO

"Trying to debug complex production issues without observability is like trying to cross a muddy field blindfolded, at night."

I've been an avid user of observability tools since I discovered Splunk over 10 years ago. The biggest downside to Splunk is the cost and I've been lucky enough to work in places where the benefits of a tool like Splunk outweight the massive licensing fees.

ELK is a very popular log aggregation & querying stack powered by open source software and while I've used it, and derivates such as Grafana, I've been consistently impressed at how much bang it provides for virtually no bucks at all.

I've found Cloudwatch Logs to be a bit lacking for my tastes so I'm exploring ElasticSearch & Kibana to see if it can give me the observability drug I need. I don't want to manage separate logins though so it needs to be integrated with AWS SSO Auth to be useful.

## Surely this had been done before?

Maybe, but my google fu has failed me so I'm documenting the steps and mappings for others.

Also, all the fields are helpfully called different things in different places, so it wasn't straightforward enough to solve without some pain.

The [official AWS Blog](https://aws.amazon.com/blogs/security/how-to-enable-secure-access-to-kibana-using-aws-single-sign-on/) suggests that you need to piggyback over AWS Cognito in order to achieve this. I've copied the main diagram below to save you a click:

![]({{ site.baseurl }}/images/Secure-Access-to-Kibana-using-AWS-Single-Sign-On-Figure1.png "Diagram showing how Cognito can be used as a SAML bridge for Kibana and AWS SSO. Lifted from the AWS blog linked above.")


It turns out that recommendation is because that blog post was written before the native SAML authentication was added to Kibana on the Amazon Elasticsearch Service (Oct 2020)

Now that we have [native SAML authentication for Kibana](https://aws.amazon.com/about-aws/whats-new/2020/10/amazon-elasticsearch-service-adds-native-saml-authentication-kibana/) we can integrate directly with AWS SSO and cut out the middle man. I also don't need AD integration so I'll skip that although your milage may vary.

![]({{ site.baseurl }}/images/Secure-Access-to-Kibana-using-AWS-Single-Sign-On-Figure2.png "Modified diagram showing how Kibana and AWS SSO can integrate directly using SAML 2.0.")

I'll update this page as I learn more. Here's the current status:
 - [X] Make it work
 - [ ] Make it good
 - [ ] Make it fast

One limitation of this approach is that AWS SSO does not appear to support passing Groups to Custom SAML 2.0 applications at this time so while AWS SSO Groups can be used to grant/deny access to an application overall, further refinement of permissions within Kibana must be handled inside Kibana based on the NameID/Subject.

## Spare me the narrative, what's the short version?

It's not one of the [AWS SSO supported apps](https://docs.aws.amazon.com/singlesignon/latest/userguide/saasapps.html#saasapps-supported) so we have to go the long way round with a [custom SAML app](https://docs.aws.amazon.com/singlesignon/latest/userguide/samlapps.html).

Here's the [AWS ES SAML guide for Kibana](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/saml.html)

You also need to [Map attributes](https://docs.aws.amazon.com/singlesignon/latest/userguide/mapawsssoattributestoapp.html) using the [supported mappings](https://docs.aws.amazon.com/singlesignon/latest/userguide/attributemappingsconcept.html)

| User attribute in the app | Maps to this string value or user attribute in AWS SSO
 | Format |
|-|-|-|
| Subject | ${user:subject} | unspecified |
| roles | sso_user | unspecified |
| name | ${user:preferredUsername} | basic |
| email | ${user:email} | basic |

_roles_ is a (comma/semi-colon) delimited list of Kibana "backend roles", these are mapped inside Kibana to "roles" which give permissions. As of the time of writing, [there is a bug](https://github.com/opendistro-for-elasticsearch/security/pull/1024) which means only one role can be sent per record.


## Actually, maybe let's go step by step.

OK, no problem..

1. Goto AWS SSO and Create a new Custom SAML Application
2. Provide a Application Name and optional description.
3. Download the **AWS SSO SAML metadata file**

4. Goto the ElasticSearch domain in AWS Console.
5. Actions -> Modify authentication
6. Under _SAML authentication for Kibana_, click _Enable SAML authentication_
7. Under _Configure your identity provider (IdP)_ copy the following values:
   1. Service provider entity ID  ->  AWS SSO / Application metadata / Application SAML audience
   2. IdP-initiated SSO URL
   3. SP-initiated SSO URL -> AWS SSO / Application metadata / Application ACS URL
8. Under _Import IdP metadata_
   1. Upload the _AWS SSO SAML metadata file_
   2. _IdP entity ID_ should be filled out automatically from the XML.
   3. SAML master username / backend role (optional)
      1. This simply updates the _all_access_ role in Kibana
      2. Use an IAM Role here to resolve access errors in the AWS console. Without this, the _Cluster Health_ and _Indices_ tabs won't load properly.
         1. Goto IAM Roles (https://console.aws.amazon.com/iam/home?region=us-east-2#/roles)
         2. Search SSO
         3. Use the _Role ARN_ in _SAML Master backend role_
         4. e.g. arn:aws:iam::123456123456:role/aws-reserved/sso.amazonaws.com/us-east-2/AWSReservedSSO_AdministratorAccess
9. Save
10. Copy the _Kibana_ or _Custom Kibana_ endpoint  ->  AWS SSO / Application start URL

1. Back in AWS SSO
2. Save _Configuration_ and change to the _Attribute Mapping_ tab
   1. Populate with the following values.
   | User attribute in the app | Maps to this string value or user attribute in AWS SSO
    | Format |
   |-|-|-|
   | Subject | ${user:subject} | unspecified |
   | roles | sso_user | unspecified |
   | name | ${user:preferredUsername} | basic |
   | email | ${user:email} | basic |
3. Save changes and go to the _Assigned Users_ tab
   1. Add an SSO group to grant access to a set of SSO users.


Done!

## Troubleshooting
