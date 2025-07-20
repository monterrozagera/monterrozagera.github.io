---
title: High-Availability Active Directory Federation Services
permalink: /ad-fs.html
---

# High-Availability Active Directory Federation Services (AD FS) Infrastructure

### Scenario Overview
A multinational organization needs a secure SSO solution using Active Directory Federation Services (AD FS) to enable seamless access to cloud and on-premises applications. The infrastructure must be highly available, support external access, and include disaster recovery mechanisms. The solution should span multiple geographic locations and comply with enterprise security standards.

---

### Prerequisites
1. **Environment**:
   - Multiple Windows Server 2022 instances (minimum 4 for AD FS servers, 2 for proxies, and 2 for domain controllers).
   - Active Directory Domain Services (AD DS) already configured with at least two domain controllers for redundancy.
   - SQL Server instance (e.g., SQL Server 2019) for AD FS configuration database (optional for high availability).
   - Valid SSL certificates from a trusted Certificate Authority (CA) for AD FS and proxies.
   - Network infrastructure with DNS configured and firewall rules allowing necessary ports (e.g., 443 for HTTPS, 49443 for certificate authentication).
   - Public IP or DNS name for external access (e.g., `fs.contoso.com`).

2. **Skills/Knowledge**:
   - Familiarity with Active Directory, DNS, and Group Policy.
   - PowerShell scripting for automation and configuration.
   - Understanding of load balancing (Windows NLB or external hardware/software load balancer).
   - Knowledge of Azure MFA or third-party MFA solutions (optional for MFA integration).

3. **Tools**:
   - Windows Server 2022 ISO or Azure VMs.
   - PowerShell 7.x for scripting.
   - SQL Server Management Studio (if using SQL Server).
   - Network Load Balancer or external load balancer (e.g., F5, Azure Load Balancer).

---

### Step-by-Step Implementation

#### Step 1: Plan the AD FS Infrastructure
1. **Define Requirements**:
   - **Topology**: Use a farm of AD FS servers for high availability (at least two servers per site for redundancy).
   - **Geographic Distribution**: Deploy AD FS servers in two or more data centers (e.g., US and Europe).
   - **External Access**: Use AD FS proxy servers (Web Application Proxy) in the DMZ for external users.
   - **Database**: Use SQL Server for the AD FS configuration database to enable high availability (Windows Internal Database (WID) is not suitable for multi-site HA).
   - **MFA**: Integrate Azure MFA or a third-party solution for enhanced security.
   - **Disaster Recovery**: Plan for failover to a secondary site using SQL Server Always On and AD FS configuration replication.

2. **Naming and Networking**:
   - Choose a federation service name (e.g., `fs.contoso.com`).
   - Ensure DNS records resolve internally and externally to the load balancer VIP (Virtual IP).
   - Configure firewall rules to allow HTTPS (443) and certificate authentication (49443) traffic.

3. **Sample Architecture**:
   - **Primary Site (US)**:
     - 2 AD FS servers (ADFSSRV1, ADFSSRV2).
     - 1 SQL Server instance for AD FS configuration database.
     - 2 Web Application Proxy servers (WAP1, WAP2) in the DMZ.
     - 2 Domain Controllers (DC1, DC2).
   - **Secondary Site (Europe)**:
     - 2 AD FS servers (ADFSSRV3, ADFSSRV4).
     - 1 SQL Server instance (part of Always On Availability Group).
     - 2 Web Application Proxy servers (WAP3, WAP4) in the DMZ.
     - 2 Domain Controllers (DC3, DC4).
   - Load balancer (e.g., Azure Load Balancer or Windows NLB) for both AD FS and WAP servers.

---

#### Step 2: Set Up the Active Directory Environment
1. **Verify Active Directory**:
   - Ensure AD DS is healthy with at least two domain controllers per site for redundancy.
   - Run `dcd Immediate Response: dcdiag` to check AD health.
   - Create an AD FS service account (e.g., `svc_adfs`) with a strong password.

2. **Configure DNS**:
   - Create an A record for `fs.contoso.com` pointing to the load balancer VIP.
   - Create CNAME records for external access if needed.

---

#### Step 3: Install and Configure AD FS Servers
1. **Install AD FS Role**:
   - On each AD FS server (ADFSSRV1, ADFSSRV2, ADFSSRV3, ADFSSRV4), install the AD FS role:
     ```powershell
     Install-WindowsFeature ADFS-Federation -IncludeManagementTools
     ```

2. **Configure the Primary AD FS Server (ADFSSRV1)**:
   - Use PowerShell to configure the AD FS farm with a SQL Server database:
     ```powershell
     $cert = Get-PfxCertificate -FilePath "C:\Certs\adfs-cert.pfx"
     Install-AdfsFarm -CertificateThumbprint $cert.Thumbprint `
                     -FederationServiceName "fs.contoso.com" `
                     -ServiceAccountCredential (Get-Credential) `
                     -SQLConnectionString "Data Source=SQLSRV1;Initial Catalog=ADFSConfiguration;Integrated Security=True"
     ```
   - Provide the AD FS service account credentials when prompted.
   - Ensure the SSL certificate is installed in the Local Machine store.

3. **Join Additional AD FS Servers to the Farm**:
   - On ADFSSRV2, ADFSSRV3, and ADFSSRV4, join the existing farm:
     ```powershell
     $cert = Get-PfxCertificate -FilePath "C:\Certs\adfs-cert.pfx"
     Add-AdfsFarmNode -CertificateThumbprint $cert.Thumbprint `
                      -SQLConnectionString "Data Source=SQLSRV1;Initial Catalog=ADFSConfiguration;Integrated Security=True"
     ```

4. **Verify AD FS Installation**:
   - Access the AD FS metadata endpoint: `https://fs.contoso.com/federationmetadata/2007-06/federationmetadata.xml`.
   - Check the AD FS Management Console to confirm all servers are listed in the farm.

---

#### Step 4: Configure Load Balancing
1. **Set Up Windows Network Load Balancing (NLB)**:
   - Install NLB on ADFSSRV1 and ADFSSRV2:
     ```powershell
     Install-WindowsFeature NLB -IncludeManagementTools
     ```
   - Create an NLB cluster:
     ```powershell
     New-NlbCluster -InterfaceName "Ethernet" -ClusterPrimaryIP "192.168.1.100" -ClusterName "ADFSCluster"
     Add-NlbClusterNode -InterfaceName "Ethernet" -NewNodeName "ADFSSRV2" -NewNodeAddress "192.168.1.102"
     ```
   - Configure port rules to balance traffic on ports 443 and 49443.

2. **Alternative: Use an External Load Balancer**:
   - Configure an external load balancer (e.g., Azure Load Balancer or F5) with health probes for HTTPS (port 443).
   - Add ADFSSRV1 and ADFSSRV2 as backend pool members.

3. **Repeat for Secondary Site**:
   - Configure NLB or an external load balancer for ADFSSRV3 and ADFSSRV4 in the secondary site.

---

#### Step 5: Set Up Web Application Proxy (WAP) for External Access
1. **Install WAP Role**:
   - On WAP1 and WAP2 (and WAP3, WAP4 for the secondary site), install the Web Application Proxy role:
     ```powershell
     Install-WindowsFeature Web-Application-Proxy -IncludeManagementTools
     ```

2. **Configure WAP**:
   - Configure each WAP server to connect to the AD FS farm:
     ```powershell
     $cert = Get-PfxCertificate -FilePath "C:\Certs\adfs-cert.pfx"
     Install-WebApplicationProxy -FederationServiceName "fs.contoso.com" `
                                 -CertificateThumbprint $cert.Thumbprint `
                                 -FederationServiceTrustCredential (Get-Credential)
     ```
   - Provide the AD FS service account credentials.

3. **Configure Load Balancing for WAP**:
   - Set up NLB or an external load balancer for WAP servers, similar to the AD FS servers, using ports 443 and 49443.

4. **Update DNS**:
   - Ensure the external DNS record for `fs.contoso.com` points to the WAP load balancer VIP.

---

#### Step 6: Configure Multi-Factor Authentication (MFA)
1. **Integrate Azure MFA**:
   - Register the AD FS farm with Azure AD:
     ```powershell
     Register-AdfsAzureMfa -TenantId "your-tenant-id" -ClientId "your-client-id" -ClientSecret "your-client-secret"
     ```
   - Enable MFA in AD FS:
     ```powershell
     Set-AdfsGlobalAuthenticationPolicy -AdditionalAuthenticationProvider AzureMfaAuthentication
     ```

2. **Alternative: Third-Party MFA (e.g., Duo)**:
   - Install the Duo Authentication Proxy on a separate server.
   - Configure AD FS to use Duo as an additional authentication provider:
     ```powershell
     Set-AdfsAdditionalAuthenticationProvider -Name "DuoAuthentication" -ConfigurationData @{ DuoApiHostname = "api-xxxx.duosecurity.com"; IntegrationKey = "your-ikey"; SecretKey = "your-skey"; ApiToken = "your-api-token" }
     ```

3. **Test MFA**:
   - Access `https://fs.contoso.com/adfs/ls` from an external client.
   - Verify that users are prompted for MFA (e.g., Azure MFA push notification or Duo app).

---

#### Step 7: Set Up Disaster Recovery
1. **Configure SQL Server Always On Availability Groups**:
   - On the primary SQL Server (SQLSRV1), create an Availability Group for the ADFSConfiguration database.
   - Add the secondary SQL Server (SQLSRV2) as a replica:
     ```sql
     CREATE AVAILABILITY GROUP ADFSAG
     WITH (AUTOMATED_BACKUP_PREFERENCE = PRIMARY)
     FOR DATABASE ADFSConfiguration
     REPLICA ON 'SQLSRV1' WITH (ENDPOINT_URL = 'TCP://SQLSRV1.contoso.com:5022', AVAILABILITY_MODE = SYNCHRONOUS_COMMIT)
     REPLICA ON 'SQLSRV2' WITH (ENDPOINT_URL = 'TCP://SQLSRV2.contoso.com:5022', AVAILABILITY_MODE = SYNCHRONOUS_COMMIT);
     ```
   - Join the secondary SQL Server to the Availability Group.

2. **Configure AD FS for Failover**:
   - Update the AD FS farm to use the Availability Group listener:
     ```powershell
     Set-AdfsProperties -SQLConnectionString "Data Source=ADFSAG-Listener;Initial Catalog=ADFSConfiguration;Integrated Security=True"
     ```

3. **Test Failover**:
   - Simulate a failure on the primary site (e.g., stop SQLSRV1).
   - Verify that AD FS continues to function using the secondary site’s SQL Server and AD FS servers.

---

#### Step 8: Secure the Infrastructure
1. **Certificate Management**:
   - Ensure all AD FS and WAP servers use the same SSL certificate for `fs.contoso.com`.
   - Configure certificate auto-renewal using a script or certificate management tool.

2. **Enable TLS 1.3**:
   - Update registry settings to enable TLS 1.3 on all servers:
     ```powershell
     Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\HTTP\Parameters\SslBinding" -Name "EnabledProtocols" -Value 0x00000800
     ```

3. **Configure Conditional Access**:
   - Use Azure AD Conditional Access policies to restrict access based on location, device compliance, or user risk level.
   - Example policy: Require MFA for external access only.

4. **Harden Servers**:
   - Apply CIS benchmarks using Group Policy:
     ```powershell
     Import-GPO -Path "C:\CIS\GPO\WinServer2022_CIS.gpo" -TargetName "ADFS_Servers"
     ```
   - Restrict RDP access and enforce strong passwords.

---

#### Step 9: Monitor and Test the Deployment
1. **Monitor AD FS**:
   - Use Event Viewer to monitor AD FS logs (Event ID 100 for successful logins, 403 for MFA failures).
   - Configure Azure Monitor or Azure Sentinel to collect and analyze logs:
     ```powershell
     Install-Module -Name Az.Monitor
     Connect-AzAccount
     New-AzDiagnosticSetting -Name "ADFSDiagnostic" -ResourceId "/subscriptions/your-sub-id/resourceGroups/your-rg/providers/Microsoft.Adfs/adfs/farm" -LogAnalyticsWorkspaceId "your-workspace-id"
     ```

2. **Test the Deployment**:
   - Test internal access: `https://fs.contoso.com/adfs/ls`.
   - Test external access via WAP: `https://fs.contoso.com/adfs/ls` from an external client.
   - Simulate failover by stopping primary AD FS servers and verifying access via secondary site.
   - Validate MFA functionality with test accounts.

---

#### Step 10: Document and Automate
1. **Document the Setup**:
   - Create runbooks detailing server roles, DNS settings, load balancer configurations, and SQL Server setup.
   - Include recovery procedures for failover and certificate renewal.

2. **Automate Maintenance**:
   - Create a PowerShell script to check AD FS health:
     ```powershell
     $health = Test-AdfsFarmHealth
     if ($health.Status -ne "Healthy") {
         Send-MailMessage -To "admin@contoso.com" -From "alerts@contoso.com" -Subject "AD FS Health Alert" -Body $health.Details
     }
     ```
   - Schedule the script using Task Scheduler.

---

### Key Considerations
- **Scalability**: Add more AD FS or WAP servers as needed to handle increased load.
- **Security**: Regularly update servers with the latest patches and monitor for vulnerabilities using tools like Microsoft Defender for Cloud.
- **Backup**: Back up the SQL Server database and AD FS certificates regularly.
- **Compliance**: Ensure configurations align with standards like NIST 800-53 or GDPR, if applicable.

---

### Testing and Validation
- Verify SSO with a test application (e.g., Office 365 or a custom SAML app).
- Check load balancer failover by stopping one AD FS or WAP server.
- Simulate a disaster recovery scenario by failing over to the secondary site.
- Audit logs to ensure MFA and conditional access policies are enforced.
