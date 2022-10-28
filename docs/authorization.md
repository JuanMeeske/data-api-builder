# Authorization

Data API builder uses a role-based authorization workflow. Any incoming request, authenticated or not, is assigned to a role. [Roles](#roles) can be [System Roles](#system-roles) or [User Roles](#user-roles). The assigned role is then checked against the defined [permissions](#permissions) specified in the [configuration file](./configuration-file.md) to understand what actions, fields, and policies are available for that role on the requested entity.

## Roles

Roles, with the exception of the system roles described below, are not pre-defined and are inferred from the claims found in the incoming request.

### System Roles

There are two system roles:

- `anonymous`: all non-authenticated requests will be assigned to the `anonymous` role
- `authenticated`: all authenticated requests will be assigned to the `authenticated` role

### User Roles

An authenticated request comes with a set of role claims that describe the requestor's role membership. When using EasyAuth authentication, the received token can be something like the following:

```json
{
  "identityProvider": "github",
  "userId": "d75b260a64504067bfc5b2905e3b8182",
  "userDetails": "username",
  "userRoles": ["author"],
  "claims": [{
    "typ": "name",
    "val": "Neo"
  }]
}
```

the roles will be used to match any defined role in the configuration file. For example, a request coming in with the aforementioned sample token will be matched with the `author` permissions if the sample `book` entity is configured like the following:

```json
"book": {
    "source": "books",
    "permissions": [
        {
            "role": "anonymous",
            "actions": [ "read" ]
        },
        {
            "role": "author",
            "actions": [ "*" ]
        }
    ]
}
```

More specifically, the above configuration is telling Data API Builder that it must allow the requestor to

- read data from the underlying database object to any non-authenticated request
- perform any CRUD operation on the underlying database object if the user making the request is in the `author` role, which is the case of the sample token mentioned before.

### Roles selection

For a request to be evaluated in the context of a user role, the request must include the `X-MS-API-ROLE` HTTP Header and set the value to a role present in the received authentication token. If the received authentication token has the following contents:

```json
{
  "identityProvider": "github",
  "userId": "d75b260a64504067bfc5b2905e3b8182",
  "userDetails": "username",
  "userRoles": ["author"],
  "claims": [{
    "typ": "name",
    "val": "Neo"
  }]
}
```

and the request must be evaluated in the context of the `author` role, then the `X-MS-API-ROLE` must be set to `author`.

A request can only be evaluated in the context of a single role. So, if the authentication token allows for more than one role, for example:

```json
"userRoles": ["author", "editor"]
```

the desired role must be specified in the `X-MS-API-ROLE` HTTP Header.

If `X-MS-API-ROLE` is not specified for an authenticated request, the request is assumed to be evaluated in the context of the `authenticated` system role.

## Permissions

Permissions and their components,  `roles`, `actions`, `fields` and `policies`, are explained in the [configuration file](./configuration-file.md#permissions) documentation.

There can be multiple roles defined in an entity's permissions configuration. However, a request is only evaluated in the context of a single role. The role evaluated for a request is either a system role automatically assigned by the Data API Builder engine or a role manually specified in the `X-MS-API-ROLE` HTTP header.

### Secure by default

By default, an entity has no permissions configured, which means that no role is allowed to perform any actions on the entity.

To allow anonymous access to an entity, for example the `book` entity, the `anonymous` permission must be defined. For example:

```json
"book": {
  "source": "dbo.books",
  "permissions": [{
    "role": "anonymous",
    "actions": [ "read" ]
  }]
}
```

To simplify permissions definition on an entity, it is assumed that if there are no specific permissions for the `authenticated` role, then the permission defined for the `anonymous` role are used. The `book` configuration shown before will therefore allow any anonymous or authenticated requests to perform read operations on the `book` entity. If only authenticated requests should be able to perform a read operation on the book entity, then the following configuration must be used:

```json
"book": {
  "source": "dbo.books",
  "permissions": [{
    "role": "authenticated",
    "actions": [ "read" ]
  }]
}
```

With such configuration any anonymous request will be denied as there are not permission set for the `anonymous` role.

As by default there are no pre-defined permission for the `anonymous` or `authenticated` roles, if only a specific role must be allowed to operate on an entity, you only need to define the permission for that role. All other roles, both system or user defined, will be automatically denied access:

```json
"book": {
  "source": "dbo.books",
  "permissions": [{
    "role": "administrator",
    "actions": [ * ]
  }]
}
```

In the above configuration sample, only request carrying the `administrator` role in the authentication token and specifying the `administrator` value in the `X-MS-API-ROLE` HTTP header, will be able to operate on the `book` entity.