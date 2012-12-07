# HTTP Subscriptions Specification
Version: 0.1 draft

HTTP Subscriptions is a simple specification that extends HTTP to support subscribing to events based on HTTP callbacks (webhooks). 

## High-level Overview
At a high-level, the protocol is similar the subscription side of PubSubHubbub. 

1. Subscriber agent (client) performs a subscription request against an HTTP Subscriptions aware endpoint known as an event source endpoint. The request contains a callback URL.
1. The server, called the subscription server, responds accordingly. It will asynchronously perform a verification step.
1. If verification succeeds, the subscription server will begin making event requests to the callback URL until the subscriber agent performs an unsubscribe request, or the subscription lease expires.

## Detailed Flow

### Subscription Request
A subscriber sends a special subscription request to an event source endpoint that supports HTTP Subscriptions. This request includes necessary information to create a subscription. The HTTP Subscription information is all in the headers, including a callback URL that represents the subscriber. This leaves the content payload available for vendor specific request details. 

The primary identifier of it being a subscription request is the `Pragma: subscribe` header. This may optionally include a topic to inform the subscription server what events the subscriber is interested in. This would look like `Pragma: subscribe=topic-string`. However, this is optional as the event source endpoint already represents a topic.

	POST /eventsource HTTP/1.1
	Host: subscription-server.com
	Pragma: subscribe
	Callback: <http://example.com/callback>; method="POST"; secret="oingoboingo"; rel="subscriber"
	Prefer: subscription-lease=604800

The server will respond with 202 Accepted and optionally provide a URL to a resource representing the subscription created. However, the subscription is not yet verified, hence the 202 Accepted, not 201 Created. 

	HTTP/1.1 202 Accepted
	Link: <http://subscription-server.com/subscription>; rel="subscription"

### Subscription Verification
The verification step is performed by the server before activating the subscription. This step is to verify that the callback URL is willing to accept a potentially high request generating subscription. There are two ways to provide verification: a whitelist/blacklist method, or a per callback URL method. 

The whitelist/blacklist method is tried first. The idea is that a given domain may always want to verify the intent of an HTTP Subscription, so implementing the per callback URL method will needlessly be inefficient. Since we are effectively talking about whether an automated request can be allowed or disallowed for a certain part of a domain, we use the robots.txt at the root of the callback URL domain to verify if a given callback URL on this domain is allowed. 

    # http://example.com/robots.txt
    User-Agent: *
    Allow: /somepath
    Disallow: /

If there is no robots.txt or robots.txt "disallows" a path, the subscription server will attempt verification against the specific callback URL. This is similar to the PubSubHubbub verification of intent mechanism. The subscription server will attempt a special request to the callback URL provide a generated challenge token. It will include other information about the subscription request, including a potential URL to a subscription.

	GET /callback HTTP/1.1
	Host: example.com
	Pragma: subscribe-verify=THIS_IS_A_CHALLENGE_STRING
	Link: <http://subscription-server.com/subscription>; rel="subscription"
	Prefer: subscribe-lease=3600

The callback URL must return a 200 OK response with the challenge token in a corresponding Pragma directive. This is an opt-in approach, making sure this URL is aware and willing to handle the subscription. 

	HTTP/1.1 200 OK
	Pragma: subscribe-confirm=THIS_IS_A_CHALLENGE_STRING
	Content-Length: 0

### Event Requests
Now the subscription server can start sending event requests to the callback URL. These requests should conform to the arguments provided in the original Callback header, including which HTTP method to use, and if a Content-HMAC signature should be provided (although the subscription server may include this anyway using an out-of-band secret). The event request should also include a reference to the subscription resource if one exists. The content payload is completely up to the subscription server.

	POST /callback HTTP/1.1
	Host: example.com
	Link: <http://subscription-server.com/subscription>; rel="subscription"
	Content-HMAC: sha1 C+7Hteo/D9vJXQ3UfzxbwnXaijM=
	Content-Length: 21
	Content-Type: application/x-www-form-urlencoded

	payload=Hello%20world

The subscription server will expect a 200 OK response. The subscription server may implement policies to cancel the subscription if the callback URL consistently returns an error code.

### Unsubscribe Request
Unsubscribing is done very similarly to subscription requests on the event source URL. However, the server may allow other ways to unsubscribe, such as performing a DELETE on the subscription resource. Regardless, the HTTP Subscription method of unsubscribing should be implemented to be compatible with HTTP Subscription clients/libraries. Alternative mechanisms should be considered a convenience for the user or to maintain consistency with the design of the rest of the API. 

	POST /eventsource HTTP/1.1
	Host: subscription-server.com
	Pragma: unsubscribe
	Callback: <http://example.com/callback>; method="POST"; secret="oingoboingo"; rel="subscriber"
	Prefer: subscription-lease=604800

The unsubscribe request should include the same HTTP Subscription details of the original subscribe request. Regardless of the mechanism used to initiate an unsubscribe, the subscription server needs to verify with the callback URL that an unsubscribe is desired. This is done again very similar to subscription verification, however only using the per callback method:

	GET /callback HTTP/1.1
	Host: example.com
	Pragma: unsubscribe-verify=THIS_IS_A_CHALLENGE_STRING
	Link: <http://subscription-server.com/subscription>; rel="subscription"
	Prefer: subscribe-lease=3600

And the callback URL should respond:

	HTTP/1.1 200 OK
	Pragma: unsubscribe-confirm=THIS_IS_A_CHALLENGE_STRING
	Content-Length: 0

### Leasing and Automatic Refreshing
TODO
