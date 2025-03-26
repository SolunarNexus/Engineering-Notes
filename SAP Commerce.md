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

### How to import Impex
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