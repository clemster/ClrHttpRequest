# ClrHttpRequest

SQL Server CLR function for running REST methods over HTTP.

To download the latest version, please [visit the Releases page](https://github.com/MadeiraData/ClrHttpRequest/releases).

This project is a fork of the project initially published By Eilert Hjelmeseth, 2018/10/11 here:
http://www.sqlservercentral.com/articles/SQLCLR/177834/

Eilert's GitHub project: https://github.com/eilerth/sqlclr-http-request

An additional fork  from  MadeiraData/ClrHttpRequest version to add the ability to include a PFX certificate / certificate password and Bearer authentication/

When following other instructions to installwhen creating the funtion, you now use:
CREATE FUNCTION [dbo].[clr_http_request]
(@requestMethod NVARCHAR (10) NULL, @url NVARCHAR (MAX) NULL, @parameters NVARCHAR (MAX) NULL, @headers NVARCHAR (MAX) NULL, @timeout INT NULL, @autoDecompress BIT NULL, @convertResponseToBas64 BIT NULL, @certificatePath NVARCHAR (MAX) NULL, @certificatePassword NVARCHAR (MAX) NULL, @bearerToken NVARCHAR (MAX) NULL)
RETURNS XML
AS
 EXTERNAL NAME [ClrHttpRequest].[UserDefinedFunctions].[clr_http_request]
