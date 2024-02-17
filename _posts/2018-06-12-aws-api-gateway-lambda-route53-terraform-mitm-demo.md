---
title: MITM Demo using AWS API Gateway and Lambda deployed via Terraform
tags: [aws, api gateway, lambda, route53, terraform, mitm, demo, IaC]
style: border
color: primary
description: Example of using AWS API Gateway and Lambda to perform a MITM deployed with terraform.
---

Imagine wanting to modify an API request or response of a production system without actually touching that production system. One of the easiest ways to do that is to put a man-in-the-middle ([MITM](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)) device between the device making the request and the production API system. There are several ways to implement a MITM, this is a 'hello world'-like implementation utilizing AWS [API Gateway](https://aws.amazon.com/api-gateway/) with [Lambda](https://aws.amazon.com/lambda/) running Python 3.6 and deployed via [Terraform](https://www.terraform.io/).

### How it works:

* A HTTP/HTTPS request is made to an API endpoint which is setup in AWS's API Gateway.
* The API Gateway then passes the domain, request type, headers, parameters, and body of the request to an [integrator](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-integration-types.html), which in this case is Lambda function.
* The Lambda temporary environment (similar to [Docker](https://docs.docker.com/) or [Jail](https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/jails.html)) is spun up with the zip package containing the Lambda code. This is then executed using a specified entry point. That code is passed a JSON/dictionary with the data of the original HTTP/HTTPS request and some other Amazon specific data points.
* The Python script within the Lambda then makes the modified request to the actual production API. Upon the actual API's response, the response is further modified, and then passed back to the original requestor.

The overview above transpires in under 500 &micro;s, impressive to say the least. Furthermore the cost savings over running a dedicated EC2 instance is substantial.

### Cost savings

Ignoring data costs as they will be similar between the two solutions. The smallest EC2 ([t2.nano](https://aws.amazon.com/ec2/pricing/on-demand/)) instance will cost $50.81/year to run. Compare that to the API-Gateway/Lambda solution which will cost $17.90 for the same application. Below is how those cost break out .

**EC2 solution (sans bandwidth costs)**:
24 (hours) &#42; 365 (days) &#42; 0.0058 (Price per hour) = **50.808**

**API Gateway / Lambda solution (sans bandwidth costs)**:

60 (seconds) &#42; 60 (minutes) &#42; 24 (hours) &#42; 365 (days) = 31,536,000 seconds in a year

31536000 / 8 (request every 8 seconds) = 3,942,000 requests the application will make in a year

**Lambda specific cost**:

$0.00000000208 ([Price per &micro;s in a 128MB env](https://aws.amazon.com/lambda/pricing/)) &#42; 500 (max processing time in &micro;s) &#42; 3942000 (requests per year) = 4.09968

**API Gateway specific cost**:

$0.0000035 ([Price per API call to API Gateway](https://aws.amazon.com/api-gateway/pricing/)) &#42; 3942000 = 13.797

**Costs per year**:

$13.797 (API Gateway) &#43; $4.09968 (Lambda) = **17.89668** (plus data)

### Example

For simplicity sake, this will concept will be demonstrated using a simple API endpoint. At [ip-api.com](http://ip-api.com) requests can be made to [various endpoints](http://www.ip-api.com/docs/) which respond with geo-IP information from the requestor's IP in the format of the endpoint name. For example if a request is made to the `/json` endpoint, the response is in JSON. Below is the base line response* when I hit that API endpoint from my local machine.

    justin@local-machine$ curl -o- -q http://ip-api.com/json
    {
        "as": "AS209 Qwest Communications Company, LLC",
        "city": "Salt Lake City",
        "country": "United States",
        "countryCode": "US",
        "isp": "CenturyLink",
        "lat": 40.7079,
        "lon": -111.8555,
        "org": "CenturyLink",
        "query": "65.1.2.3",
        "region": "UT",
        "regionName": "Utah",
        "status": "success",
        "timezone": "America/Denver",
        "zip": "84106"
    }
    justin@local-machine$

[After I deploy via Terraform](https://github.com/EpiJunkie/mitm-demo-aws-api-gw-lambda-terraform), I can now hit the MITM proxy endpoint setup in AWS. You will notice that the 'org' field has been modified by the Lambda environment.

    justin@local-machine$ curl -o- -q http://mitm-demo.justinholcomb.me/json
    {
        "as": "AS16509 Amazon.com, Inc.",
        "city": "Columbus",
        "country": "United States",
        "countryCode": "US",
        "isp": "Amazon.com",
        "lat": 39.9653,
        "lon": -83.0235,
        "org": "Amazon.com ThisHasBeenManipulatedDict",
        "query": "18.221.15.77",
        "region": "OH",
        "regionName": "Ohio",
        "status": "success",
        "timezone": "America/New_York",
        "zip": "43215"
    }
    justin@local-machine$

As you can see above, because the requestor is the Lambda environment, the Lambda's IP is returned. The API at ip-api.com accepts using `/{format}/{ip}` endpoint to return IP information for a different IP address. See below:

    justin@local-machine$ curl -o- -q http://mitm-demo.justinholcomb.me/json/65.1.2.3
    {
        "as": "AS209 Qwest Communications Company, LLC",
        "city": "Salt Lake City",
        "country": "United States",
        "countryCode": "US",
        "isp": "CenturyLink",
        "lat": 40.7079,
        "lon": -111.8555,
        "org": "CenturyLink ThisHasBeenManipulatedDict",
        "query": "65.1.2.3",
        "region": "UT",
        "regionName": "Utah",
        "status": "success",
        "timezone": "America/Denver",
        "zip": "84106"
    }
    justin@local-machine$

Furthermore, [the code has some commented out lines](https://github.com/EpiJunkie/mitm-demo-aws-api-gw-lambda-terraform/blob/master/lambda-function.py#L52) to utilize the 'X-Forwarded-For' request header and this endpoint. However it was omitted in this for clarity.

As demonstrated, this is an easy and cost effective way to implement a man-in-the-middle for a production system to observe and manipulate request/response traffic for legitimate/nefarious reasons. This can be very powerful as anything within the request or response can be modified from the headers, parameters, and or body. This ability can be particularly helpful for QA teams trying to push the limits of an application to the point of breaking. Furthermore, Terraform makes it easy to deploy within minutes and because the infrastructure is built using code ([IaC](https://en.wikipedia.org/wiki/Infrastructure_as_Code)) rather than direct human interaction, changes are easy to document to pin to a particular behavior characteristic and then revert if necessary.
