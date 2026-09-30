![](images/logo.png?raw=true)
# GLPI webhook

## Overview

This guide describes how to integrate your Zabbix installation with your GLPI installation using the Zabbix webhook feature, providing instructions on setting up a media type, user, and action in Zabbix.

The webhook supports both legacy REST API (V1) and RESTful API (V2).
This webhook creates tickets in the GLPI Assistance section. Created tickets have the following urgency mapping:

|Severity in Zabbix|Urgency in GLPI|
|-|-|
0 - Not classified| Very low|
1 - Information| Very low|
2 - Warning| Low|
3 - Average| Medium|
4 - High| High|
5 - Disaster| Very high|

- When a problem is created in Zabbix, the webhook creates a ticket with the status "New" and sets the webhook user (or the user defined in the `glpi_requester_id` parameter) as the ticket requester. The Zabbix event ID is added to the ticket description.
- When a problem is updated in Zabbix, the webhook updates the ticket's urgency in GLPI and adds a followup entry with the update comment. The ticket title is not changed.
- When a problem is resolved in Zabbix, the webhook updates the ticket's title and adds a solution with resolution details. GLPI then sets the ticket status to "Solved" - this works regardless of the requesters, observers or assignees already set on the ticket.
- Due to the specifics of the webhook, the number of retries is, by default, set to 1. We recommend not changing this setting; should a transaction error occur, additional duplicate objects (tickets, followups) may be created during the retry.

## Tested on
- GLPI 11.0.10 (RESTful API V2 and REST API V1), 12.0.0-rc3 (RESTful API V2)

## Requirements

Zabbix version: 7.4 and higher.

## Parameters

After importing the webhook, you can configure it using webhook parameters.

### Configurable parameters

The configurable parameters are intended to be changed according to the webhook setup as well as the user's preferences and environment.

|Name|Value|Description|
|----|-----|-----------|
|tls_verify|\{$HTTP\.TLS\.VERIFY:"GLPI"\}|TLS certificate verification for HTTP requests: "none" - disabled, "peer" - verify the certificate chain and expiration, "full" - full verification. Any other value enables full verification. To override the setting for this media type only, define the global macro with the context "GLPI", e.g. {$HTTP.TLS.VERIFY:"GLPI"}.|
|zabbix_url|\{$ZABBIX\.URL\}|Current Zabbix URL.|
|glpi_legacy_api|false|Boolean value (true/false) to set API version: `false` (default) enables Modern API v2 with OAuth2 Bearer token authentication, `true` enables Legacy API v1 with Session-Token authentication.|
|glpi_app_token||GLPI application token (optional; specify if the token is set in the API client settings).|
|glpi_user_token|\<PLACE GLPI USER TOKEN\>|GLPI user token.|
|glpi_client_id|\<PLACE GLPI CLIENT ID\>|GLPI client ID.|
|glpi_client_secret|\<PLACE GLPI CLIENT SECRET\>|GLPI client secret.|
|glpi_username|\<PLACE GLPI USERNAME\>|GLPI username.|
|glpi_password|\<PLACE GLPI USER PASSWORD\>|GLPI user password.|
|glpi_url|\<PLACE GLPI URL\>|URL of GLPI installation.|
|glpi_requester_id||GLPI user ID to set as the ticket requester (optional). If empty, the webhook user is used (detected via the API session).|
|glpi_debug||Set to `1` to report errors when setting the ticket requester as failed notifications in the action log (optional). If empty, such errors are only logged and the ticket is created without a requester. Use it for troubleshooting only: when the notification fails, the ticket ID is not stored in the problem tags, so later updates and the recovery of that problem are not sent to the ticket.|
|glpi_urgency_autoregistration|Low|String value of GLPI urgency to assign to autoregistration event tickets.|
|glpi_urgency_discovery|Low|String value of GLPI urgency to assign to discovery event tickets.|
|glpi_urgency_internal|Low|String value of GLPI urgency to assign to internal event tickets.|
|severity_not_classified|Very low|GLPI urgency to assign to tickets when event has Zabbix severity "Not classified".|
|severity_information|Very low|GLPI urgency to assign to tickets when event has Zabbix severity "Information".|
|severity_warning|Low|GLPI urgency to assign to tickets when event has Zabbix severity "Warning".|
|severity_average|Medium|GLPI urgency to assign to tickets when event has Zabbix severity "Average".|
|severity_high|High|GLPI urgency to assign to tickets when event has Zabbix severity "High".|
|severity_disaster|Very high|GLPI urgency to assign to tickets when event has Zabbix severity "Disaster".|

### Internal parameters

Internal parameters are reserved for predefined macros that are not meant to be changed.

|Name|Value|Description|
|----|-----|-----------|
|event_source|\{EVENT\.SOURCE\}|Numeric value of the event source. Possible values: 0 - Trigger, 1 - Discovery, 2 - Autoregistration, 3 - Internal, 4 - Service.|
|event_value|\{EVENT\.VALUE\}|Numeric value of the event that triggered an action (1 for problem, 0 for recovering).|
|event_severity|\{EVENT\.SEVERITY\}|Name of the event severity.|
|event_nseverity|\{EVENT\.NSEVERITY\}|Numeric value of the event severity. Possible values: 0 - Not classified, 1 - Information, 2 - Warning, 3 - Average, 4 - High, 5 - Disaster.|
|event_update_nseverity|\{EVENT\.UPDATE\.NSEVERITY\}|Numeric value of the event update severity. Possible values: 0 - Not classified, 1 - Information, 2 - Warning, 3 - Average, 4 - High, 5 - Disaster.|
|event_update_severity|\{EVENT\.UPDATE\.SEVERITY\}|Name of the event update severity.|
|event_update_status|\{EVENT\.UPDATE\.STATUS\}|Numeric value of the problem update status. Possible values: 0 - Webhook was called because of problem/recovery event, 1 - Update operation.|
|alert_subject|\{ALERT\.SUBJECT\}|'Default subject' value from action configuration.|
|alert_message|\{ALERT\.MESSAGE\}|'Default message' value from action configuration.|
|event_id|\{EVENT\.ID\}|Numeric ID of the event that triggered an action.|
|trigger_id|\{TRIGGER\.ID\}|Numeric ID of the trigger of this action.|
|glpi_problem_id|\{EVENT\.TAGS\.\_\_zbx\_glpi\_problem\_id\}|GLPI problem ID.|

> Please be aware that each webhook supports an HTTP proxy. To use this feature, add a new media type parameter with the name `http_proxy` and set its value to the proxy URL.

## Service setup

### RESTful API (V2) with OAuth2 - recommended configuration for GLPI 11+:

1. Enable access to the GLPI API:
  - In the GLPI web interface, go to *Setup* > *General* > *API*.
  - Switch the toggle to activate *Enable API* and click the *Save* button.

[![](images/thumb.1.png?raw=true)](images/1.png)

2. Add an [OAuth client](https://help.glpi-project.org/documentation/modules/configuration/oauth-clients):
  - Go to *Setup* > *OAuth clients*.
  - Click the *Add* button on the top of the page.
  - Set the client name; enter *api* in the *Scopes* field and *Password* in *Grants*.
  - Click the *Add* button.
  - Open the settings of the created client, and then copy and save the client ID and client secret.

[![](images/thumb.2.png?raw=true)](images/2.png)

3. Create a new [user profile](https://glpi-user-documentation.readthedocs.io/fr/latest/modules/administration/profiles/profiles.html) with permissions to create tickets and followups (alternatively, you can use an existing profile with sufficient privileges):
  - Go to *Administration* > *Profiles* and click the *Add* button on the top of the page.
  - Specify the profile name and set the *Profile's Interface* option to *Standard Interface*, and then click the *Add* button.
  - Open the created profile and click the *Assistance* tab.
  - In the *Tickets* section, set the *Update*, *Create*, *See all tickets*, and *Assign* permissions.
  - In the *Followups/Tasks* section, set the *Add (Requester)* permission for the *Followups* row.
  - Click the *Save* button.

[![](images/thumb.4.png?raw=true)](images/4.png)
[![](images/thumb.5.png?raw=true)](images/5.png)
[![](images/thumb.6.png?raw=true)](images/6.png)

> The *Assign* permission is required to set the ticket requester through RESTful API (V2): the webhook adds the requester as a ticket team member, which GLPI allows only for users who can assign tickets. Without it, tickets are created without a requester.

4. Create a new [user](https://glpi-user-documentation.readthedocs.io/fr/latest/modules/administration/users/users.html):
  - Go to *Administration* > *Users* and click the *Add* button on the top of the page.
  - Specify the user login and set the *Authorization* > *Profile* option to the profile you created in the previous step (or any other existing profile with permissions to create tickets and followups).
  - Set the password for the user.
  - Click the *Add* button.

[![](images/thumb.7.png?raw=true)](images/7.png)

> GLPI 11+ continues to support REST API (V1) without requiring OAuth2/V2. Note that V1 is not recommended for new integrations.

### REST API (V1) - legacy configuration for GLPI 10:

1. Enable access to the GLPI REST API:
  - In the GLPI web interface, go to *Setup* > *General* > *API*.
  - Set the *Enable Rest API* and *Enable login with external token* options to *Yes* and click the *Save* button.

[![](images/thumb.legacy_1.png?raw=true)](images/legacy_1.png)

2. Add a new API client:
  - Click the *Add API client* button.
  - Specify the API client name and set the *Active* option to *Yes*.
  - For security reasons, you may want to restrict the API client to the IP address of Zabbix server and/or create an additional application token (will be generated by default; you can uncheck the *Regenerate* checkbox if you don't want to use it).
  - Click the *Add* button.
  - If you've opted to create an application token, open the settings of the created API client, and then copy and save the generated application token.

[![](images/thumb.legacy_2.png?raw=true)](images/legacy_2.png)
[![](images/thumb.legacy_3.png?raw=true)](images/legacy_3.png)

3. Create a new [user profile](https://glpi-user-documentation.readthedocs.io/fr/latest/modules/administration/profiles/profiles.html) with permissions to create tickets and followups (alternatively, you can use an existing profile with sufficient privileges):
  - Go to *Administration* > *Profiles* and click the *Add* button on the top of the page.
  - Specify the profile name and set the *Profile's Interface* option to *Standard Interface*, and then click the *Add* button.
  - Open the created profile and click the *Assistance* tab.
  - Set the *Update*, *Create*, and *See all tickets* permissions in the *Tickets* section.
  - Set the *Add followup (Requester)* permission for the *Followups* line in the *Followups/Tasks* section.
  - Click the *Save* button.

[![](images/thumb.4.png?raw=true)](images/4.png)
[![](images/thumb.legacy_5.png?raw=true)](images/legacy_5.png)
[![](images/thumb.legacy_6.png?raw=true)](images/legacy_6.png)

4. Create a new [user](https://glpi-user-documentation.readthedocs.io/fr/latest/modules/administration/users/users.html):
  - Go to *Administration* > *Users* and click the *Add User* button on the top of the page.
  - Specify the user login and set the *Profiles* option to the profile that you created in the previous step (or any other existing profile with permissions to create tickets and followups).
  - Click the *Add* button.
  - Open the profile of the created user and check the *Regenerate* checkbox of the *API token* option; click *Save*.
  - Copy and save the generated user API token.

[![](images/thumb.legacy_7.png?raw=true)](images/legacy_7.png)
[![](images/thumb.legacy_8.png?raw=true)](images/legacy_8.png)
[![](images/thumb.legacy_9.png?raw=true)](images/legacy_9.png)

## Zabbix configuration

1. Before you can start using the GLPI webhook, you need to set the global macro `{$ZABBIX.URL}`:
  - In the Zabbix web interface, go to *Administration* > *Macros* in the top-left drop-down menu.
  - Set up the global macro `{$ZABBIX.URL}` which will contain the URL to the Zabbix frontend. The URL should be either an IP address, a fully qualified domain name, or a localhost.
  - Specifying a protocol is mandatory, whereas the port is optional. Depending on the web server configuration, you might also need to append `/zabbix` to the end of the URL. Good examples:
    - `http://zabbix.com`
    - `https://zabbix.lan/zabbix`
    - `http://server.zabbix.lan/`
    - `http://localhost`
    - `http://127.0.0.1:8080`
  - Bad examples:
    - `zabbix.com`
    - `http://zabbix/`

[![](images/thumb.8.png?raw=true)](images/8.png)

2. Import the media type:
  - In the *Alerts* > *Media types* section, import the [`media_glpi.yaml`](media_glpi.yaml) file.

3. It is also possible to set GLPI ticket urgency by parameters. Predefined parameter values are already in place; note that they use the default GLPI urgency.

**Please adjust these values to suit your GLPI environment.**

The following parameters apply to Zabbix events that support severities:
  - `severity_not_classified` - for Zabbix severity "Not Classified"
  - `severity_information` - for Zabbix severity "Information"
  - `severity_warning` - for Zabbix severity "Warning"
  - `severity_average` - for Zabbix severity "Average"
  - `severity_high` - for Zabbix severity "High"
  - `severity_disaster` - for Zabbix severity "Disaster"

And the following for Zabbix events that do not have severities:
  - `glpi_urgency_internal` - for Zabbix internal events
  - `glpi_urgency_discovery` - for Zabbix discovery events
  - `glpi_urgency_autoregistration` - for Zabbix autoregistration events

4. Open the imported GLPI media type and set the following webhook parameters:
  - `glpi_url` - the frontend URL of your GLPI installation, without any path suffix (e.g. http://glpi.example.com:8080).
  - `glpi_legacy_api` - determine whether you are using RESTful API V2 with OAuth2 authorization (default, parameter value `false`) or legacy API (parameter value `true`).

  If using RESTful API V2 with OAuth2, set the required parameters:
  - `glpi_client_id` - the client ID that was generated during the creation of the OAuth client.
  - `glpi_client_secret` - the client secret that was generated during the creation of the OAuth client.
  - `glpi_username` - GLPI username created for the webhook.
  - `glpi_password` - password for the GLPI webhook user.

[![](images/thumb.9.png?raw=true)](images/9.png)

  If using legacy API, set the parameters:
  - `glpi_app_token` - if you've opted to use an application token during the creation of API client, specify it here; otherwise leave it empty.
  - `glpi_user_token` - the user token that was generated during the creation of the GLPI user.

[![](images/thumb.10.png?raw=true)](images/10.png)

  Optionally, set the following parameters (for both API versions):
  - `glpi_requester_id` - ID of the GLPI user to set as the ticket requester. Leave it empty to use the webhook user.
  - `glpi_debug` - set to `1` to report errors when setting the ticket requester as failed notifications. Leave it empty in production.

5. Click the *Enabled* checkbox to enable the media type and click the *Update* button to save the webhook settings.

6. Create a dedicated Zabbix user and add media:
  - To create a new user,  go to the *Users* > *Users* section and click the *Create user* button in the top-right corner. In the *User* tab, fill in all the required fields (marked with red asterisks).
  - In the *Media* tab, click *Add* and select *GLPI* from the *Type* drop-down list. Though the *Send to* field is not used in the GLPI webhook, it cannot be left empty. To comply with frontend requirements, enter any symbol in the field.
  - Make sure this user has access to all the hosts for which you would like problem notifications to be sent to GLPI.
  - Use a user that is not used to update problems (acknowledge, comment, change severity). Zabbix does not send update notifications to the user who performed the update, so if that user is the only recipient, problem updates are never sent to GLPI.

[![](images/thumb.11.png?raw=true)](images/11.png)

7. Configure the action:
  - Go to *Alerts* > *Actions* > *Trigger actions* and open (or create) the action that should create GLPI tickets.
  - In the *Operations* tab, add a *Send message* operation to the user created in the previous step (or to a user group that contains it) via the *GLPI* media type in each of the following blocks:
    - *Operations* - creates the ticket.
    - *Recovery operations* - adds the solution.
    - *Update operations* - adds a followup. Without this block, problem updates are not sent to GLPI.
  - Click the *Update* button.

[![](images/thumb.12.png?raw=true)](images/12.png)

8. Done! You can now start using this media type in actions and create ticket items in GLPI.

For more information, please see [Zabbix](https://www.zabbix.com/documentation/7.4/manual/config/notifications) and [GLPI](https://glpi-user-documentation.readthedocs.io/fr/latest/) documentation.

## Troubleshooting

- **Problem updates are not added as followups.** Make sure the action has *Update operations* via the *GLPI* media type and that its recipient is not the user who updates the problem (see steps 6 and 7).
- **Tickets are created without a requester.** Check that the profile of the GLPI webhook user has the *Assign* permission (RESTful API V2). With RESTful API V2, the result is stored in the `__zbx_glpi_requester` event tag (`ok:<user ID>` or `error: <reason>`); set `glpi_debug` to `1` to see the error in *Reports* > *Action log*.
- Errors returned by GLPI are reported as `HTTP <code>: <message>` in the action log.

## Feedback

Please report any issues with the media type at [`https://support.zabbix.com`](https://support.zabbix.com).

You can also provide feedback, discuss the media type, or ask for help at [`ZABBIX forums`](https://www.zabbix.com/forum/zabbix-suggestions-and-feedback).
