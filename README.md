# element-call-traefik

This is my personal documentation on how i host and integrate element-call to conduct private video-chats.
It assumes that you have traefik setup and running, but provides a working config as example.

This setup looks somewhat like this:
1. Traefik as proxy and for ssl-termination
1. a matrix instance (synapse) on `matrix.<your-domain>`
   - It provides authorization and manages your rooms/groups
1. a matrix-sliding-proxy on `syncv3.<your-domain>` if you want to use the Element X app 
1. the element-call webapp on `element-call.<your-domain>`
1. a livekit instance on `livekit.<your-domain>`
   - livekit does the heavy lifting and is basically the videocall-backend to the element-call webapp

The whole setup here is split into multiple compose files. I hope this makes it easier to focus on one topic at a time. On my on server, everything is in one folder and a single compose file.

# 0. Setup the subdomains

make sure the required subdomains (see above) point to your server

# 1. Setup the Element-Call Webapp

1. Create a folder for the elment-call webapp to live in. In this repo this is the  `element-call` folder. If you are like me and have everything matrix related in a single folder, feel free to use that one.
1. Go into that folder and create an `element-call-config.json`. It tells your element-call instance where to find your matrix homeserver. It could also tell element-call where to finde the livekit backend, but the prefered way to set this is the `.well-known/matrix/client`-file that we will setup later. See the example in this repo for what the element-call-config looks like (you have to replace `<your-domain>`)
1. Create a compose.yaml (or use your existing one) and add the `element-call` service as is shown in this repo.

You should now be able to `docker compose up -d element-call` and visit `element-call.<your-domain>`. You should already see the element-call login. If you already configured your matrix-server, you might be able to login, though calls won't work yet because livekit is not yet setup.

# 2. Setup Livekit

1. Go into the folder that contains the `compose.yaml` that contains the element-call service
1. Copy over the `livekit.yaml` and `livekit-redis.conf` from this repo as a starting point.
   - Replace `<your-domain>` in these files
1. Add the `livekit`, `livekit-redis` and `livekit-jwt-service` to the `compose.yaml` as shown in this repo
   - Replace `<your-domain>` in the environments and labels of these services
   - In an ideal world you wouldn't need the [livekit-jwt-service](https://github.com/element-hq/lk-jwt-service). It only provides two routes, and traefik is configured to route only these two to the jwt-service.
   - the jwt-service is using the same subdomain as the livekit backend to prevent cors issues
1. Create a temporary folder somewhere in which you run the livekit/generate container as described [in their docs](https://docs.livekit.io/home/self-hosting/vm/). This will get you a `livekit.yaml` with a key and a secret.
1. Replace `<livekit-key>` and `<livekit-secret>` with the values from the generated `livekit.yaml`
1. Remove the temporary folder. It is no longer needed.
1. forward the following ports to your server if you have a firewall:
   - 3478 (udp, STUN)
     - skip this one if you don't want to have your STUN-traffic be unencrypted. STUN allowes clients to find each others public ip address. This is usally not considered sensitive data though.
   - 7883 (udp, WebRTC)
   - 7881 (tcp, WebRTC-Fallback)
   - 50000-50255 (udp, TURN relay range)
   - 80 (tcp+udp, http)
   - 443 (tcp+udp, https+wss)

> Here is why these Ports can be forwarded (mostly) without exposing unencrypted traffic:  
> - Port 80 goes to traefik, which just redirects http-traffic to port 443 with encryption
> - Port 443 also goes to traefik and is encrypted (HTTPS/wss)
> - Port 3478 is the only one actually handling unencrypted traffic (see above)
> - 7881 and 7883 Go to livekit. They handle WebRTC-Traffic, which is as part of its spec, must be encrypted. They therfore don't need the ssl-termination through traefik
> - Ports 50000-50255 are used to funnel traffic between clients that can't directly connect to each other, but can connect to the server. If i understand it correctly, it should only ever contain WebRTC-Traffic in this scenario, which is always encrypted. (see https://stackoverflow.com/a/23300741)

You should now be able to `docker compose up -d` These services. It will not be used by element-call yet, because we have to tell element-call where to find the livekit backend. This is done later via the file served at `https://matrix.<your-domain>/.well-known/matrix/client`

# 3. Setup well-known files for Matrix/synapse

Services that belong to the Element ecosystem want to know where they can find each other. To do so, they usually ask the `./well-known/matrix/client`-config of the homeserver.
In order to easily adjust what is served here i decided to create it myself, instead of relying on synapse to create it for me.

To serve the required `well-known`-configs do the following:
1. Add the `well-known-nginx` service to the compose.yaml that contains your synapse service (see this repos `matrix/compose.yaml`)
1. Add the `nginx.conf`
1. replace `<your-domain>` in the services labels and the `nginx.conf`

You should now be able to `docker compose up -d well-known-nginx`. treafik should then route well-known requests for the homeserver and the syncv3 proxy to that service

If you look inside the `nginx.conf` you'll see that it serves 3 "well-know" files.
- The `/client` one is so that Element X knows where to find the syncv3-proxy, and **so that element-call knows where to find the livekit backend**
- The `element.json`-one is so that Element X knows where to find the element-call webapp
- I have to recheck where the `/server`-one is used, not sure about that at the moment. Could be Element X related

**The Element-Call Webapp should be usable now**
- Go to https://element-call.<your-domain>
- Login with a user on your homeserver
- Try starting a call


# 4. Configure Matrix/synapse (optional)

As per the [element-call docs](https://github.com/element-hq/element-call?tab=readme-ov-file#configuration), you can add this to your synapses `homeserver.yaml`:
```
experimental_features:
    msc3266_enabled: true
```
MSC3266 is the Room Summary API ([Proposal](https://github.com/deepbluev7/matrix-doc/blob/room-summaries/proposals/3266-room-summary.md)).
This option is supposed to only be required for "guest access / knocking" (https://github.com/element-hq/element-call/pull/2343#issuecomment-2085260557). One user had problems with invite-links not working without this setting (https://github.com/element-hq/element-call/issues/2340).

As i only video-call with users on my homeserver, i've had no issues so far that were caused by this feature not being enabled.

# App-Support
## Videocall-button in Element X
you matrix-domain has to serve the correctly configured `/.well-known/element/element.json` in order for the element x app uses your self-hosted element-call website when using the video-call button

## Videocall-button and Video-Rooms in the Element Webapp
If you self-host the element-web app (see the element-web/compose.yaml for an example), you can provide a json-config-file. In there you can specify this:
```json
{
  "element_call": {
    "url": "https://element-call.<your-domain>"
  },
  "features": {
    "feature_video_rooms": true,
    "feature_group_calls": true,
    "feature_element_call_video_rooms": true
  }
}
```
**This will make visitors who use your self-hosted element-web to directly use your self-hosted element-call instance when choosing to initiate an element-call call. It also enables
- video-rooms + element-call-video-rooms (special rooms where the main focus is a perpetuate video call, but that also includes a text-chat)
- group-calls (make sure you have permissions to make element-call calls in your groups)

# Element Desktop app

The desktop app just wraps element-web. for now, it seems to be impossible to
configure the well-known in a way that tells element-desktop what element-call url to use.
It could be that in the future, this can be configured via `/.well-known/element/element.json` just like it can be for the Element X app..

For now i suggest "[installing](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Installing#installing_pwas)" a configured, self-hosted element-web instance as PWA

For more Info, see https://github.com/element-hq/element-meta/issues/2441 and https://github.com/element-hq/element-meta/issues/2441
