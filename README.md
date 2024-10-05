# element-call-traefik

I'll have to update the readme, but the important parts are:

- run the livekit/generate container as described [in their docs](https://docs.livekit.io/home/self-hosting/vm/) to get a `livekit.yaml` with a key and a secret. Use them in the `livekit.yaml` and `element-call/compose.yaml`
- go into the element-call folder and clone [element-call](https://github.com/element-hq/element-call) into it (`git clone https://github.com/element-hq/element-call.git`)
- replace <your-domain> the element-call/public/config.json and move it to the corresponding folder in the cloned element-call repo
- Replace the Dockerfile in the cloned element-call repo with the one from here, so that the webapp can be built
- forward ports 3478 (udp, STUN), 7883 (udp, WebRTC), 7881 (tcp, WebRTC-Fallback), 50000-50255 (udp, TURN relay range), 80 (tcp+udp, http), 443 (tcp+udp, https+wss) to the machine running this all

to start the element-call-webapp you first should build it with `docker compose build element-call`. After that, it can be `docker compose up -d`'d like the rest of the services in that compose file. This is because the webapp has no official docker image yet, so we build it. This is also th reason we have to adjust their `Dockerfile`.
Updating the service is therefore done via `git pull` + rebuild - which is exactly what the `update-element-call.sh` does

if you already have synapse running you don't need most of what is in the `matrix` folder.
The one interesting thing in there is the `well-known-nginx`-server. It is configured so that traffic to the well-known-urls for server and client are routed to that nginx, so we can easily change them. They are important so that the separate servers here find one another
- the livekit-config in `/.well-known/matrix/client` is required for the element-call website to use your livekit-instance
- the `/.well-known/element/element.json` is required so that the element x app uses your self-hosted element-call website

Something similar is done with the `livekit-jwt-service` in the `element-call/compose.yaml`. Only the two routes [this service](https://github.com/element-hq/lk-jwt-service) actually provides are routed to it. The rest is routed to livekit, under the same domain (to prevent CORS issues)

If i remember and find the time, i'll update this to a more complete guide
