# Proxy
A **proxy server** is an intermediary (software, or occasionally dedicated hardware) that sits between a client and a server. It receives a request from one side and relays it to the other, so the two never talk directly. Because every request passes through it, a proxy is a natural place to:

- **Filter** traffic (block domains, IPs or malicious payloads)
- **Log and monitor** requests
- **Transform** requests or responses (add or remove headers, rewrite URLs, compress, encrypt or decrypt)
- **Cache** responses
- **Route** requests to the right backend

Which side the proxy acts for decides its type: a **forward proxy** acts for **clients**, and a **reverse proxy** acts for **servers**.

```
Forward proxy:   [Client A] ─┐
                 [Client B] ─┼─► [Forward Proxy] ───► Internet ───► [any server]
                 [Client C] ─┘       (client side)

Reverse proxy:                                  ┌─► [Server 1]
                 [any client] ───► Internet ───► [Reverse Proxy] ─┼─► [Server 2]
                                                   (server side)  └─► [Server 3]
```

## Forward Proxy

A forward proxy (often just called a "proxy", "proxy server" or "web proxy") sits **in front of a group of client machines**. When those clients make requests to the internet, the proxy intercepts them and talks to the web servers **on the clients' behalf**. The destination server sees the proxy's IP, not the client's.

![forward-proxy](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-I/proxy/forward-proxy.png)

**How it works**
1. The client is configured to use the proxy, through browser or OS settings, PAC files, or a transparent network redirect.
2. For plain HTTP, the client sends the full URL to the proxy (`GET http://example.com/page`). The proxy fetches it and returns the response.
3. For HTTPS, the client sends `CONNECT example.com:443`. The proxy opens a TCP tunnel and passes encrypted bytes through without seeing the content. Corporate proxies that need to inspect HTTPS do **TLS interception**: they decrypt and re-encrypt traffic using a company root CA installed on employee devices.

**Use cases and advantages**
- **Access control and content filtering:** organizations and schools block certain sites or categories.
- **Bypassing geo-restrictions:** reach [geo-restricted](https://en.wikipedia.org/wiki/Geo-blocking) content by exiting from another region.
- **Anonymity and privacy:** hide the client's IP address from the destination.
- **Caching:** a shared cache (e.g. Squid) saves bandwidth when many users fetch the same content.
- **Egress control in backend systems:** services in a private network reach external APIs through a single egress proxy. That gives you a fixed outbound IP to allowlist, audit logs, and a way to block exfiltration.
- **Monitoring and compliance:** log or scan outbound traffic for malware or data leaks.

**Disadvantages**
- The proxy operator **can see and log** your traffic (all of it for HTTP, the destinations for HTTPS), so "anonymity" only means hidden *from the destination*.
- Setup, maintenance and configuration take effort and cost money.
- It is an extra hop, which adds latency and can become a bottleneck or single point of failure.
- TLS interception weakens end-to-end security and breaks certificate pinning.

**Variants**
- **Transparent proxy:** traffic is redirected at the network level, so clients need no configuration and often don't know the proxy exists. Common with ISPs, captive portals and corporate networks.
- **Anonymous / high-anonymity (elite) proxy:** strips identifying headers such as `X-Forwarded-For` and `Via`.
- **SOCKS proxy (SOCKS5):** works at the session/transport level, so it can relay any TCP or UDP traffic, not just HTTP. It does no inspection or caching.
- **VPN (related, not a proxy):** tunnels *all* of a device's traffic at the network layer (L3) and encrypts it. A proxy usually handles only certain applications or protocols.

**Examples:** Squid, Zscaler and Blue Coat (corporate), Tor (chained anonymizing proxies), Smokescreen (egress proxy).

## Reverse Proxy

A reverse proxy sits **in front of one or more web servers** and intercepts requests from clients. Clients think they are talking to the origin server. In fact they talk to the proxy, which forwards each request to a backend and returns the response. The backends stay hidden from clients.

> **The key difference:** a forward proxy sits in front of **clients** and makes sure no origin server ever talks directly to them. A reverse proxy sits in front of **origin servers** and makes sure no client ever talks directly to them.

![reverse-proxy](https://raw.githubusercontent.com/karanpratapsingh/portfolio/master/public/static/courses/system-design/chapter-I/proxy/reverse-proxy.png)

**How it works**
1. DNS for `api.example.com` resolves to the reverse proxy, not to the app servers.
2. The proxy terminates the client connection, usually decrypting TLS as well.
3. It picks a backend using routing rules (host, path, header) and a load-balancing algorithm, then forwards the request over a pooled, reused connection.
4. It may modify the request, for example by adding `X-Forwarded-For`, `X-Forwarded-Proto` or `X-Request-ID`.
5. It returns the backend's response, possibly compressing or caching it.

**Use cases and advantages**
- **Security:** hides backend IPs and topology, so only the proxy is exposed. It is a single choke point to add a **WAF**, DDoS absorption, IP allow and deny lists, and request size limits.
- **Load balancing:** spreads traffic across backends and removes unhealthy ones using health checks. See [Load Balancing](load_balancing.md).
- **SSL/TLS termination:** decrypts HTTPS at the edge, which offloads crypto from app servers and centralizes certificate management.
- **Caching:** serves static or cacheable responses without hitting the backend. See [Caching](caching.md) and [CDN](cdn.md).
- **Compression:** gzip or brotli responses to save bandwidth.
- **Connection management:** absorbs many slow client connections and multiplexes them onto a few persistent backend connections. It also buffers slow clients so app threads aren't tied up.
- **Routing, rewriting and versioning:** sends `/api/*` to one service and `/static/*` to another. Also handles canary or blue-green releases and A/B tests.
- **Rate limiting and authentication** at the edge (often in an [API Gateway](api_gateway.md)).
- **Protocol translation:** HTTP/2 or HTTP/3 toward clients and HTTP/1.1 toward backends, or HTTP/JSON to gRPC.
- **Scalability and flexibility:** add, remove or replace backends without clients noticing.

**Disadvantages**
- **More complexity:** another component to deploy, configure, monitor and secure.
- **Single point of failure** if there is only one. Running several for high availability adds more complexity (see below).
- **Extra latency** from the additional hop, though connection reuse and caching often offset it.
- **Potential bottleneck:** it must be sized to handle all traffic.
- **Lost client context:** backends see the proxy's IP unless the proxy forwards the original IP in `X-Forwarded-For` or the PROXY protocol.

**Examples:** Nginx, HAProxy, Envoy, Traefik, Caddy, Apache (`mod_proxy`), and cloud services such as AWS ALB, GCP Load Balancer and Cloudflare.

## TLS Handling at a Reverse Proxy
| Mode                 | What happens                                                               | Trade-off                                                    |
|----------------------|----------------------------------------------------------------------------|--------------------------------------------------------------|
| **TLS termination**  | The proxy decrypts traffic and sends plain HTTP to the backends.           | Fastest and enables L7 features, but traffic inside the network is unencrypted. |
| **TLS re-encryption (bridging)** | The proxy decrypts, inspects, then re-encrypts to the backends. | Supports L7 features and encrypts internal traffic (zero trust), at extra CPU cost. |
| **TLS passthrough**  | The proxy forwards encrypted bytes without decrypting (L4, often routed by SNI). | End-to-end encryption, but no L7 routing, caching or header changes. |

## Preserving Client Information
Once the connection goes through a proxy, backends see the **proxy's** IP. Common ways to keep the original details:
- `X-Forwarded-For: <client>, <proxy1>, <proxy2>` gives the original client IP and each proxy in the chain. **Only trust entries added by your own proxies**, because clients can forge this header.
- `X-Forwarded-Proto` and `X-Forwarded-Host` give the original scheme (http or https) and host.
- `Forwarded:` is the standardized (RFC 7239) form of the headers above.
- `Via:` lists the proxies the request passed through.
- **PROXY protocol** (HAProxy) adds the client IP and port at L4, for non-HTTP or passthrough traffic.

## High Availability for Proxies
Since every request goes through the proxy, it must not become a single point of failure:
- **Active-passive pair with a floating virtual IP:** keepalived/VRRP moves the IP to the standby if the active one fails.
- **Active-active behind DNS** (multiple A records) or behind an **L4 load balancer** in front of L7 proxies.
- **Anycast:** the same IP is advertised from many locations, and BGP routes users to the nearest healthy one. CDNs and Cloudflare work this way.
- **Managed cloud LBs** (ALB, NLB) are redundant across availability zones out of the box.
- Keep proxies **stateless**, or keep shared state such as rate-limit counters in Redis, so they scale horizontally.

## Forward vs Reverse Proxy
| Aspect               | Forward Proxy                                  | Reverse Proxy                                      |
|----------------------|------------------------------------------------|----------------------------------------------------|
| Acts on behalf of    | Clients                                        | Servers                                            |
| Hides                | Client identity from servers                   | Server identity and topology from clients          |
| Deployed by          | Client side (company, school, user)            | Server side (service owner)                        |
| Client awareness     | Client is usually configured to use it (unless transparent) | Client is unaware; it looks like the origin |
| Typical uses         | Filtering, anonymity, egress control, shared cache | Load balancing, TLS termination, caching, WAF, routing |
| Examples             | Squid, Zscaler, Tor, corporate web proxy       | Nginx, HAProxy, Envoy, Traefik, AWS ALB, Cloudflare |

## Load Balancer vs Reverse Proxy vs API Gateway
These overlap heavily, and one product (e.g. Nginx or Envoy) can play all three roles. The difference is mainly the **primary purpose**:

| Component         | Primary purpose                                                          | Typical features                                                    |
|-------------------|--------------------------------------------------------------------------|---------------------------------------------------------------------|
| **Load balancer** | Spread traffic across **multiple identical servers** for scale and availability. | Algorithms (round robin, least connections, hashing), health checks, L4 or L7. |
| **Reverse proxy** | Be the **front door** for one or more servers. Useful even with **a single backend**. | TLS termination, caching, compression, security, path routing, buffering. |
| **API gateway**   | Reverse proxy specialized for **APIs and microservices**.                | Authentication, rate limiting, quotas, request validation and transformation, API keys, aggregation, analytics. See [API Gateway](api_gateway.md). |

- A load balancer only makes sense with **multiple** servers. A reverse proxy is useful even with **one** (TLS, caching, security, buffering).
- An **L7 load balancer is effectively a reverse proxy** with load balancing as its main job.
- An **L4 load balancer** (e.g. AWS NLB, LVS) forwards TCP or UDP connections without reading HTTP. It is closer to a packet or connection forwarder than to a full reverse proxy. It is faster but can't route by URL or headers, cache, or modify requests.

## Proxies in Modern Architectures
- **Edge or CDN proxy:** a globally distributed reverse proxy that caches content near users. See [CDN](cdn.md).
- **Ingress controller (Kubernetes):** the reverse proxy that routes external traffic to services in the cluster (Nginx Ingress, Traefik, Envoy-based gateways).
- **Sidecar proxy / service mesh:** each service instance runs a local proxy (e.g. Envoy in Istio, or Linkerd) that handles service-to-service traffic. It provides mutual TLS, retries, timeouts, circuit breaking, load balancing and telemetry without changes to app code. See [Resilience Patterns](resilience_patterns.md) and [Service Discovery](service_discovery.md).
- **Database proxy:** connection pooling and routing for databases (PgBouncer, ProxySQL, RDS Proxy). It protects the DB from too many connections and can split reads and writes across replicas.
- **Egress gateway:** a forward proxy for outbound traffic from a cluster or VPC.

## Interview Tips
- When you draw a box between users and app servers, say what it does: **"an L7 reverse proxy / load balancer that terminates TLS, routes by path, and health-checks backends."**
- Always address the **SPOF** concern: run at least 2 instances behind a virtual IP, DNS or anycast, or use a managed LB.
- Mention **TLS termination vs passthrough** when security or compliance comes up.
- Know which layer you mean: **L4** works on IP and port (fast, content-blind) and **L7** works on HTTP (content-aware). See [OSI Model](osi_model.md).
- For microservices, mention an **API gateway** at the edge and a **service mesh** for east-west (service-to-service) traffic.

## Examples

Commonly used proxy technologies:

- [Nginx](https://www.nginx.com): web server and reverse proxy, load balancer, cache
- [HAProxy](http://www.haproxy.org): high-performance L4/L7 load balancer and proxy
- [Traefik](https://doc.traefik.io/traefik): cloud-native reverse proxy with automatic service discovery (Docker, Kubernetes)
- [Envoy](https://www.envoyproxy.io): L7 proxy built for service meshes (Istio) and API gateways
- [Caddy](https://caddyserver.com): reverse proxy with automatic HTTPS
- [Squid](http://www.squid-cache.org): caching forward proxy

# References
- https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/
- https://github.com/karanpratapsingh/system-design#proxy
- https://www.nginx.com/resources/glossary/reverse-proxy-vs-load-balancer/
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Forwarded-For
- https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy
