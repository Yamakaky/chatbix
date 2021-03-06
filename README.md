# Chatbix, the Chatbox with a typo

## Build

```sh
$ echo 'DATABASE_URL=postgres://dbuser:dbpassword@dbaddress/chatbox' > .env
$ echo 'LISTEN_URL=0.0.0.0:PORT' >> .env
$ cargo run --bin chatbix
```

## API

Every route below has for base URI `http(s)://address.of.chat/api/`

### Structs and objects

#### Timestamp

Timestamps can either be of the format 1485402097 or of the format
2017-01-26T04:41:37Z

the integer format is preferred because it doesn't leave place to interpretation,
plus the parsing doesn't cost as much (even though it's still minimal)

#### Tags

tags are on 32bits:

* tags & 1 = logged\_in (generated by the server)
* tags & 2 >> 1 = generated // is this message generated from another message
* tags & 4 >> 2 = bot // is this message sent by a bot ?
* tags & 8 >> 3 = no\_notif // should this message ignore notifications rules and not notify him
* tags & (2^4 + 2^5 + 2^6 + 2^7) >> 4 = show\_value
* tags & (2^8 | 2^9) >> 8 = text_format
* everything else: reserved for later use
  
`show_value`:

* show\_value : u4; 
* show\_value = 0: no change;
* show\_value = 1 to 4; 'hidden' message, with 4 being more hidden than 1
* show\_value = 9 to 12; 'important' message, with 12 being more important

^ these are basically ignored by the server and are only implementation dependant
you can set up your client to never show show_value = 4, and show show_value = 1 but not
notify, ...

* text\_format = 0: plain text
* text\_format = 1: markdown
* text\_format = 2: preformatted / code -> content that must not trim spaces and use a monospaced to display the content
* text\_format = 3: (reserved for later use)

^ these are ignored by the server as well, ...

#### Auth Key

Auth Key describes a alphanumeric string of length 16, for instance "f4xVbuTlR9bTl0i6"

It is retrieved when logging in, and can be used to confirm one's identity (to be sure that
messages come from the owner and not some fake). It can also be used to have access to
admin commands, but this happens only when the user is an admin (of course).

### Retrieving messages

Method: GET

* Retrieving the last X messages : `/api/get_messages`
  (Note: X is hardcoded for now)
* Retrieving all messages since T : `/api/get_messages?timestamp=T`
* Retrieving all messages since message of id I : `/api/get_messages?message_id=I`
* Retrieving all messages between T1 and T2 `/api/get_messages?timestamp=T1&timestamp_end=T2`
* Retrieving all messages of the default channel plus the channel C `/api/get_messages?channel=C`
* Retrieving all messages of the default channel plus multiple channels C1, C2, ... : `/api/get_messages?channel=C1?channel=C2`, `/api/get_messages?channels=C1,C2,C3`, or any combination of both
* If you want to only retrieve a channel without the default one: `/api/get_messages?channel=C?no_default_channel?message_id=I`

### Sending a new message

The URI is always POST `/api/new_message`

The body must always be valid JSON.

These values are required:

* username: string
* content: string

These values are optional:

* tags: integer, see [the tags section](#Tags)
* color: a value of "#RRGGBB" is expected, but it is not checked. You could input "red", "blue" or whatever as well, but don't expect it to be parsed by other clients
* channel: string, name of the channel this should be sent to
* auth\_key: string, see the section Auth Key

### Logging in

POST `/api/login`

Required values in the JSON body:

* username: string
* password: string

your password is sha512'd before entering the DB, so you can either input a hashed password or the raw password.
Default clients will sha512 the password before sending it via this request though, so take that into account
if you want to be compatible with other clients.

Returns `{"auth_key":AUTH_KEY}` on success

The AUTH\_KEY will stay the same until the server is restarted (and thus the cache is discarded), or until you
call `/api/logout`. This means that multiple clients can be connected with the same auth\_key.

### Logging out

POST `/api/logout`

Required values:

* username: string
* auth\_key: see Auth Key

### Registering

POST `/api/register`

Required values in the JSON body:

* username: string
* password: string

Returns `{"auth_key":AUTH_KEY}` on success

### Heartbeat

GET `/api/heartbeat`

Get the last messages since timestamps for the given channels, and also get the "connected" users.

Parameters are the same as `get\_messages`, without `timestamp\_end`

And these are added as well:

* username: (your username)
* (optionnal) auth\_key (your auth\_key)
* active: TRUE/true/1 OR FALSE/false/0 , with default "true"

When the window is not focused in the chat anymore, you should set `active` to `false`.

## License

Dual licensed under MIT / Apache-2.0
