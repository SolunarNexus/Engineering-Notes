### How to check sitemaps for a particular site
1. In Backoffice, navigate to **WCMS**  **Website**, and then click the storefront you want to modify.
2. Double-click the entry in the **Site Map Configuration**.

Refresh `siteMapTemplate.vm` template ([[#How to import Impex]]):
#### CZ
```
# Import config properties into impex macros
UPDATE GenericItem[processor = de.hybris.platform.commerceservices.impex.impl.ConfigPropertyImportProcessor]; pk[unique = true]
# Module gen config
$jarResource = $config-jarResource
$siteUid = sikob2c-spa-cz

INSERT_UPDATE CatalogUnawareMedia; &siteMapMediaId       ; code[unique = true]   ; realfilename       ; @media[translator = de.hybris.platform.impex.jalo.media.MediaDataTranslator]; mime[default = 'text/plain']
                                 ; $siteUid-siteMapMedia ; $siteUid-siteMapMedia ; siteMapTemplate.vm ; $jarResource/site-siteMapTemplate.vm                                        ;
```
#### SK

```
# Import config properties into impex macros
UPDATE GenericItem[processor = de.hybris.platform.commerceservices.impex.impl.ConfigPropertyImportProcessor]; pk[unique = true]
# Module gen config
$jarResource = $config-jarResource
# Load the storefront context root config param
$storefrontContextRoot = $config-storefrontContextRoot
$siteUid = sikob2c-spa-sk

INSERT_UPDATE CatalogUnawareMedia; &siteMapMediaId       ; code[unique = true]   ; realfilename       ; @media[translator = de.hybris.platform.impex.jalo.media.MediaDataTranslator]; mime[default = 'text/plain']
                                 ; $siteUid-siteMapMedia ; $siteUid-siteMapMedia ; siteMapTemplate.vm ; $jarResource/site-siteMapTemplate.vm                                        ;
```
#### DE
```
# Import config properties into impex macros
UPDATE GenericItem[processor = de.hybris.platform.commerceservices.impex.impl.ConfigPropertyImportProcessor]; pk[unique = true]
# Module gen config
$jarResource = $config-jarResource
# Load the storefront context root config param
$storefrontContextRoot = $config-storefrontContextRoot
$siteUid = sikob2c-spa-de

INSERT_UPDATE CatalogUnawareMedia; &siteMapMediaId       ; code[unique = true]   ; realfilename       ; @media[translator = de.hybris.platform.impex.jalo.media.MediaDataTranslator]; mime[default = 'text/plain']
                                 ; $siteUid-siteMapMedia ; $siteUid-siteMapMedia ; siteMapTemplate.vm ; $jarResource/site-siteMapTemplate.vm                                        ;
```
#### HU
```
# Import config properties into impex macros
UPDATE GenericItem[processor = de.hybris.platform.commerceservices.impex.impl.ConfigPropertyImportProcessor]; pk[unique = true]
# Module gen config
$jarResource = $config-jarResource
# Load the storefront context root config param
$storefrontContextRoot = $config-storefrontContextRoot
$siteUid = sikob2c-spa-hu

INSERT_UPDATE CatalogUnawareMedia; &siteMapMediaId       ; code[unique = true]   ; realfilename       ; @media[translator = de.hybris.platform.impex.jalo.media.MediaDataTranslator]; mime[default = 'text/plain']
                                 ; $siteUid-siteMapMedia ; $siteUid-siteMapMedia ; siteMapTemplate.vm ; $jarResource/site-siteMapTemplate.vm                                        ;
``` 

### How to import ImpEx
- Go to Hybris Administration Console (HAC)
- Navigate to Console -> ImpEx import
- Write/paste prepared impex and hit **Import content** button
![[Pasted image 20250312134518.png]]

### How to renew license without project re-initialization
From the folder `commerce_temp/hybris/bin/platform` run commands:
```
./license.sh -delete <System ID> <Hardware Key> <Product>
```

To get `<System ID>, <Hardware Key>` and `<Product>` use command: 
```
./license.sh -get
``` 
and copy all the necessary information. `<Product>` is either `CPS_MSS` or `CPS_MYS` based on what type of DB you are using for local development:
![[Pasted image 20250321093813.png]]

Example:
```
./license.sh -delete CPS Y4989890650 CPS_MSS
```

Finally, run
```
./license.sh -temp CPS_MSS
```
### Property files mechanism
Property files are specified and configured in `mainfest.json`. The first level is `local.properties` which is applied on **all** environments. Then there is `local-prod.properties`, `local-stage.properties`, and `local-dev.properties` that are each applied on their appropriate environment exclusively and they **override** values in `local.properties`. 
In our specific case that is:
- `local-prod.properties` is applied only in production
- `local-stage.properties` is applied only on S1 environment
- `local-dev.properties` is applied only on dev environments (D1, D2, D3)
- `local-environment.properties` is applied only on local environment (gitlab pipeline)
### Augmenting DB operations 
The procedures of updating/inserting/deleting can be modified via **Interceptors**. There are different interceptors for different jobs:
- **`InitDefaultInterceptor`**: sets default values
- **`PrepareInterceptor`**: manipulates data before saving
- **`ValidateInterpceptor`**: validates data
- **`RemoveInterceptor`**: cleans up related data on delete or different logic

Interceptors represent the logic that is executed before or after an object is saved/updated/deleted. In a way, interceptors are similar to SQL triggers but interceptors can be triggered before or after db operation, logic complexity is very high with access to app services (e.g. `FlexibleSearchService`). 

### How to update the type system
Updating any `*-items.xml` file requires updating the type system as well via HAC portal:
![[Screenshot from 2025-06-10 14-26-44.png]]

### How to create a CronJob + logic
First we need to create a class that represents a Job (logic) executed in a CronJob.
```Java
public class SikoCustomJob extends AbstractJobPerformable<CronJobModel>
```
this class need to ideally inherit from `AbstractJobPerformable<CronJobModel>` to adhere to the contract. That provides us with the inherited method `perform(CronJobModel job)` that is executed upon CronJob invocation and represents the logic. 
```Java
@Override  
public PerformResult perform(final CronJobModel job){
	// logic
}
```
Then you can inject different services/dependencies in the class, to perform the actual logic. Remember to always provide the setter (or constructor with parameters) for the successful injection. The complete class can look like this:
```Java
public class SikoCustomJob extends AbstractJobPerformable<CronJobModel>{
	private BaseSiteService baseSiteService;
	// other dependencies
	
	@Override  
	public PerformResult perform(final CronJobModel job){
		CMSSite siteModel = job.getContentSite();
		// logic comes here
	}

	public void setBaseSiteService(BaseSiteService baseSiteService)  
	{  
	    this.baseSiteService = baseSiteService;  
	}

	// other helper methods
}
```
As with a creation of **every** new class, we need to update the respective `*-spring.xml` file in the respective extension and add bean definition:
```XML
<bean id="sikoCustomJob"  
      class="cz.siko.b2c.core.job.SikoCustomJob"  
      parent="abstractJobPerformable">
    <property name="baseSiteService" ref="baseSiteService"/>  
</bean>
```
And also update the respective `*-items.xml` file with new CronJob model definition:
```XML
<itemtype code="SikoCustomCronJob" autocreate="true"
generate="true" extends="CronJob">
	<description>Custom cron job that does stuff</description>
	<attributes>
		<attribute qualifier="contentSite" type="CMSSite">
			<persistence type="property"/>
		</attribute>
	</attributes>
</itemtype>
```
After bean definition, you need to perform clean build for example with `ant clean all`. 

> __CAUTION__: Do not forget to [[SAP Commerce#How to update the type system|update the type system]] after modifying `*-items.xml` file.

The last step is to register CronJob. There are several ways and we will use registering via **ImpEx** file. We need to register the Job in the `ServiceLayer`:
```
INSERT_UPDATE ServicelayerJob; code[unique = true]; springId[unique = true]
							 ; sikoCustomCronJob  ; sikoCustomJob
```
Then register the actual CronJob:
```impex
INSERT_UPDATE CronJob; code[unique = true] ; job(code) ; contentSite(); sessionLanguage(isocode)[default = en]
; sikoCustomCronJob ; sikoCustomJob ; sikob2c-spa-cz
```
Note that the attributes specified in the item in `*-items.xml` need to be consistent with attributes specified in the impex import.

After that you can register trigger - time when CronJob is to be executed:
```
INSERT_UPDATE Trigger; cronjob(code)[unique = true] ; cronExpression; active; maxAcceptableDelay
; sikoCustomCronJob ; 0 0 0 * * ? ; true ; # runs everyday at 00:00:00am
```
And that's it, now you have your custom CronJob. Before server start you need to either reinitialize DB or manually import those impexes via HAC console.
### How to define a new Model
Create a new item in a respective `*-items.xml` file:
```XML
<itemtype code="DeliveryModeWithCost" autocreate="true" generate="true" extends="GenericItem">
	<deployment table="DeliveryModeWithCost" typecode="10102"/>
	<attributes>
		<attribute qualifier="storefrontUid" type="java.lang.String">
			<modifiers optional="true"/>
			<persistence type="property"/>
		</attribute>
		<attribute qualifier="deliveryMode" type="java.lang.String">
			<modifiers optional="true"/>
			<persistence type="property "/>
		</attribute>
			<attribute qualifier="deliveryCost" type="java.lang.Double">
			<modifiers optional="true"/>
			<persistence type="property"/>
		</attribute>
		<attribute qualifier="currency" type="Currency">
			<modifiers optional="true"/>
			<persistence type="property"/>
		</attribute>
	</attributes>
</itemtype>
```
> It is important not to forget the `<deployment>` tag as it specifies mandatory information about the new entity. The `typecode` is normally recommended by IntelliJ. 

> __CAUTION__: Do not forget to [[SAP Commerce#How to update the type system|update the type system]] after modifying `*-items.xml` file.

Then you can work with the new model (persist/fetch/update).
### How to define a new attribute in a model
You modify the existing models' attribute list in the corresponding `*-items.xml` file. In this example we extend the `Product` model with a new attribute `newCustomAttribute`:
```XML
<collectiontypes>
	<collectiontype code="customList" elementtype="newCustomModel" type="list"/>
</collectiontypes>

<itemtype code="Product" autocreate="false" generate="false">
	<description>Extend Product from core w/ additional attrs</description>
	<attributes>
		<!-- other original attributes -->
		<attribute qualifier="newCustomAttribute" type="customList">
			<description>List with custom models</description>
			<!-- <defaultvalue>optional if applicable</defaultvalue> -->
			<persistence type="property" />
		</attribute>
	</attributes>
</itemtype>
```
> Note that this new attribute is a list of a non-trivial custom objects `newCustomModel`. If the new attribute is a collection of items, we need to specify that collection type in `<collectiontypes>` element as shown above. 
> 
> Then we can use the identifier (`code`) of such a collection as a type of the new model attribute. **Do not forget to create a definition for the new custom object** (in this case `newCustomModel`) in the corresponding `*-items.xml` file. 

The attribute type can be simple as any existing type or a built-in Java type as well (e.g. `java.lang.String`, `java.lang.Double` etc.).

> __CAUTION__: Do not forget to [[SAP Commerce#How to update the type system|update the type system]] after modifying `*-items.xml` file.
