# MediaTailor session initialization micro-service for prefetch

## Overview
MediaTailor is a server side ad insertion (SSAI) service that can scale to support millions of concurrent streaming sessions. As SSAI scales, the ad tech stack (ad buying & selling, and ad decisioning) must also scale, but traffic for ad requests are very spiky and maintaining scale for Ad Decisioning systems can be costly. For many reasons, MediaTailor implemented a feature called live prefetch, which is designed to make calls to the ad server before an ad break is seen in the manifest.

Instructions on how to create and manage prefetch schedules can be found in the MediaTailor documentation pages.

The prefetch schedule has an optional setting called `stream id`, the value of which is used to logically group active sessions into different prefetch windows, allowing you to employ your own custom traffic shaping/throttling to the ADS

See the below 2 examples of ad requests with and without prefetch enabled...

### No prefetch

![](images/adsrequests1.png?width=50pc&classes=border,shadow)
With no prefetch, requests may time out if the ADS doesn't have the capacity to serve all requests within an acceptable latenxy timeframe.

### Prefetch
![](images/adsrequests2.png?width=50pc&classes=border,shadow)
With the use of prefetch schedules, MediaTailor is able to keep within the ad serving capacity of the ADS. This uses the concept of `stream id` to group requests together in their own retrieval windows

This github solution is a reference architecture to show you have a session initialization micro-service (in this case, a Lambda@Edge function) can be used to randomly assign a `stream id` for each session.

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
1. The initial call is a GET to the HLS playback prefix, with `/session-initializer/` in the path
2. The Lambda@Edge function is invoked and a random stream id is generated
3. The Lambda@Edge function returns an HTTP 302 redirect to the client, with the `/session-initializer/` path removed, and an aws.streamId query parameter appended
4. The client will send a request to the HTTP 302 location, which will get proxied through CloudFront to MediaTailor, where MediaTailor will create a session with the stream id value


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
1. The initial call is a GET to the DASH playback prefix, with `/session-initializer/` in the path
2. The Lambda@Edge function is invoked and a random stream id is generated
3. The Lambda@Edge function returns an HTTP 302 redirect to the client, with the `/session-initializer/` path removed, and an aws.streamId query parameter appended
4. The client will send a request to the HTTP 302 location, which will get proxied through CloudFront to MediaTailor, where MediaTailor will create a session with the stream id value

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
This example uses cURL to show the call flow:
1. The initial call is a POST to the session initialization playback prefix, with `/session-initializer/` in the path. There is a small body being sent in the call, to represent ads parameters
2. The Lambda@Edge function is invoked and a random stream id is generated
3. The Lambda@Edge function creates the session on MediaTailor using the parameters provided
4. The Lambda@Edge function returns an HTTP 200 response to the client, along with the relative root path structure to the manifest and ad tracking endpoints

```shell
REQUEST:
% curl -X POST https://123.cloudfront.net/session-initializer/v1/session/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/out/v1/4d6b21805fb24291abf3534adfea8966/index.m3u8 --data '{"adsParams":{"device":"smartphone","subscription":"tier2"}}' | jq .

RESPONSE:
{
  "manifestUrl": "/v1/master/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/out/v1/4d6b21805fb24291abf3534adfea8966/index.m3u8?aws.sessionId=171e4158-6ad1-492b-b543-b4f06138cf2d",
  "trackingUrl": "/v1/tracking/94063eadf7d8c56e9e2edd84fdf897826a70d0df/prefetch/171e4158-6ad1-492b-b543-b4f06138cf2d"
}

```

| Important                                                                 |
|---------------------------------------------------------------------------|
| This solution does not create or manage MediaTailor prefetch schedules    |