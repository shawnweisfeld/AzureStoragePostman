# Using Azure Storage with Postman

Azure Storage provides a REST API that allows you to interact with it from any device with an internet connection, regardless of what language you are programming in. 

See the full details in the [Azure Storage REST API Reference](https://docs.microsoft.com/en-us/rest/api/storageservices/) docs. 

When I am learning/investigating a new API the first thing I do is exercise it with Postman. Postman is a GUI tool that allows you to manually create an test REST services, and as a bonus you can export sample code many popular languages. You can [Download Postman](https://www.postman.com/) here. 

  > NOTE: I am going to assume you know how to use Postman. If you need a refresher the Postman website has a good [Learning Center](https://learning.postman.com/).

## How to use this sample

### Setup

 - First you will need to import the Postman collection into the tool. You can [download it here](Azure%20Storage.postman_collection.json). 
 - In the Postman collection you will see 5 requests. They are numbered so that you can execute them in the correct order.
 
   ![5 requests](/img/5requests.png)
 
 - Now that we have our collection, we need to setup some variables. I created an environment in Postman to hold them all.

    ![variables](/img/variables.png)

    - **directoryId** - The ID of the AAD instance you plan to authenticate against
    - **client_id** - The ID of the AAD object (in our case a service principal) that you are going to authenticate with
    - **client_secret** - This string is the "password" for our service principal
    - **subscriptionId** - The ID of the subscription we are going to use
    - **resourceGroupName** - This is the resource group we want to put our storage account in
    - **accountName** - This is the name of the storage account we will create and use
    - **containerName** - This is the name of the blob container we will create and use
    - **blob_name** - This is the name of the blob we will create

 - Now lets create our service principal.
   - The first step is to [Create an Azure Active Directory application](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal).
   - Next you need to get the Directory ID & Client ID and place these in those variables you created in Postman. [Instructions here](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#get-values-for-signing-in)
   - Now lets create and get the client secret and place that in the Postman variables. [Instructions here](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#create-a-new-application-secret)
   - Now you need to give your application the permissions to create a storage account and contribute content to the storage account. You can learn more about [assigning a role to your applicaiton here](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#assign-a-role-to-the-application). 

      > I gave my service principal Owner and storage blob data contributor access for my subscription.

        ![role assignments](/img/roleassignments.png)

  - Now would also be a good time to update the rest of the Postman variables. 
    - Grab your subscription id.
    - Create the resource group you want to use.
    - Fill in values for the storage account, container and blob names. **Don't create these in Azure, this is what our Postman sample will do.** 

### Request Details

> NOTE: Here I am walking through the configuration of each request. If you imported the Postman collection during the setup you will have this all configured already for you. However, it is good to step through it, to understand what it is doing. 

  - 01 GET Management OAuth2 token

    The first thing we want to do is create our storage account. This is done by making a request to the Azure resource manager directly. To that end we need to get an authorization token against the management plane in Azure. 

    We get the token by making a `GET` request to `https://login.microsoftonline.com/{{directoryId}}/oauth2/token`. We want to set the `Content-Type` header to `application/x-www-form-urlencoded`, and pass in the following key value pares in the body. 

      | Key | Value |
      | --- | ----- |
      | grant_type | client_credentials |
      | client_id | {{client_id}} |
      | client_secret | {{client_secret}} |
      | resource | https://management.azure.com |

    Finally we want to use a bit of code to parse the response and pull the access token we generated into a variable we can use later. 

      ``` javascript
      var data = JSON.parse(responseBody);
      postman.setGlobalVariable("mgmt_access_token", data.access_token);
      ```

    Now we can send this request to Azure and get back our auth token for the management plane. 

  - 02 PUT create storage account

    Now with the proper token, we can ask Azure to create our storage account.

    We do that by making a `PUT` request to `https://management.azure.com/subscriptions/{{subscriptionId}}/resourceGroups/{{resourceGroupName}}/providers/Microsoft.Storage/storageAccounts/{{accountName}}`. We want to set a query parm for `api-version` to `2018-02-01`, and a header for the `Content-Type` to `application/json`.     Now we want to set the Authorization to use the `Bearer Token` type and the token from our variable `{{mgmt_access_token}}`. 

    Now we will need to send a chunk of json as the body telling azure the details of the storage account we want to create. 

      ``` json
      {
        "sku": {
        "name": "Standard_GRS"
        },
        "kind": "StorageV2",
        "location": "southcentralus"
      }
      ```

    When you send this request you will get back a `202 Accepted` response. This tells us that Azure has started to create our storage account. If you resend the request a few seconds later you should get back a `200 OK` with the full details of the storage account you just created. 

  - 03 GET Storage OAuth2 token

    In the first step we authenticated against the Azure management plane, to talk to Azure storage we will need to authenticate against the storage management plane. It has a different "scope". So you will see that this request looks just like the first one, except for the `resource` key is set to `https://storage.azure.com` and we are putting the returned access token in a different Postman variable. 

  - 04 PUT create container

    Now lets create a container to put our blob in. We do this by sending a `PUT` request to `https://{{accountName}}.blob.core.windows.net/{{containerName}}`, with a query string parameter `restype` set to `container`, and set the following key value pairs as headers. 

      | Key | Value |
      | --- | ----- |
      | Content-Type | application/json |
      | x-ms-version | 2019-07-07 |
      | x-ms-blob-public-access | container |

    Now we want to set the Authorization to use the `Bearer Token` type and the token from our variable `{{storage_access_token}}`. 

    Now, when you send the request you should get back a `201 created`

  - 05 PUT a blob in the container

    Finally, lets upload a blob to the container we just created. We do this by sending a `PUT` request to `https://{{accountName}}.blob.core.windows.net/{{containerName}}/{{blob_name}}`. As before we set the Authorization to use the `Bearer Token` type and the token from our variable `{{storage_access_token}}`. We also set headers for `x-ms-blob-type` of `BlockBlob` and `x-ms-version` of `2019-07-07`. Finally, we can place whatever we want in the body, this will be the contents of our file. NOTE: There are limits to the size of the blob you can upload in a single request, if your blob is too big you will have to break it into chunks.


That is it, you should now have a storage account, container and blob in Azure. We just scratched the surface of what these APIs can do, but this should help you get started. 

![done](/img/done.png)



## Reference Links
- Microsoft Docs
  - [Azure Storage REST API Reference](https://docs.microsoft.com/en-us/rest/api/storageservices/)
  - [How to: Use the portal to create an Azure AD application and service principal that can access resources](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)
  - [Authorize with Azure Active Directory](https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-with-azure-active-directory)
  - [Create an Azure Storage account with the REST API](https://docs.microsoft.com/en-us/rest/api/storagerp/storage-sample-create-account)
  - [Create Container API](https://docs.microsoft.com/en-us/rest/api/storageservices/create-container)
  - [Put Blob API](https://docs.microsoft.com/en-us/rest/api/storageservices/put-blob)
- Blogs
  - [Azure REST APIs with Postman in 2 Minutes](https://blog.jongallant.com/2017/11/azure-rest-apis-postman/)
  - [How to create Azure Storage Account with REST API using Postman](http://raaviblog.com/how-to-create-azure-storage-account-with-rest-api-using-postman/)
  - [How to use Azure blob storage service REST API operations using POSTMAN](http://raaviblog.com/how-to-use-azure-blob-storage-service-rest-api-operations-using-postman/)
  - [Examples of the Windows Azure Storage Services REST API](https://convective.wordpress.com/2010/08/18/examples-of-the-windows-azure-storage-services-rest-api/)