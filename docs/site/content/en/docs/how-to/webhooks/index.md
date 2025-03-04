---
title : "Webhooks"
description: "Webhooks"
lead: ""
date: 2020-10-06T08:48:45+00:00
lastmod: 2020-10-06T08:48:45+00:00
draft: false
images: []
weight: 121
---

The project uses [Webhooks](https://en.wikipedia.org/wiki/Webhook) to send the messages and events from WhatsApp to your
application.

Webhooks are a way for two different applications to communicate with each other in real-time.
When a certain event happens in one application, it sends a message to another application through a webhook URL.
The receiving application can then take action based on the information received.

## Setup webhooks
### Session webhooks
You can define webhooks configuration per session when you start it with `POST /api/sessions/start` request data.

Here's a simple example:
```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://httpbin.org/post",
        "events": [
          "message"
        ]
      }
    ]
  }
}

```

Here's available configuration options for webhooks
```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://httpbin.org/post",
        "events": [
          "message"
        ],
        "hmac": {
          "key": "your-secret-key"
        },
        "retries": {
          "delaySeconds": 2,
          "attempts": 15
        },
        "customHeaders": [
          {
            "name": "X-My-Custom-Header",
            "value": "Value"
          }
        ]
      }
    ]
  }
}
```

### Global webhooks
There's a way how you can configure webhooks for ALL sessions - by settings these environment variables:

- `WHATSAPP_HOOK_URL=https://httpbin.org/post`  - to set up a URL for the webhook
- `WHATSAPP_HOOK_EVENTS=message,message.any,state.change` - specify events. Do not specify all of
  them, it's too heavy payload, choose the right for you.
- `WHATSAPP_HOOK_EVENTS=*` - subscribe to all events. It's not recommended for production, but it's fine for
development.

That webhook configuration **does not appear** in `session.config` field in `GET /api/sessions/` request.

## Webhook payload

On the URL that you set you'll receive **HTTP POST** request with a JSON string with following format:

```json
{
  "event": "message",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    ...
  }
}
```

Where `event` value helps you identify the incoming event with `payload` for that events.

Below the list of all events that WhatsApp HTTP API sends to your.

{{< alert icon="👉" text="If you want to look at a webhook body - use the url https://httpbin.org/post" />}}

Run the bellow command and see look at the logs - it prints body request for all events that happen in your WhatsApp!

```bash
docker run -it -e "WHATSAPP_HOOK_EVENTS=*" -e WHATSAPP_HOOK_URL=https://httpbin.org/post -p 3000:3000 devlikeapro/whatsapp-http-api
```



## Events

### message

Incoming message (text/audio/files)

```json
{
  "event": "message",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    "id": "true_11111111111@c.us_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
    "timestamp": 1667561485,
    "from": "11111111111@c.us",
    "fromMe": true,
    "to": "11111111111@c.us",
    "body": "Hi there!",
    "hasMedia": false,
    "ack": 1,
    "vCards": [],
    "_data": {
      "id": {
        "fromMe": true,
        "remote": "11111111111@c.us",
        "id": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
        "_serialized": "true_11111111111@c.us_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
      },
      "body": "Hi there!",
      "type": "chat",
      "t": 1667561485,
      "notifyName": "MyName",
      "from": "11111111111@c.us",
      "to": "11111111111@c.us",
      "self": "in",
      "ack": 1,
      "isNewMsg": true,
      "star": false,
      "kicNotified": false,
      "recvFresh": true,
      "isFromTemplate": false,
      "pollInvalidated": false,
      "latestEditMsgKey": null,
      "latestEditSenderTimestampMs": null,
      "broadcast": false,
      "mentionedJidList": [],
      "isVcardOverMmsDocument": false,
      "isForwarded": false,
      "hasReaction": false,
      "ephemeralOutOfSync": false,
      "productHeaderImageRejected": false,
      "lastPlaybackProgress": 0,
      "isDynamicReplyButtonsMsg": false,
      "isMdHistoryMsg": false,
      "stickerSentTs": 0,
      "isAvatar": false,
      "requiresDirectConnection": false,
      "pttForwardedFeaturesEnabled": true,
      "isEphemeral": false,
      "isStatusV3": false,
      "links": []
    }
  }
}
```

### message.any

Fired on all message creations, including your own. The payload is the same as for [message](#message) event.

```json
{
  "event": "message.any",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    ...
  }
}
```

### message.ack

```json
{
  "event": "message.ack",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    ...
  }
}
```

### state.change

It's an internal engine's state, not **session** `status`.

```json
{
  "event": "state.change",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    ...
  }
}
```

### group.join

```json
{
  "event": "group.join",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    ...
  }
}
```

### group.leave

```json
{
  "event": "group.left",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    ...
  }
}
```

### presence.update

- `payload.id` indicates the chat - either direct chat with a contact or a group chat.
- `payload.id.[].participant` - certain participant presence status. For a direct chat there's only one participant.

```json
{
    "event": "presence.update",
    "session": "default",
    "engine": "NOWEB",
    "payload": {
        "id": "111111111111111111@g.us",
        "presences": [
            {
                "participant": "11111111111@c.us",
                "lastKnownPresence": "typing",
                "lastSeen": null
            }
        ]
    }
}
```

## Webhooks Advanced ![](/images/versions/plus.png)
### HMAC authentication

You can authenticate webhook sender by using [HMAC Authentication](https://www.okta.com/identity-101/hmac/).

1. Define you secret key in `config.hmac.key` field when you start session with `POST /api/sessions/start`:

```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://httpbin.org/post",
        "events": [
          "message"
        ],
        "hmac": {
          "key": "your-secret-key"
        }
      }
    ]
  }
}
```

2. After that you'll receive all webhooks payload with two additional headers:
- `X-Webhook-Hmac` - message authentication code for the raw **body** in HTTP POST request that send to your endpoint.
- `X-Webhook-Hmac-Algorithm` - `sha512` - algorithm that have been used to create `X-Webhook-Hmac` value.

3. Implement the authentication algorithm by hashing body and using secret key and then verifying it with `X-Webhook-Hmac`
value. Please [check your implementation here ->](https://www.devglan.com/online-tools/hmac-sha256-online)

Here's example for
```
# Full body
{"event":"message","session":"default","engine":"WEBJS"}
# Secret key
my-secret-key
# X-Webhook-Hmac-Algorithm
sha512
# X-Webhook-Hmac
208f8a55dde9e05519e898b10b89bf0d0b3b0fdf11fdbf09b6b90476301b98d8097c462b2b17a6ce93b6b47a136cf2e78a33a63f6752c2c1631777076153fa89
```


### Retries
**WAHA** retries to reach your webhook URL **15 times** with **2 seconds delay** between attempts by default in
[Plus Version →]({{< relref "plus-version" >}})

You can configure those parameters by settings `config.retries` structure when `POST /api/sessions/start`:

```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://httpbin.org/post",
        "events": [
          "message"
        ],
        "retries": {
          "delaySeconds": 2,
          "attempts": 15
        }
      }
    ]
  }
}

```

### Custom Headers
You can send any customer headers by defining `config.webhooks.customHeaders` fields this way:
```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://httpbin.org/post",
        "events": [
          "message"
        ],
        "customHeaders": [
          {
            "name": "X-My-Custom-Header",
            "value": "Value"
          }
        ]
      }
    ]
  }
}
```

## Examples
Here's few examples of how to handle webhook in different languages:
1. [Python guide]({{< relref "/docs/examples/python" >}})

**Do you use another language?**

Please create a short guide how to handle webhook and send message after you finish your setup!
You can create a pull request with your favorite language in the
[GitHub, in examples folder ->](https://github.com/devlikeapro/whatsapp-http-api/tree/core/examples).
