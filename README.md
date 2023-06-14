# MediaTailor session initialization micro-service for prefetch

## Overview
.

## Release Notes
| Date       | Version | Update Notes                                                           |
|------------|-----|------------------------------------------------------------------------|
| 2023-06-14 | 1.0 | Initial release |

## How To Deploy

1. Login to the AWS Console
2. Navigate to the AWS CloudFormation service in the `us-east-1` region (The solution deploys a Lambda@Edge function, [which run in us-east-1 only at this time](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-restrictions.html#lambda-at-edge-restrictions-region))
3. Choose to upload a file; use [this](#) CloudFormation template to deploy the micro-service. As this is a reference architecture, the CloudFormation template also creates a MediaTailor configuration and CloudFront distribution.

There is x input parameters, for the `StreamId` parameter, specify a number between 1 and 10

**Note: the number you put in here should be the same as the number of prefetch schedules you are planning to make per ad break**

## How To Use

MediaTailor configurations provide three prefixes when created (Check the CloudFormation stack output tab for this information):
* Session initialization prefix
* HLS playback prefix
* DASH playback prefix

In order to use the micro-service, a path is inserted into the prefix path at the first position; `session-initializer`. This will trigger a request on a preconfigured CloudFront behavior to invoke a Lambda function that will randomly create a stream id for the session. The Lambda will then issue a redirect back to the client (if the HLS or DASH playback prefixes were used), or create the session on behalf of the client (if the session initialization prefix was used)

### Example 1: Create a session using HLS playback prefix
[  example ]

### Example 2: Create a session using DASH playback prefix
[  example ]

### Example 3: Create a session using session initialization playback prefix
[  example ]

| Important                                                                 |
|---------------------------------------------------------------------------|
| This solution does not create or manage MediaTailor prefetch schedules    |