# Kubernetes User Management Guide

This guide provides a step-by-step process for creating a new team member, granting them access to a Kubernetes cluster, and revoking their permissions when necessary. It covers generating certificates, configuring RBAC, and managing `kubeconfig` files.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Steps to Create a New Team Member](#steps-to-create-a-new-team-member)
   - [Generate a Private Key](#1-generate-a-private-key)
   - [Create a Certificate Signing Request (CSR)](#2-create-a-certificate-signing-request-csr)
   - [Sign the CSR with the Kubernetes CA](#3-sign-the-csr-with-the-kubernetes-ca)
   - [Create a Dedicated `kubeconfig` File](#4-create-a-dedicated-kubeconfig-file)
   - [Configure RBAC for the Team Member](#5-configure-rbac-for-the-team-member)
3. [Steps to Revoke a Team Member's Access](#steps-to-revoke-a-team-members-access)
   - [Remove RBAC Permissions](#1-remove-rbac-permissions)
   - [Revoke the Certificate (Optional)](#2-revoke-the-certificate-optional)
   - [Remove the `kubeconfig` File](#3-remove-the-kubeconfig-file)
4. [Best Practices](#best-practices)
5. [Conclusion](#conclusion)

---

## Prerequisites
- Access to a Kubernetes cluster.
- `kubectl` installed and configured.
- OpenSSL installed.
- Access to the Kubernetes CA certificate and key (typically located in `/etc/kubernetes/pki/` on the master node).

---

## Steps to Create a New Team Member

### 1. Generate a Private Key
Generate a private key for the new team member (e.g., `team-member`):

```bash
openssl genpkey -algorithm RSA -out team-member.key
```

This creates a private key file named `team-member.key`.

---

### 2. Create a Certificate Signing Request (CSR)
Create a CSR for the team member. The CSR includes the user's details and is used to request a certificate from the Kubernetes CA.

```bash
openssl req -new -key team-member.key -out team-member.csr -subj "/CN=team-member/O=team-group"
```

- `CN=team-member`: Sets the Common Name (username) to `team-member`.
- `O=team-group`: Sets the Organization (group) to `team-group`.

---

### 3. Sign the CSR with the Kubernetes CA
Sign the CSR using the Kubernetes CA certificate and key:

```bash
openssl x509 -req -in team-member.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out team-member.crt -days 365
```

- `-CA`: Path to the Kubernetes CA certificate.
- `-CAkey`: Path to the Kubernetes CA private key.
- `-CAcreateserial`: Creates a serial number file if it doesn't exist.
- `-days 365`: Sets the certificate validity period to 365 days.

---

### 4. Create a Dedicated `kubeconfig` File
Create a dedicated `kubeconfig` file for the team member. This file will include the cluster details, user credentials, and context.

1. **Set the cluster details**:
   ```bash
   kubectl config set-cluster kubernetes \
     --kubeconfig=team-member-kubeconfig \
     --server=https://<CLUSTER-IP>:6443 \
     --certificate-authority=/etc/kubernetes/pki/ca.crt
   ```

   Replace `<CLUSTER-IP>` with the IP address or hostname of your Kubernetes API server.

2. **Set the user credentials**:
   ```bash
   kubectl config set-credentials team-member \
     --kubeconfig=team-member-kubeconfig \
     --client-certificate=team-member.crt \
     --client-key=team-member.key
   ```

3. **Set the context**:
   ```bash
   kubectl config set-context team-member-context \
     --kubeconfig=team-member-kubeconfig \
     --cluster=kubernetes \
     --user=team-member
   ```

4. **Set the current context**:
   ```bash
   kubectl config use-context team-member-context \
     --kubeconfig=team-member-kubeconfig
   ```

---

### 5. Configure RBAC for the Team Member
Create a `Role` or `ClusterRole` and a `RoleBinding` or `ClusterRoleBinding` to grant the team member the necessary permissions.

#### Example: Read-Only Access to Pods
1. **Create a Role**:
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: default
     name: pod-reader
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "watch", "list"]
   ```

   Apply the Role:
   ```bash
   kubectl apply -f role.yaml
   ```

2. **Create a RoleBinding**:
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: read-pods
     namespace: default
   subjects:
   - kind: User
     name: team-member
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: Role
     name: pod-reader
     apiGroup: rbac.authorization.k8s.io
   ```

   Apply the RoleBinding:
   ```bash
   kubectl apply -f rolebinding.yaml
   ```

---

## Steps to Revoke a Team Member's Access

### 1. Remove RBAC Permissions
Delete the `RoleBinding` or `ClusterRoleBinding` associated with the team member.

#### Example: Delete a RoleBinding
```bash
kubectl delete rolebinding read-pods -n default
```

#### Example: Delete a ClusterRoleBinding
```bash
kubectl delete clusterrolebinding <clusterrolebinding-name>
```

---

### 2. Revoke the Certificate (Optional)
To revoke the team member's certificate, update the Kubernetes CA or use a Certificate Revocation List (CRL).

#### Option 1: Regenerate the CA Certificate
1. Regenerate the CA certificate and key:
   ```bash
   openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt
   ```

2. Distribute the new CA certificate to all cluster nodes and update the `kubeconfig` files for other users.

#### Option 2: Use a Certificate Revocation List (CRL)
1. Generate a CRL:
   ```bash
   openssl ca -gencrl -keyfile ca.key -cert ca.crt -out crl.pem
   ```

2. Update the Kubernetes API server to use the CRL:
   Add the `--client-ca-file` and `--client-crl-file` flags to the API server configuration:
   ```yaml
   - --client-ca-file=/etc/kubernetes/pki/ca.crt
   - --client-crl-file=/etc/kubernetes/pki/crl.pem
   ```

3. Restart the API server to apply the changes.

---

### 3. Remove the `kubeconfig` File
Communicate with the team member to ensure they delete their `kubeconfig` file (`team-member-kubeconfig`) from their machine.

---

## Best Practices
1. **Least Privilege**: Grant only the permissions the team member needs.
2. **Secure Sharing**: Encrypt the `kubeconfig` file before sharing it.
3. **Audit Regularly**: Periodically review RBAC permissions and `kubeconfig` files.
4. **Short-Lived Certificates**: Issue certificates with short expiration times.
5. **Monitor Activity**: Use Kubernetes audit logs to monitor user activity.

---

## Conclusion
This guide provides a comprehensive workflow for creating a new team member, granting them access to a Kubernetes cluster, and revoking their permissions when necessary. By following these steps, you can securely manage access to your cluster while maintaining control over user permissions.

For further reading, refer to the official Kubernetes documentation:
- [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
