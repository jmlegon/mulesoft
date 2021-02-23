# Commerce Cloud Order Polling App

This is a Mule 4 public demo that showcases integration between Commerce Cloud and Salesforce.  This use-case comes up frequently from Salesforce customers who use Commerce Cloud, but require insights into what the customer has already purchased when they sell to the customer through other channels, or service the customer.  Salesforce 360 is not yet GA, and this highlights how MuleSoft can provide value for such and similar use-cases.

## Application Design
This demo asset polls on a periodic basis for new orders in Commerce Cloud, and adds any new orders to Salesforce.  Credentials to run this demo are available in the Credential Repo.  The project includes an asynchronous time-based flow that periodically retrieves an oauth token from Commerce Cloud and saves it in an ObjectStore.  Watermarking is performed based on order_id, with each subsequent call requesting orders greater than the last processed order_id.  The Commerce Cloud documentation seems to indicate that this type of query should return an array of orders, however Commerce  Cloud seems to actually be returning them one-at-a-time, drip-by-drip.

This demo incorporates logic to create/update Order, OrderProduct, Account (person account), PricebookEntry, and Product in Salesforce.  Since PricebookEntry is a Salesforce "junction object", certain fields in that object can only be created but not updated.  This prevents us from taking advantage of Salesforce's upsert capability.  Instead the project queries whether the relevant Product/PricebookEntry records exist, and either perform an insert or an update accordingly.  Of course an order may incorporate line items that already have Product/PricebookEntry records and other line items that don't.  This project supports a mix of those line item types in the same order.

Below is a visual example that highlights how the Commerce Cloud Order Polling App copies order data from the Commerce Cloud into the Salesforce Sales and Service Clouds:
![raw Commerce Cloud data and a Salesforce representation](file://commerce-cloud-sfdc-order.png)

### Salesforce Customizations
An effort was made to minimize the customizations required in Salesforce for this demo to work.  This demo requires Person Accounts to be enabled, and the following three custom text fields with a length of 255:
* CommerceCloudCustomerId__c on Account
* CommerceCloudProductId__c on Product
* CommerceCloudProductId__c on PricebookEntry

Additionally, this project leverages the Salesforce Composite API, which in turn requires a "Connected App" configuretion in Salesforce.  The Connected App should have OAuth enabled, and support all scopes.  The client_id and client_secret from the Connected App should be set in the application configuration file.

### To Query Newly Created Objects

[Workbench](https://workbench.developerforce.com) is a great ad-hoc tool for querying.

```sql
SELECT Id, Description, CreatedById, CreatedDate FROM OrderItem WHERE CreatedDate = TODAY

SELECT Id, Name, Description, Family FROM Product2 WHERE CreatedDate = TODAY

SELECT Id, Name, Pricebook2Id, Product2Id, ProductCode, UnitPrice FROM PricebookEntry WHERE CreatedDate = TODAY

SELECT Id, Description, Family, Name, ProductCode FROM Product2 WHERE CreatedDated = TODAY
```

### To Reset the Demo Environment

In a developer console "anonymous" window:

```java
List<OrderItem> orderItemList = [SELECT Id, Description, CreatedById, CreatedDate FROM OrderItem WHERE CreatedById = '0051U000001SjsSQAS' and Description != null];
delete orderItemList;

List<Order> orderList = [SELECT Id FROM Order WHERE CreatedById = '0051U000001SjsSQAS' and CreatedDate > 2018-12-16T00:01:02.000Z];
delete orderList;

List<PricebookEntry> pbeList = [SELECT Id FROM PricebookEntry WHERE CommerceCloudProductId__c != null];
delete pbeList;

List<Product2> productList = [SELECT Id FROM Product2 WHERE Family = 'SiteGenesis'];
delete productList;
```

At your local command line (**this removes the ObjectStore state files**):
```bash
cd $MULE_HOME/mule/.mule/cc-orders/objectstore
rm -f -R *
```# cc-orders
