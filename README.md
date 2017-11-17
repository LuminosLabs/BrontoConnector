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
