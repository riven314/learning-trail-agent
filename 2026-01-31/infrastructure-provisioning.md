# Infrastructure Provisioning & Orchestration

Context: patterns and tools for automated infrastructure provisioning, from simple cloud-init scripts to full orchestration platforms.

## Cloud-Init Pattern

Auto-runs scripts on VM first boot. No manual SSH needed.

```yaml
#cloud-config
write_files:
  - path: /etc/app/config.env
    content: |
      SERVICE_ID=node-3
      REDIS_URL=redis://10.0.1.5:6379

runcmd:
  - systemctl start myapp
```

## Golden Image Pattern

```
Slow: Provision VM → Install deps → Download code → Configure → Start
Fast: Provision VM (pre-baked) → Inject config via cloud-init → Start
      (60 seconds vs 5+ minutes)
```

Pre-bake dependencies and code into image, inject only environment-specific config at runtime.

## Orchestration Spectrum

```
Simple ←─────────────────────────────────────────→ Complex

Direct API    Terraform    Ansible    Nomad    Kubernetes
+ Systemd     + Cloud-Init
```

**Rule**: use the simplest tool that meets your needs. K8s is overkill for <10 services.

## Cost Optimization

### Where costs come from

1. **Compute**: number × size of instances
2. **Storage**: disk size × IOPS tier
3. **Network**: egress traffic (internal often free)
4. **Redundancy**: 3× replication = 3× storage cost

### Optimization strategies

- Right-size instances (don't over-provision)
- Reserved instances for stable workloads (~30% discount)
- Spot instances for stateless, fault-tolerant workloads (~70% discount)
- Start minimal, scale when needed

#SYSTEM-DESIGN #INFRASTRUCTURE #DEVOPS #CLOUD
