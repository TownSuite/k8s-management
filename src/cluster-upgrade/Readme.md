# MicroK8s Cluster Node Upgrade Script

A bash script to simplify rolling upgrades of self-hosted MicroK8s cluster nodes.

**Intended for small clusters and small sets of clusters that are not handled by managed Kubernetes installations.**

## Overview

This script automates the process of upgrading MicroK8s nodes in a Kubernetes cluster by:
1. Draining workloads from a node
2. Connecting via SSH to upgrade MicroK8s
3. Waiting for the node to be ready
4. Uncordoning the node to resume scheduling

## Prerequisites

### Required Tools
- **kubectl**: Kubernetes command-line tool
  - Download from: https://kubernetes.io/docs/tasks/tools/
  - Must be configured with access to your cluster

### Required Access
- SSH access to all nodes being upgraded
- Kubernetes admin permissions to drain/uncordon nodes
- Sudo privileges on target nodes (for snap refresh)

## Configuration

Create an `upgrade-node.env` file in the same directory as the script with the following variables:

```bash
# Kubectl command (can be full path or alias)
k="kubectl"

# SSH user for connecting to nodes
USER="your-ssh-user"

# MicroK8s version channel to upgrade to
MICROK8S_VERSION="1.34"

# Array of node names/IPs to upgrade
NODE_LIST=("node1" "node2" "node3")
```

### Configuration Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `k` | kubectl command or path | `kubectl` or `/usr/local/bin/kubectl` |
| `USER` | SSH username for node access | `ubuntu` or `admin` |
| `MICROK8S_VERSION` | Target MicroK8s version | `1.33` or `1.34` |
| `NODE_LIST` | Array of node hostnames/IPs | `("192.168.1.10" "192.168.1.11")` |

## Usage

```bash
./upgrade-node
```


### Example

```bash
# Make script executable
chmod +x upgrade-node

# Run the upgrade
./upgrade-node
```

## How It Works

For each node in `NODE_LIST`, the script:

1. **Drains the node**: Safely evicts pods using `kubectl drain --ignore-daemonsets`
2. **Shows pod status**: Displays current pod distribution across the cluster
3. **Upgrades MicroK8s**: SSH into the node and runs:
   - `snap refresh microk8s --channel=${MICROK8S_VERSION}-strict/stable`
   - Waits for MicroK8s to be ready
4. **Uncordons the node**: Re-enables pod scheduling with `kubectl uncordon`

## Important Considerations

### Before Running

- **Backup your cluster**: Always backup critical data before upgrades
- **Test in non-production**: Verify the script works in a test environment first
- **Check compatibility**: Ensure the target MicroK8s version is compatible with your workloads
- **Review the .env file**: The script sources this file directly - only use trusted configuration

### During Execution

- **One node at a time**: The script processes nodes sequentially to maintain cluster availability
- **DaemonSets remain**: The `--ignore-daemonsets` flag leaves DaemonSet pods running during drain
- **SSH connectivity**: Ensure SSH keys are configured for passwordless access
- **Monitor progress**: Watch for any errors in the output

### Error Handling

The script uses:
- `set -e`: Exits immediately if any command fails
- `set -u`: Exits if an undefined variable is used

If the script fails mid-upgrade, manually check:
- Node status: `kubectl get nodes`
- Pod distribution: `kubectl get pods -A -o wide`
- Uncordon stuck nodes: `kubectl uncordon <node-name>`

## Troubleshooting

### Script exits with "unbound variable"
Ensure all required variables are defined in `upgrade-node.env`

### Node won't drain
- Check for pods with PodDisruptionBudgets: `kubectl get pdb -A`
- Force drain (use with caution): `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force`

### SSH connection fails
- Verify SSH key access: `ssh $USER@<node> echo "Connected"`
- Check firewall rules on target nodes

### MicroK8s upgrade hangs
- SSH into the node manually and check: `microk8s status`
- Review snap logs: `snap logs microk8s`

## Example Workflow

```bash
# 1. Create environment file
cat > upgrade-node.env << 'EOF'
k="kubectl"
USER="ubuntu"
MICROK8S_VERSION="1.33"
NODE_LIST=("node1.local" "node2.local" "node3.local")
EOF

# 2. Verify cluster status
kubectl get nodes

# 3. Run upgrade
./upgrade-node.sh dummy

# 4. Verify all nodes upgraded
kubectl get nodes -o wide
```

## Security Notes

⚠️ **The script uses `source` to load the environment file, which executes any code in that file. Only use trusted configuration files.**

