Creating a Custom GPT Search Experience for a Specific SharePoint Site Using Azure Functions, Microsoft Graph API, and GPT Actions
Overview
ChatGPT Enterprise allows workspace users to create Custom GPTs with Actions that call external APIs. This guide explains how to create a Custom GPT that calls an Azure Function, which then uses Microsoft Graph API to search SharePoint or OneDrive content.
The default Microsoft SharePoint connector in ChatGPT searches across the SharePoint and OneDrive content available to the signed-in user. In some cases, a department or business unit may need a more focused experience that searches only a specific SharePoint site, subsite, document library, or OneDrive location.
For example:
•	HR may want a GPT that searches only the HR benefits site.
•	Finance may want a GPT that searches only Finance and Accounting sites.
•	A project team may want a GPT that searches only a dedicated project SharePoint site.
This guide covers how to create that experience using:
•	Azure Function App
•	Microsoft Entra App Registration
•	Microsoft Graph API
•	OAuth 2.0
•	GPT Actions
•	Optional SharePoint site-level access restrictions
________________________________________
Prerequisites
Before starting, you will need:
1.	Access to the Azure Portal with permission to create Azure Function Apps and Microsoft Entra App Registrations.
2.	Access to create or modify Microsoft Graph API permissions.
3.	Postman or another API testing tool.
4.	Familiarity with OAuth 2.0 flows.
5.	Familiarity with Node.js and JavaScript.
6.	A SharePoint site, subsite, document library, or OneDrive location that the target end users can access.
7.	ChatGPT Enterprise access with permission to create Custom GPTs and configure Actions.
An OpenAI API key is required for the Azure Function to call the OpenAI API directly. It is not required just to configure a GPT Action that calls your Azure Function. This key can be created via platform.openai.com. Ensure that your project/key has access to use gpt-4o mini model.
________________________________________
Step 1: Create the Azure Function App
1.	Go to the Azure Portal.
2.	Create a new Function App.
3.	Choose the following runtime settings:
o	Runtime stack: Node.js
o	Version: Node.js 22, or the current supported Node.js version approved by your organization.
4.	Under the Networking tab, ensure public access is enabled if ChatGPT needs to call the function endpoint directly.
5.	Leave the remaining settings as default, or adjust them according to your organization’s Azure standards.
6.	Enable Application Insights. This is recommended for debugging API calls, authentication issues, and Graph API responses.
<br>
<img width="1791" height="922" alt="CreateFnApp4" src="https://github.com/user-attachments/assets/caf071b9-9ca7-41db-a50b-1842083dc3c2" />
<br>
<img width="1343" height="892" alt="CreateFnApp2" src="https://github.com/user-attachments/assets/69fa2cee-2bf1-43ad-99d0-feb948041aea" />
<br>

Step 2: Configure Authentication for the Function App
1.	Open the Function App created in Step 1.
2.	In the left navigation, go to Settings > Authentication.
3.	Select Add identity provider.
4.	Choose Microsoft as the identity provider.
5.	Select tenant type: Workforce.
6.	Leave the remaining settings as default unless your organization requires a different configuration.
7.	Create the identity application.
After the identity application is created, configure Microsoft Graph permissions.
Required Microsoft Graph Delegated Permissions
Add the following Microsoft Graph delegated permissions:
•	Files.Read.All
•	Sites.Read.All
•	Sites.Selected
Delegated permissions are used so the search runs in the context of the signed-in user. This helps ensure that results honor the user’s existing SharePoint and OneDrive permissions.
After adding permissions, have an administrator grant admin consent.
App Service Permission
Also confirm that the Azure App Service permission scope is available:
•	user_impersonation
This is used by the OAuth flow for the Function App.
<br>
<img width="918" height="783" alt="CreateFnApp3" src="https://github.com/user-attachments/assets/eee2a418-972b-4dcf-812e-d5dd90ce4b72" />
<br>
________________________________________
Step 3: Capture App Registration Values
When the Function App authentication provider is configured, an App Registration is created in Microsoft Entra ID.
1.	Go to Microsoft Entra ID > App registrations.
2.	Find and open the App Registration associated with your Function App.
3.	On the Overview page, copy and save:
o	Application client ID
o	Directory tenant ID
4.	Select Endpoints.
5.	Copy and save:
o	OAuth 2.0 authorization endpoint
o	OAuth 2.0 token endpoint
6.	Go to Authentication.
7.	Add the following Postman redirect URIs:
o	https://oauth.pstmn.io/v1/callback
o	https://oauth.pstmn.io/v1/browser-callback
You will add the ChatGPT OAuth redirect URI later when configuring the GPT Action.
<br>
<img width="1791" height="922" alt="CreateFnApp4" src="https://github.com/user-attachments/assets/59f625ab-f741-43f7-8915-4154cf0643c8" />
<br>

________________________________________
Step 4: Expose an API Scope
1.	In the same App Registration, go to Expose an API.
2.	Add a scope named:
 	user_impersonation
3.	Copy the full scope value and save it for later use.
The scope value usually looks similar to:
api://<application-client-id>/user_impersonation
You will use this value in Postman and in the GPT Action OAuth configuration.
________________________________________
Step 5: Capture the Client Secret
The Function App authentication configuration creates or uses a secret value.
1.	Open the Function App.
2.	Go to Settings > Environment variables.
3.	Locate:
 	MICROSOFT_PROVIDER_AUTHENTICATION_SECRET
4.	Save this value securely.
This value is used as the OAuth client secret when testing with Postman and when configuring the GPT Action authentication.
Do not store this secret in code or publish it in documentation.
________________________________________
Step 6: Create the HTTP Trigger Function
1.	Open the Function App.
2.	Create a new function.
3.	Choose HTTP trigger.
4.	Select the authorization level appropriate for your security model.
5.	After the function is created, select Get Function URL.
6.	Save the function URL.
At this point, the sample function should return a response similar to:
This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.
________________________________________
Step 7: Test the Function App with Postman
Before adding Microsoft Graph logic, test the basic Function App endpoint.
1.	Open Postman.
2.	Create a new request.
3.	Configure authorization using OAuth 2.0.
4.	Enter:
o	Authorization URL
o	Token URL
o	Client ID
o	Client Secret
o	Scope
5.	Set client credentials to be sent in the request body.
6.	Generate a new access token.
7.	Use the token to call the Azure Function endpoint.
If the configuration is correct, you should receive the default HTTP trigger response.
________________________________________
Step 8: Add Microsoft Graph Search Logic
After confirming the Function App works, replace the sample HTTP trigger code with your Node.js logic.
The Function App should accept a POST body similar to:
{
  "query": "What question should the GPT answer?",
  "searchTerm": "keyword or phrase to search"
}
The function should then:
1.	Read the incoming request body.
2.	Validate the user token.
3.	Call Microsoft Graph API.
4.	Search SharePoint or OneDrive content.
5.	Return the matching files, snippets, links, or extracted content to the GPT Action.
Depending on your approach, you can use:
•	Microsoft Search API for SharePoint and OneDrive search.
•	DriveItem search for searching within a specific drive or folder hierarchy.
•	Site ID or Drive ID filtering for more controlled targeting.
________________________________________
Step 9: Configure the Custom GPT
1.	In ChatGPT Enterprise, create a new Custom GPT.
2.	Open the Configure tab.
3.	Add clear GPT instructions that explain:
o	What site or document library the GPT should search.
o	What kinds of questions it should answer.
o	That answers should be grounded only in returned search results.
4.	Under Actions, create a new Action.
5.	Add your OpenAPI schema.
6.	Set the authentication type to OAuth.
7.	Configure OAuth using the same values tested in Postman:
o	Authorization URL
o	Token URL
o	Client ID
o	Client Secret
o	Scope
8.	Copy the OAuth redirect URI generated by ChatGPT.
9.	Return to the Azure App Registration.
10.	Add the ChatGPT redirect URI under the Web redirect URI settings.
11.	Save the App Registration changes.
12.	Test the GPT Action from the Custom GPT builder.
________________________________________
Step 10: Restrict Search to a Specific SharePoint Site, Subsite, Library, or OneDrive Location
At this point, your Function App can search using Microsoft Graph. However, without additional scoping, it may search more content than intended, depending on the Graph endpoint and permissions used.
There are two main approaches to restricting results.
________________________________________
Option 1: Restrict Search Programmatically
Use site IDs, drive IDs, list IDs, or folder paths in your Azure Function code.
Recommended approach:
1.	Use Microsoft Graph Explorer or a Graph API request to get the target SharePoint site ID.
2.	Get the drive ID for the target document library.
3.	Modify the function logic so it searches only within that site, drive, library, or folder.
4.	Store allowed site IDs or drive IDs in Azure Function environment variables instead of hardcoding them.
5.	Validate incoming requests so users cannot override the allowed site or drive scope.
Example environment variables:
ALLOWED_SITE_ID=<target-site-id>
ALLOWED_DRIVE_ID=<target-drive-id>
ALLOWED_FOLDER_PATH=/Shared Documents/Benefits
This option is useful when the same app registration can technically access more content, but your application code enforces the search boundary.
________________________________________
Option 2: Restrict App Access with Sites.Selected
Use Sites.Selected to grant the application access only to selected SharePoint sites.
High-level process:
1.	In the Function App, go to Settings > Identity.
2.	Turn the managed identity status On.
3.	Save the Object ID / Principal ID.
4.	In Microsoft Entra, confirm the App Registration has Sites.Selected.
5.	Grant the app permission to the specific SharePoint site.
Using PnP.PowerShell, the pattern is:
Connect-PnPOnline -Url "https://contoso.sharepoint.com/sites/YourSite" -Interactive

Grant-PnPAzureADAppSitePermission `
  -AppId "YOUR_APP_CLIENT_ID" `
  -DisplayName "YOUR_APP_NAME" `
  -Site "https://contoso.sharepoint.com/sites/YourSite" `
  -Permissions Read
You will then grant site permissions using Microsoft Graph API:
POST https://graph.microsoft.com/v1.0/sites/{siteId}/permissions
Content-Type: application/json
{
  "roles": ["read"],
  "grantedToIdentities": [
    {
      "application": {
        "id": "YOUR_APP_CLIENT_ID",
        "displayName": "YOUR_APP_NAME"
      }
    }
  ]
}
Use read unless the function truly needs write access.
This option is stronger than programmatic filtering because it limits what the app can access at the permission layer.
________________________________________
Recommended Security Model
For production use, combine both controls:
1.	Use Sites.Selected to restrict the app’s permission boundary.
2.	Use site ID, drive ID, or folder path filtering in code.
3.	Store approved site and drive IDs in environment variables.
4.	Never let the GPT or user supply unrestricted Graph paths.
5.	Log search requests and Graph responses in Application Insights.
6.	Return only the minimum content needed to answer the user’s question.
7.	Do not expose access tokens, secrets, raw Graph errors, or internal IDs to the GPT response.
________________________________________
Testing Checklist
Before publishing the GPT, validate the following:
•	The Azure Function endpoint works with OAuth.
•	Postman can retrieve an access token.
•	Postman can call the Function App successfully.
•	The Function App can call Microsoft Graph.
•	Search results only come from the intended SharePoint site or OneDrive location.
•	A user without access to a document cannot retrieve that document through the GPT.
•	The GPT Action schema validates successfully.
•	The GPT can authenticate through OAuth.
•	The GPT returns grounded answers based on the Azure Function response.
•	Application Insights logs enough detail for troubleshooting without exposing secrets.
________________________________________
Troubleshooting Tips
OAuth token fails
Check:
•	Client ID
•	Client secret
•	Tenant ID
•	Authorization URL
•	Token URL
•	Scope
•	Redirect URI
•	Admin consent
Postman works but ChatGPT OAuth fails
Check:
•	The ChatGPT redirect URI was copied correctly.
•	The redirect URI was added to the App Registration.
•	The OAuth scope in the GPT Action matches the exposed API scope.
•	The client secret is correct and has not expired.
Graph API returns too many results
Check:
•	Whether the code is searching all SharePoint and OneDrive content.
•	Whether the request includes the correct site ID, drive ID, list ID, or folder path.
•	Whether Sites.Selected has been configured correctly.
Graph API returns no results
Check:
•	The signed-in user has access to the target SharePoint content.
•	The app has the required Graph permissions.
•	Admin consent has been granted.
•	The target site permission was granted when using Sites.Selected.
•	The search query is valid.
•	The target library or folder contains indexed content.
________________________________________
Summary
A Custom GPT with Actions can provide a focused search experience for a specific SharePoint site, document library, subsite, or OneDrive location.
The recommended architecture is:
1.	Custom GPT with OAuth-secured Action.
2.	Azure Function App as the API layer.   
4.	Microsoft Graph API for SharePoint and OneDrive search.
5.	Microsoft Entra authentication.
6.	Sites.Selected and code-level filtering to restrict scope.
This approach gives teams a controlled, department-specific GPT search experience while still respecting Microsoft 365 identity, permissions, and governance.

Sample Open AI Schema
Replce tenantID, function url with your azure values
openapi: 3.1.0
info:
  title: SharePoint Search API
  description: API for searching SharePoint documents.
  version: 1.0.0
servers:
  - url: https://{your_function_app_name}.azurewebsites.net/api
    description: SharePoint Search API server
paths:
  /{your_function_name}?code={enter your specific endpoint id here}:
    post:
      operationId: searchSharePoint
      summary: Searches SharePoint for documents matching a query and term.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query:
                  type: string
                  description: The full query to search for in SharePoint documents.
                searchTerm:
                  type: string
                  description: A specific term to search for within the documents.
      responses:
        '200':
          description: Search results
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    documentName:
                      type: string
                      description: The name of the document.
                    snippet:
                      type: string
                      description: A snippet from the document containing the search term.
                    url:
                      type: string
                      description: The URL to access the document.
