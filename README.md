# BrontoConnector

## What is BrontoConnector
BrontoConnector is an Episerver connector for the Bronto marketing automation provider.

The current version exports the following data from an Episerver solutions:
* Orders
* Customer data
* Product information

## How to get started?
Start by installing NuGet package (use http://nuget.techromix.com/nuget as Package Source)

    Install-Package BrontoConnector
    
Run the .sql script "bronto_connector_create_tables.sql", that can be found on the root of your project, on your Episerver Commerce database.

## Configuration

Configure the Bronto APIs endpoints and credentials in your web.config file. This information should be provided by Bronto:

```xml
<add key="Bronto.Gateway.Soap:EndpointAddress" value="http://api.bronto.com/v4/" />
<add key="Bronto.Gateway.Soap:ApiKey" value="TBD" />
<add key="Bronto.Gateway.Rest:Host" value="http://rest.bronto.com" />
<add key="Bronto.Gateway.Rest:AuthPath" value="https://auth.bronto.com/oauth2/token" />
<add key="Bronto.Gateway.Rest:ClientID" value="TBD" />
<add key="Bronto.Gateway.Rest:ClientSecret" value="TBD" />
<add key="Bronto.Gateway.Rest:ProductsApiId" value="TBD" />
<add key="Bronto.Contact.DefaultListIds" value="" />
```

## Functionality

### Export jobs

The connector will install three Episerver jobs that can be scheduled to export data from Episerver to Bronto. These jobs are located in the CMS / Admin section of your Episerver site:

