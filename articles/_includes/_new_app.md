## Configure Auth0

### Get Your Application Keys

  When you signed up for Auth0, a new Client was created for you, or you could have created a new one.

  Your application needs some details about that client to communicate with Auth0. You can get these details from the [Client Settings](${manage_url}/#/clients/${account.clientId}/settings) section fin the Auth0 dashboard.

  You need the following information:
  * **Client ID** : $(account.clientId)
  * **Domain** : $(account.domain)

  ![App Dashboard](/media/articles/dashboard/client_settings.png)