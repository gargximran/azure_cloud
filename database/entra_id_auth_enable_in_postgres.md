
# Use Microsoft Entra ID for authentication with Azure Database for PostgreSQL - Flexible Server


You can configure Microsoft Entra authentication for Azure Database for PostgreSQL flexible server either during server provisioning or later. Only Microsoft Entra administrator users can create or enable users for Microsoft Entra ID-based authentication. 
> [!WARNING]  
>Recommend not using the Microsoft Entra administrator for regular database operations because that role has elevated user permissions (for example, CREATEDB).

>[!TIP]  
>You can have multiple Microsoft Entra admin users with Azure Database for PostgreSQL flexible server. Microsoft Entra admin users can be a user, a group, or service principal.

## Prerequisites


**Configure network requirements**

Microsoft Entra ID is a multitenant application. It requires outbound connectivity to perform certain operations, like adding Microsoft Entra admin groups. Additionally, you need network rules for Microsoft Entra connectivity to work, depending on your network topology:

- **Public access (allowed IP addresses)**: No extra network rules are required.
- **Private access (virtual network integration)**:

  - You need an outbound network security group (NSG) rule to allow virtual network traffic to only reach the `AzureActiveDirectory` service tag.
  - If you're using a route table, you need to create a rule with the destination service tag `AzureActiveDirectory` and next hop `Internet`.
  - Optionally, if you're using a proxy, you can add a new firewall rule to allow HTTP/S traffic to reach only the `AzureActiveDirectory` service tag.
- **Custom DNS**:
   There are additional considerations if you are using custom DNS in your Virtual Network (VNET). In such cases, it is crucial to ensure that the following **endpoints** resolve to their corresponding IP addresses:
**login.microsoftonline.com**: This endpoint is used for authentication purposes. Verify that your custom DNS setup enables resolving login.microsoftonline.com to its correct IP addresses
**graph.microsoft.com**: This endpoint is used to access the Microsoft Graph API. Ensure your custom DNS setup allows the resolution of graph.microsoft.com to the correct IP addresses.

To set the Microsoft Entra admin during server provisioning, follow these steps:

1. In the Azure portal, during server provisioning, select either **PostgreSQL and Microsoft Entra authentication** or **Microsoft Entra authentication only** as the authentication method.
1. On the **Set admin** tab, select a valid Microsoft Entra user, group, service principal, or managed identity in the customer tenant to be the Microsoft Entra administrator.

  You can optionally add a local PostgreSQL admin account if you prefer using the **PostgreSQL and Microsoft Entra authentication** method.

  > [!NOTE]  
  > You can add only one Azure admin user during server provisioning. You can add multiple Microsoft Entra admin users after the Server is created.



<img src="https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/media/how-to-configure-sign-in-Azure-ad-authentication/set-Azure-ad-admin-server-creation.png"
     alt="Screenshot that shows selections for setting a Microsoft Entra admin during server provisioning."
     style="margin-right: 10px;" />  



To set the Microsoft Entra administrator after server creation, follow these steps:

1. In the Azure portal, select the instance of Azure Database for PostgreSQL flexible server that you want to enable for Microsoft Entra ID.
2. Under **Security**, select **Authentication**. Then choose either **PostgreSQL and Microsoft Entra authentication** or **Microsoft Entra authentication only** as the authentication method, based on your requirements.
3. Select **Add Microsoft Entra Admins**. Then select a valid Microsoft Entra user, group, service principal, or managed identity in the customer tenant to be a Microsoft Entra administrator.
4. Select **Save**.



  <img src="https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/media/how-to-configure-sign-in-Azure-ad-authentication/set-Azure-ad-admin.png"
     alt="Screenshot that shows selections for setting a Microsoft Entra admin after server creation."
     style="margin-right: 10px;" />  

> [!IMPORTANT]  
> When setting the administrator, a new user is added to Azure Database for PostgreSQL flexible server with full administrator permissions.


## Connect to Azure Database for PostgreSQL by using Microsoft Entra ID

The following high-level diagram summarizes the workflow of using Microsoft Entra authentication with Azure Database for PostgreSQL:



<img src="https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/media/how-to-configure-sign-in-Azure-ad-authentication/authentication-flow.png"
     alt="Diagram of authentication flow between Microsoft Entra ID, the user's computer, and the server."
     style="margin-right: 10px;" />  

Microsoft Entra integration works with standard PostgreSQL tools like psql, which aren't Microsoft Entra aware and support only specifying the username and password when you're connecting to PostgreSQL. As shown in the preceding diagram, the Microsoft Entra token is passed as the password.



## Authenticate with Microsoft Entra ID

Use the following procedures to authenticate with Microsoft Entra ID as an Azure Database for PostgreSQL flexible server user. You can follow along in Azure Cloud Shell, on an Azure virtual machine, or on your local machine.

### Sign in to the user's Azure subscription

Start by authenticating with Microsoft Entra ID by using the Azure CLI. This step isn't required in Azure Cloud Shell.

```azurecli-interactive
az login
```

The command opens a browser window to the Microsoft Entra authentication page. It requires you to give your Microsoft Entra user ID and password.


### Retrieve the Microsoft Entra access token



For Azure CLI version 2.0.71 and later, you can specify the command in the following convenient version for all clouds:

```azurecli-interactive
az account get-access-token --resource-type oss-rdbms
```

After authentication is successful, Microsoft Entra ID returns an access token:

```json
{
  "accessToken": "TOKEN",
  "expiresOn": "...",
  "subscription": "...",
  "tenant": "...",
  "tokenType": "Bearer"
}
```

The token is a Base64 string. It encodes all the information about the authenticated user and is targeted to the Azure Database for PostgreSQL service.

### Use a token as a password for signing in with client psql

When connecting, it's best to use the access token as the PostgreSQL user password.

While using the psql command-line client, the access token needs to be passed through the `PGPASSWORD` environment variable. The reason is that the access token exceeds the password length that psql can accept directly.



```bash
export PGPASSWORD=<copy/pasted TOKEN value from step 2>
```

You can also combine step 2 and step 3 together using command substitution. The token retrieval can be encapsulated into a variable and passed directly as a value for `PGPASSWORD` environment variable:

```bash
export PGPASSWORD=$(az account get-access-token --resource-type oss-rdbms --query "[accessToken]" -o tsv)
```

Now you can initiate a connection with Azure Database for PostgreSQL as you usually would:

```sql
psql "host=mydb.postgres... user=user@tenant.onmicrosoft.com dbname=postgres sslmode=require"
```


## Authenticate with Microsoft Entra ID as a group member

<a name='create-azure-ad-groups-in-azure-database-for-postgresql---flexible-server'></a>

### Create Microsoft Entra groups in Azure Database for PostgreSQL flexible server

To enable a Microsoft Entra group to access your database, use the same mechanism you used for users, but specify the group name instead. For example:

```sql
select * from  pgaadauth_create_principal('Prod DB Readonly', false, false).
```

>[!IMPORTANT]  
>When group members sign in, they use their access tokens but specify the group name as the username.

> [!NOTE]  
> Azure Database for PostgreSQL flexible server supports managed identities and service principals as group members.

### Sign in to the user's Azure subscription

Authenticate with Microsoft Entra ID by using the Azure CLI. This step isn't required in Azure Cloud Shell. The user needs to be a member of the Microsoft Entra group.

```azurecli-interactive
az login
```

### Retrieve the Microsoft Entra access token

For Azure CLI version 2.0.71 and later, you can specify the command in the following convenient version for all clouds:

```azurecli-interactive
az account get-access-token --resource-type oss-rdbms
```

After authentication is successful, Microsoft Entra ID returns an access token:

```json
{
  "accessToken": "TOKEN",
  "expiresOn": "...",
  "subscription": "...",
  "tenant": "...",
  "tokenType": "Bearer"
}
```

### Use a token as a password for signing in with psql or PgAdmin

These considerations are essential when you're connecting as a group member:

- The group name is the name of the Microsoft Entra group that you're trying to connect.
- Be sure to use the exact way the Microsoft Entra group name is spelled. Microsoft Entra user and group names are case-sensitive.
- When you're connecting as a group, use only the group name and not the alias of a group member.
- If the name contains spaces, use a backslash (`\`) before each space to escape it.
- The access token's validity is 5 minutes to 60 minutes. We recommend you get the access token before initiating the sign-in to Azure Database for PostgreSQL.

You're now authenticated to your PostgreSQL server through Microsoft Entra authentication.