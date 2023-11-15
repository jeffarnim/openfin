---
title: "ServiceNow integration"
slug: "servicenow-openfin"
hidden: false
category: "6554e8e7ecc618083df24cd5"
parentDocSlug: "servicenow-integration"
---
## Integrate your OpenFin solution with your data in ServiceNow

OpenFin provides an API to help simplify connecting your OpenFin app with ServiceNow. The API integrates with the ServiceNow REST API via OAuth 2.0 to provide secure interoperability between an OpenFin app and a ServiceNow instance.

## Prerequisites

* OpenFin
  * Minimum recommended runtime version 20.91.64.3.
* A ServiceNow instance
  * Account with the Administrator (admin) role.

## Installation

### Install npm package

Add the @openfin/servicenow NPM package as a dependency for your app:

```shell
npm install @openfin/servicenow
```

### Install OpenFin app from ServiceNow store

Navigate to the [OpenFin application on the ServiceNow Store](https://store.servicenow.com/sn_appstore_store.do#!/store/application/c959f40297db6510dbcefbbe2153af53) and install the application into the ServiceNow instance that you wish to integrate with your OpenFin app(s).

## Configuration

### Get Client ID and Instance Name

1. In a web browser, navigate to the ServiceNow instance that you wish to integrate with.
1. In the "All" menu, navigate to System OAuth > Application Registry.
1. Click the "New" button and when prompted click "Create an OAuth API endpoint for external clients".
1. On the form that appears, set "Name" to an appropriate value such as the name of your app, or "OpenFin".
1. Ensure the following:

   * The "Active" and "Public Client" checkboxes are checked.
   * The "Redirect URL" value points to your ServiceNow instance, i.e. "https://YOUR_INSTANCE_NAME.service-now.com/oauth_redirect.do", replacing "YOUR_INSTANCE_NAME" with the name of your ServiceNow instance.
   * Set the "Refresh Token Lifespan" to "86,400", or to a different value that is acceptable for your organization.
   * Set the "Access Token Lifespan" to "7,200", or to a different value that is acceptable for your organization. You may update these values but be aware that reducing the refresh token lifespan value will likely result in users having to re-connect to the instance. For more details, see [Working with ServiceNow authorization](doc:servicenow-auth).

2. Click the "Submit" button to create the OAuth registration.
3. From the created OAuth registration, copy the "Client ID" value as you will provide this to the OpenFin ServiceNow API "connect" function to connect to the instance.

## CORS Rules

CORS rules are required for your OpenFin application to make requests to the OpenFin Integration scripted web service, as well the relevant ServiceNow REST APIs:

1. In a web browser, navigate to your target ServiceNow instance.
1. In the "All" menu, navigate to System Web Services > REST > CORS Rules.
1. Click the "New" button and complete the form ensuring the following values are provided:

   * Name: OpenFin Integration
   * REST API: OpenFin Integration [x_opfi_openfin/integration]
   * Domain: (Provide your OpenFin application's domain)
   * Max age: 86,400
   * HTTP Methods: GET, POST

1. Click the "Submit" button to create the CORS rule.
1. Create additional CORS rules for the various ServiceNow REST APIs that your OpenFin application will utilize. For example, the most common REST API is the Table API:

   * Name: OpenFin – Table API
   * REST API: Table API [now/table]
   * Domain: (Provide your OpenFin application's domain)
   * Max age: 86,400
   * HTTP Methods: GET, POST, PUT, PATCH

## Usage

> 📘 Note
> 
> Take a look at the [basic](https://github.com/built-on-openfin/workspace-starter/tree/main/how-to/integrate-with-service-now-basic) and [advanced](https://github.com/built-on-openfin/workspace-starter/tree/main/how-to/integrate-with-service-now) starter projects for help getting started with OpenFin's ServiceNow integration.

## Connect your OpenFin app to ServiceNow

Call the `connect` function to begin the authorization flow needed to connect your app to the target ServiceNow instance. Note that after calling this function, the user will be prompted to authorize the app to act on their behalf (for more information see [Working with ServiceNow authorization > The authorization process](doc:servicenow-auth#the-authorization-process)).

```typescript
// Import the OpenFin ServiceNow integration.
import { connect } from '@openfin/servicenow';

try {
  // Update the below statements for your target instance
  const instanceUrl = 'https://YOUR_INSTANCE_NAME.service-now.com';
  const clientId = 'YOUR_CLIENT_ID';

  // Initiate the authorization flow to connect to the instance
  const serviceNowConnection = await connect(instanceUrl, clientId);

  // If connection succeeded, output connection info to the console
  const { currentUser, instanceUrl: connectedInstance } = serviceNowConnection;

  // You are now connected to ServiceNow.
  console.log(`Connected to ${connectedInstance} as ${currentUser.user_name}`);
} catch (err) { 
  // If connection failed, output error info to the console
  console.error('Connection failed: ', err.message);
}
```

Make sure to provide your own values for the following:

* Instance Name: Replace `YOUR_INSTANCE_NAME` with the instance name of the target ServiceNow instance.
* Client ID: Replace `YOUR_CLIENT_ID` with your Client ID, as described in [Get Client ID and Instance Name](#get-client-id-and-instance-name).

## Executing requests to ServiceNow's REST APIs

The CORS rules described here are intended for use with the ServiceNow Table API.
If you wish to use any of the other APIs, you can do so with this integration.
See the configuration step to make sure you have the CORS Rule for that specific API you want to call, and refer to the ServiceNow documentation regarding how to use that specific API.

### Get the user's active cases

This code sample queries the ServiceNow database for the current user's active cases.

```typescript
// Get the current user's active Cases.
try {
  // Create the query parameters to return the current user's active cases
  const queryParams = new URLSearchParams();
  queryParams.set(
    'sysparm_query',
    `assigned_to=${serviceNowConnection.currentUser.sys_id}^active=true^ORDERBYDESCsys_updated_on`
  );
  queryParams.set(
    'sysparm_fields',
    'sys_id,sys_class_name,task_effective_number,short_description,state,sys_updated_on'
  );
  queryParams.set('sysparm_limit', '30');
  queryParams.set('sysparm_display_value', 'true');

  // Execute the request to ServiceNow's Table API passing the expected response type as a parameter along with the request URI.
  const response = await serviceNowConnection.executeApiRequest<ServiceNowEntities.CSM.Case[]>(`/api/now/v2/table/sn_customerservice_case?${queryParams.toString()}`);

  // Parse the response from the TableAPI.
  const { data: myActiveCases } = response;

  // In this sample, we write log entries for the number of cases, and some details from each case.
  console.log(`Found ${myActiveCases!.length} cases`);
  myActiveCases!.forEach((activeCase) => {
    console.log(`Case ${activeCase.task_effective_number} is ${activeCase.state}`);
  });
} catch (err) {
  // Make sure to properly handle any errors that might occur when executing API requests.
  if (err instanceof AuthTokenExpiredError) {
    console.error('Connection expired, re-connect then try again');
  } else if (err instanceof ApiRequestError) {
    console.error(`Request failed: ${err.message}`);
  } else {
    console.error(err);
  }
}
```

### Get a specific Financial Services Case by ID

This code sample queries the ServiceNow database for a specific Financial Services Case.

```typescript
// Specify the specific Financial Services Case ID to retrieve.
const caseId = '850057664b874cbdb7ad18d535960380';

// Create the query parameters.
const queryParams = new URLSearchParams();
queryParams.set('sysparm_exclude_reference_link', 'true');
queryParams.set('sysparm_display_value', 'true');

// Call the ServiceNow TableAPI to get the specific Financial Services Case.
const response = await serviceNowConnection.executeApiRequest<ServiceNowEntities.FSO.FinancialServicesCase>(`/api/now/v2/table/sn_bom_financial_service/${caseId}?${queryParams.toString()}`);

// Parse the case.
const { data: fsoCase } = response;

// Log the result.
console.log(`Case ${fsoCase!.task_effective_number} is ${fsoCase!.state}`);
```

### Create a new Financial Services Case

This code sample shows the steps to create a new Financial Services Case.

```typescript
// Create the query parameters.
const queryParams = new URLSearchParams();
queryParams.set('sysparm_fields', 'sys_id,task_effective_number');

// Call the ServiceNow TableAPI to POST the case.
const response = await serviceNowConnection.executeApiRequest<ServiceNowEntities.FSO.FinancialServicesCase>(
  `/api/now/v2/table/sn_bom_financial_service?${queryParams.toString()}`,
  'POST',
  { short_description: 'New case for customer' } as ServiceNowEntities.FSO.FinancialServicesCase
);

// Parse the response.
const { data: newFsoCase } = response;

// Log the result.
console.log(`Created new Financial Services Case with ID: ${newFsoCase!.sys_id}`);
```

### Add a comment to a case

This code sample shows how to add a comment to a specific case.

```typescript
// Specify the case ID to add a comment to.
const caseId = 'dfa83216971eb110d16bf1d11153af8c';

// Call the ServiceNow TableAPI to add the comment to the case.
await serviceNowConnection.executeApiRequest(
  `/api/now/v2/table/sn_bom_financial_service/${caseId}`,
  'PATCH',
  {
    comments: 'Currently investigating this case',
  } as ServiceNowEntities.FSO.FinancialServicesCase,
  true // Suppress the response body for this request as it is not needed
);
```

### Close a specific case

This code sample shows how to close a specific case.

```typescript
// Specify the case ID to close.
const caseId = 'dfa83216971eb110d16bf1d11153af8c';

// Call the ServiceNow TableAPI to close the case.
await serviceNowConnection.executeApiRequest(
  `/api/now/v2/table/sn_bom_financial_service/${caseId}`,
  'PATCH',
  {
    comments: 'The case has been resolved.',
    state: '3',
  } as ServiceNowEntities.FSO.FinancialServicesCase,
  true // Suppress the response body for this request as it is not needed
);
```

## Disconnect from ServiceNow.

Call disconnect to terminate the connection to ServiceNow and clean up utilized resources:

```typescript
// Disconnect from ServiceNow.
await serviceNowConnection.disconnect();
```
