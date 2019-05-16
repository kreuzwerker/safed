# Simple AWS Federation for startups and lean organizations (`safed`)

`safed` (pronounced _safety_) is serverless federation utility that's using AWS managed services for it's security critical components. When deployed it enables a federation workflow for console and CLI based operations using OpenID Connect (OIDC) IdP's such as Google's GSuite or Github (planned). It works like this:

1. users navigate to an AWS [ALB](https://aws.amazon.com/elasticloadbalancing/features/#Details_for_Elastic_Load_Balancing_Products) which is configured to authenticate with OIDC and to route it's traffic to the `safed` [λ](https://aws.amazon.com/lambda) function - during this interaction users will be redirected to their IdP (where they may need to log in again or pass multifactor authentication) and then get redirected back to the λ function
2. the λ is invoked with certain headers, one of them containing a [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) token that attests the user has been authenticated
3. the λ calls API's from the IdP to get all the groups or teams the users is in and matches those to roles in organization accounts on the AWS side
4. depending on the mode the λ function is invoked with, the user is either presented with lists of roles she can assume in various accounts (_console_ mode) or receives temporary credentials for CLI operations (_cli_ mode)
5. when assuming a role or building a console link, the λ uses the configured web federation in the respective AWS accounts and assumes the appropriate role, using the token from the OIDC login flow previously passed as header

## Installing

Installing `safed` means one of two things: installing and configuring the infrastructure as well as installing and using the commandline client.

### Infrastructure

TODO (CF / SAM, TF)

#### Configuring GSuite

TODO (create and configure OAuth credentials, configure GSuite)

### Commandline client

TODO (`brew` / relase tab)

Note: the commandline client is build using reproducible Golang builds. Building source code from scratch using the supplied `Makefile` should generate binaries with equivalent hashes. Check the URL path `/version` for the correct Golang and `upx` compiler versions as well as the correct git tag that needs to be checked out from the repository.

## About `safed`

`safed` consists out of the following infrastructure components:

* ALB that receives traffic on
  * port 80: redirects all traffic to port 443
  * port 443: implements the API (see below) over HTTPS with OIDC authentication in place for some of it's paths
* λ that implements the business logic of `safed` in Golang
* IAM role for the λ function with a matching policy that allow listing all roles, their tags, the managed policies and organization members of a _master_ account as well as access to a DynamoDB table (see below)
* IAM cross-account role (to be deployed in all accounts with roles `safed` should be able to provide access to) with a matching policy that allows listing all roles, theirs tags and managed policies in the respective _member_ account
* IAM web-identity federation integration (to be deployed in all accounts with roles `safed` should be able to provide access to)
* IAM roles that use the federation integrations as trusted entities (to be deployed in all accounts with roles `safed` should be able to provide access to) - these are the roles that users will be able to assume
* A DynamoDB table which serves as a cache for the organization accounts and the discovered roles and managed policies

### Mapping roles and policies to users

Roles and polices are mapped to users on two sides:

- On the GSuite sides the mailing list groups a user is in are requested through additional scopes (TODO: document) using the OIDC flow - the λ then [retrieves these groups](https://developers.google.com/admin-sdk/directory/v1/guides/manage-groups#get_all_member_groups) from the GSuite API

- On the AWS side the user itself and / or the groups she is in are mapped to roles and policies through [tags on the roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_tags.html) a user may assume

Note that this mapping mechanism is subject to the [limits](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-limits.html) AWS imposes on tags.

#### Key syntax

Keys identify a subject, e.g. a user email address or a GSuite mailing list. In order to specify the type of subject, prefixes are used:

- `safe:email`: a `verified_email` claim from the OIDC response
- `safe:gsgr`: the name of a GSuite mailing list

#### Value syntax

Tag values can be blank or a comma-separated list of policy ARNs (without the leading `arn:aws:iam` prefix). In case policies are specified, these polices are used to [further restrict](https://aws.amazon.com/blogs/security/create-fine-grained-session-permissions-using-iam-managed-policies/) the effective policy associated with a role.

#### Examples

- `safe:gsgr:Super Admins` → `nil`: allow members of the "Super Admins" group to assume this role without further restriction
- `safe:gsgr:BI Team` → `aws:policy/RDSReadOnlyAccess,policy/IAMReadOnlyAccess`: allow members of the "BI Team" group to assume this role with an effective policy that is the subset of this roles permission and the AWS managed policies `RDSReadOnlyAccess`and `IAMReadOnlyAccess`.  

## Using `safed`

### Console mode usage

- `/` (requires authentication) - starts _console_ mode by presenting an `application/json` object of alphabetically sorted organization accounts and organizational units (OUs) with roles and optionally the managed policies a user has access to; as of now _console_ mode requires a browser that renders a "nice" representation of a JSON tree
- `/cli?port=:port&public_key=:public_key` (requires authentication) - starts _cli_ mode (see "protocol" below)
- `/version` (no authentication) - return the version of λ function
- `.well-known/jwks.json` (no authentication) - returns the public key used for signing JWTs in JWK format (generated on first usage of the λ function)

### CLI mode usage

After being installed the commandline client can be used as a `credential_process` using an [appropriate profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html), making it compatible with tooling build with recent AWS SDKs. The commandline takes the following parameters (in long-, short-form and as environment variable):

- `--duration / -d / SAFED_DURATION` (optional): the duration for which a user wants to assume the role - between `1h` and `12h`, depending on the configuration of the role

- `--role / -r / SAFED_ROLE`: the ARN of the role you want to assume
- `—timeout / -t`: the timeout of the entire process of aquiring credentials (default: `60s`)
- `--url / -u / SAFED_URL`: the URL of the ALB e.g. `aws.mystartup.com`
- `—verbose / -v / SAFED_VERBOSE` : enables verbose mode (default: `false`)

TODO: perhaps we should also support `awsu` style "exec" mode for legacy?

#### Protocol

The commandline protocol retrieves fresh temporary credentials in a format suitable for an `credential_process`. This format is a JSON document looking like this:

```
{
  "Version": 1,
  "AccessKeyId": "an AWS access key",
  "SecretAccessKey": "your AWS secret access key",
  "SessionToken": "the AWS session token for temporary credentials", 
  "Expiration": "ISO8601 timestamp when the credentials expire"
}  
```

 In order to aquire this document, the commandline protocol performs the followings steps:

1. Select random free local port ≥ 10000 and bind an HTTP service to it
2. Create random JWK `ED25519` keypair and store it a secure location in-memory using [memguard](https://godoc.org/github.com/awnumar/memguard)
3. Retrieves the JWK of the API from it's `.well-known` endpoint to get the public signing key of the API
4. Open the local standard web browser and call the API using the query part parameters
   1. `port` passing the free local port
   2. `public_key` passing the public key of the previously generated JWK
5. The API goes through the OIDC workflow and after completion redirects the browser to `localhost` using the `http` schema and the `port` passed in locally - this redirect contains a single query paramter `credentials` which contains a JWE that is encrypted from the API to the public key of previously created random JWK keypair; this JWE is then decrypted, it's signature is verified (using the public key from the JWK) and the `credentials` claims (in the format mentioned) above are extract and returned as-is
6. If the above mentioned steps do not complete within the `--timeout` the process completes with an exit code of `1`
