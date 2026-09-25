# KiwiIRC - Audio / Video conferencing

This plugin integrates the [Jitsi Meet](https://jitsi.org/jitsi-meet/) conference software into KiwiIRC.

Features -
* Individual conference rooms for channels and private messages
* Video, audio, or both, directly within Kiwi IRC itself
* Continue using Kiwi IRC and other channels without dropping the conference call
* Channel operators automatically promoted to conference room moderators

### Building
~~~shell
yarn && yarn build
~~~

Copy `dist/plugin-conference.js` to your Kiwi plugins folder

### Loading the plugin into Kiwi IRC
Add the plugin javascript file to your kiwiirc `config.json` and configure the settings:

```json
{
    "plugins": [
        {
            "name": "conference",
            "url": "static/plugins/plugin-conference.js"
        }
    ],
    "conference": {
        "server": "meet.jit.si",
        "secure": false
    }
}
```

### Security note!
By default, this plugin uses Jisti's public servers. It should be noted that by using these public servers, your conference calls are not secure in that anybody can join them if they can guess the room name.

Note that the "secure" option enables JWT authentication, but will not work on Jitsi's public server.

### Extra configuration
Jitsi Meet supports extra configuration to customise its interface and functions. You can configure these via the optional `interfaceConfigOverwrite` and `configOverwrite` config options.

The defaults are:
~~~json
"conference": {
    "secure": false,
    "server": "meet.jit.si",
    "queries": true,
    "channels": true,
    "buttonIcon": "fa-phone",
    "viewHeight": "40%",
    "enabledInChannels": ["*"],
    "groupInvitesTTL": 30000,
    "maxParticipantsLength": 60,
    "participantsMore": "more...",
    "inviteText": "{{ nick }} is inviting you to a private call.",
    "joinText": "{{ nick }} has joined the conference.",
    "joinButtonText": "Join now!",
    "showLink": false,
    "useLinkShortener": false,
    "linkShortenerURL": "https://x0.no/api/?{{ link }}",
    "linkShortenerAPIToken": "API_KEY_HERE",
    "interfaceConfigOverwrite": {
        "SHOW_JITSI_WATERMARK": false,
        "SHOW_WATERMARK_FOR_GUESTS": false,
        "TOOLBAR_BUTTONS": [
            "microphone", "camera", "fullscreen", "hangup",
            "settings", "videoquality", "filmstrip", "fodeviceselection",
            "stats", "shortcuts",
        ],
    },
    "configOverwrite": {
        "startWithVideoMuted": true,
        "startWithAudioMuted": true,
    },
}
~~~

The 'showLink' item, if enabled, will append a direct link to the broadcasted join message which opens the jitsi conference for non-Kiwi users.
The 'useUseLinkShortener' item, if enabled (requires showLink to also be enabled), will shorten the link displayed using a link shortening service like Bitly. If you like, you can sign up for a free Bitly account to use the API (https://bitly.com/). Once registered, follow the instructions to generate an access token, then provide that in Kiwi's config @ "linkShortenerAPIToken". Note that some services, like x0.no do not require API tokens, in which case the token can be omitted.

Examples of linkShortenerURL data are:

If using Bitly:

    `https://api-ssl.bitly.com/v3/shorten`

Alternative shortener that doesn't require an API token:

    `https://x0.no/api/?{{ link }}`

note: for link shorteners other than Bitly `{{ link }}` is replaced with the conference url and the response is read from the body

More info about Jitsi's options can be found in these files:
* https://github.com/jitsi/jitsi-meet/blob/master/interface_config.js
* https://github.com/jitsi/jitsi-meet/blob/master/config.js

You may also choose to hide the conference call icon in either channels or private messages:
```json
{
    "channels": false,
    "queries": false
}
```
### Running your own conference server
Running your own conference server allows you to secure your conference rooms. We make use of the Jitsi Meet server to handle the conference calls, the installation steps can be found here: https://github.com/jitsi/jitsi-meet/blob/master/doc/quick-install.md

### Self-hosted Jitsi via docker-jitsi-meet (fork addition)
This section documents a known-working setup for pointing this plugin at your
own [jitsi/docker-jitsi-meet](https://github.com/jitsi/docker-jitsi-meet)
instance, reverse-proxied through Apache/nginx (e.g. behind Cloudflare) rather
than exposing Jitsi's own web container directly to the internet.

**Docker Compose (`.env`)**
- `PUBLIC_URL=https://meet.example.com`
- `JVB_ADVERTISE_IPS=<your real public IPv4>` -- required regardless of
  Cloudflare/any CDN in front of the web UI. Call media (JVB's UDP port) is a
  raw RTP/UDP port that clients connect to directly; it never goes through
  Cloudflare (Cloudflare's proxy only carries HTTP(S)/WebSocket on 80/443,
  not arbitrary UDP), so JVB must advertise the real IP for ICE to work.
- `JVB_PORT=<any free UDP port>` -- must be open/forwarded on your firewall.
- `DISABLE_HTTPS=1` -- if Apache/nginx terminates TLS in front of the web
  container (recommended, keeps cert management in one place with your other
  vhosts), Jitsi's own web container should stay plain-HTTP internally.
- Web container ports bound to `127.0.0.1` only (e.g. `HTTP_PORT=127.0.0.1:8000`)
  so it's unreachable except through your reverse proxy.

**Reverse proxy**
The Jitsi web container's own nginx handles all app paths internally
(`/http-bind`, `/xmpp-websocket`, static assets, etc.) -- your reverse proxy
just needs to forward everything to it with WebSocket upgrade support for
`/xmpp-websocket`. An Apache example:
```apache
RequestHeader set X-Forwarded-For expr=%{REMOTE_ADDR}
RequestHeader set X-Forwarded-Proto "https"
ProxyPreserveHost On
RequestHeader set Connection "Upgrade" env=HTTP_UPGRADE
ProxyPass /xmpp-websocket ws://127.0.0.1:8000/xmpp-websocket
ProxyPassReverse /xmpp-websocket ws://127.0.0.1:8000/xmpp-websocket
ProxyPass / http://127.0.0.1:8000/
ProxyPassReverse / http://127.0.0.1:8000/
```
(nginx: proxy the same paths with `proxy_set_header Upgrade`/`Connection` on
the `/xmpp-websocket` location.)

**Plugin config** -- point it at your instance:
```json
"conference": {
    "server": "meet.example.com",
    "secure": false
}
```

**`secure: true` (JWT-authenticated rooms)** -- this plugin sends the IRCd's
own `EXTJWT` command to fetch a room-scoped JWT before joining, so it only
works if:
1. Your IRCd supports the `EXTJWT` command and is configured with a signing
   key.
2. Your Jitsi instance has `AUTH_TYPE=jwt` set, with `JWT_APP_SECRET`
   matching that same signing key (`JWT_APP_ID` matching your IRCd's issuer).

Without a matching IRCd, leave `secure: false` -- rooms are then only as
private as their (unguessable-ish) generated room name, same as the public
`meet.jit.si` default.

**A note on moderation**: self-hosted Jitsi with no auth configured (the
`secure: false` default above) already grants moderator to whoever joins an
empty room first -- no channel-op lookup or extra plugin logic needed for
"whoever starts the call is the moderator."

## License

[ Licensed under the Apache License, Version 2.0](LICENSE).
