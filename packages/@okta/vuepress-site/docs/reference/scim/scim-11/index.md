---
title: SCIM 1.1 Protocol Reference
icon: /assets/img/icons/scim.svg
meta:
  - name: description
    content: Your SCIM API must support specific SCIM 1.1 API endpoints to work with Okta. Those endpoints and their explanations are detailed here.
---

# Okta and SCIM V1.1

This reference focuses on how Okta API endpoints share information with System for Cross-domain Identity Management (SCIM) specific API calls.

This document specifically covers **Version 1.1** of the SCIM specification. For Version 2.0 of the SCIM specification, see our [SCIM 2.0 reference](/docs/reference/scim/scim-20/).

The SCIM Protocol is an application-level REST protocol for provisioning and managing identity data on the web. The protocol supports creation, discovery, retrieval, and modification of core identity resources.

To better understand SCIM and the specific implementation of SCIM using Okta, see our [Understanding SCIM](/docs/concepts/scim/) guide or our blog post on [What is SCIM?](https://www.okta.com/blog/2017/01/what-is-scim/).

## SCIM User operations

### Creating users

The User creation operation brings the user's application profile from Okta over to the Service Provider. A user's application profile represents the key-value attributes defined on the **Profile** tab when a user is added.

To enable user provisioning, an Okta administrator must configure the provisioning options in the Okta Admin Console. In the Okta Admin Console, select your SCIM application from your list of applications. Under the **Provisioning** tab, click **To App** and **Edit**. In the **Create User** option, click **Enable** and then **Save**.

For more information on enabling the provisioning features of your SCIM application, see [Provisioning and Deprovisioning](https://help.okta.com/en/prod/okta_help_CSH.htm#ext_Provisioning_Deprovisioning_Overview) under the **Accessing Provisioning Features** section.

After saving the application configuration, Okta makes two requests:

```http
GET /scim/v1/Users?startIndex=1&count=2 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

and

```http
GET /scim/v1/Groups?startIndex=1&count=100 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

> **Note:** The query parameters that Okta sends at this point are always constant.

After you complete this step, whenever a user is assigned to the application in Okta, the following operation requests are made against the SCIM server:

* Determine if the user already exists
* Create the user if the user isn't found

#### Determine if the user already exists

**GET** /Users

Okta checks that the user exists on the SCIM server through a GET operation with the `filter=userName` parameter (or any other filter parameter that was configured with the SCIM application). This check is performed using the `eq` (equal) operator and is the only one necessary to successfully provision users with Okta.

> **Note:** The filter must check an attribute that is _unique_ for all users in the Service Provider profiles.

For Okta Integration Network (OIN) applications, this filter is configured during the review process done by the Okta Application Analyst assigned to evaluate the application submission. Application submissions are processed through the [OIN Manager](https://oinmanager.okta.com).

The requests from Okta to the Service Provider are of the form:

```http
GET /scim/v1/Users?filter=userName%20eq%20%22test.user%40okta.local%22&startIndex=1&count=100 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

The SCIM application checks the filter provided and returns an empty response if no users match the filter criteria. For example:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 06:39:03 GMT
Content-Type: application/json

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "totalResults": 0,
    "startIndex": 1,
    "itemsPerPage": 0,
    "Resources": []
}
```

If the SCIM server does return a user, Okta automatically matches the result with the user in Okta, sending the user application profile to the SCIM server.

**PUT** /Users/*$userID*

```http
PUT /scim/v1/Users/e155dc98-aeee-4f58-b683-12b93dbc86b3 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "e155dc98-aeee-4f58-b683-12b93dbc86b3",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test",
        "familyName": "User"
    },
    "displayName": "Test User",
    "locale": "en_US",
    "active": true,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "groups": []
}
```

#### Create the user

**POST** /Users

If the user isn't found on the SCIM server, then Okta attempts to create it through a POST operation that contains the user application profile. The request looks like the following:

```http
POST /scim/v1/Users HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "userName": "test.user@okta.local",
    "name": {
        "givenName": "Test",
        "familyName": "User"
    },
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "displayName": "Test User",
    "locale": "en_US",
    "externalId": "00b443dd40SOK2eup0h7",
    "groups": [],
    "password": "1mz050nq",
    "active": true
}
```

The response from the SCIM server contains the created user profile:

```http
HTTP/1.1 201 Created
Date: Fri, 18 Oct 2019 06:39:04 GMT
Content-Type: application/json

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "e155dc98-aeee-4f58-b683-12b93dbc86b3",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test",
        "familyName": "User"
    },
    "displayName": "Test User",
    "locale": "en_US",
    "active": true,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "groups": [],
    "meta": {
        "created": "2019-10-18T06:39:04+00:00",
        "lastModified": "2019-10-18T06:39:04+00:00",
        "version": "fe2a80129b16ff934d485c3c9c45892f"
    }
}
```

If the SCIM server returns an empty response body to the provisioning request, then Okta marks the operation as invalid, and the Okta administrator is shown an error in the Admin Console:

"Automatic provisioning of user `userName` to app `AppName` failed: Error while creating user `displayName`: Create new user returned empty user."

If the user that Okta tries to create already exists in the Service Provider application, then the Service Provider needs to respond with an error schema to stop the provisioning job. The response looks like the following:

```http
HTTP/1.1 409 Conflict
Date: Fri, 18 Oct 2019 06:50:25 GMT
Content-Type: application/json

{
    "Errors": [{
        "description": "User already exists in the database.",
        "code": 409
    }]
}
```

### Retrieving users

**GET** /Users

When importing users from the SCIM server, Okta accesses the `/Users` endpoint and processes them page by page, using `startIndex`, `count`, and `totalResults` as pagination references. Similarly, when returning large lists of resources, your SCIM implementation must support pagination. Using a limit of `count` results and an offset of `startIndex` returns smaller groupings of resources in a request.

> **Note:** The `itemsPerPage`, `startIndex`, and `totalResults` values need to be exchanged as integers, not as strings.

Okta uses `count=100` as the pagination reference to return up to 100 elements. If the value of `totalResults` is higher than 100, then after Okta finishes retrieving the first 100 resources, the `startIndex` becomes `startIndex+100` and is passed as a query parameter along with `count` in a new request to the `/Users` endpoint. This pagination operation repeats until all pages are viewed.

The SCIM server must consistently return the same ordering of results for the requests, regardless of which values are provided for the `count` and `startIndex` pagination references. For more information on pagination, see [Section 3.2.2.3](http://www.simplecloud.info/specs/draft-scim-api-01.html#query-resources) of the V1.1 specification.

A sample request from Okta to retrieve the users from the SCIM app:

```http
GET /scim/v1/Users?startIndex=1&count=100 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

The response to this request is a JSON listing of all the resources found in the SCIM app.

### Retrieving a specific user

**GET** /Users/*$userID*

Okta can also run a GET operation to check if a specific user still exists on the SCIM server. The request looks like the following:

```http
GET /scim/v1/Users/48e0a2da-0999-4f2c-87f4-80432cfe6617 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

The response from the server is the user profile:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 06:54:35 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test",
        "familyName": "User"
    },
    "displayName": "Test User",
    "locale": "en_US",
    "active": true,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "groups": [],
    "meta": {
        "created": "2019-10-18T06:50:23+00:00",
        "lastModified": "2019-10-18T06:50:23+00:00",
        "version": "6d8ef4c9b0ae96fae6623e549f820a60"
    }
}
```

### Updating a specific user (PUT)

Updating a user refers to modifying an attribute in the Okta user profile that is mapped with an attribute in the SCIM application.

To update a user, the functionality needs to be enabled in Okta. In the Okta Admin Console, select the SCIM application from your list of applications. Under the **Provisioning** tab, click **To App**. In the **Update User Attributes** option, click **Enable** and then **Save**.

#### Retrieve the user profile

**GET** /Users/*$userID*

To update a user profile, Okta first makes a GET request to `/Users/${userID}` and retrieves the body of the user's SCIM profile:

```http
GET /scim/v1/Users/48e0a2da-0999-4f2c-87f4-80432cfe6617 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

When the SCIM server receives this request, it responds with the user's SCIM profile:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 06:56:38 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test",
        "familyName": "User"
    },
    "displayName": "Test User",
    "locale": "en_US",
    "active": true,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "groups": [],
    "meta": {
        "created": "2019-10-18T06:50:23+00:00",
        "lastModified": "2019-10-18T06:56:20+00:00",
        "version": "a7fc04c68e3f5551e29a448031d45425"
    }
}
```

#### Update the user profile

**PUT** /Users/*$userID*

After the user's profile is retrieved from the SCIM server, Okta modifies the attributes that were changed and runs a PUT request with the new body to the `/Users/${userID}` endpoint:

```http
PUT /scim/v1/Users/48e0a2da-0999-4f2c-87f4-80432cfe6617 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test1",
        "familyName": "User"
    },
    "displayName": "Test1 User",
    "locale": "en_US",
    "active": true,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "meta": {
        "created": "2019-10-18T06:50:23+00:00",
        "lastModified": "2019-10-18T06:56:20+00:00",
        "version": "a7fc04c68e3f5551e29a448031d45425"
    },
    "groups": []
}
```

The response from the SCIM server needs to be the user's updated SCIM profile:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 06:56:39 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test1",
        "familyName": "User"
    },
    "displayName": "Test1 User",
    "locale": "en_US",
    "active": true,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "groups": [],
    "meta": {
        "created": "2019-10-18T06:50:23+00:00",
        "lastModified": "2019-10-18T06:56:39+00:00",
        "version": "7c9d14512d03434910940e5574f6f8f0"
    }
}
```

### Updating a specific user (PATCH)

**PATCH** /Users/*$userID*

A user resource can be updated through a PATCH operation for the following actions:

* Activating a user
* Deactivating a user
* Syncing the user password

The `active` attribute in a user profile represents the user's current status.

Other updates to attributes in a user profile should be handled through a PUT operation.

To deactivate users, the functionality needs to be enabled in Okta. In the Okta Admin Console, under the **Provisioning** tab, click **To App** and **Edit**. In the **Deactivate Users** option, click **Enable** and then **Save**.

When a user is deactivated, Okta sends this request:

```http
PATCH /scim/v1/Users/48e0a2da-0999-4f2c-87f4-80432cfe6617 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
    "active": false
}
```

The response from the SCIM server needs to be the user's updated SCIM profile:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 07:00:47 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
    "userName": "test.user@okta.local",
    "externalId": "00b443dd40SOK2eup0h7",
    "name": {
        "givenName": "Test1",
        "familyName": "User"
    },
    "displayName": "Test1 User",
    "locale": "en_US",
    "active": false,
    "emails": [{
        "primary": true,
        "value": "test.user@okta.local",
        "type": "work"
    }],
    "groups": [],
    "meta": {
        "created": "2019-10-18T06:50:23+00:00",
        "lastModified": "2019-10-18T07:00:47+00:00",
        "version": "6feac5cd8e2e3a617037e2543e3e83d0"
    }
}
```

> **Note:** The SCIM server response to PATCH operation requests can also be a HTTP 204 response, with no body returned.

### Deleting users

**DELETE** /Users/*$userID*

Okta doesn't perform DELETE operations on users.

If a user is suspended, deactivated, or removed from the application in Okta, then Okta sends a PATCH request to set the `active` attribute to `false`.

## SCIM Group operations

### Creating groups

**POST** /Groups

To create a group on the SCIM server, an Okta administrator must push the group using the Okta Admin Console. In the Okta Admin Console, select the SCIM application from your list of applications. On the **Push Groups** tab, click **Push Groups**. You can select which existing Okta group to push, either by specifying a name or a rule. For more information, see the [Using Group Push topic](https://help.okta.com/en/prod/okta_help_CSH.htm#ext_Directory_Using_Group_Push) in the Okta Help Center.

After the group is selected, Okta makes a POST request to the Service Provider:

```http
POST /scim/v1/Groups HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "displayName": "Test SCIMv1"
}
```

When it receives this request, the SCIM server responds with the group details as it would for a GET operation to the `/Groups/${groupID}/`:

```http
HTTP/1.1 201 Created
Date: Fri, 18 Oct 2019 07:02:44 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "74094a55-c9ee-47ae-9fd4-9137deb43497",
    "displayName": "Test SCIMv1",
    "members": [],
    "meta": {
        "created": "2019-10-18T07:02:44+00:00",
        "lastModified": "2019-10-18T07:02:44+00:00",
        "version": "12ba4e7e158aa0b78d78c7e9ae15d917"
    }
}
```

> **Note:** If the group has a description, it is sent automatically to the SCIM server:

```http
{
"schemas": ["urn:scim:schemas:core:1.0", "urn:okta:custom:group:1.0"],
    "displayName": "Group 10",
    "urn:okta:custom:group:1.0": {
        "description": "All Users West of the Rockies"
    }
}
```

The SCIM server isn't required to save and return the group description. However, there is no option to prevent the description from being sent.

### Retrieving groups

**GET** /Groups

When importing groups from the SCIM server, Okta accesses the `/Groups` endpoint and processes them page by page, using the `startIndex`, `count`, and `totalResults` values for reference. Similarly, when returning large lists of resources, your SCIM implementation must support pagination. Using a limit of `count` results and an offset of `startIndex` returns smaller groupings of resources in a request.

> **Note:** The `itemsPerPage`, `startIndex`, and `totalResults` values need to be exchanged as integers, not as strings.

Okta uses `count=100` as the pagination reference to return up to 100 elements. If the value of `totalResults` is higher than 100, then after Okta finishes retrieving the first 100 resources, the `startIndex` becomes `startIndex+100` and is passed as a query parameter along with `count` in a new request to the `/Groups` endpoint. This pagination operation repeats until all pages are viewed.

The SCIM server must consistently return the same ordering of results for the requests, regardless of which values are provided for the `count` and `startIndex` pagination references.

A sample request from Okta to retrieve the groups from the SCIM app:

```http
GET /scim/v1/Groups?startIndex=1&count=100 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

The response to this request is a JSON listing of all the group resources found in the SCIM app.

### Retrieving specific groups

**GET** /Groups/*$groupID*

There are situations where Okta needs to run a GET operation on a specific `${groupID}`, for example to see if the group still exists on the SCIM server. The request looks like the following:

```http
GET /scim/v1/Groups/74094a55-c9ee-47ae-9fd4-9137deb43497 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

The response from the server is the group details:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 07:08:51 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "74094a55-c9ee-47ae-9fd4-9137deb43497",
    "displayName": "Test SCIMv1",
    "members": [],
    "meta": {
        "created": "2019-10-18T07:02:44+00:00",
        "lastModified": "2019-10-18T07:02:44+00:00",
        "version": "12ba4e7e158aa0b78d78c7e9ae15d917"
    }
}
```

### Updating a specific group name

**PATCH** /Groups/*$groupID*

Updates to existing group names for existing Okta groups are handled by a PATCH operation. The group must already be pushed out to the SCIM server.

```http
PATCH /scim/v1/Groups/74094a55-c9ee-47ae-9fd4-9137deb43497 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "74094a55-c9ee-47ae-9fd4-9137deb43497",
    "displayName": "Test SCIMv11"
}
```

The group name Update operation occurs each time there is a group membership Update operation.

The SCIM server response is to return the updated group details:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 07:11:00 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "74094a55-c9ee-47ae-9fd4-9137deb43497",
    "displayName": "Test SCIMv11",
    "members": [],
    "meta": {
        "created": "2019-10-18T07:02:44+00:00",
        "lastModified": "2019-10-18T07:11:00+00:00",
        "version": "3a0c8354ccfcaab56019859469078a0b"
    }
}
```

> **Note:** The SCIM server response to PATCH operation requests can also be a HTTP 204 response, with no body returned.

### Updating specific group membership

**PATCH** /Groups/*$groupID*

To add or remove users inside a specific pushed group on the SCIM server, Okta requires the following:

* The user must be a member of the group in Okta
* The user has been added under the **Assignments** tab of the SCIM application inside the Okta Admin Console
* The group is pushed under the **Push Groups** tab of the SCIM application inside the Okta Admin Console

If these three requirements are met, Okta sends a request to add the specified users to the group on the SCIM server. The operation can be sent as a PUT or a PATCH operation depending on the configuration of the SCIM application.

```http
PATCH /scim/v1/Groups/74094a55-c9ee-47ae-9fd4-9137deb43497 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "members": [{
        "value": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
        "display": "test.user@okta.local"
    }]
}
```

The SCIM server response is to return the updated group details:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 07:11:01 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "74094a55-c9ee-47ae-9fd4-9137deb43497",
    "displayName": "Test SCIMv11",
    "members": [{
        "value": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
        "display": "test.user@okta.local"
    }],
    "meta": {
        "created": "2019-10-18T07:02:44+00:00",
        "lastModified": "2019-10-18T07:11:01+00:00",
        "version": "cca3a6ab18596cc450377eddcc4b5a58"
    }
}
```

When a user is removed from the group, Okta sends the `members` array, specifying the `${userID}` along with an `operations` element that has the value `delete`.

```http
PATCH /scim/v1/Groups/74094a55-c9ee-47ae-9fd4-9137deb43497 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>

{
    "schemas": [
        "urn:scim:schemas:core:1.0"
    ],
    "members": [
        {
            "value": "b4327d81-fc79-47ad-a7ff-182d9e103291",
            "display": "test.user2@okta.local",
            "operation": "delete"
        }
    ]
}
```

The SCIM server response is to return the updated group details:

```http
HTTP/1.1 200 OK
Date: Fri, 18 Oct 2019 07:11:01 GMT
Content-Type: text/json;charset=UTF-8

{
    "schemas": ["urn:scim:schemas:core:1.0"],
    "id": "74094a55-c9ee-47ae-9fd4-9137deb43497",
    "displayName": "Test SCIMv11",
    "members": [{
        "value": "48e0a2da-0999-4f2c-87f4-80432cfe6617",
        "display": "test.user@okta.local"
    }],
    "meta": {
        "created": "2019-10-18T07:02:44+00:00",
        "lastModified": "2019-10-18T07:11:01+00:00",
        "version": "cca3a6ab18596cc450377eddcc4b5a58"
    }
}
```

> **Note:** The SCIM server response to PATCH operation requests can also be a HTTP 204 response, with no body returned.

### Deleting a specific group

**DELETE** /Groups/*$groupID*

Okta administrators can remove pushed groups from the Okta Admin Console, under the **Push Groups** tab of the SCIM application.

On the **Push Groups** tab, click **Active** then click **Unlink pushed group**. In the dialog box that appears, you can choose whether you want to **Delete the group in the target app** or **Leave the group in the target app** on the SCIM server.

When the admin clicks **Unlink**, Okta sends a DELETE request:

```http
DELETE /scim/v1/Groups/74094a55-c9ee-47ae-9fd4-9137deb43497 HTTP/1.1
User-Agent: Okta SCIM Client 1.0.0
Authorization: <Authorization credentials>
```

The SCIM server can return an empty response:

```http
HTTP/1.1 204 No Content
Date: Fri, 18 Oct 2019 07:16:10 GMT
```

### Additional references

* [What is SCIM?](https://www.okta.com/blog/2017/01/what-is-scim/)
* [SCIM Provisioning using Okta Lifecycle Management](/docs/concepts/scim/)
* [Build a provisioning app using SCIM](/docs/guides/build-provisioning-integration/)
* [SCIM 1.1 RFC: Core Schema](http://www.simplecloud.info/specs/draft-scim-core-schema-01.html)
* [SCIM 1.1 RFC: Protocol](http://www.simplecloud.info/specs/draft-scim-api-01.html)
