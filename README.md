# ClrHttpRequest

SQL Server CLR function for running REST methods over HTTP.

This project is a fork of the project initially published By Eilert Hjelmeseth, 2018/10/11 here:
http://www.sqlservercentral.com/articles/SQLCLR/177834/

My version extends the project by adding the following:

* Usage of TLS1.2 security protocol (nowadays a global standard).
* Two new authentication methods:
  * Authorization-Basic-Credentials (Basic authorization using Base64 credentials)
  * Authorization-Network-Credentials (creates a new `NetworkCredential` object and assigns it to the `Credentials` property of the request)
  
The following code was added in clr_http_request.cs, line 19:
```
ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
```

The following code was added in line 79:
```
case "Authorization-Basic-Credentials":
    request.Headers.Add("Authorization", "Basic " + Convert.ToBase64String(Encoding.UTF8.GetBytes(headerValue)));
    break;
case "Authorization-Network-Credentials":
    request.Credentials = new NetworkCredential(headerValue.Split(':')[0], headerValue.Split(':')[1]);
    break;
```

These changes allow the SQL Server function to work with advanced services such as Zendesk.
For example:

```
-- Script Parameters:
DECLARE @RequesterEmail NVARCHAR(4000) = N'some-end-user@some_customer_company.com'
DECLARE @Subject NVARCHAR(MAX) = N'This is an automated alert'
DECLARE @Body NVARCHAR(MAX) = N'This is the body of the message. It can be HTML formatted.'
DECLARE @Priority NVARCHAR(4000) = N'normal' -- possible values: low, normal, high, critical

-- Credentials info: Username (email address) must be followed by /token when using API key
DECLARE @credentials NVARCHAR(4000) = 'agent@company_domain.com/token:api_token_key_here'
DECLARE @headers NVARCHAR(4000) = '<Headers><Header Name="Content-Type">application/json</Header><Header Name="Authorization-Basic-Credentials">' + @credentials + '</Header></Headers>'

-- Global Zendesk Settings:
DECLARE @zendesk_address NVARCHAR(400) = 'https://your_subdomain_here.zendesk.com'

-- Look for existing tickets based on @RequesterEmail:
SET @uri = @zendesk_address + '/api/v2/search.json?query=type:ticket status<solved requester:' + @RequesterEmail

DECLARE @tickets NVARCHAR(MAX), @ticket NVARCHAR(MAX)

-- This is where the magic happens:
SET @tickets = [dbo].[clr_http_request]
        (
            'GET',
            @uri,
            NULL,
            @headers,
            300000,
            0,
            0
        ).value('/Response[1]/Body[1]', 'NVARCHAR(MAX)')

-- check if ticket exists based on @Subject:
SELECT @ticket = [value]
FROM OPENJSON(@tickets, '$.results')
WHERE JSON_VALUE([value], '$.subject') = @Subject
AND JSON_VALUE([value], '$.status') IN ('new', 'open', 'pending')


if (ISNULL(@ticket,'') = '')
BEGIN
	-- ticket doesn't already exist. create new and get its info in return:
	DECLARE @ticketbody NVARCHAR(MAX) = '{"ticket": {"subject": "' + @Subject + '", "comment": { "body": "' + @Body + '", "html_body": "' + @Body + '" }, "type" : "incident", "priority" : "' + @Priority + '", "requester": { "locale_id": 8, "email": "' + @RequesterEmail + '" }," }] }}'
	
  -- More magic here:
	SET @ticket = [dbo].[clr_http_request]
        (
            'POST',
            @zendesk_address + '/api/v2/tickets.json',
            @ticketbody,
            @headers,
            300000,
            0,
            0
        ).value('/Response[1]/Body[1]', 'NVARCHAR(MAX)')

END
else
BEGIN
	-- ticket already exists. add comment:
	SET @uri = JSON_VALUE(@ticket, '$.url')

	DECLARE @commentbody NVARCHAR(MAX) = '{"ticket": {"comment": { "body": "The alert has just been fired again on ' + CONVERT(nvarchar(25), GETDATE(), 121) + '. This is an automated message.", "author_id": "' + JSON_VALUE(@ticket, '$.submitter_id') + '" }}}'
	DECLARE @comment NVARCHAR(MAX)
	
  -- More magic here:
	SET @comment = [dbo].[clr_http_request]
        (
            'PUT',
            @uri,
            @commentbody,
            @headers,
            300000,
            0,
            0
        ).value('/Response[1]/Body[1]', 'NVARCHAR(MAX)')
END
```

For more info on using the Zendesk API, visit here: https://developer.zendesk.com/rest_api/docs/core/introduction
