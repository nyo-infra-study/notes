# Security and Identity

AWS operates on a Shared Responsibility Model: AWS is responsible for security *OF* the cloud (infrastructure, hardware), and the customer is responsible for security *IN* the cloud (data, OS, firewall config).

## AWS IAM (Identity and Access Management)
Securely manage access to AWS services and resources.
*   **Users**: Individuals or applications needing access.
*   **Groups**: Collections of users.
*   **Roles**: Assumable identities with specific permissions (often used by EC2 instances to access S3 seamlessly without hardcoding credentials).
*   **Policies**: JSON documents attached to identities defining exactly what they can or cannot do (Allow/Deny, Actions, Resources).

## AWS KMS (Key Management Service)
Managed service to easily create and control the encryption keys used to encrypt your data.
*   Integrated with most AWS services (S3, EBS, RDS).

## AWS Shield and AWS WAF
*   **AWS Shield**: Managed Distributed Denial of Service (DDoS) protection service. Shield Standard is enabled automatically for all AWS customers at no extra cost.
*   **AWS WAF (Web Application Firewall)**: Helps protect web apps from common web exploits (e.g., SQL injection, Cross-Site Scripting) by defining customizable web security rules.


---

### ➡️ Next: [Management and Governance](./07-management-governance.md)
