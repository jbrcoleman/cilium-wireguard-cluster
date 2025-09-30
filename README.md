# cilium-wireguard-cluster
Following EKS blue print to that uses Cilium configured in CNI chaining mode with the VPC CNI and with Wireguard transparent encryption enabled on an Amazon EKS cluster.

Source: [Wireguard with Cilium Pattern](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/network/wireguard-with-cilium/)
# Transparent Encryption with Cilium and Wireguard

This pattern demonstrates how to configure Cilium in CNI chaining mode with AWS VPC CNI and enable Wireguard transparent encryption on an Amazon EKS cluster.

## Overview

This setup provides transparent encryption of pod-to-pod traffic using Wireguard, while leveraging AWS VPC CNI for IP address management. All network traffic between pods is automatically encrypted without requiring application-level changes.

## Prerequisites

- Terraform installed
- AWS CLI configured with appropriate credentials
- kubectl installed
- Linux Kernel version 5.10+ on EKS nodes
- See the [full prerequisites guide](https://aws-ia.github.io/terraform-aws-eks-blueprints/getting-started/#prerequisites)

## Key Components

### Infrastructure Files

- **`eks.tf`** - Contains the EKS cluster configuration and Cilium deployment
- **`wireguard-blog-demo.yaml`** - Sample application for demonstrating encrypted connectivity (optional)

### Architecture Highlights

- **CNI Chaining**: Cilium runs in chaining mode with AWS VPC CNI
- **Wireguard Encryption**: Provides transparent encryption at the network layer
- **No Application Changes**: Existing applications automatically benefit from encryption

## Deployment

Follow the standard Terraform deployment process:

```bash
terraform init
terraform plan
terraform apply
```

For detailed deployment steps, refer to the [getting started guide](https://aws-ia.github.io/terraform-aws-eks-blueprints/getting-started/#prerequisites).

## Validation

### 1. Deploy Example Application

Deploy the sample pods to test encrypted connectivity:

```bash
kubectl apply -f example.yaml
```

### 2. Verify Cilium Wireguard Status

Get the status from a Cilium pod:

```bash
kubectl exec -n kube-system ds/cilium -- cilium status | grep Encryption
```

Expected output should show:
```
Encryption: Wireguard [NodeEncryption: Disabled, cilium_wg0 (Pubkey: ..., Port: 51871, Peers: 1)]
```

**Note**: `NodeEncryption: Disabled` is expected since node-to-node encryption was not enabled in the Helm values.

### 3. Verify Traffic Encryption with Packet Capture

Open a shell in a Cilium container:

```bash
kubectl exec -it -n kube-system ds/cilium -- bash
```

Install tcpdump:

```bash
apt-get update && apt-get install -y tcpdump
```

Start packet capture on the Wireguard interface:

```bash
tcpdump -n -i cilium_wg0
```

You should see clear text in the output, which confirms the traffic is encrypted by Wireguard (the unencrypted payload is only visible inside the Wireguard interface).

Exit the container when done:

```bash
exit
```

### 4. Run Cilium Connectivity Tests

Deploy the official Cilium connectivity test suite:

```bash
kubectl create ns cilium-test
kubectl apply --server-side -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/v1.14.1/examples/kubernetes/connectivity-check/connectivity-check.yaml
```

View the logs of any connectivity test pod to check results:

```bash
kubectl logs -n cilium-test deployment/pod-to-a -f
```

You should see successful HTTP requests (200 responses) indicating proper connectivity through the encrypted network.

## Cleanup

Destroy resources in the correct order to avoid dependency issues:

```bash
# Remove add-ons first
terraform destroy -target="module.eks_blueprints_addons" -auto-approve

# Remove EKS cluster
terraform destroy -target="module.eks" -auto-approve

# Remove remaining resources
terraform destroy -auto-approve
```

For more details on cleanup, see the [destroy documentation](https://aws-ia.github.io/terraform-aws-eks-blueprints/getting-started/#destroy).

## Understanding the Encryption

- **Wireguard** operates at Layer 3 and encrypts all IP packets between nodes
- **Transparent** means applications don't need to be modified or aware of encryption
- **Performance** impact is minimal due to Wireguard's efficient implementation
- **Key Management** is handled automatically by Cilium

## Troubleshooting

If encryption is not showing as enabled:

1. Verify kernel version is 5.10+ on all nodes
2. Check Cilium Helm values include Wireguard configuration
3. Review Cilium agent logs: `kubectl logs -n kube-system ds/cilium`
4. Ensure proper security group rules allow Wireguard traffic (UDP port 51871)

## Additional Resources

- [Cilium Documentation](https://docs.cilium.io/)
- [Wireguard Official Site](https://www.wireguard.com/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [EKS Blueprints Documentation](https://aws-ia.github.io/terraform-aws-eks-blueprints/)