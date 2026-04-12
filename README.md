# chatgpt reverse-proxy

## IMPORTANT

To adapt this project to your own domain, run the following command beforehand :

On windows, start with :

    docker run -v .:/data --rm -it debian:trixie-slim
    cd /data

Then as on linux :

    sed -i -e 's#FOO\.YOURDOMAIN\.EXT#yoursub.domain.ext#g' README.md
    sed -i -e 's#FOO\.YOURDOMAIN\.EXT#yoursub.domain.ext#g' chatgpt-proxy.conf

Where `yoursub.domain.ext` is your own subdomain for this chatgpt proxy.

## DNS

Point all names to your server's IP :

    # chatgpt.com
    FOO.YOURDOMAIN.EXT

    # cdn.oaistatic.com
    cdn.FOO.YOURDOMAIN.EXT

    # auth0.openai.com
    auth0.FOO.YOURDOMAIN.EXT

    # api.openai.com
    api.FOO.YOURDOMAIN.EXT

    # files.oaiusercontent.com
    files.FOO.YOURDOMAIN.EXT

    # ab.chatgpt.com (A/B testing)
    ab.FOO.YOURDOMAIN.EXT

    # telemetry SDK
    prodregistryv2.FOO.YOURDOMAIN.EXT

## Certbot

Generate wildcard certificate using dns challenge

    certbot certonly \
        --quiet \
        --non-interactive \
        --keep-until-expiring \
        --no-reuse-key \
        --dns-ovh \
        --dns-ovh-propagation-seconds 10 \
        --dns-ovh-credentials /etc/letsencrypt/dns*.ini \
        --domain "FOO.YOURDOMAIN.EXT" \
        --domain "*.FOO.YOURDOMAIN.EXT" \
        --post-hook "systemctl restart nginx"

**IMPORTANT**: this is for a domain managed by OVH, and a DNS challenge for ACME (letsencrypt) : adapt your command to your own DNS provider and Letsencrypt configuration.

## Nginx configuration

Install nginx

    sudo apt-get install --yes --no-install-recommends nginx

Place `chatgpt-proxy.conf` it in `/etc/nginx/sites-available/`

    ln -s -f /etc/nginx/sites-available/chatgpt-proxy.conf \
        /etc/nginx/sites-enabled/chatgpt-proxy.conf

    nginx -t

    systemctl restart nginx

Check logs at the same time

    tail -f /var/log/nginx/*.log

    journalctl -f

Then try to navigate the site

Going forward, every time you see a new failing domain in the Firefox network log, the fix is always the same pattern: add a sub_filter line + a new server block pointing to that domain.

## Cookies

Install the "Cookie Editor" extention in firefox

Allow it to work on private browsing (for testing)

### On your "normal" firefox

Export cookies from the computer where you're logged in

Install the Cookie-Editor browser extension (available on Chrome and Firefox). Then:

- Go to <https://chatgpt.com>
- Open Cookie-Editor
- Click Export → Export as JSON
- Save the JSON somewhere (file, pastebin, etc.)

### Modify the domain for the cookies

The exported cookies will have domain: chatgpt.com. Cookies must be scoped to domain you're currently on, so you need to find and edit the relevant cookies in the JSON before importing.

Modify the json file by opening it and

- searching for `chatgpt.com"`
- replacing with `FOO.YOURDOMAIN.EXT"`

And save it.

The session cookie stay valid for about 30 days.

### On your "proxy" firefox

Import cookies on the proxy computer

- Go to <https://FOO.YOURDOMAIN.EXT> (your proxy)
- Open Cookie-Editor
- Click Import, paste the JSON
- **Hard refresh the page (Ctrl+Shift+R)**

While you do that, also double-check your cookies in Firefox. Open DevTools → Storage tab → Cookies → <https://FOO.YOURDOMAIN.EXT>. You should see `__Secure-next-auth.session-token` in the list.

## What now ?

After that, your "proxied" access will work, until the cookie expires (30 days)

### Routine maintenance

Every ~30 days you'll need to re-export and re-import the session cookie from the source machine

If it suddenly breaks, that's almost always the first thing to check

### If new features break

OpenAI periodically adds new domains or endpoints — the fix is always the same: spot the failing domain in the Network tab, add a sub_filter line + server block, reload nginx
Keep the Network tab habit, it'll tell you everything

### Cloudflare may eventually get suspicious

If many users hammer the proxy, Cloudflare might start throwing 403s or JS challenges at your server's IP

Not much you can do about that from the nginx side alone

So do not share the proxy too widely !
