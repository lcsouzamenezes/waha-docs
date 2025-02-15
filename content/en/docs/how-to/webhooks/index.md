---
title : "🔄 Webhooks"
description: "Webhooks, how to set up and handle them."
lead: ""
date: 2020-10-06T08:48:45+00:00
lastmod: 2020-10-06T08:48:45+00:00
draft: false
images: []
weight: 225
---

The project uses 
[Webhooks](https://en.wikipedia.org/wiki/Webhook) and [Websockets](https://en.wikipedia.org/wiki/WebSocket) 
to notify your application about the messages and events from WhatsApp.

![](webhooks.png)

Webhooks are a way for two different applications to communicate with each other in real-time.
When a certain event happens in one application, it sends a message to another application through a webhook URL.
The receiving application can then take action based on the information received.

## Setup
### Session webhooks
You can define webhooks configuration per session when you start it with `POST /api/sessions/` request data.

Here's a simple example:
```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://webhook.site/11111111-1111-1111-1111-11111111",
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
        "url": "https://webhook.site/11111111-1111-1111-1111-11111111",
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

- `WHATSAPP_HOOK_URL=https://webhook.site/11111111-1111-1111-1111-11111111`  - to set up a URL for the webhook
- `WHATSAPP_HOOK_EVENTS=message,message.any,state.change` - specify events. Do not specify all of
  them, it's too heavy payload, choose the right for you.
- `WHATSAPP_HOOK_EVENTS=*` - subscribe to all events. It's not recommended for production, but it's fine for
development.

That webhook configuration **does not appear** in `session.config` field in `GET /api/sessions/` request.

### Connect Websockets
Alternatively, you can use Websockets to receive messages in real-time. 
```bash
websocat -E ws://localhost:3000/ws
```

👉 Read more about it in the [**Websockets section**](#websockets) below.

## Webhook payload

On the URL that you set you'll receive **HTTP POST** request with a JSON string with following format:

```json
{
  "event": "message",
  "session": "default",
  // 'metadata' provided when you created the session
  "metadata": {
    "user.id": "123",
    "user.email": "email@example.com"
  },
  // me - your own contact, if authenticated and WORKING
  "me": {
    "id": "71111111111@c.us",
    "pushName": "~"
  },
  "payload": {
    ... // event specific data
  },
  "environment": {
    "tier": "PLUS",
    "version": "2023.10.12"
  },
  "engine": "WEBJS"
}
```

Where `event` value helps you identify the incoming event with `payload` for that events.

Below the list of all events that WhatsApp API sends to your.

💡 You can open [https://webhook.site](https://webhook.site) and paste UUID from it to `url` field,
and you'll see all requests immediately in your browser to intercept the webhook's payload.

Run the bellow command and see look at the logs - it prints body request for all events that happen in your WhatsApp!

```bash
docker run -it -e "WHATSAPP_HOOK_EVENTS=*" -e WHATSAPP_HOOK_URL=https://webhook.site/11111111-1111-1111-1111-11111111 -p 3000:3000 devlikeapro/waha
```

### Metadata
You can provide additional `metadata` when you start the session with 
[**Start Session**]({{< relref "/docs/how-to/sessions#start-session" >}}) 
request data.

```json
{
  "event": "message",
  "session": "default",
  // 'metadata' provided when you created the session
  "metadata": {
    "user.id": "123",
    "user.email": "email@example.com"
  },
  ...
}
```


## Webhooks
Here's the list of features that are available by [**🏭 Engines**]({{< relref "/docs/how-to/engines" >}}):

{{< include file="content/en/docs/how-to/webhooks/features.md" >}}

### session.status
The `session.status` event is triggered when the session status changes.
- `STOPPED` - session is stopped
- `STARTING` - session is starting
- `SCAN_QR_CODE` - session is required to scan QR code or login via phone number
  - When you receive the `session.status` event with `SCAN_QR_CODE` status, you can [**fetch updated QR ->**]({{< relref "/docs/how-to/sessions#get-qr" >}})
  - The `SCAN_QR_CODE` is issued every time when QR updated (WhatsApp requirements)
- `WORKING` - session is working and ready to use
- `FAILED` - session is failed due to some error. It's likely that authorization is required again or device has been disconnected from that account.
Try to restart the session and if it doesn't help - logout and start the session again.

```json
{
    "event": "session.status",
    "session": "default",
    "me": {
        "id": "7911111@c.us",
        "pushName": "~"
    },
    "payload": {
        "status": "WORKING"
    },
    "engine": "WEBJS",
    "environment": {
        "version": "2023.10.12",
        "engine": "WEBJS",
        "tier": "PLUS"
    }
}
```

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

### message.reaction
Receive events when a message is reacted to by a user (or **yourself** reacting to a message).
- `payload.reaction.text` - emoji that was used to react to the message. It'll be an empty string if the reaction was removed.
- `payload.reaction.messageId` - id of the message that was reacted to.

```json
{
    "event": "message.reaction",
    "session": "default",
    "me": {
        "id": "79222222222@c.us",
        "pushName": "WAHA"
    },
    "payload": {
        "id": "false_79111111@c.us_11111111111111111111111111111111",
        "from": "79111111@c.us",
        "fromMe": false,
        "participant": "79111111@c.us",
        "to": "79111111@c.us",
        "timestamp": 1710481111.853,
        "reaction": {
            "text": "🙏",
            "messageId": "true_79111111@c.us_11111111111111111111111111111111"
        }
    },
    "engine": "WEBJS",
    "environment": {
        "version": "2024.3.3",
        "engine": "WEBJS",
        "tier": "PLUS",
        "browser": "/usr/bin/google-chrome-stable"
    }
}
```

{{< alert icon="👉" text="NOWEB engine note - reactions were sent in 'message' and 'message.any' events, not it's available only in 'message.reaction'!" />}}

### message.ack
Receive events when server or recipient gets the message, read or played it.

`ackName` field contains message status (`ack` has the same meaning, but show the value in int, but we keep it for backward compatability, they much to each other)

Possible message ack statuses:
- `ackName: ERROR, ack: -1`
- `ackName: PENDING, ack: 0`
- `ackName: SERVER, ack: 1`
- `ackName: DEVICE, ack: 2`
- `ackName: READ, ack: 3`
- `ackName: PLAYED, ack: 4`


The payload may have more fields, it depends on the engine you use, but here's a minimum amount that all engines send:
```json
{
  "event": "message.ack",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    "id":"true_11111111111@c.us_4CC5EDD64BC22EBA6D639F2AF571346C",
    "from":"11111111111@c.us",
    "participant": null,
    "fromMe":true,
    "ack":3,
    "ackName":"READ"
  }
}
```

### message.waiting
Happens when you see
[Waiting for this message. This may take a while.](https://faq.whatsapp.com/3398056720476987)
on your phone.

![waiting for this message](waiting-for-this-message.jpg)

```json
{
  "event": "message.waiting",
  "session": "default",
  "engine": "WEBJS",
  "payload": {
    "id": "true_11111111111@c.us_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
    "timestamp": 1667561485,
    "from": "11111111111@c.us",
    "fromMe": true,
    "to": "11111111111@c.us",
    ...
    "_data": {
      ...
    }
  }
}
```

### message.revoked
The `message.revoked` event is triggered when a user, whether it be you or any other participant,
revokes a previously sent message.

```json
{
  "event": "message.revoked",
  "session": "default",
  "payload": {
    "before": {
      "id": "some-id-here",
      "timestamp": "some-timestamp-here",
      "body": "Hi there!"
    },
    "after": {
      "id": "some-id-here",
      "timestamp": "some-timestamp-here",
      "body": ""
    }
  }
}
```

**Important notes**:
1. The above messages' ids don't match any of the ids you'll receive in the `message` event, it's a different id.
2. In order to find the message that was revoked, you'll need to search for the message with
   the same timestamp and chat id as the one in the `after` object.
3. `before` field can be null in some cases.

### chat.archive
{{< include file="content/en/docs/how-to/chats/webhooks-chat-archive.md" >}}

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
  "event": "group.leave",
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

### poll.vote
We have a dedicated page [how to send polls and receive votes]({{< relref "/docs/how-to/polls" >}})!

```json
{
  "event": "poll.vote",
  "session": "default",
  "payload": {
    "vote": {
      "id": "false_1111111111@c.us_83ACBE602A05C79B234B54415E95EE8A",
      "to": "me",
      "from": "1111111@c.us",
      "fromMe": false,
      "selectedOptions": ["Awesome!"],
      "timestamp": 1692861427
    },
    "poll": {
      "id": "true_1111111111@c.us_BAE5F2EF5C69001E",
      "to": "1111111111@c.us",
      "from": "me",
      "fromMe": true
    }
  },
  "engine": "NOWEB"
}
```

### poll.vote.failed
We have a dedicated page [how to send polls and receive votes]({{< relref "/docs/how-to/polls" >}})!

```json
{
  "event": "poll.vote.failed",
  "session": "default",
  "payload": {
    "vote": {
      "id": "false_11111111111@c.us_2E8C4CDA89EDE3BC0BC7F605364B8451",
      "to": "me",
      "from": "111111111@c.us",
      "fromMe": false,
      "selectedOptions": [],
      "timestamp": 1692956972
    },
    "poll": {
      "id": "true_1111111111@c.us_BAE595F4E0A2042C",
      "to": "111111111@c.us",
      "from": "me",
      "fromMe": true
    }
  },
  "engine": "NOWEB"
}
```

### labels.upsert

```json
{
  "event": "label.upsert",
  "session": "default",
  "payload": {
    "id": "10",
    "name": "Label Name",
    "color": 14,
    "colorHex": "#00a0f2"
  },
  "engine": "NOWEB",
  ...
}

```

### labels.deleted

```json
{
  "event": "label.deleted",
  "session": "default",
  "payload": {
    "id": "10",
    "name": "",
    "color": 14,
    "colorHex": "#00a0f2"
  },
  "engine": "NOWEB",
  ...
}

```

### labels.chat.added

```json
{
  "event": "label.chat.added",
  "session": "default",
  "payload": {
    "labelId": "6",
    "chatId": "11111111111@c.us",
    "label": null <=== right after scanning QR it can be null. 
  },
  "engine": "NOWEB",
  ...
}
```

### labels.chat.deleted

```json
{
  "event": "label.chat.deleted",
  "session": "default",
  "payload": {
    "labelId": "6",
    "chatId": "11111111111@c.us",
    "label": null
  },
  "engine": "NOWEB",
  ...
}
```

### call.received

{{< include file="content/en/docs/how-to/calls/webhooks-call-received.md" >}}

### call.accepted

{{< include file="content/en/docs/how-to/calls/webhooks-call-accepted.md" >}}

### call.rejected

{{< include file="content/en/docs/how-to/calls/webhooks-call-rejected.md" >}}

### state.change
⚠️ **DEPRECATED**, use `session.status` event instead.

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


## Webhooks Advanced ![](/images/versions/plus.png)
### HMAC authentication

You can authenticate webhook sender by using [HMAC Authentication](https://www.okta.com/identity-101/hmac/).

1. Define you secret key in `config.hmac.key` field when you start session with `POST /api/sessions/`:

```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://webhook.site/11111111-1111-1111-1111-11111111",
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
[Plus Version →]({{< relref "waha-plus" >}})

You can configure those parameters by settings `config.retries` structure when `POST /api/sessions/`:

```json
{
  "name": "default",
  "config": {
    "webhooks": [
      {
        "url": "https://webhook.site/11111111-1111-1111-1111-11111111",
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
        "url": "https://webhook.site/11111111-1111-1111-1111-11111111",
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

## Websockets
You can use Websockets to receive messages in real-time.

Install [websocat](https://github.com/vi/websocat?tab=readme-ov-file#installation) first.

```bash
# Listen all sessions and events
# -E to end the connection when the server closes it
websocat -E ws://localhost:3000/ws
# Listen all sessions and events (explicitly)
websocat -E ws://localhost:3000/ws?session=*&events=*

# Listen certain events
# (!) Only 'session.status' event is supported now
websocat -E ws://localhost:3000/ws?session=*&events=session.status

# If you're using HTTPS (SSL) connection
websocat -E wss://localhost:3000/ws?session=*&events=session.status

# If you're using Api Key - make sure to add it to the URL
websocat -E ws://localhost:3000/ws?x-api-key=123

# If you want to see the logs and ping the server every 10 seconds
websocat -v --ping-interval=10 -E ws://localhost:3000/ws

# Listen certain session
# NOT SUPPORTED YET
# websocat -E ws://localhost:3000/ws?session=default&events=session.status
```

⚠️ Right now websockets has limited support for events, but we're working on it to add more events in the future. 
- No session filtering, you can listen to all sessions only.
- Only [session.status](#session.status) event is supported now.

Fill free to [**Create Feature Request**](https://github.com/devlikeapro/waha/issues/new/choose)
if you need more events or per session filters.

## Examples
Here's few examples of how to handle webhook in different languages:
1. [Python guide]({{< relref "/docs/integrations/python" >}})

**Do you use another language?**

Please create a short guide how to handle webhook and send message after you finish your setup!
You can create a pull request with your favorite language in the
[GitHub, in examples folder ->](https://github.com/devlikeapro/waha/tree/core/examples).
