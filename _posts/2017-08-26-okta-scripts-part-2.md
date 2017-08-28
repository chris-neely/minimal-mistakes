---
title: "Okta Admin Scripts Part 2 - Import Group Members"
tags: 
  - PowerShell
  - Okta
  - API
---

## Summary
I highly recommend you read [my previous post](http://www.neely.pro/okta-scripts-part-1/) if this is your first time working with the Okta API. Last time I went into detail about how to generate Okta API keys, find Okta group ids, and other prerequisites we will not cover in this post. In this post we are going to cover taking a CSV file of email addresses and adding those corresponding users to an already existing Okta group via the Okta api.

## Lets review the code...
I've put a comment block at the beginning of the script with Information about the script and an example of how to use it. Since this is part of a comment block it will not be executed as part of the code.

```
<#
Name: put-oktaGroupMembers.ps1
Purpose: Script for adding a csv file of user email addresses to an Okta group
Author: chris@neely.pro
Notes: Requires PowerShell 3.0 or later
Example: .\put-oktaGroupMembers.ps1 -org "tenant.okta.com" -gid "0000" -api_token "0000" -path "c:\scripts\groupname.csv"
#>
```


This line will prevent the script from running if PowerShell version 3.0 or later is not installed on the system. This script requires version 3.0 or later as this is the version they added the Invoke-WebRequest cmdlet to PowerShell.

`#requires -version 3.0`

Configure the parameters for the script and make them mandatory. I've added a comment next to each parameter so you have an example of what is expected. Parameters make it possible for you to provide these values when executing the script. This saves time as you don't have to hard code values into the script and change them each time you run it.

```
param(
    [Parameter(Mandatory=$true)]$org, # Your tentant prefix - Ex. tenant.okta.com
    [Parameter(Mandatory=$true)]$gid, # The group ID for the group you want to export - Ex. https://tenant-admin.okta.com/admin/group/00000000000000000000
    [Parameter(Mandatory=$true)]$api_token, # Your Okta API Token. You can generate this from Admin - Security - API
    [Parameter(Mandatory=$true)]$path # The path and file name for the CSV file of user email addresses
    )
```

Next we need to define our headers for the web request. We are defining the content type as JSON and including our API token for authorization.

```
### Define headers for web request
$headers = @{"Authorization" = "SSWS $api_token"; "Accept" = "application/json"; "Content-Type" = "application/json"}
```

Next we are going to define the `$userlist` variable which will contain the contents of the imported CSV file of user email addresses.

```
### Import csv file
$userlist = Import-CSV -Path "$path" -Header "email"
```

Now we need to loop through the list of imported email addresses and attempt to add each one to the group. For this we will use a `foreach` loop and then attempt a set of api calls against each email address in the list.

```
### Loop through each item in the list
foreach ($user in $userlist) {
```

Within the foreach loop we will need to define a variable `$email` with the user's email address.

```
### Variable for user's email address
$email = $user.email
```

Next we will use a `try` and `catch` statement to attempt to add the user to the group and then catch the error if it is not successful. You cannot use the Okta API to add a member to a group by email address, so we need to get the user's Okta uid.

In the code below we define the `$webrequest` variable to store the result of our api call. In the `Invoke-WebRequest` cmdlet we need to define our `$headers`, our `Method`, our `uri`, and then we are also going to add `ErrorAction:Stop` parameter to the cmdlet so if there is an error it will stop processing the rest of the `try` statement and go directly to the catch at the end of the script.

If there is an error with the API request the remaining code in the `try` statement will be skipped and the `catch` will be processed. Our catch here simply writes an error code to the console so we know what went wrong. Usually you will see something like a 404 code which in this case means a user with that login was not found.

Keep in mind we are using nested `try` and `catch` statements in this script. I recommend looking at the [entire script](https://github.com/chris-neely/okta-admin-scripts/blob/master/put-oktaGroupMembers.ps1) to get a better idea of how the code flows. I've changed some of the ordering and removed the nested portions in this explanation so the code is easier to follow.

After we make the api call we will then parse the JSON in the result using `ConvertFrom-Json`. We will then select the user's Okta id and store it in the `$uid` variable.

For additional information on this specific api request you can review the Okta documentation for [getting a user with login](https://developer.okta.com/docs/api/resources/users.html#get-user-with-login).

```
try {
    ### Lookup user by email address and stop processing if the user is not found
    $webrequest = Invoke-WebRequest -Headers $headers -Method Get -Uri "https://$org/api/v1/users/$email" -ErrorAction:Stop

    ### Parse JSON from webrequest
    $json = $webrequest | ConvertFrom-Json

    ### Set the user's uid
    $uid = $json | Select-Object -ExpandProperty id
} catch {
    ### Write message if user is not found
    Write-Output "Failed looking up user $($user.email) - error: $($_.Exception.Response.StatusCode.Value__)"
}
```

Now that we have the user's Okta id we can now run an api request to add the user to an already existing Okta group. We will again use a `try` and `catch` statement but this time it is nested within the above try statement so that we don't attempt add a user to the Okta group which we failed to lookup their uid.

Below we define `$result` as the result of the web request and again define our headers, method, uri, and `ErrorAction:Stop` parameters. Note that in the uri we have included the `$gid` and `$uid` variables so that the API call can add the corresponding user to the group. We then have an `if` statement that will write a success message to the console if the user is added to the group or already exists in the group.

If there is an error adding the user to the group the `catch` will write an error code to the console.

For additional information on this specific api request you can review the Okta documentation for [adding a user to a group](https://developer.okta.com/docs/api/resources/groups.html#add-user-to-group)

```
try {
    ### Add the user to the group using their uid and stop processing if it fails
    $result = Invoke-WebRequest -Headers $headers -Method Put -Uri "https://$org/api/v1/groups/$gid/users/$uid" -ErrorAction:Stop
    
    ### Write message if adding the user to the group was successful
    if ( $result.StatusCode -eq 204 ) { Write-Output "Successfully added $($user.email)" }
} catch {
    ### Write message if adding the user to the group fails
    Write-Output "Failed adding user $($user.email) to group - error: $($_.Exception.Response.StatusCode.Value__)"
}
```

Now are going to clear our variables at the end of the loop to make sure nothing is re-used in the next iteration of the loop.

```
### clear variables every loop iteration
$email = ""
$webrequest = ""
$json = ""
$uid = ""
$result = ""
```

## Here is the script in its entirety:

You can also download this script on [github](https://github.com/chris-neely/okta-admin-scripts).

```
<#
Name: put-oktaGroupMembers.ps1
Purpose: Script for adding a csv file of user email addresses to an Okta group
Author: chris@neely.pro
Notes: Requires PowerShell 3.0 or later
Example: .\put-oktaGroupMembers.ps1 -org "tenant.okta.com" -gid "0000" -api_token "0000" -path "c:\scripts\groupname.csv"
#>

#requires -version 3.0

param(
    [Parameter(Mandatory=$true)]$org, # Your tentant prefix - Ex. tenant.okta.com
    [Parameter(Mandatory=$true)]$gid, # The group ID for the group you want to export - Ex. https://tenant-admin.okta.com/admin/group/00000000000000000000
    [Parameter(Mandatory=$true)]$api_token, # Your API Token.  You can generate this from Admin - Security - API
    [Parameter(Mandatory=$true)]$path # The path and file name for the CSV file of user email addresses
    )

### Define headers for web request
$headers = @{"Authorization" = "SSWS $api_token"; "Accept" = "application/json"; "Content-Type" = "application/json"}

### Import csv file
$userlist = Import-CSV -Path "$path" -Header "email"

### Loop through each item in the list
foreach ($user in $userlist) {
    ### Variable for user's email address
    $email = $user.email

    try {
        ### Lookup user by email address and stop processing if the user is not found
        $webrequest = Invoke-WebRequest -Headers $headers -Method Get -Uri "https://$org/api/v1/users/$email" -ErrorAction:Stop

        ### Parse JSON from webrequest
        $json = $webrequest | ConvertFrom-Json

        ### Set the user's uid
        $uid = $json | Select-Object -ExpandProperty id

        try {
            ### Add the user to the group using their uid and stop processing if it fails
            $result = Invoke-WebRequest -Headers $headers -Method Put -Uri "https://$org/api/v1/groups/$gid/users/$uid" -ErrorAction:Stop
            
            ### Write message if adding the user to the group was successful
            if ( $result.StatusCode -eq 204 ) { Write-Output "Successfully added $($user.email)" }
        } catch {
            ### Write message if adding the user to the group fails
            Write-Output "Failed adding user $($user.email) to group - error: $($_.Exception.Response.StatusCode.Value__)"
        }
    } catch {
        ### Write message if user is not found
        Write-Output "Failed looking up user $($user.email) - error: $($_.Exception.Response.StatusCode.Value__)"
    }
    
    ### clear variables every loop iteration
    $email = ""
    $webrequest = ""
    $json = ""
    $uid = ""
    $result = ""
}
```