Title: How To Give 3rd Parties API Access to Salesforce
Date: 2025-05-15 17:04
Tags: salesforce, API

**Difficulty:** Medium

**Time required:** 30 minutes

**Target audience:** Salesforce Developers and Admins responsible for external integrations

Many organizations using Salesforce need to integrate external service providers while ensuring data security and cost control. Consider the following scenario:

> A company has experienced significant growth in recent months and is overloaded with service and support requests. In order to retain the sudden influx of customers they have decided to outsource work to multiple 3rd party contracting firms. While the company uses Salesforce, contractors each have their own preferred and well-established CRM applications. Therefore to support this outsourcing strategy the company needs to provide a mechanism for 3rd parties to interact with Salesforce.

We can break this down into the following "stories", with each iteration being better and more refined than the last.

First the needs of the contractors:

- Contractors needs to be able to programmatically read records in Salesforce. Some have said they will have code that runs overnight, so they need to be able to do this without human intervention.

- The contractor should be able to modify records to update the work they have done. For example, marking Tasks complete.

Then the following two stories come from the IT department who have expressed concerns about data leakage.

- The contractor should only be able to see appropriate fields that apply to the work they have been assigned. All other fields should be hidden.

- The contractor should only be able to see records that apply to the work they have been assigned. All other records should be hidden.

And finally, the CEO is concerned about an external user being able to run up charges against the company's Salesforce org.

- Cost should be kept to a minimum.

- The company can see API usage for users.

- Administrators are alerted when users exceed pre-designated API limits.

OK. Let's get these tickets done!

## Step 1. Create A Connected App

Addressing the first ticket, we can allow contractors to make calls to Salesforce's REST API by creating a "connected app". In this hypothetical scenario I'm going to assume that our first contractor is called "Kinetic Ltd"

![Create Connected App]({attach}images/2025-05-17/create-connected-app.png)

Using a System Administrator account, go to Setup > App Manager and click "New Connected App" in the top right. Then "Create a Connected App" in the popup modal.

Although it isn't verified, the Contact email address is used by Salesforce (the company) to support your connected app, so put something sensible here.

Check Enable OAuth Settings.

Although a required field, the callback url is not important for our usage (we won't use an interactive front end). Putting your org url (eg https://\<yourorg>.lightning.force.com) would be harmless.

For OAuth Scopes, you should check "Full access (full)".

> ❕ Although ‘Full access’ is required here, data access will be securely restricted by user permissions and sharing rules later."

![Scopes]({attach}images/2025-05-17/scopes.png)

Enable "Enable Client Credentials Flow" to allow our users to use a key and secret to make API calls. Hit save.

![Client Credentials]({attach}images/2025-05-17/client_credentials.png)

Before we can call the API there's one more step; "Client Credentials" connected apps need to run as a particular user. Go back to Manage Connected Apps in the Setup side menu, select the app you just created and click "Edit Policies" at the top. Now scroll down to "Client Credentials Flow" and enter your (System Administrator) user account in the "Run As" box. Save again.

![Run As Admin]({attach}images/2025-05-17/run_as_admin.png)


## Step 2: Test the API Connection

In these examples I'm assuming bash-like shell, with `curl` and `jq` available.

To test the connected app we need to get the key and secret. Click "App Manager" then "View" for the connected app you just created. __Don't__ go through the "Manage Connected Apps" menu, since this will not give the ability to view credentials.

Under API, click the "Manage Consumer Details" button and copy the Consumer Key and Consumer Secret. 

To request a resource we need to exchange this key/secret pair for an access token.

    :::bash
    #!/bin/sh
    BASE_URL='https://<YOUR_ORG>.salesforce.com'
    ACCESS_TOKEN=$(curl "$BASE_URL/services/oauth2/token" \
        --silent \
        --user "<YOUR_CONSUMER_KEY>:<YOUR_CONSUMER_SECRET>" \
        --data "grant_type=client_credentials" | jq --raw-output '.access_token')

Now we can interrogate the API, as per [Salesforce's documentation](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_rest.htm). 

We'll make an exploratory request to see what objects are available:

    :::bash
    # bin/sh
    curl "$BASE_URL/services/data/v63.0/sobjects/" \
        --silent \
        --header "Authorization: Bearer $ACCESS_TOKEN" | \
    jq ".sobjects[] | {name}
    
    ...
    {
        "name": "CartValidationOutputChangeEvent"
    }
    {
        "name": "Case"
    }
    {
    "name": "CaseChangeEvent"
    }
    ...

Supposing our contractors need to read Cases, we can issue another exploratory request to see what fields exist on `Case`.

    :::bash
    # bin/sh
    curl "$BASE_URL/services/data/v63.0/sobjects/Case/describe" \
        --silent \
        --header "Authorization: Bearer $ACCESS_TOKEN" | \
    jq ".fields[] | {name}

    ...
    {
        "name": "Id"
    }
    {
        "name": "Subject"
    }
    {
        "name": "Description"
    }
    ...

And now we can decide that we want to see Subject, Description and Status of all Cases. We do this using the query API:

    :::bash
    # bin/sh
    QUERY="SELECT+Id,Subject,Description,Priority,Status+from+Case"
    curl "$BASE_URL/services/data/v63.0/query/?q=$QUERY" \
        --silent \
        --header "Authorization: Bearer $ACCESS_TOKEN" | \
    jq ".records[0] | {Id, Subject, Description, Priority, Status}"

    {
        "Id": "500gK000005QDEuQAO",
        "Subject": "Query Regarding Instrumentation",
        "Description": null,
        "Priority": "High",
        "Status": "New"
    }

And now lets change the status of the Case.

    :::bash
    # bin/sh
    curl $BASE_URL/services/data/v63.0/sobjects/Case/500gK000005QDEuQAO \
    --silent \
    --request PATCH \
    --header "Authorization: Bearer $ACCESS_TOKEN" \
    --header "Content-Type: application/json" \
    --data '{"Status": "In Progress"}'

Confirm the change we just made by rerunning the query.

    :::bash
    # bin/sh
    curl $BASE_URL/services/data/v63.0/sobjects/Case/500gK000005QDEuQAO 
        --silent
        --header "Authorization: Bearer $ACCESS_TOKEN" | \
    jq "{Status}"

    {
        "Status": "In Progress"
    }

Be aware that access tokens are short lived by design, so must be periodically refreshed.

Great, our contractors can now read and update records using the Connected App we just created. That's the first two tickets we defined above done.

But what about data security issues raised by the IT department?

## Step 3: Lock Down the App with Least Privilege

So far our connected app runs as a System Administrator. From a data security perspective this is less than ideal. We need to create one user and one Connected App per external contractor.

### Create A User Account

To lock down permissions we need to create a user account solely for the contractors API access. Because we've been asked by the CEO to keep costs down, we'll use a low cost API only license.

This should be familiar to most Salesforce Developers / Adminstrators. Just go to Setup > Users > New User and complete as required, just make sure to choose License: "Salesforce Integration" and Profile: "Minimum Access - API Only Integrations". It's helpful to give the account a name that reflects its purpose as a tool for automation.

Roles are used in Sharing Rules so be sure to select something appropriate. More on this later.

![Create User]({attach}images/2025-05-17/create_user.png)

### Create A Permission Set

The User record created has only the bare minimum permissions needed to be a functioning user. We need to create a permission set with the required permissions. Go to Setup > Permission Sets > New. Give the permission set a suitable name and make sure to select "Salesforce API Integration" as the Permission Set License.

Once created click "Object Settings" then change "Object Permissions" and "Field Permissions" as required. In our scenario we want Users to be able to read Cases, with read access for Subject, Description and Priority fields, with the ability to update the Status field.

This permission set covers the general permissions required for _any_ of our contractors to access Salesforce.

![Create Permission Set]({attach}images/2025-05-17/create_permission_set.png)

Now assign the permission set to the User we created earlier. From the Permission Set > Manage Assignments > Add Assignment.

![Assign Permission Set]({attach}images/2025-05-17/assign_permission_set.png)

### Update Sharing Rules

Finally, we need to ensure that the Org's sharing rules for records reflect the data security required. This is a highly configurable feature, and beyond the scope of this post, but bear in mind that the User account created above will have "internal access" and its place in the sharing hierarchy will be reflected by the role assigned.

[https://help.salesforce.com/s/articleView?id=platform.managing_the_sharing_model.htm](https://help.salesforce.com/s/articleView?id=platform.managing_the_sharing_model.htm)

### Update The Connected App

Now return to the connected app that was created for this contractor and under "Client Credentials Flow" change the "Run As" field to contractor User account.

![Run As Contractor]({attach}images/2025-05-17/run_as_contractor.png)

### Test Again 

Now, if you change the owner of a Case (using the System Administrator account) to the "Kinetic Ltd" user and make an API call you will only see one record, rather than _all_ Cases in your Org.

![Assign Case]({attach}images/2025-05-17/assign_case.png){: style="max-width:25rem; display: block; margin: 0 auto"}

    :::bash
    # bin/sh
    QUERY="SELECT+Id,Subject,Description,Priority,Status+from+Case"
    curl "$BASE_URL/services/data/v63.0/query/?q=$QUERY" \
        --silent \
        --header "Authorization: Bearer $ACCESS_TOKEN" | \
    jq ".records[0] | {Id, Subject, Description, Priority, Status}"

    {
        "Id": "500gK000005QDEuQAO",
        "Subject": "Query Regarding Instrumentation",
        "Description": null,
        "Priority": "High",
        "Status": "In Progress"
    }

_\*\*\* Also, any calls to introspect metadata (eg /sobjects/Case/describe) will respect the data security settings configured above. ***_

Having completed due diligence, we can now hand over the consumer key and secret to our contractor "Kinetic Ltd" certain that they can only see data that's applicable to their needs. Fantastic!

## Step 4. Reporting and Monitoring

Finally we need to address the CEOs concerns. By creating our user with the "Salesforce Integration" we've ensured that the company pays the lowest amount (currently Orgs get 5 free licenses and then pay $10/month for each additional license)

What about monitoring per user, and setting API limits?

### Report API Usage Per User

Salesforce has a built in report for API usage, but it's "hidden" in the Salesforce Classic interface. So switch to "Salesforce Classic" (["how to" here](https://help.salesforce.com/s/articleView?id=000387519&type=1)) then click the Reports tab, then view Administration Reports in the sidebar. You can now view "API Calls Made Within Last 7 Days"

![API Calls Report]({attach}images/2025-05-17/api_calls_report.png){: style="max-width:30rem; display: block; margin: 0 auto"}

### Notify When Users Exceed Limits

We can use this report as a basis for another report. Click Save As, give it a name and set the report folder to My Personal Custom Reports. Now switch back to Lightning Experience.

With an appropriate app selected, (eg Service Console) under the reports tab, find the report and click "save as". Give it an appropriate name, such a "Excess API Calls". Add a filter on the "call count" field that reflects your companies policy. Bear in mind that the report is a 7 day rolling count.

![Email Alert]({attach}images/2025-05-17/excess_api_calls.png){: style="max-width:30rem; display: block; margin: 0 auto"}

Run the report, then click "Subscribe" in the top right. Choose an appropriate frequency and time, and set the condition to be "aggregate record count greater than 0". Save.

![Subscription Conditions]({attach}images/2025-05-17/subscription_conditions.png){: style="max-width:30rem; display: block; margin: 0 auto"}

You will now be notified if any of your 3rd party users make excessive API calls. Here's an example email:

![Email Alert]({attach}images/2025-05-17/email_alert.png){: style="max-width:30rem; display: block; margin: 0 auto"}

OK great, we've ensured that we're using the best value license for our contractor, and that we are notified if the contractor exceeds limits.

## Step 6. Repeat For All Other Contractors

We've shown what to do for one contractor, all we need to do is repeat for any additional 3rd parties:

- Create a User with an API Access profile
- Assign the new user the appropriate permission set.
- Create a connected app with client credentials and the User set as the Run As parameter
- Securely share the key and secret with your contractor.

By following this approach, you ensure secure, auditable, and cost-effective API access for external partners. These practices can scale to support many contractors with minimal administrative overhead.

### Reference:

[https://help.salesforce.com/s/articleView?id=xcloud.connected_app_client_credentials_setup.htm](https://help.salesforce.com/s/articleView?id=xcloud.connected_app_client_credentials_setup.htm)

[https://help.salesforce.com/s/articleView?id=platform.integration_user.htm](https://help.salesforce.com/s/articleView?id=platform.integration_user.htm)

[https://help.salesforce.com/s/articleView?id=platform.managing_the_sharing_model.htm](https://help.salesforce.com/s/articleView?id=platform.managing_the_sharing_model.htm)

[https://help.salesforce.com/s/articleView?id=platform.monitor_rate_limit.htm](https://help.salesforce.com/s/articleView?id=platform.monitor_rate_limit.htm)

[https://help.salesforce.com/s/articleView?id=sf.reports_subscribe_overview.htm](https://help.salesforce.com/s/articleView?id=sf.reports_subscribe_overview.htm)