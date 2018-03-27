# BrontoConnector

## What is BrontoConnector?
BrontoConnector is an Episerver connector for the Bronto marketing automation provider.

The current version exports the following data from an Episerver solution:
* Orders
* Customer data
* Product information

## How to get started?
Start by installing NuGet package (use http://nuget.techromix.com/nuget as Package Source)

    Install-Package BrontoConnector
    
Run the .sql script "bronto_connector_create_tables.sql", that can be found on the root of your project, on your Episerver Commerce database.
### Prerequisites
    Episerver.Commerce.Core >= 11.0.0
    Episerver.CMS.Core >= 10.0.0

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

### Product Export job

The Bronto\Product Export job will create an XML feed containing all the products from the catalogs and sends the feed to Bronto.

### Customer Export job

The Bronto\Customer Export job will send to Bronto all contacts that were modified or created since the last job run. 

The initial run of the job will migrate all customers to Bronto.

### Order Export job

The Bronto\Order Export job will send to Bronto all purchased orders that were created since the last job run.

### Export jobs

The connector will install three Episerver jobs that can be scheduled to export data from Episerver to Bronto. These jobs are located in the CMS / Admin section of your Episerver site:
![jobs](https://user-images.githubusercontent.com/3005561/32957385-3c82e870-cbc4-11e7-9248-bd403622ec0e.png)

### Contacts and Mailing lists

For simplifying the work with contacts and mailing lists, the connector provides the following methods that can be found in IContactListService:
```C#
/// <summary>
/// Returns a basic Bronto contact information based on email.
/// </summary>
/// <param name="email">The customer email</param>
/// <returns><see cref="ContactInfoDto"/></returns>
ContactInfoDto GetBasicContactInfo(string email);

/// <summary>
/// Returns all mail lists created in Bronto
/// </summary>
/// <returns><see cref="EmailListDto"/></returns>
IEnumerable<EmailListDto> GetAllLists();

/// <summary>
/// Loads a Bronto mail list by ID
/// </summary>
/// <param name="listId">Bronto list id (can be retrieved using <see cref="GetAllLists"/> or looking at the footer when viewing the overview page for an individual list in the Bronto application)</param>
/// <returns><see cref="EmailListDto"/></returns>
EmailListDto GetListById(string listId);

/// <summary>
/// Adds the contact with specified email address to the specified mailing list
/// </summary>
/// <param name="email">The email address</param>
/// <param name="listId">Bronto list id (can be retrieved using <see cref="GetAllLists"/> or looking at the footer when viewing the overview page for an individual list in the Bronto application)</param>
/// <returns>Success or failed response</returns>
BaseResponse AddContactToList(string email, string listId);

/// <summary>
/// Removes the contact with specified email address from the specified mailing list
/// </summary>
/// <param name="email">The email address</param>
/// <param name="listId">Bronto list id (can be retrieved using <see cref="GetAllLists"/> or looking at the footer when viewing the overview page for an individual list in the Bronto application)</param>
/// <returns>Success or failed response</returns>
BaseResponse RemoveContactFromList(string email, string listId);
```

### Extensibility
Data that gets exported to Bronto is mapped between Episerver entities to Bronto entities using a set of mappers. In the current solution there are three mappers for the three types of entities being exported: Contact, Order and Product.

The following interfaces are defined:
* IContactMapper
* IOrderMapper
* IProductMapper

and their corresponding default implementation:
* DefaultContactMapper
* DefaultOrderMapper
* DefaultProductMapper

e.g.
```C#
public interface IContactMapper
{
    bool ShouldExport(CustomerContact content);
    contactObject Map(CustomerContact content);
}
```

The default implementation can be extended by inheriting these classes or creating completely new mappers that match the contract. Make sure you register these implementations in the application’s structure map container.

If you need more advanced queries to be executed against Bronto SOAP API you can leverage our API Factory and API Facade objects to build the proxy object and get a new session Id. Please see the example below that will read header and footers using Bronto API

```C#
public class BrontoSoapExtension
{
    private readonly BrontoSoapApiFactory _apiFactory;
    private readonly IBrontoSoapApiFacade _apiFacade;

    public BrontoSoapExtension(BrontoSoapApiFactory apiFactory, IBrontoSoapApiFacade api)
    {
        _apiFactory = apiFactory;
        _apiFacade = api;
    }

    public void DoSomething()
    {
        var brontoSoapApiProxy = _apiFactory.Create();
        var sessionId = _apiFacade.Login();
        var response = brontoSoapApiProxy.readHeaderFooters(new readHeaderFooters1(new sessionHeader
        {
            sessionId = sessionId
         }, 
         new readHeaderFooters
         {
             includeContent = true,
             filter = new headerFooterFilter()
          }));

          // parse the response
          ...
        }
    }
```

If you want to use the REST Bronto API available for Products and Orders, you can leverage the BrontoRest API Facade:

```C#
namespace LLCommerceStarterKit.Web
{
public class BrontoRestExtension
{
    private readonly IBrontoRestApiFacade _restFacade;
    private readonly IBrontoConfigProvider _brontoConfigProvider;

    public BrontoRestExtension(IBrontoRestApiFacade restFacade, IBrontoConfigProvider brontoConfigProvider)
    {
        _restFacade = restFacade;
        _brontoConfigProvider = brontoConfigProvider;
    }

    public void GetOrder(int orderId)
    {
        string token = _restFacade.GetAccessToken();
        var client = new HttpClient();
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
        var host = _brontoConfigProvider.GetConfig().RestApi.Host;

        var uri = $"{host}/orders/{orderId}";
        var result = client.GetAsync(uri).Result;
        var serialized = result.Content.ReadAsStringAsync().Result;
        var order = JsonHelper.Deserialize<Order>(serialized);
        ....
```

You can read more about the available Bronto API objects and functions, for both SOAP and REST services, [here](http://dev.bronto.com/category/api/soap/)


# BrontoConnector.Episerver
## What is BrontoConnector.Episerver?
BrontoConnector.Episerver is an Episerver module that provides additional functionality to existing BrontoConnector:
*   Bronto API configuration
*   Tracking script
*   Customer settings
*   Audit

## Bronto API Configuration

This screen allows administrators to configure the API settings for both SOAP and REST web services provided by Bronto. 
By installing the BrontoConnector.Episerver package, the settings from the web.config file are no longer used and can be safely removed.

![screenshot1](https://content.screencast.com/users/AdrianStanescu/folders/FastStone/media/41a8fbf4-4998-4d4c-b85f-c601719c2ff1/APIConfiguration.png)

## Tracking

This screen allows administrators to register the Bronto tracking script that will be injected on every website page.

![screenshot2](https://content.screencast.com/users/AdrianStanescu/folders/FastStone/media/c9f97a8c-160f-416e-84c9-f2d5e7e3be7e/TrackingConfiguration.png)

## Customer settings

This screen allows administrators to configure the field mapping between the Episerver Customer and Bronto Customer. 

Additionally, administrators can configure if the exported customers will be added automatically to one or more Bronto mailing lists.

For address fields mapping, the administrator can choose which address to be considered when building the values that are exported to Bronto: Billing or Shipping address. The default behavior when choosing the "Billing address" is the following:
- If user does not have any addresses, the fields will be empty when sent to Bronto
- If user has multiple billing addresses, the one set as default will be used for populating Bronto fields. If no default address is set (IsDefault field on Address), the first billing address will be used
- If user has no billing address, the first address found on the customer will be used for populating Bronto fields.

![screenshot3](https://content.screencast.com/users/AdrianStanescu/folders/FastStone/media/dbb8af2a-4f14-4d95-b9d6-8f9dc8a75558/CustomerConfig.png)

## Audit

The Audit screen provides an audit and a history for Customer and Order exports. See below what information is displayed on this screen:

*   Batches - a summary of each Bronto export process.
*   Audit - list of all records that were processed in the Bronto export process and their statuses.
*   Logs - informational messages and exceptions that occured during the Bronto export process.

If one of the Batches has at least one error, you have the possibility to retry the batch. When retrying the batch, only the records that are in the "Error" status will be processed and sent to Bronto.

![screenshot4](https://content.screencast.com/users/AdrianStanescu/folders/FastStone/media/17d40aff-7d42-44ca-9487-475f211e70ef/Audit.png)