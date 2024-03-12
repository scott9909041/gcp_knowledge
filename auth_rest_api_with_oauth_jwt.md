# Authentication of GCP REST API with JWT

# Context

The story starts from a requirement of keeping files on GCP cloud storage to save the cost on a third party platform. The task is to create a general solution and best practice of authentication can be followed since.

# Solution Overview
There are many ways to involve GCP resource. Through client library, GCP command line tool and REST API. Since the third-party platform is built with their own language, it is not possible to use the client library and GCP command line tool. As far as we know every operation can be perform only through REST API call. So, the focus of this article is to explain how to authenticate REST API fired from the platform. An overview step is described as follow.

Preparation:
* Create necessary service accounts on GCP
* Store key file safely on the platform

At runtime:
* Create a OAuth2.0 JWT
* Exchange for an access token
* Call GCP REST API to perform operations


# Solution In Detail
## The Design Of Service Account Management
Since the service account could be granted with various kind of permission and the key file stored on a 3rd party platform. Leaking the key file could lead to serious consequences. So we design to create 2 service account.

* The caller service account
* The privilege service account

The caller service account only holds permission that can request access token representing the privilege service account. And the privilege service account is the one that possesses various of GCP permissions. Only the key file of the caller service account is uploaded and stored on the platform. We will discuss more in detail.
## Create necessary service accounts and setup permission
Since the knowledge of GCP is out of scope, please follow the [google documentation](https://cloud.google.com/docs/authentication/use-service-account-impersonation) of this step.
## Store key file safely
This is also out of scope, please check documentation of the platform.


After the preparation steps are completed. Let's discuss how to implement the functionality.
## Create a OAuth2.0 JWT

Now you are implementing a functionality that needs to involve GCP, first step is to retrieve the content of key file which you uploaded to the platform. Then please look for a JWT library which is compatible with the platform and follow the sample code below to create JWT.

The sample code is in node.js


```javascript
import * as jose from 'jose';

const alg = 'RS256';
const header = {
    alg,
    typ: 'JWT',
    kid: '#{The private_key_id field in key.json}'
};
const payload = {
	scope: 'https://www.googleapis.com/auth/cloud-platform',
	iss: '#{The client_email field in key.json}',
	aud: 'https://oauth2.googleapis.com/token',
	// Issued at, a number value represents the seconds from UTC.
	iat: 17215454,
	// Expire time.
	exp: 17216565
}
const jwt = await new jose.SignJWT(payload).sign('#{private_key field in key file}')
console.log(jwt)
```

## Request An Access Token From Google Auth Server
Fire a API to auth server of Google to request an access token. These are the key metrics.

API information:

| Type        | Value                               |
| ----------- | ----------------------------------- |
| Endpoint    | https://oauth2.googleapis.com/token |
| Http Method | POST                                |
| Content-Type | application/x-www-form-urlencoded |

Request body:

| Key        | Value                                       |
| ---------- | ------------------------------------------- |
| grant_type | urn:ietf:params:oauth:grant-type:jwt-bearer |
| assertion | #{The JWT you just created} |

The API should return format like:

| Key | Type |
| -------- | -------- |
| access_token | string |
| expires_in | number |
| token_type | "Bearer" |

Now you receive the access token. **Remember, this token represents the caller service account, which can only be used to request access token of the privilege service account.**

## Request An Access Token of The Privilege Service Account
If you setup the service account and permissions correctly, you should be able to get another access token directly by call the below API.

API information:

| Type        | Value                               |
| ----------- | ----------------------------------- |
| Endpoint    | https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/#{account id}:generateAccessToken |
| Http Method | POST                                |
| Content-Type | application/json |

**Field the account id with the id of the privilege service account.**

HTTP header:
| Key | Value |
| -------- | -------- |
| Authorization | "Bearer #{access token}" |

**Field the access token of the caller service account**

Request body:

| Key | Type | Value |
| -------- | -------- | -------- |
| scope | string or string[] | Refer to https://developers.google.com/identity/protocols/oauth2/scopes |

The API should return format like:

| Key | Type |
| -------- | -------- |
| access_token | string |
| expires_in | number |
| token_type | "Bearer" |

## Perform the GCP operations you like
Congratulations! The complex part is over. The access token you get represents the privilege service account, which can be used to perform any operations that is allowed with GCP IAM. If you encounter an 403 error, please check if you have set your IAM permission correctly. If you encounter a 401, please check if you have exactly follow the format when creating JWT, such as the "scope" field. Or check if the JWT has expired.

To locate the API information, please refer to Google REST API documentation. For example, if you want to operate cloud storage, visit [here](https://cloud.google.com/storage/docs/json_api).

