Red Hat 3scale API Gateway Set-Up Procedure
===========================================

Required Resources
------------------

For the deployment of 3scale on OCP, we need the S3 3scale AMP Template for
OpenShift, available from the
[templates section of the 3scale GitHub repository](https://github.com/3scale/3scale-amp-openshift-templates).

More information on how to deploy and configure 3scale is available from
[Red Hat 3scale Installation Guide](https://access.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.5/html-single/installing_3scale/index).

Deploying the API Gateway
-------------------------

First, obviously, a project needs to be created:

    $ oc new-project 3scale

We'll be using the GitHub repository:

    $ export API_GIT_URL=https://raw.githubusercontent.com/3scale/3scale-amp-openshift-templates/master

Create the template. We need the S3 template on AWS.

    $ oc create -f ${API_GIT_URL}/amp/amp-s3.yml

***NOTE***: Technically, we could skip the above step and simply use the
``--file`` option to ``oc new-app``, pointing it at the above repository URL.

The creation of the deployment is then very simple:

    $ oc new-app --name=api-gw \
            --template=3scale-api-management-s3 \
            -p WILDCARD_DOMAIN=api.apps.agile.ocp.aws.p0f.net \
            -p WILDCARD_POLICY=Subdomain

It takes a couple of minutes for all the components to deploy.

Post-Install Configuration
--------------------------

### Configuring Red Hat SSO for 3scale

After deploying 3scale, we need to configure SSO integration.

In the internal SSO, create a new client:

    Realm: Ocp-agile
        -> Clients
        [Create]

    Client ID: 3scale-sso
    Client Protocol: openid-connect
    [Save]

In the subsequent client configuration, make sure the following settings apply:

    Enabled: ON
    Access Type: confidential
    Standard Flow Enabled: ON
    Implicit Flow Enabled: OFF
    Direct Access Grants Enabled: OFF
    Service Accounts Enabled: ON
    Authorization Enabled: OFF
    Valid Redirect URIs:
        https://3scale-admin.api.apps.agile.ocp.aws.p0f.net/*
        https://3scale.api.apps.agile.ocp.aws.p0f.net/*         # only needed for developer portal
    Advanced Settings:
        Access Token Lifespan: 1 Minutes
    [Save]

Change the service account roles in the ``Service Account Roles`` tab:

    Client Roles: realm-management
    Available Roles: manage-clients
    [Add selected >>]

Obtain the client secret from the ``Credentials`` tab.

Configure the ``email_verified`` mapper in the ``Mappers`` tab.

    [Add Builtin]
        -> email verified: CHECK
    [Add Selected]

To automatically map Users to Organizations, create an ``org_name`` mapper in
the ``Mappers`` tab:

    [Create]
        Name: org name
        Mapper Type: User Attribute
        User Attribute: org_name
        Token Claim Name: org_name
        Claim JSON Type: String
        Add to ID token: ON
        Add to access token: ON
        Add to userinfo: ON
        Multivalued: OFF
        Aggregate attribute values: OFF
    [Save]

Now the ``org_name`` attribute could be used to automatically assign any user
to an organization within 3scale (*if it worked*).

### Configuring 3scale API Management for Red Hat SSO

In the 3scale management console, configure the following:

    Account Settings
        -> Users
        -> SSO Integrations
        [New SSO Integration]

    SSO Provider: Red Hat Single Sign-On
    Client: 3scale-sso
    Client Secret: c6...cc
    Realm or Site: https://sso-internal.apps.agile.ocp.aws.p0f.net/auth/realms/ocp-agile
    Do not verify SSL certificate: YES
    [Save]

After saving, test OAuth flow by clicking the ``Test authentication flow``
link. Log in as some existing user.

If successful, make the SSO provider active by clicking ``Publish``.

For accessing 3scale, users do not need any roles. However, to do anything
useful, they have to be assigned at least one role **in the SSO admin
console**.

### Configuring 3scale Developer Portal for Red Hat SSO

In the 3scale management console, configure the following:

    Audience
        -> Developer Portal
        -> SSO Integrations
        [Red Hat Single Sign-On]

Fill in the settings:

    Client: 3scale-sso
    Client Secret: c6...cc
    Realm or Site: https://sso-internal.apps.agile.ocp.aws.p0f.net/auth/realms/ocp-agile
    Do not verify SSL certificate: YES
    Published: YES
    [Save]

Testing the portal is possible at
<https://3scale.api.apps.agile.ocp.aws.p0f.net/>, but only **after** the
``Developer Portal Access Code`` had been removed from ``Audience -> Developer
Portal -> Domains & Access``.

