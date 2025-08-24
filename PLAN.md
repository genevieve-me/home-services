# Home Services Improvement Plan

This plan addresses security, operational, and configuration issues identified during the repository review.

### Container Resource Limits
**Issue**: No resource constraints on containers, potential for resource contention.
**Action Items**:
- Add memory limits to prevent OOM scenarios
- Set CPU limits to prevent single container monopolizing system
- Configure appropriate restart policies
- Add health checks where applicable

**Implementation**: Add to Podman Quadlet configurations

### Secure Podman Secrets Storage
**Issue**: Comment in code indicates secrets stored unencrypted on host by default.
**Action Items**:
- Research and implement encrypted secret storage options for Podman
- Consider using systemd credentials or external secret management
- Document the security implications and mitigation strategies
- Audit all secrets currently stored as Podman secrets

**Files to review**: `playbooks/pods.yml`, vault files

### Container Network Isolation
**Issue**: Container networking may allow unnecessary inter-container communication.
**Action Items**:
- Review container network policies
- Implement network segmentation where appropriate
- Restrict container-to-container communication to necessary services only
- Add network policy documentation

**Files to modify**: Container network configurations

### 8. Standardize Auto-Update Policies

Some containers are pinned to specific versions because I don't want to always update to `latest`.
However, I run the risk of forgetting to check for updates and staying on an old version for a long time.

We're setting up ntfy, maybe we can use it for update notifications?
