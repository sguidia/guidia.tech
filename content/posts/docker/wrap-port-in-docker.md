---
title: "How to Properly Wrap Your Ports in Docker"
date: 2025-07-05
menu:
  sidebar:
    name: How to Properly Wrap Your Ports in Docker
    identifier: wrap-port-in-docker
    parent: docker
    weight: 10
summary: "Your ports are not as protected as you think. Here's how to expose them securely and stop living dangerously."

---
So, you just set up Docker, launched a container, forwarded some ports—looks clean, right?  
Not so fast. There’s one big problem: your exposed ports are hanging out like a drunk dude at Mardi Gras.  
Anyone with a pulse (and nmap) can hit them.

**Amateur hour.**

## The Myth: “I’m Safe with Nginx!”

If you’re running nginx as a reverse proxy, you might think everything’s locked down. Example:

```bash
curl https://lnkd.in/vault/
```

That request gets proxied internally to `localhost:4002`, where your Vault (HashiCorp) container is running.

**Wrong.**

```bash
curl XXX.XXX.XXX.XXX:4002
```

💥 Boom. Direct access to the container, bypassing nginx and all its rules. Garbage setup.

## “I’ll Just Use iptables!” — Said the Fool

Yeah, you could mess around with iptables, but Docker dynamically creates rules,  
so you’ll be playing an endless game of whack-a-mole.

## The Real Fix: Bind to 127.0.0.1

Let’s take Gitea as an example. Here’s a properly configured `docker-compose.yml`:

```yaml
networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:latest
    networks:
      - gitea
    volumes:
      - gitea:/data
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2221:22"
```

### Why this works:

- The `127.0.0.1` binding exposes the ports only to the host machine  
- ✅ nginx can still access `127.0.0.1:3000`  
- ❌ But the outside world? They get nothing

```bash
curl XXX.XXX.XXX.XXX:3000
```

404 on your dreams.

## Bonus: Stop Using `docker-compose`

If you’re still running this:

```bash
docker-compose up -d
```

You’re basically using a fossilized binary. The modern, supported way is:

```bash
docker compose up -d
```

Subtle? Yes.  
Important? Also yes.

**Your move.**