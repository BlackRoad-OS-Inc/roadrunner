# INTEGRATION — roadrunner

> How this fork connects to the rest of BlackRoad OS

## Node Assignment

| Property | Value |
|----------|-------|
| **Primary Node** | Octavia (.101) |
| **Fork Of** | rquickjs |
| **RoundTrip Agent** | RoadRunner Agent |
| **NLP Intents** | 'run script' / 'execute js' |
| **NATS Subject** | `blackroad.roadrunner.>` |
| **GuardRail Monitor** | `https://guard.blackroad.io/status/roadrunner` |

## Deployment

Deploy via blackroad-operator:

```bash
# From blackroad-operator
cd ~/blackroad-operator
./scripts/deploy/deploy-roadrunner.sh

# Or via fleet coordinator
./fleet-coordinator.sh deploy roadrunner

# Manual deploy to Octavia (.101)
ssh blackroad@$(echo "Octavia (.101)" | grep -oP '[0-9.]+' || echo "Octavia (.101)") \
  "cd /opt/blackroad/roadrunner && git pull && sudo systemctl restart roadrunner"
```

## Systemd Service

```ini
[Unit]
Description=BlackRoad roadrunner (rquickjs fork)
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=blackroad
WorkingDirectory=/opt/blackroad/roadrunner
ExecStart=/opt/blackroad/roadrunner/start.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## NATS Integration (CarPool)

```bash
# Subscribe to roadrunner events
nats sub "blackroad.roadrunner.>" --server nats://192.168.4.101:4222

# Publish status
nats pub "blackroad.roadrunner.status" '{"node":"Octavia (.101)","status":"running"}' \
  --server nats://192.168.4.101:4222
```

## RoundTrip Agent

The **RoadRunner Agent** manages this service via RoundTrip:

```bash
# Check agent status
curl -s https://roundtrip.blackroad.io/api/agents | jq '.[] | select(.name=="RoadRunner Agent")'

# Send command to agent
curl -X POST https://roundtrip.blackroad.io/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"agent":"RoadRunner Agent","message":"status","channel":"fleet"}'
```

## GuardRail Monitoring

Add to Uptime Kuma (Alice :3001):

| Check | URL/Command | Interval |
|-------|------------|----------|
| HTTP Health | `http://Octavia (.101):PORT/health` | 30s |
| Process | `systemctl is-active roadrunner` | 60s |
| NATS Heartbeat | `blackroad.roadrunner.heartbeat` | 60s |

## Memory System Integration

```bash
# Log actions
~/blackroad-operator/scripts/memory/memory-system.sh log deploy roadrunner "Deployed to Octavia (.101)"

# Add solutions to Codex
~/blackroad-operator/scripts/memory/memory-codex.sh add-solution "roadrunner" "How to restart" \
  "sudo systemctl restart roadrunner"

# Broadcast learnings
~/blackroad-operator/scripts/memory/memory-til-broadcast.sh broadcast "roadrunner" "Config change: ..."
```

## Related Components

| Component | Role | Connection |
|-----------|------|-----------|
| **TollBooth** (WireGuard) | VPN mesh | All traffic between nodes |
| **CarPool** (NATS) | Messaging | Event pub/sub on `blackroad.roadrunner.>` |
| **GuardRail** (Uptime Kuma) | Monitoring | Health checks every 30s |
| **RoadMem** (Mem0) | Memory | Persistent agent state |
| **OneWay** (Caddy) | TLS Edge | HTTPS termination on Gematria |
| **RearView** (Qdrant) | Vector Search | Semantic search over roadrunner logs |
| **BackRoad** (Portainer) | Containers | Docker management if containerized |
| **PitStop** (Pi-hole) | DNS | Internal `roadrunner.blackroad.local` resolution |
