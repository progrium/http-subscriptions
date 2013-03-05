# HTTP Subscriptions Specification
Version: 0.1 draft

HTTP Subscriptions is a simple specification that extends HTTP to support subscribing to events based on HTTP callbacks (webhooks). 

## High-level Overview
At a high-level, the protocol is similar the subscription side of PubSubHubbub. 

1. Subscriber agent (client) performs a subscription request against an HTTP Subscriptions aware endpoint known as an event source endpoint. The request contains a callback URL.
1. The server, called the subscription server, responds accordingly.
1. The subscription server makes event requests to the callback URL until the subscriber agent performs an unsubscribe request, or the subscription lease expires.

## Detailed Flow

### Subscription Request
A subscriber sends a special subscription request to an event source endpoint that supports HTTP Subscriptions. This request includes necessary information to create a subscription. The HTTP Subscription information is all in the headers, including a callback URL that represents the subscriber. This leaves the content payload available for vendor specific request details. 

The primary identifier of it being a subscription request is the `Pragma: subscribe` header. This may optionally include a topic to inform the subscription server what events the subscriber is interested in. This would look like `Pragma: subscribe=topic-string`. However, this is optional as the event source endpoint already represents a topic.

	POST /eventsource HTTP/1.1
	Host: subscription-server.com
	Pragma: subscribe
	Callback: <http://example.com/callback>; method="POST"; secret="oingoboingo"; rel="subscriber"
	Prefer: subscription-lease=604800

The server will respond with 201 Created and optionally provide a URL to a resource representing the subscription created. 

	HTTP/1.1 201 Created
	Link: <http://subscription-server.com/subscription>; rel="subscription"

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

The unsubscribe request should include the same HTTP Subscription details of the original subscribe request. The server responds with 200 OK if the details match.

### Leasing and Automatic Refreshing
TODO
