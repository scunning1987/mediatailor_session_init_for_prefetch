# MediaTailor session initialization micro-service for prefetch

## Overview
MediaTailor is a server side ad insertion (SSAI) service that can scale to support millions of concurrent streaming sessions. As SSAI scales, the ad tech stack (ad buying & selling, and ad decisioning) must also scale, but traffic for ad requests are very spiky and maintaining scale for Ad Decisioning systems can be costly. For many reasons, MediaTailor implemented a feature called live prefetch, which is designed to make calls to the ad server before an ad break is seen in the manifest.

The prefetch feature is configured by 



![](images/prefetch1.png?width=50pc&classes=border,shadow)


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
This example uses cURL to show the call flow:
```shell
REQUEST:
% curl -v https://123.cloudfront.net/session-initializer/v1/master/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/out/v1/4d6b21805fb24291abf3534adfea8966/index.m3u8"

RESPONSE:
< HTTP/2 302 
< content-length: 0
< location: https://123.cloudfront.net/v1/master/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/out/v1/4d6b21805fb24291abf3534adfea8966/index.m3u8?aws.streamId=3
< server: CloudFront
< date: Wed, 14 Jun 2023 22:34:07 GMT
< x-cache: Miss from cloudfront
< via: 1.1 123.cloudfront.net (CloudFront)
< x-amz-cf-pop: PHX50-P1
< x-amz-cf-id: -H2aaJCxrDpnVJtxQI0J3bSNgMpVSF6nkVponFOVHcYTrY35b-REhA==
< vary: Origin
```


### Example 2: Create a session using DASH playback prefix
This example uses cURL to show the call flow:
```shell
REQUEST:
% curl -vvv "https://123.cloudfront.net/session-initializer/v1/dash/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/c09ec45a7b484eea88aefc30376ff6e2/index.mpd"

RESPONSE:
< HTTP/2 302 
< content-length: 0
< location: https://123.cloudfront.net/v1/dash/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/c09ec45a7b484eea88aefc30376ff6e2/index.mpd?aws.streamId=4
< server: CloudFront
< date: Wed, 14 Jun 2023 22:30:12 GMT
< x-cache: Miss from cloudfront
< via: 1.1 123.cloudfront.net (CloudFront)
< x-amz-cf-pop: PHX50-P1
< x-amz-cf-id: 6ZgAeVo15lwctsJKKmySi1keUrjr8-fHJ8-bugXFxElEK2VSOZCEDg==
< vary: Origin
```


### Example 3: Create a session using session initialization playback prefix


```shell
REQUEST:
% curl -v https://123.cloudfront.net/session-initializer/v1/session/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/out/v1/4d6b21805fb24291abf3534adfea8966/index.m3u8 --data '{"adsParams":{"device":"smartphone","subscription":"tier2"}}' | jq .

RESPONSE:
{
  "manifestUrl": "/v1/master/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/out/v1/4d6b21805fb24291abf3534adfea8966/index.m3u8?aws.sessionId=171e4158-6ad1-492b-b543-b4f06138cf2d",
  "trackingUrl": "/v1/tracking/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/171e4158-6ad1-492b-b543-b4f06138cf2d"
}

```

| Important                                                                 |
|---------------------------------------------------------------------------|
| This solution does not create or manage MediaTailor prefetch schedules    |