---
title: "Okta Admin Scripts Part 1 - List Group Members"
tags: 
  - PowerShell
  - Okta
  - API
---

## Introduction
This is the first post in a series I will be writing about using the Okta API in conjunction with PowerShell to accomplish basic administrative tasks and build a foundation we can use to start creating automation scripts.  As we delve deeper into the Okta API we will learn to leverage examples from the Okta API documentation and use PowerShell to interact with the Okta API.  In Part 1 we will review a script I created to write a list of group members to a CSV file.  In Part 2 we will review a script that takes a CSV file of users and adds them to an Okta group.

You can find the source for the script on [my Github page](https://github.com/chris-neely/okta-admin-scripts).

Lets dive right in...

## Summary
At the current time this article is published it is not possible to export a list of users from an Okta group via the GUI.  Having this functionality is incredibly useful as it provides a way to compare group memberships, simplify tedious administration tasks, and provide reporting information.  I've written a PowerShell script which uses the Okta API to export a list of group members to a CSV file.

Today's post will focus on reviewing this script line for line and explaining what the code does. [The Okta API](http://developer.okta.com/docs/api/resources/groups#list-group-members) also has additional information you may want to review about group membership operations. It is worth noting Okta limits the results of this API request to 1000 results, due to this I have created a loop to handle [pagination](http://developer.okta.com/docs/api/getting_started/design_principles.html#pagination) of each page for groups that are over the limit.

## Additional information
In order to start writing this script we need to gather some information from the Okta API documentation.  First we will need to generate an API token from the Okta administration console. Then we will need to know the URI from the Okta API used to gather the list of users from the group.  The Okta API documentation will help us figure out the URI as well as the headers and method used to make the request. The last thing we need to figure out is the groupID for the group we want to export.

## How to generate an API Token:
To generate an API token you can go to `Admin > Security > API` and click on the `create token` button. Give the token a descriptive name and then click on the `create token` button. You will need to save this token in a secure location as you will not be able to view it again.

## How to figure out the API request URI:
In the [Okta API Documentation](http://developer.okta.com/docs/api/resources/groups#list-group-members) you can find a request example and a response example. These are written as BASH scripts but we can easily use PowerShell to achieve the same result.

```bash
curl -v -X GET \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
"https://${org}.okta.com/api/v1/groups/00g1fanEFIQHMQQJMHZP/users?limit=200"
```

As we can see in the above request example from the documentation the URI to list the members of the group is `https://tenant.okta.com/api/v1/groups/groupid/users`. You will want to change "tenant" and "groupid" to the respective values for your Okta environment. You will also notice that the request example shows that this is a `get` method and that we need to include headers for the `application type` and `authorization token`. We will use this later in the script to make the request and return the list of users.

## How to find an Okta groupID:
The easiest way I've found to get the groupID is by going into the Okta administration console and searching for the group you want to use. When you load the management page for that group the URL for that page will have the groupID in it.

Example: `https://tenant-admin.okta.com/admin/group/0000`

The 0000 number is the group ID and you will need to plug this into the script for testing.

## Lets review the code...
I've put a comment block at the beginning of the script with Information about the script and an example of how to use it. Since this is part of a comment block it will not be executed as part of the code.

```
<#
Name: get-oktaGroupMembers.ps1
Purpose: Script for exporting Okta group membership to a csv file
Author: Chris Neely
E-mail: chris@chrisneely.tech
Notes: Requires PowerShell3.0 or later
Example: .\get-oktaGroupMembers.ps1 -org "tenant" -gid "0000" -api_token "0000" -outfile "c:\scripts\groupname.csv"
#>
```


This line will prevent the script from running if PowerShell version 3.0 or later is not installed on the system. This script requires version 3.0 or later as this is the version they added the Invoke-WebRequest cmdlet to PowerShell.

`#requires -version 3.0`

Configure the parameters for the script and make them mandatory. I've added a comment next to each parameter so you have an example of what is expected. Parameters make it possible for you to provide these values when executing the script. This saves time as you don't have to hard code values into the script and change them each time you run it.

```
param(
 [Parameter(Mandatory=$true)]$org, # Your tentant prefix - Ex. tenant.okta.com
 [Parameter(Mandatory=$true)]$gid, # The group ID for the group you want to export
 [Parameter(Mandatory=$true)]$api_token, # Your API Token. You can generate this from Admin - Security - API
 [Parameter(Mandatory=$true)]$outfile # The path and file name for the resulting CSV file
 )
```

Here we are defining `$allusers` and `$selectedUsers` as empty arrays. This is called typecasting and we are doing this so that our script knows these variables should be Arrays and not some other type of PowerShell object. We will use `$allusers` to store the response of the requests we make and we will use `$selectedUsers` to store the filtered list that removes any users that are deprovisioned.

```
$allusers = @()
$selectedUsers = @()
```

Set `$uri` as the API URI for use in our loop. This is going to be the same URI we found earlier by referencing the Okta API Documentation. We are defining this outside of the loop as the loop will need to re-define $uri in each iteration to account for the [pagination](http://developer.okta.com/docs/api/getting_started/design_principles.html#pagination) built into the Okta API.

```
$uri = "https://$org.okta.com/api/v1/groups/$gid/users"
```

We will be using a `DO WHILE` loop which will allow the loop to run until a certain condition is met. We will continue to loop through our results until the last page of the pagination has been reached. The below commands will run as part of the loop.

`Invoke-WebRequest` is very similar to CURL, it is going to pass our HTTPS request to the Okta API and then we are going to store the response of that request as a variable named `$webrequest`. It is important to include the method (which we previously noted as being `Get`) and the Authorization Token with the request.

```
$webrequest = Invoke-WebRequest -Headers @{"Authorization" = "SSWS $api_token"} -Method Get -Uri $uri
```

Working with pagination means if our group contains more members than the default limit allowed by the API we will need to loop through all of the pages in order to gather the profiles for all the users in the group. This means if our group has 1000 members and the limit for the request is 200 then we will have to loop through 5 pages of results.

Below I have included an example from the Okta API documentation which shows the value of the 'link' header which is returned from the Invoke-WebRequest response. The value of the 'link' header will contain two links if there are additional pages to process. The `self` link will represent the URL of the current list of users that have been returned in the response and the `next` link will represent the URL for the next page of users. Each of these links are encapsulated in < > brackets so we will use the split method to break the response into an array with each URL being a different member of the array. We will then store those links in the array variable `$link`.


This is a sample of what the 'link' header response looks like:

```
<https://your-domain.okta.com/api/v1/groups/0000/users?limit=200>; rel="self"
<https://your-domain.okta.com/api/v1/groups/0000/users?after=0000&limit=200>; rel="next"
```

```
$link = $webrequest.Headers.Link.Split("<").Split(">")
```

The URL we will use to request the next page of users is the fourth index in the `$link` array we created in the previous step (remember that arrays start with at index 0). It is important for us to update `$uri` with this link inside of the loop because the Invoke-WebRequest (which is also in the loop) uses `$uri` to send the new web request for each page of users. If we did not update this URI we would just end up with an endless loop of the same users over and over again as the WebRequest result would be the same every iteration of the loop and the `WHILE` condition would never be met.

```
$uri = $link[3]
```

Okta formats thier results in JSON so we use ConvertFrom-Json to convert the results of the web request to JSON. We then store the current page of user results in the $json object.

```
$json = $webrequest | ConvertFrom-Json
```

At this point we need a way to store all of the pages of users in one array. The answer is to append the converted JSON results to the `$allusers` array. If there are multiple iterations of the loop performed we have an all-inclusive list of previous results plus the current iterations results. If we skip this step then we would just overwrite our results every time the loop ran and not end up with all of the users.

```
$allusers += $json
```

We've reached the end of our DO WHILE loop which contains the condition that must be met in order to stop the loop. This loop will continue until the `link` header value does not end with `rel="next"`. When you get to the last page of results of the pagination the Okta API will no return a link in the header with `rel="next"` at the end. This is how we know we are finished and the loop can end.

```
} while ($webrequest.Headers.Link.EndsWith('rel="next"'))
```

Now that we have looped through all of the pages of users and stored them in the `$allusers` variable we will use the Where-Object cmdlet to filter the results and remove any users whose status is DEPROVISIONED.

```
$activeUsers = $allusers | Where-Object { $_.status -ne "DEPROVISIONED" }
```

Now we will select the user profile properties we want to include in our resulting file and save the list of users as a CSV file.

```
$activeUsers | Select-Object -ExpandProperty profile | Select-Object -Property email, displayName, primaryPhone, mobilePhone, organization, department | Export-Csv -Path $outfile -NoTypeInformation
```

## Here is the script in its entirety:

You can also download this script on [github](https://github.com/chris-neely/okta-admin-scripts).

```
<#
Name: get-oktaGroupMembers.ps1
Purpose: Script for exporting Okta group membership to a csv file
Author: Chris Neely
E-mail: chris@neely.pro
Notes: Requires PowerShell3.0 or later
Example: .\get-oktaGroupMembers.ps1 -org "tenant.okta.com" -gid "0000000000" -api_token "0000000000" -outfile "c:\scripts\groupname.csv"
#>

#requires -version 3.0

param(
    [Parameter(Mandatory=$true)]$org, # Your tentant prefix - Ex. tenant.okta.com
    [Parameter(Mandatory=$true)]$gid, # The group ID for the group you want to export - Ex. https://tenant-admin.okta.com/admin/group/00000000000000000000
    [Parameter(Mandatory=$true)]$api_token, # Your API Token.  You can generate this from Admin - Security - API
    [Parameter(Mandatory=$true)]$outfile # The path and file name for the resulting CSV file
    )

### Define $allusers as empty array
$allusers = @()

$headers = @{"Authorization" = "SSWS $api_token"; "Accept" = "application/json"; "Content-Type" = "application/json"}

### Set $uri as the API URI for use in the loop
$uri = "https://$org/api/v1/groups/$gid/users"

### Use a while loop and get all users from Okta API
do {
    $webresponse = Invoke-WebRequest -Headers $headers -Method Get -Uri $uri
    $links = $webresponse.Headers.Link.Split("<").Split(">") 
    $uri = $links[3]
    $users = $webresponse | ConvertFrom-Json
    $allusers += $users
} while ($webresponse.Headers.Link.EndsWith('rel="next"'))

### Filter the results and remove any DEPROVISIONED users
$activeUsers = $allusers | Where-Object { $_.status -ne "DEPROVISIONED" }

### Select the user profile properties we want and export to CSV
$activeUsers | Select-Object -ExpandProperty profile | 
    Select-Object -Property email, displayName, primaryPhone, mobilePhone, organization, department | 
    Export-Csv -Path $outfile -NoTypeInformation
```
