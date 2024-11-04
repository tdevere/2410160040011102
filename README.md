# Steps to Force IIS to Use Kerberos Authentication for Azure DevOps Server 2022

We recommend re-configuring Azure DevOps Server to use Kerberos authentication instead of NTLM, if you haven’t already. Azure DevOps Server has supported Kerberos for quite some time (see [Reconfigure Azure DevOps Server to use Kerberos instead of NTLM](https://devblogs.microsoft.com/devops/reconfigure-azure-devops-server-to-use-kerberos-instead-of-ntlm/))

To force IIS to use Kerberos authentication on a website hosting Azure DevOps Server 2022, you need to configure both your IIS server and Active Directory properly. Below are the detailed steps:

### 1. Ensure the Application Pool Identity is a Domain Account

- **Access IIS Manager**:
  - Open **Internet Information Services (IIS) Manager** on the server hosting Azure DevOps Server.
- **Locate the Application Pool**:
  - In the **Connections** pane, expand the server node and click on **Application Pools**.
  - Find the application pool used by your Azure DevOps Server website.
- **Change Identity**:
  - Right-click the application pool and select **Advanced Settings**.
  - Under the **Process Model** section, find the **Identity** property.
  - Change the **Identity** to a **Domain User Account** that has appropriate permissions.
    - Click on the **Identity** value, then select **Custom Account** and click **Set**.
    - Enter the domain account credentials and click **OK**.

### 2. Register the Service Principal Name (SPN) for the Domain Account

- **Open Command Prompt as Administrator**:
  - Run **Command Prompt** with administrative privileges.
- **Use the `setspn` Command**:
  - Register the SPN using the following command:

    ```bash
    setspn -S HTTP/hostname domain\username
    ```

    - Replace `hostname` with the fully qualified domain name (FQDN) or the NetBIOS name of the server.
    - Replace `domain\username` with the domain account used as the application pool identity.

  - **Example**:

    ```bash
    setspn -S HTTP/devopsserver.contoso.com CONTOSO\DevOpsAppPoolAccount
    ```

- **Verify SPN Registration**:

  ```bash
  setspn -L domain\username
  ```

### 3. Configure IIS Authentication Settings

- **Navigate to Your Website**:
  - In **IIS Manager**, expand the server node and select your Azure DevOps Server website.
- **Open Authentication Settings**:
  - Double-click on **Authentication** in the **IIS** section.
- **Enable Windows Authentication**:
  - Ensure **Windows Authentication** is **Enabled**.
- **Disable Anonymous Authentication**:
  - Ensure **Anonymous Authentication** is **Disabled**.
- **Configure Providers for Windows Authentication**:
  - Select **Windows Authentication** and click on **Providers** in the **Actions** pane.
  - **Remove All Existing Providers**:
    - Remove both **NTLM** and **Negotiate** if they are listed.
  - **Add "Negotiate:Kerberos"**:
    - Click **Add** and select **Negotiate:Kerberos** from the list.
    - If **"Negotiate:Kerberos"** is not listed, you can type it manually.
  - **Ensure "Negotiate:Kerberos" is the Only Provider**:
    - **Negotiate:Kerberos** should be the only provider in the list.

### 4. Set `useAppPoolCredentials` to `True`

- **Edit Configuration Settings**:
  - Still in IIS Manager, select your website.
- **Open Configuration Editor**:
  - Double-click on **Configuration Editor** under the **Management** section.
- **Navigate to the Correct Section**:
  - In the **Section** dropdown, navigate to:

    ```
    system.webServer/security/authentication/windowsAuthentication
    ```

- **Set `useAppPoolCredentials`**:
  - Find the **`useAppPoolCredentials`** attribute and set it to **`True`**.
- **Apply the Changes**:
  - Click **Apply** in the **Actions** pane.

### 5. Disable Kernel-mode Authentication

- **In the Same Configuration Editor**:
  - Locate the **`useKernelMode`** attribute.
- **Set `useKernelMode` to `False`**:
  - Change **`useKernelMode`** to **`False`**.
- **Apply the Changes**:
  - Click **Apply**.

### 6. Restart IIS

- **Restart IIS Services**:
  - Open **Command Prompt** as an administrator.
  - Execute the following command:

    ```bash
    iisreset
    ```

### 7. Configure Delegation in Active Directory (If Necessary)

- **Open Active Directory Users and Computers**:
  - On your domain controller, open **Active Directory Users and Computers**.
- **Find the Application Pool Account**:
  - Locate the domain account used as the application pool identity.
- **Set Delegation Options**:
  - Right-click the account and select **Properties**.
  - Navigate to the **Delegation** tab.
  - Select **Trust this user for delegation to specified services only**.
  - Choose **Use any authentication protocol**.
  - Click **Add**, then select the services (SPNs) that the account can delegate to.
- **Apply and Close**:
  - Click **OK** to save changes.

### 8. Verify Kerberos Authentication

- **Use a Browser to Access the Website**:
  - From a client machine, access the Azure DevOps Server website using its FQDN.
- **Check Authentication Method**:
  - Use tools like **KerbTray** or **Klist** to verify that Kerberos tickets are being used.
  - Alternatively, enable **Kerberos logging** on the client machine and check the **Event Viewer** for Kerberos events.

# Considerations for Kerberos Authentication with a Load Balancer
Key Points
* `Common Service Account:` Use a single domain account as the application pool identity on all backend servers.
* `SPN Registration:` Register the SPN for the load-balanced DNS name to the common service account.
* `Consistent Configuration:` Ensure IIS and application settings are identical across all servers.
* `Load Balancer Configuration:` Configure the load balancer to support Kerberos authentication.
Client Access: Clients should access the service using the load balancer's DNS name.

## Configure the Load Balancer
Session Persistence (Affinity/Sticky Sessions):
* Configure the load balancer to use Client IP or Cookie-Based session persistence to maintain session state if necessary.
### Support for Kerberos Delegation:
* Ensure the load balancer is configured to pass through Kerberos authentication headers without modification.
* Avoid SSL Offloading at the load balancer unless properly configured, as it can interfere with Kerberos authentication.
### Load Balancer as Transparent Proxy:
* The load balancer should act as a transparent proxy, forwarding client requests to backend servers without altering authentication tokens.

## Client Access Configuration
* Access via Load Balancer DNS Name:
* Clients must access the Azure DevOps Server using the load balancer's DNS name (https://devops.contoso.com).

## DNS Resolution:
* Ensure devops.contoso.com resolves to the load balancer's IP address.
Browser Configuration:
* Clients may need to add the load balancer's DNS name to the Local Intranet zone in their browser settings to enable Kerberos authentication.

## Configure Delegation in Active Directory (If Necessary)
Delegation Settings:
* On the domain account CONTOSO\DevOpsAppPoolAccount, configure delegation if your application needs to access resources on behalf of the user (double-hop scenario).
* In Active Directory Users and Computers:
  * Right-click CONTOSO\DevOpsAppPoolAccount and select Properties.
  * Go to the Delegation tab.
  * Choose Trust this user for delegation to specified services only.
  * Select Use any authentication protocol.
  * Add the SPNs of services that the application needs to access.

## Restart IIS on All Backend Servers
* Do this on each backend server to apply the changes.

# Further Considerations for Kerberos Authentication with a Load Balancer
## Load Balancer Configuration
* SPN Registration for Load-Balanced Name:
  * Register the SPN using the load balancer's DNS name to the common service account.
  ```bash
  setspn -S HTTP/devops.contoso.com CONTOSO\DevOpsAppPoolAccount
  ```

## SSL Offloading:
If SSL offloading is used, ensure that the load balancer re-encrypts the traffic to the backend servers or configure Kerberos Constrained Delegation (KCD) on the load balancer.
* Client IP Preservation:
  * Configure the load balancer to preserve the client’s IP address if session persistence is based on client IP.

# Client Access Configuration

* DNS and Hostname Consistency:
  * Clients must access the service using the exact DNS name for which the SPN is registered.
  * Ensure that all DNS records (A or CNAME) for devops.contoso.com point to the load balancer.

* Browser Configuration: 
For browsers like Chrome and Firefox, additional settings might be required to enable
  * Kerberos authentication.
    * Chrome:
    ```
    --auth-server-whitelist="devops.contoso.com"
    ```
  * Firefox:
    * Navigate to about:config and set:
    ```bash
    network.negotiate-auth.trusted-uris = devops.contoso.com
    ```
---

# Impact on Build Agents

Determine the Build Agent Authentication method. If the agent uses a PAT token for authentication, there is no expected impact as the agent already uses token based authenitcation. 

However, for agents using Windows Authentication, those agents will need to particpate in the Active Directory, as clients of the network in order to satify kerberos requirements. 

## Impact on Linux Build Agents
### Authentication Method
Personal Access Tokens (PATs):
* Default for Linux Agents: Linux build agents commonly use PATs for authentication to Azure DevOps Server.
* No Direct Impact: Since PATs are not affected by the server's authentication method (NTLM or Kerberos), the switch to Kerberos should not directly impact Linux agents using PATs.

### Network and Connectivity
Firewall and Ports:

* No Change in Requirements: The network requirements for Linux agents remain the same. Ensure that the agents can reach the Azure DevOps Server over the necessary ports (usually HTTPS over TCP port 443).

DNS Resolution:

* Consistent Access: Linux agents must resolve the Azure DevOps Server's hostname correctly. 
* Ensure that DNS settings are accurate and that the agents can reach the server using the Fully Qualified Domain Name (FQDN).
### Server URL and Certificates
Server URL:
* Use FQDN: Ensure that the agents are configured to connect to the server using its FQDN, which matches the SPN registered for Kerberos.
* Consistency: Inconsistent URLs might lead to SSL certificate validation issues.

SSL Certificates:
* Trusted Root CA: The SSL certificate used by Azure DevOps Server should be trusted by the Linux agents.
* Certificate Errors: If the agents encounter SSL certificate errors, you may need to install the server's SSL certificate on the agents or use certificates signed by a trusted Certificate Authority (CA).

### Impact on Windows Build Agents
# Authentication Method Change
* Kerberos Requirement: Build agents will now need to authenticate to the Azure DevOps Server using Kerberos instead of NTLM.
* Domain Membership: Agents must be domain-joined and able to obtain Kerberos tickets from the Key Distribution Center (KDC) in Active Directory.

# Service Account Configuration
* Agent Service Account: The account under which the build agent service runs must have the necessary permissions and be properly configured for Kerberos authentication.
* SPN Registration: In most cases, build agents do not require SPNs unless they are acting as a server for other services. However, ensure that no conflicting SPNs exist for the agent service account.
# Network and DNS Considerations
* DNS Resolution: Agents must resolve the Azure DevOps Server's fully qualified domain name (FQDN) correctly to obtain the proper Kerberos tickets.
* Firewall Rules: Ports required for Kerberos (TCP/88 and UDP/88) must be open between the build agents and the domain controllers.

# Double-Hop Authentication Scenarios
*  Potential Issues with Double-Hop: If build agents need to access resources on behalf of the user (e.g., accessing file shares or databases), additional Kerberos delegation configuration may be required.
* Constrained Delegation: Configure constrained delegation for the build agent service account if double-hop authentication is necessary.


## References

- [Microsoft Docs - Windows Authentication](https://docs.microsoft.com/en-us/iis/configuration/system.webServer/security/authentication/windowsAuthentication/)
- [Understanding Kerberos Double Hop](https://techcommunity.microsoft.com/t5/ask-the-directory-services-team/understanding-kerberos-double-hop/ba-p/395463)
- [Microsoft Tech Community - SPN configurations for Kerberos Authentication – A quick reference](https://techcommunity.microsoft.com/t5/iis-support-blog/spn-configurations-for-kerberos-authentication-a-quick-reference/ba-p/330547)

- [Microsoft Learn - Kerberos authentication troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/kerberos-authentication-troubleshooting-guidance)

---

