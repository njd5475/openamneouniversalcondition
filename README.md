# OpenAM Universal Neo Policy Evaluation Plugin

This project introduces an OpenAM plugin library
that implements ....

**Note**

The current OpenAM version that the project depends on is OpenAM 13.0.0-SNAPSHOT, but the plugin has been tested to workwith OpenAM 12.0.0 and OpenAM 12.0.2 as well.


## About the Plugin

.... 


## Building the Plugin

Before building the plugin,
update the POM property `<openam.version>` to match your OpenAM version.

The line to update is:

    <openam.version>13.0.0-SNAPSHOT</openam.version>

Build the plugin using Apache Maven.

    mvn install


## Installing the Plugin

After successfully building the plugin,
copy the library to the `WEB-INF/lib/` directory where you deployed OpenAM.
For OpenAM deployed on Apache Tomcat under `/openam`:

    cp target/*.jar /path/to/tomcat/webapps/openam/WEB-INF/lib/

Next, edit the `policyEditor/locales/en/translation.json` file
to add the strings used by the policy editor
so that the policy editor shows the custom subject and condition.

    "conditionTypes": {
      ...
       "NeoUniversal": {
              "title": "Neo Universal Condition",
              "props": {
                  "dbURL": "DB Transactional Endpoint URL",
                  "dbUsername": "DB Username",
                  "dbPassword": "DB Password",
                  "cypherQuery": "Cypher Query",
                  "paramsJson": "Query Parameters (JSON)",
                  "allowCypherResult": "Cypher Result for Allow-Access",
                  "denyCypherResult": "Cypher Result for Deny-Access"
              }
          },
        ...

Restart OpenAM or the container in which it runs.

    /path/to/tomcat/bin/shutdown.sh
    ...
    /path/to/tomcat/bin/startup.sh

Your custom policy plugin can now be used for new policy applications.


## Adding Custom Policy Implementations to Existing Policy Applications

In order to use your custom policy in existing applications,
you must update the applications.
Note that you cannot update an application that already has policies configured.
When there are already policies configured for an application,
you must instead first delete the policies, and then update the application.

The following example updates the `iPlanetAMWebAgentService` application
in the top level realm of a fresh installation.

    curl \
     --request POST \
     --header "X-OpenAM-Username: amadmin" \
     --header "X-OpenAM-Password: password" \
     --header "Content-Type: application/json" \
     --data "{}" \
     http://openam.example.com:8080/openam/json/authenticate

    {"tokenId":"AQIC5wM2...","successUrl":"/openam/console"}

    curl \
     --request PUT \
     --header "iPlanetDirectoryPro: AQIC5wM2..." \
     --header "Content-Type: application/json" \
     --data '{
        "name": "iPlanetAMWebAgentService",
        "resources": [
            "*://*:*/*?*",
            "*://*:*/*"
        ],
        "actions": {
            "POST": true,
            "PATCH": true,
            "GET": true,
            "DELETE": true,
            "OPTIONS": true,
            "HEAD": true,
            "PUT": true
        },
        "description": "The built-in Application used by OpenAM Policy Agents.",
        "realm": "/",
        "conditions": [
            "AuthenticateToService",
            "AuthLevelLE",
            "AuthScheme",
            "IPv6",
            "SimpleTime",
            "OAuth2Scope",
            "IPv4",
            "AuthenticateToRealm",
            "OR",
            "AMIdentityMembership",
            "LDAPFilter",
            "AuthLevel",
            "SessionProperty",
            "Session",
            "NOT",
            "AND",
            "ResourceEnvIP",
            "NeoUniversal"
        ],
        "resourceComparator": null,
        "applicationType": "iPlanetAMWebAgentService",
        "subjects": [
            "JwtClaim",
            "AuthenticatedUsers",
            "Identity",
            "NOT",
            "AND",
            "NONE",
            "OR"
        ],
        "attributeNames": [],
        "saveIndex": null,
        "searchIndex": null,
        "entitlementCombiner": "DenyOverride"
    }' http://openam.example.com:8088/openam/json/applications/iPlanetAMWebAgentService

Notice that the command adds `"NeoUniversal"` to `"conditions"`.


## Trying the Environment Condition

# 'TODO change the following for a graphDB example ....'



Using OpenAM policy editor, create a policy
in the "iPlanetAMWebAgentService" of the top level realm
that allows HTTP GET access to `"http://www.example.com:80/*"`
and that makes use of the custom subject and condition.

    {
        "name": "Sample Policy",
        "active": true,
        "description": "Try sample policy plugin",
        "resources": [
            "http://www.example.com:80/*"
        ],
        "applicationName": "iPlanetAMWebAgentService",
        "actionValues": {
            "GET": true
        },
        "subject": {
            "type": "SampleSubject",
            "name": "demo"
        },
        "condition": {
            "type": "SampleCondition",
            "nameLength": 4
        }
    }

With the policy in place, authenticate
both as a user who can request policy decisions
and also as a user trying to access a resource.
Both of these calls return "tokenId" values
for use in the policy decision request.

    curl \
     --request POST \
     --header "X-OpenAM-Username: amadmin" \
     --header "X-OpenAM-Password: password" \
     --header "Content-Type: application/json" \
     --data "{}" \
     http://openam.example.com:8080/openam/json/authenticate

     {"tokenId":"AQIC5wM2LY4Sfcw...","successUrl":"/openam/console"}

    curl \
     --request POST \
     --header "X-OpenAM-Username: demo" \
     --header "X-OpenAM-Password: changeit" \
     --header "Content-Type: application/json" \
     --data "{}" \
     http://openam.example.com:8080/openam/json/authenticate

     {"tokenId":"AQIC5wM2LY4Sfcy...","successUrl":"/openam/console"}

Use the administrator token ID as the header of the policy decision request,
and the user token Id as the subject "ssoToken" value.

    curl \
     --request POST \
     --header "iPlanetDirectoryPro: AQIC5wM2LY4Sfcw..." \
     --header "Content-Type: application/json" \
     --data '{
        "subject": {
          "ssoToken": "AQIC5wM2LY4Sfcy..."},
        "resources": [
            "http://www.example.com:80/index.html"
        ],
        "application": "iPlanetAMWebAgentService"
     }' \
     http://openam.example.com:8080/openam/json/policies?_action=evaluate

     [
         {
             "resource": "http://www.example.com:80/index.html",
             "actions": {
                 "GET": true
             },
             "attributes": {},
             "advices": {}
         }
     ]

To use the custom resource attribute, add "resourceAttributes" to the policy.

    curl \
     --request PUT \
     --header "iPlanetDirectoryPro: AQIC5wM2LY4Sfcw..." \
     --header "Content-Type: application/json" \
     --data '{
        "name": "Sample Policy",
        "active": true,
        "description": "Try sample policy plugin",
        "resources": [
            "http://www.example.com:80/*"
        ],
        "applicationName": "iPlanetAMWebAgentService",
        "actionValues": {
            "GET": true
        },
        "subject": {
            "type": "SampleSubject",
            "name": "demo"
        },
        "condition": {
            "type": "SampleCondition",
            "nameLength": 4
        },
        "resourceAttributes": [
            {
                "type": "SampleAttribute",
                "propertyName": "test"
            }
        ]
    }' http://openam.example.com:8088/openam/json/policies/Sample%20Policy

When you now request a policy decision,
the plugin also returns the "test" attribute that you configured.

    curl \
     --request POST \
     --header "iPlanetDirectoryPro: AQIC5wM2LY4Sfcw..." \
     --header "Content-Type: application/json" \
     --data '{
        "subject": {
          "ssoToken": "AQIC5wM2LY4Sfcy..."},
        "resources": [
            "http://www.example.com:80/index.html"
        ],
        "application": "iPlanetAMWebAgentService"
     }' \
     http://openam.example.com:8080/openam/json/policies?_action=evaluate

    [
        {
            "resource": "http://www.example.com/profile",
            "actions": {
                "GET": true
            },
            "attributes": {
                "test": [
                    "sample"
                ]
            },
            "advices": {}
        }
    ]

If you made it this far, then you have successfully tested this plugin.
Good luck building your own custom policy plugins!

* * * * *

Everything in this repository is licensed under the ForgeRock CDDL license:
<http://forgerock.org/license/CDDLv1.0.html>

Copyright 2013-2014 ForgeRock AS

