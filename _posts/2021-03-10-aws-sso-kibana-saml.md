---
toc: true
layout: post
description: A guide to setting up AWS Elasticsearch Kibana to authenticate vis AWS SSO and SAML 2.0.
categories: [markdown, aws, security, sso, elasticsearch, kibana, saml]
title: Using Elasticsearch Kibana with AWS SSO
image: images/Secure-Access-to-Kibana-using-AWS-Single-Sign-On-Figure2.png
---
# Securing Elasticsearch Kibana with AWS SSO

> "Trying to debug complex production issues without observability is like trying to cross a muddy field, blindfolded, at night." - _anon_

I've been an avid user of observability tools since I discovered Splunk over a decade ago. The biggest downside to Splunk is the cost and I've been lucky enough to work in places where the benefits of a tool like Splunk outweight the massive licensing fees.

Recently while exploring native AWS services, I found Cloudwatch Logs to be a bit lacking for my tastes so I'm hoping Elasticsearch can give me the observability drug I need without venturing too far outside the walls of AWS. 

[ELK stack](https://www.elastic.co/what-is/elk-stack) is a very popular log aggregation & querying stack powered by open source software and while I've used it, and derivates such as Grafana, I've been consistently impressed at how much bang it provides for virtually no bucks at all. If you want to follow along, [Open Distro Elasticsearch](https://opendistro.github.io/for-elasticsearch/) can be launched [within the AWS free tier](https://aws.amazon.com/elasticsearch-service/pricing/) as long as you use a single node and choose t2.small.elasticsearch or t3.small.elasticsearch.

I don't want to manage separate passwords though so it needs to be fully integrated with <abbr title="Amazon Web Services - Single Sign-On">AWS SSO</abbr> Auth to be viable.

## Surely you could have just Googled this?

Maybe, but my google fu has failed me so I'm documenting the steps and mappings for others.

The [official AWS Blog](https://aws.amazon.com/blogs/security/how-to-enable-secure-access-to-kibana-using-aws-single-sign-on/) suggests that you need to piggyback over AWS Cognito in order to achieve this. I've copied the main diagram below to save you a click:

![]({{ site.baseurl }}/images/Secure-Access-to-Kibana-using-AWS-Single-Sign-On-Figure1.png "Diagram showing how Cognito can be used as a SAML bridge for Kibana and AWS SSO. Lifted from the AWS blog linked above.")


That blog post was written before the native SAML authentication was added to Kibana on the Amazon Elasticsearch Service (Oct 2020). So now that we have [native SAML authentication for Kibana](https://aws.amazon.com/about-aws/whats-new/2020/10/amazon-elasticsearch-service-adds-native-saml-authentication-kibana/) we can integrate directly with AWS SSO and cut out the middle man. I also don't need <abbr title="Active Directory">AD</abbr> integration so I'll skip that although <abbr title="Your Milage May Vary">YMMV</abbr>.

![]({{ site.baseurl }}/images/Secure-Access-to-Kibana-using-AWS-Single-Sign-On-Figure2.png "Modified diagram showing how Kibana and AWS SSO can integrate directly using SAML 2.0.")

I'll update this page as I learn more. Here's the current status:
 - [X] Make it work
 - [ ] Make it good
 - [ ] Make it fast

One limitation of this approach is that AWS SSO does not appear to support passing Groups to Custom SAML 2.0 applications at this time so while AWS SSO Groups can be used to grant/deny access to an application overall, further refinement of permissions must be handled inside Kibana based on username.

## Spare me the story, what's the short version?

It's not one of the [AWS SSO supported apps](https://docs.aws.amazon.com/singlesignon/latest/userguide/saasapps.html#saasapps-supported) so we have to go the long way round with a [custom SAML app](https://docs.aws.amazon.com/singlesignon/latest/userguide/samlapps.html).

Here's the [AWS ES SAML guide for Kibana](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/saml.html).

All the fields are helpfully called different things in different places so you need to map these fields between AWS SSO, Elasticsearch & Kibana:

| AWS SSO App Configuration | Elasticsearch | typical format |
|-|-|-|
| Application properties > Application start URL | Overview > Kibana / Custom Kibana | https://your.domain.here/_plugin/kibana/ |
| Application metadata > Application ACS URL | Modify Auth > SP-initiated SSO URL | https://your.domain.here/_plugin/kibana/_opendistro/_security/saml/acs | 
| Application metadata > Application SAML audience | Modify Auth > Service provider entity ID  | https://your.domain.here |

| Elasticsearch | AWS SSO | typical format |
|-|-|-|
| Auth > Import IdP metadata | Application Configuration > SAML metadata file | An XML manifest file |
| Auth > SAML master backend role (optional) | IAM Role ARN | arn:aws:iam::123456123456:role/aws-reserved/sso.amazonaws.com/us-east-2/AWSReservedSSO_AdministratorAccess |

You also need to [Map attributes](https://docs.aws.amazon.com/singlesignon/latest/userguide/mapawsssoattributestoapp.html) using the [supported mappings](https://docs.aws.amazon.com/singlesignon/latest/userguide/attributemappingsconcept.html)

| User attribute in the app | Maps to this string value or user attribute in AWS SSO | Format |
|-|-|-|
| Subject | ${user:subject} | unspecified |
| roles | sso_user | basic |
| name | ${user:preferredUsername} | basic |
| email | ${user:email} | basic |

Note: _roles_ is a (comma/semi-colon) delimited list of Kibana "backend roles", these are mapped inside Kibana to "roles" which give permissions. As of the time of writing, [there is a bug](https://github.com/opendistro-for-elasticsearch/security/pull/1024) which means only one role can be sent per record. 

Also, I highly recommend you get the [_SAML tracer_ Chrome plugin](https://chrome.google.com/webstore/detail/saml-tracer/mpdajninpobndbfcldcmbpnnbhibjmch). It's super helpful when debugging.

## I need a bit more help, let's go step by step.

OK, no problem..

![]({{ site.baseurl }}/)

### AWS SSO - Create SAML Application
1. Go to AWS SSO and Create a new __Custom SAML 2.0 Application__
   <img src="{{ site.baseurl }}/images/2021-03-11-SSO-AddNewApp.png" style="margin-left: 0; max-height: 150px" />
2. Provide a __Display Name__ and optional __Description__.
   <img src="{{ site.baseurl }}/images/2021-03-11-SSO-ConfigureCustomApp2.png" style="margin-left: 0; max-height: 150px" />
3. Download the **AWS SSO SAML metadata file**

### Elasticsearch - Modify Authentication
4. Go to the Elasticsearch domain in AWS Console.
5. Actions -> Modify authentication
      <img src="{{ site.baseurl }}/images/2021-03-11-ElasticSearch-ModifyAuth.png" style="margin-left: 0; max-height: 250px" />
6. Under __SAML authentication for Kibana__, click __Enable SAML authentication__
7. Under __Configure your identity provider (IdP)__ copy the following values:
   1. Service provider entity ID  ->  AWS SSO / Application metadata / Application SAML audience
   2. IdP-initiated SSO URL
   3. SP-initiated SSO URL -> AWS SSO / Application metadata / Application ACS URL
    <div style="display: flex;">
    <figure class="image" style="flex: 1 1 100px; margin: 1em 10px;">
      <img src="{{ site.baseurl }}/images/2021-03-11-ElasticSearch-SAMLAuth2.png" style="margin-left: 0; max-height: 250px;" />
      <figcaption>Elasticsearch >> Configure your identity provider (IdP)</figcaption>
    </figure>
    <figure class="image" style="flex: 1 1 100px; margin:1em 0;">
      <img src="{{ site.baseurl }}/images/2021-03-11-SSO-ConfigureCustomApp6.png" style="margin-left: 0; max-height: 250px;" />
      <figcaption>AWS SSO >> Application metadata</figcaption>
    </figure>
    </div>
8. Under __Import IdP metadata__
   1. Upload the __AWS SSO SAML metadata file__
   2. __IdP entity ID__ should be populated automatically from the XML.
   3. __SAML master username / backend role__ (optional)
         ![]({{ site.baseurl }}/images/2021-03-11-ElasticSearch-ClusterHealth-Error.png "Without an _IAM Role ARN_ in _SAML Master backend role_, the _Cluster Health_ and _Indices_ tabs won't load properly.")
      2. Use an IAM Role here to resolve access errors in the AWS console. 
         1. Goto [IAM Roles](https://console.aws.amazon.com/iam/home?region=us-east-2#/roles)
         2. Search SSO
         3. Use the __Role ARN__ in __SAML Master backend role__
            - e.g. arn:aws:iam::123456123456:role/aws-reserved/sso.amazonaws.com/us-east-2/AWSReservedSSO_AdministratorAccess
      3. Note that these fields simply update the _all_access_ and _security_manager_ __Role Mappings__ in Kibana. The details won't be reloaded next time you edit this page but can be found in Kibana under Security > Role Mappings. 
9.  Save
10. Copy the __Kibana__ or __Custom Kibana__ endpoint from the Overview page ->  AWS SSO / Application start URL

### AWS SSO - complete configuration
1. Back in AWS SSO, save _Configuration_
2. Change to the _Attribute Mapping_ tab:
   1. Populate with the following values.
        
        | User attribute in the app | Maps to this string value or user attribute in AWS SSO | Format |
        |-|-|-|
        | Subject | ${user:subject} | unspecified |
        | roles | sso_user | basic |
        | name | ${user:preferredUsername} | basic |
        | email | ${user:email} | basic |
   2. Save changes.
3. Change to the _Assigned Users_ tab
   1. Add an SSO group to grant access to a set of SSO users.
   2. Save changes.

### Elasticsearch - Modify Access Policy (optional)

### Elasticsearch - VPC (optional)


Done!

## Troubleshooting

I highly recommend you get the [_SAML tracer_ Chrome plugin](https://chrome.google.com/webstore/detail/saml-tracer/mpdajninpobndbfcldcmbpnnbhibjmch). You can easily observe the SAML tokens being passed between AWS SSO & Kibana.