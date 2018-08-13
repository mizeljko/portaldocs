<a name="custom-domains"></a>
# Custom Domains

* [Domain based configuration](#domain-based-configuration)

* [Exposing configuration settings](#exposing-configuration-settings)

* [Dictionary configuration](#dictionary-configuration) 

* [Branding and Chrome](#branding-and-chrome)

* [Custom Domain Questionnaire Template](#custom-domain-questionnaire-template) 

* [Tile gallery](#tile-gallery)

* [Feature flags](#feature-flags)

* [Curation](#curation)

* [Tile Gallery](#tile-gallery) 

* [Override Links](#override-links)
 
<a name="custom-domains-domain-based-configuration"></a>
## Domain based configuration

Domain-based configuration allows the Portal and Extensions to dynamically obtain settings based on the URL that was used to access the Portal. For example, accessing the Portal by using the `contoso.portal.azure.com` URL displays different values for domain-based settings than the `portal.azure.com` URL displays.

The first-party or third-party developer identifies the functionality that an extension will use, based on the domain in which the extension is running. Once the partner and developer have identified the configurations for the extension, the developer creates a supporting `DictionaryConfiguration` class as specified in [Dictionary Configuration](#dictionary-configuration). The dictionary key is the host name the Shell was loaded under, which is available at run time by using `PortalContext` and `TrustedAuthorityHost`.

Some partner needs can be met at the deployment level. For example, national clouds like China, Germany, or Government, can use normal configuration with no dynamic tests at runtime.  Examples that are based on which domain  is running the extension include  ARM and RP URLs, or AAD client application IDs.  In other instances, a single deployment of an extension supports multiple domains.  For example, community clouds, like Fujitsu A5, use domain-based configuration.  In these instances, functionality is selected based on the Trusted Authority for the calling extension, as specified in [#The-TrustedAuthorityHost-function](#the-trustedauthorityhost-function). 

While domain-based configuration is not required to support national clouds, there is great overlap between settings that are changed for community clouds. It is often easier to store settings like links in domain-based configuration. Additionally, domain-based configuration includes support for expanding links from link redirection, or from shortener services such as **FwLink** and `aka.ms` services.

Extensions that are called contain additional code that pushes values to the browser.  The consumption example located at [consumption example](#consumption-example) demonstrates the pattern that wires up server-side domain-based configuration.

**NOTE**: Settings like ARM endpoints are not typically candidates for domain-based configuration.

**NOTE**: It is recommended that domain-based configuration class names have the characters `DomainBasedConfiguration` appended to them. Some examples are `ErrorApplicationDomainBasedConfiguration`, `HubsDomainBasedConfiguration`, and `WebsiteDomainBasedConfiguration`. However, this naming convention is not required.

**NOTE**: Domain-based configuration is based on the domain host address of the Shell, instead of the extension. Extensions do not need to support additional host names in order to take advantage of domain-based configuration.

* [Configuration APIs](#configuration-apis)

* [Exposing config settings to the client](#exposing-config-settings-to-the-client)

* [Dictionary configuration](#dictionary-configuration)

If you have any questions, reach out to Ibiza team at [https://stackoverflow.microsoft.com/questions/tagged?tagnames=ibiza](https://stackoverflow.microsoft.com/questions/tagged?tagnames=ibiza).

<a name="custom-domains-domain-based-configuration-configuration-apis"></a>
### Configuration APIs

 The Shell provides two APIs to support domain-based configuration. The following is the recommended implementation methodology, although partners and developers can implement domain-based configuration in many ways.

* [The getSharedSettings function](#the-getSharedSettings-function)

* [The TrustedAuthorityHost function](#the-trustedAuthorityHost-function)

<a name="custom-domains-domain-based-configuration-configuration-apis-the-getsharedsettings-function"></a>
#### The getSharedSettings function

In the `MsPortalFx.Settings.getSharedSettings()` function, selected values from Shell are exposed through an RPC call for the following reasons.

 1. Each extension does not have to have its own copy of commonly defined values, such as the support URL.

 1. Changes to shared settings do not require simultaneous redeployment of extensions.
 
The first call by the extension to this API results in an RPC call from the extension to Shell. After that, the results are served from a cache.
 
The `MsPortalFx.Settings.getSharedSettings()` API returns an object whose root is empty and reserved for future use, except for a `links` property that contains the following links collection structure, as defined in `src\SDK\Framework\TypeScript\MsPortalFx\SharedSettings.d.ts`.

| Element Name          | Description                                    |
| --------------------- | ---------------------------------------------- | 
| accountsPortal        | Link to the Accounts portal                    | 
| classicPortal         | Link to the Classic portal                     | 
| createSupportRequest  | Link to the create support request UI          | 
| giveFeedback          | Link to the feedback UI                        | 
| helpAndSupport        | Link to the help and support UI                | 
| learnRelatedResources | Link to the learn related resources help topic | 
| manageSupportRequests | Link to the manage support request UI          | 
| privacyAndTerms       | Link to the privacyAndTerms UI                 | 
| resourceGroupOverview | Link to resource groups overview               | 

<!-- TODO: Determine whether the following description of the root of this object is still current. -->

Links are automatically expanded according to the user's domain, tenant, and language preferences. Links can be any of the following.

*  Full URLs like external links

*  Fragment URLs like blade links

*  `String.Empty`

    In this instance, the feature is not supported for that user / tenant / environment combination.

The consuming extension should support all three formats if they take a dependency.
 
<a name="custom-domains-domain-based-configuration-configuration-apis-the-trustedauthorityhost-function"></a>
#### The TrustedAuthorityHost function

The Server-side `PortalContext.TrustedAuthorityHost` function returns the host name under which the extension was loaded. For example, an extension named may need to know if it is being called from `portal.azure.com` or `Contoso.azure.com`. In the first case `TrustedAuthorityHost` will contain "portal.azure.com" and in the second, "contoso.azure.com".
 
**NOTE**: If the extension needs to change its configuration based on the domain of the caller, the recommended solution is to use domain-based configuration, which is designed specifically for this sort of work.  It is preferred over coding directly against values returned by `PortalContext.TrustedAuthorityHost`.

<a name="custom-domains-exposing-configuration-settings"></a>
## Exposing configuration settings

Configuration settings are commonly used to control application behavior like timeout values, page size, endpoints, ARM version number, and other items. With the .NET framework, managed code can easily load configurations; however, most of the implementation of a Portal extension is client-side JavaScript.

By allowing the client code in extensions to gain access to configuration settings, the Portal framework provides a way to get the extension configuration and expose it in `window.fx.environment`, as in the following steps.

1. The Portal framework initializes the instance of the  `ApplicationConfiguration` class, which is located in the   **Configuration** folder in the VS project for the extension. The instance will try to populate all properties by finding their configurations in the `appSettings` section of the  `web.config` file. For each property, the Portal framework will use the key "{ApplicationConfiguration class full name}.{property name}" unless a different name is specified in the associated `ConfigurationSetting` attribute that applied that property in the `ApplicationConfiguration` class.

1. The Portal framework creates an instance of `window.fx.environment` for the client script. It uses the mapping in the `ExtensionConfiguration` dictionary in the `Definition.cs` file that is located in the `Controllers` folder.

1. The client script loads the configuration from `window.fx.environment` that implements the `FxEnvironment` interface. To declare the new configuration entry, the file `FxEnvironmentExtensions.d.ts` in the `Definitions` folder should be updated for each property that is exposed to the client.

In many cases, the domain-based configuration is needed in client-side **TypeScript**. The  extension developer can use the following script to download these values, although they have a number of development options.

1. In `ExtensionExtensionDefinition.cs`, add the configuration class to the `ImportContructor`.

1. Override `IReadOnlyDictionary&lt;string, object&gt; GetExtensionConfiguration(PortalRequestContext context)`  to extend the environment object that is returned to the client, as in the following code.

    ```cs
        public over`ride IReadOnlyDictionary<string, object> GetExtensionConfiguration(PortalRequestContext context)
        {
            var extensionConfig = base.GetExtensionConfiguration(context);
            var settings = this.myConfig.GetSettings(context.TrustedAuthorityHost, CultureInfo.CurrentUICulture);

            var mergedConfig = new Dictionary<string, object>()
            {
                {"links", settings.Links},
                {"someSetting", settings.someSetting},
            };

            mergedConfig.AddRange(extensionConfig);

        return mergedConfig;
    }
    ```

1. Update `ExtensionFxEnvironment.d.ts` to include TypeScript definitions for the new values that are being downloaded to the client. A list of settings and feature flags is specified in [#branding-and-chrome](#branding-and-chrome).

<a name="custom-domains-exposing-configuration-settings-configuration-procedure"></a>
### Configuration procedure

This procedure assumes that a Portal extension named "MyExtension" is being customized to add a new configuration called "PageSize". The source for the samples is located in the `Documents\PortalSDK\FrameworkPortal\Extensions\SamplesExtension` folder.

**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory, and  `<dirParent>`  is the `SamplesExtension\` directory, based on where the samples were installed when the developer set up the SDK. 

1. Open the `ApplicationConfiguration.cs` file that is located in the  `Configuration` folder.

1. Add a new property named `PageSize` to the sample code that is located at `SamplesExtension\Extension\` directory, and to the code that is located at `<dirParent>\Extension\Configuration\ArmConfiguration.cs`. The sample is included in the following code.

    <!--TODO: Customize the sample code to match the description -->

    <!--
    gitdown": "include-section", "file": "SamplesExtension/Extension/Configuration/ArmConfiguration.cs", "section": "config#configurationsettings"}
-->

1. Save the file.

    **NOTE**: The namespace is `Microsoft.Portal.Extensions.MyExtension`, the full name of the class is `Microsoft.Portal.Extensions.MyExtension.ApplicationConfiguration`, and the configuration key is `Microsoft.Portal.Extensions.MyExtension.ApplicationConfiguration.PageSize`.

1. Open the `web.config` file of the extension.

1. Locate the `appSettings` section. Add a new entry for PageSize.

    ```xml
    ...
      <appSettings>
            ...
            <add key="Microsoft.Portal.Extensions.MyExtension.ApplicationConfiguration.PageSize" value="20"/>
      </appSettings>
      ...
    ```

1. Save and close the `web.config` file.

1. Open the `Definition.cs` file that is located in the `Controllers` folder. Add a new mapping in `ExtensionConfiguration` property.

    ```csharp
        /// <summary>
        /// Initializes a new instance of the <see cref="Definition"/> class.
        /// </summary>
        /// <param name="applicationConfiguration">The application configuration.</param>
        [ImportingConstructor]
        public Definition(ApplicationConfiguration applicationConfiguration)
        {
            this.ExtensionConfiguration = new Dictionary<string, object>()
            {
                ...
                { "pageSize", applicationConfiguration.PageSize },
            };
            ...
        }
    ```

1. Open the `FxEnvironmentExtensions.d.ts` file that is located in the  `Definitions` folder, and add the `pageSize` property in the environment interface.

    ```ts
        interface FxEnvironment {
            ...
            pageSize?: number;
        } 
    ```

1. The new configuration entry is now defined. To use the configuration, add code like the following in the script.

    ```JavaScript
        var pageSize = window.fx.environment && window.fx.environment.pageSize || 10;
    ```

An extended version of this procedure is used to transfer domain based configurations, like correctly formatted FwLinks, to the client. 

<a name="custom-domains-dictionary-configuration"></a>
## Dictionary configuration

The `DictionaryConfiguration` class allows strongly-typed JSON blobs to be defined in the configuration file, and selected based on an arbitrary, case-insensitive string key. For example, the Shell and Hubs use the class to select between domain-specific configuration sets. Two configuration classes can be created.

1. A configuration class that is derived from `DictionaryCollection` that manages and exposes the instances.

1. A stand-alone settings class that contains the setting values associated with a specific key and user culture.

Like other configuration classes, these are named and populated from the config based on namespace, class name, and the Settings property name. For example, if the namespace is `Microsoft.MyExtension.Configuration` and the configuration class is `MyConfiguration`, then the configuration setting name is `Microsoft.MyExtension.Configuration.MyConfiguration.Settings`.

Instances of configuration classes are normally obtained through MEF constructors, and this is unchanged for `StringDictionaryConfiguration` and its sub-classes.

At runtime, the strongly typed settings for a specific key are obtained by using the `GetSettings` method that is inherited by the configuration class, as in the following code.

 `T settings = configClass.GetSettings(string key, CultureInfo culture)` 
 
 The `culture` parameter is optional, and is used when expanding settings that are marked with the special `[Link]` attribute. If the `culture` parameter  is not specified, the default is  `CultureInfo.CurrentUICulture`.

  A boilerplate example is located at [consumption example](#consumption-example).

* The configuration class

    To create a configuration class, derive a class from `StringDictionaryConfiguration&lt;T&gt;`, where `T` is the type of the settings class.

    Remember to mark the class as MEF exportable if the config will be made available in the normal fashion, as in the following example.

    ```cs
    namespace Microsoft.MyExtension.Configuration
    {
        [Export]
        public class MyConfiguration : DictionaryConfiguration<MySettings>
        {
        }
    }
    ```

    Nested objects, like `Billing.EA.ShowPricing`, are fully supported, as in the example code located at [consumption example](#consumption-example).

* The settings class

    The settings class is a data transport object. The following example  contains the configuration class name `MySettings`, in addition to the settings class's namespace `Microsoft.MyExtension.Configuration`.

    All properties that are populated from the JSON blob in the configuration file are marked as `[JsonProperty]` so that the configuration system `ConfigurationSettingKind.Json` option can be used. If the properties are not marked, they will not be deserialized and will remain null.

    ```
    <table>
        <thead><tr><th>Example settings class</th><th>Example config*</th></tr></thead>
        <tr>
            <td>
                <pre>
    namespace Microsoft.MyExtension.Configuration
    {
        public class MySettings
        {
            [JsonProperty]
            public bool ShowPricing { get; private set; }
        }
    }
                </pre>
            </td>
            <td>
    <pre>
    &lt;add key="Microsoft.MyExtension.Configuration.MyConfiguration.Settings" value="{
        'default': {
            'showPricing': true
        },
        'someOtherKey' : {
            'showPricing': false
        }
    }" /&gt;
    </pre>
            </td>
        </tr>
    </table>
    ```

    **NOTE**: The deserializer handles camel-case to pascal-case conversion when the code uses JSON property name conventions in the config file and C# name conventions in the configuration classes.
 
<a name="custom-domains-dictionary-configuration-the-link-attribute"></a>
### The link attribute

Expansion logic is required for properties that are marked `[Link]`. The format string is specified in the `LinkTemplate` property that is located at the root of the object. A `LinkTemplate` value of `https://go.microsoft.com/fwLink/?LinkID={linkId}&amp;clcid=0x{lcid}` is the correct template for FwLinks.

Expansion in the format string is applied according to the following rules.

1. If the string is numeric, then occurrences of `{linkId}` in the string are expanded to the numeric value. If no `LinkTemplate` property is specified, the value will be left unexpanded.

1. Occurrences of `{lcid}` are replaced with the hex representation of the user's preferred .NET LCID value.  For example, 409 is the .NET LCID value for US English. 

1. Occurrences  of `{culture}` are replaced with the user's preferred .NET culture code. For example, en-US is the .NET culture code for US English.

An exception will be thrown if the target of a `[Link]` attribute is not in one of the following string formats.

* Numeric, for example,  '12345' 

* A URL hash-fragment, like "#create\Microsoft.Support"

* A http or https URL

<a name="custom-domains-dictionary-configuration-the-default-key"></a>
### The default key

If no exact match is found for the specified key, or if the caller sends a value of null, the `Get` function returns the settings associated with a key whose name is `default`.

<a name="custom-domains-dictionary-configuration-setting-inheritance"></a>
### Setting inheritance

`DictionaryConfiguration` supports a simplistic one-level inheritance model to avoid repeating unchanged settings in 'subclassed' config. 

If a property inside the settings for a non-default key is missing or explicitly set to null,  the value for that property is set to the value from the default key, as in the following example.

``` xml
    <add key="Microsoft.Portal.Framework.WebsiteDomainBasedConfiguration.Settings" value="{
        'default': { 'setting1': 'A1', 'setting2': 'A2' },
        'config2': { 'setting1': 'B1', 'setting2': 'B2' },
        'config3': { 'setting1': 'C1' },
        'config4': { 'setting1': null },
    }"/>
```

These settings return the following values.

| Key     | Setting 1 | Setting 2 | Notes |
| ------- | --------- | --------- | ----- |
| Default | A1        | A2        |       |
| Config1 | A1        | A1        | Since there is no config1 entry, the default is returned |
| Config2 | B1        | B2        | Both values were overridden |
| Config3 | C1        | A2        | Only setting1 was overridden |
| Config4 | A1        | A2        | Assigning null is the same as skipping the property |


<a name="custom-domains-branding-and-chrome"></a>
## Branding and Chrome

<!-- TODO:  Determine whether the Custom Domain Questionnaire should include a screen shot of the new extension that is similar to the one described in this section. -->

The following image is the Azure Dashboard. The titles and labels that it displays are the settings in the extension and its configuration files.

![alt-text](../media/top-extensions-custom-domains/branding-and-chrome.png "Branding and Chrome")

The following table specifies the parts in the dashboard image.

<!-- TODO: Determine whether "Public value" can be changed to "Default value". -->

| Setting or feature flag              | Description  | Default Value |
| ---------------------- | ----------- | ------------ | 
| URL                    | The address of the extension. |              |              |
| Title                  | The text that appears on the Browser tab/title bar. It is located in the page’s `<TITLE>` element | Micro`soft Azure |
| internalonly           | Controls whether the Orange ‘Preview’ tag appears to the left of the site name  | false  | 
| Product Name           | The text that appears on the top bar | Microsoft Azure | 
| hideSearchBox          | Hides the “Search resources” box  | false |
| feedback |  A value of `False` hides the  feedback  icon on top bar  | true |
| hideDashboardShare     | Hides the Dashboard Share Button  | false | 
| hideCreateButton       | Hides the Create ("+ New") Button | false | 
| defaultTheme           | The default color theme for the portal | blue |  

* Feature flags

These feature flags impact dashboard settings that are not immediately visible. The recommended values are prepopulated, although you can modify them for your extension.

| Setting  | Description  | Public Value  | 
| -------- | ------------ | --------------------------------------  | 
| hidesupport                 | Hides support functionality in property and other select dialogs. | Not set | 
| nps                         | Controls whether the Net Promoter Score (‘How likely are you to recommend this site?’) prompt can be displayed for the site. | true |
| hubsextension_skipeventpoll | Disables ‘what’s new’ and subscription level notifications like  deployment complete for VMs  | Not set  | 
| Description            | SEO text that is included in the page source that is not visible to the end-user | Microsoft Azure Management Portal | 

<a name="custom-domains-curation"></a>
## Curation

Curation allows items that are displayed on the left navigation bar to be added, removed, and reordered. This is an alternative to hiding items programmatically or making them accessible by using deep links. For example, you can hide the ability to create new storage accounts from users, while still allowing the extension to open the `Storage Accounts` property and then open the `usage logs` blades for a storage account that was created for one of the extension's assets. Curation is optional because the extension can inherit from the production environment.

<!-- TODO:  If the storage account example is fictitious, locate one that is not fictitious. -->

A browsable asset type is one that is defined in the `PDL` file by using the `Browse` tag. Curation controls the visibility and grouping of browsable asset types in the **Favorites and Browse** portions of the left navigation bar, or the  **Favorites and Category** section, as in the following image.

![alt-text](../media/top-extensions-custom-domains/curation.png "Categories and asset types")

<!-- TODO: Determine whether  "Public value" can be changed to "Default value". -->
 
<a name="custom-domains-curation-curation"></a>
### Curation

Curation provides a significant degree of flexibility, which can be overwhelming because options are NOT mutually exclusive. For example, the production curation definition can be programmatically modified in the following ways.

* Remove everything except the assets from three extensions

* For the third extension in the example, change the category in which items are located.
 
**NOTE**: Empty categories are automatically hidden.
 
The most common configurations are Community Clouds that display `Help & Support`, configs that automatically add new browsable assets, and configs that display only assets from the extension.

Use `Curation by AssetType` to list only items from the extension, and perhaps some items from `Help & Support`. It can be combined with `Curation by Extension` to allow new asset types you deploy to be displayed  while the  updates to your `Curation by AssetType` are waiting on the next Shell deployment, combined with a `DiscoveryPolicy` of "Hide" to prevent the display of new asset types from other extensions.
		
* All AAD extensions except "KeyVault" and "Help + Support"

* Default all items to go under "Security + Identity" category, including "Help + Support"
 
<a name="custom-domains-curation-curation-default-favorites"></a>
#### Default Favorites

When a new user visits your Community Cloud for the first time, the system places several asset types in the far left drawer. You can control which items are placed there, in addtion to the order in which they are displayed. The only restriction is that these items must also exist in the Category Curation.
 
| Extension Name    | Asset ID                       | Comments            |
| ----------------- | ------------------------------ | ------------------- |
| MICROSOFT_AAD_IAM | AzureActiveDirectory           |                     |
| MICROSOFT_AAD_IAM | AzureActiveDirectoryQuickStart | Not implemented yet | 
| MICROSOFT_AAD_IAM | UserManagement                 |                     | 
| MICROSOFT_AAD_IAM | Application                    |                     | 
| MICROSOFT_AAD_IAM | Licenses                       | Not implemented yet | 
 
* Azure Active Directory
* Quick start
* Users and groups
* Enterprise apps
* Licenses
	
<a name="custom-domains-curation-curation-tile-gallery"></a>
#### Tile Gallery

<!-- TODO: Determine what the tile gallery has to do with custom domains and/ or the questionnaire template -->

The tile gallery is visible when the user clicks on `Edit dashboard`. It displays a collection of tiles that can be dragged and dropped on the dashboard. Available tiles can be searched by Category, by Type, by resource group, by tag, or by using the Search string.

* The `hidePartsGalleryPivot` flag disables all the search types except the Category search. The Category selector will be displayed only if any tile has a category assigned to it.

* The `hiddenGalleryParts` list allows this extension to hide specific parts that are made available by other extensions. For example, by default, the Service Health part is always displayed, but it can be hidden by adding it to this list.

Items are added in the order listed.
 

| Setting or feature flag              | Description  | Default Value |
| ---------------------- | ----------- | ------------ | 
| hidePartsGalleryPivots | Hides parts types picker from parts gallery. Does not disable the category picker  | false | 
| hiddenGalleryParts     | Hides listed parts from the parts gallery, like `All Resources`, `Service Health`, and others  | empty | 


<a name="custom-domains-curation-curation-default-dashboard"></a>
#### Default Dashboard

The default dashboard JSON controls what parts appear on the dashboard for new users. Existing users need to use the `Reset Dashboard` option to see updated versions of the default dashboard. The following steps generate the JSON. 

1. Use the Portal’s `customize` mode to manually create the dashboard.  Pin or drag and drop the dashboard contents into place.

1. Use the Portal’s `Share` feature to publish the dashboard to an Azure subscription.

 **NOTE**: The subscription is  not related to your content; this is just a means to an end.

1. Navigate to `More Services -> Resource Explorer` and find the dashboard that was  just published. The `Resource Explorer` will display the JSON representation of the dashboard.  You can review and adjust  the tile sizes, positions, and settings.  

1. Inspect the JSON to confirm that it contains no user-specific values, because this dashboard will be applied to all new users.

1. Paste the following `AzureActiveDirectoryDashboardv2.json` File from AAD into the Default Dashboard JSON. The file is also located at [](). 

    ```
    {
        "properties": {
            "lenses": {
                "0": {
                    "order": 0,
                    "parts": {
                        "0": {
                            "position": {
                                "x": 0,
                                "y": 0,
                                "rowSpan": 2,
                                "colSpan": 4
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/OrganizationIdentityPart"
                            }
                        },
                        "1": {
                            "position": {
                                "x": 4,
                                "y": 0,
                                "rowSpan": 2,
                                "colSpan": 4
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/AzurePortalWelcomePart"
                            }
                        },
                        "2": {
                            "position": {
                                "x": 0,
                                "y": 2,
                                "rowSpan": 2,
                                "colSpan": 4
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/Blade/ActiveDirectoryBlade/Lens/ActiveDirectoryLens/PartInstance/ActiveDirectory_UserManagementSummaryPart"
                            }
                        },
                        "3": {
                            "position": {
                                "x": 0,
                                "y": 4,
                                "rowSpan": 2,
                                "colSpan": 4
                            },
                            "metadata": {
                                "inputs": [
                                    {
                                        "name": "userObjectId",
                                        "isOptional": true
                                    },
                                    {
                                        "name": "startDate",
                                        "isOptional": true
                                    },
                                    {
                                        "name": "endDate",
                                        "isOptional": true
                                    },
                                    {
                                        "name": "fromAppsTile",
                                        "isOptional": true
                                    }
                                ],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/UsersActivitySummaryReportPart"
                            }
                        },
                        "4": {
                            "position": {
                                "x": 0,
                                "y": 6,
                                "rowSpan": 1,
                                "colSpan": 2
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/ADConnectStatusPart"
                            }
                        },
                        "5": {
                            "position": {
                                "x": 2,
                                "y": 6,
                                "rowSpan": 1,
                                "colSpan": 2
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/AuditEventsDashboardPart"
                            }
                        },
                        "6": {
                            "position": {
                                "x": 4,
                                "y": 2,
                                "rowSpan": 4,
                                "colSpan": 4
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/ActiveDirectoryRecommendationPart",
                                "viewState": {
                                    "content": {
                                        "selection": {
                                            "selectedItems": [],
                                            "activatedItems": []
                                        }
                                    }
                                }
                            }
                        },
                        "7": {
                            "position": {
                                "x": 8,
                                "y": 0,
                                "rowSpan": 2,
                                "colSpan": 2
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/ActiveDirectoryQuickTasksPart"
                            }
                        },
                        "8": {
                            "position": {
                                "x": 8,
                                "y": 2,
                                "rowSpan": 1,
                                "colSpan": 2
                            },
                            "metadata": {
                                "inputs": [],
                                "type": "Extension/Microsoft_AAD_IAM/PartType/AzurePortalPart"
                            }
                        }
                    }
                }
            },
            "metadata": {
                "model": {
                    "timeRange": {
                        "value": {
                            "relative": {
                                "duration": 24,
                                "timeUnit": 1
                            }
                        },
                        "type": "MsPortalFx.Composition.Configuration.ValueTypes.TimeRange"
                    }
                }
            }
        },
        "id": "/subscriptions/8bd095e7-b4a6-4f9c-b826-8b83943111fa/resourceGroups/dashboards/providers/Microsoft.Portal/dashboards/438319b8-6761-4c27-9f1f-97b12413c30b",
        "name": "438319b8-6761-4c27-9f1f-97b12413c30b",
        "type": "Microsoft.Portal/dashboards",
        "location": "eastasia",
        "tags": {
            "hidden-title": "Dashboard"
        }
    }
    ```

    <!--TODO:  Locate a copy that can be linked to from here, instead of the following OneNote link.
    [https://microsoft.sharepoint.com/teams/azureteams/aapt/azureux/portalfx/_layouts/OneNote.aspx?id=%2Fteams%2Fazureteams%2Faapt%2Fazureux%2Fportalfx%2FSiteAssets%2FPortalFx%20Notebook&wd=target%28Execution%2FFundamentals%2FDeployments.one%7C9B8BE2F4-DDEF-4504-982B-560AF50A892C%2FCustom%20Domain%20-%20Questionnaire%20Template%7C90BDECEB-D69D-4BA0-B60A-8A9EBB877CC4%2F%29](https://microsoft.sharepoint.com/teams/azureteams/aapt/azureux/portalfx/_layouts/OneNote.aspx?id=%2Fteams%2Fazureteams%2Faapt%2Fazureux%2Fportalfx%2FSiteAssets%2FPortalFx%20Notebook&wd=target%28Execution%2FFundamentals%2FDeployments.one%7C9B8BE2F4-DDEF-4504-982B-560AF50A892C%2FCustom%20Domain%20-%20Questionnaire%20Template%7C90BDECEB-D69D-4BA0-B60A-8A9EBB877CC4%2F%29)
    -->


<a name="custom-domains-override-links"></a>
## Override links

Your cloud has the option of using settings to override specific links that are displayed by the system. Overriding is optional, and in many cases no overrides are required. Where supported, settings use FwLinks for links instead of absolute URLs because FwLinks do not require the Shell to be redeployed in order for the extension to change the destination. Also, FwLinks support the user’s in-product language selection, which is often different from the browser’s default language.  For example, if the user has set the language in the Portal to Chinese, it should display Chinese-language pages.

Each link can be specified as one of the following five values.

* A numeric FwLink ID

* A blade reference

* An absolute URL

    Absolute URL's can be assigned friendly names by using the site located at [http://aka.ms](http://aka.ms).

* The word "blank"

    The link should be hidden and not displayed to the user.

* The word "same"

    The default production value should be used. If the default production value changes, that change is automatically propagated to the extension.

Links are separated into the following three sections.

* [Shell Links](#shell-links)

* [Hubs links](#hubs-links) 

* [Error Page Links](#error-page-links)

<sup>1</sup> Items do not support FwLinks or blade references because  they are  download links, or they are base URLs to which  parameter or path information is added dynamically at run-time.

<sup>2</sup> Items may not currently support the "blank" setting, or they may not support  values of the types other than the ones listed in the `Public Value` column. If you need to use them, reach out to  <a href="mailto:ibizapxfm@microsoft.com?subject=Settings and Links">ibizapxfm@microsoft.com</a>so that we can work with the owner team.

**NOTE**: Changing to support the unsupported values may result in a delay in delivery time.

* * *

<a name="custom-domains-override-links-shell-links"></a>
### Shell Links

| Setting name / notes | Public Value | Extension Value |
| -------------------- | ------------ | ---------- |
| accessDetails<sup>2</sup> | #blade/HubsExtension/MyAccessBlade/resourceId/ |	blank (verify that menu item is hidden) | 
| accountPortal<sup>1</sup> | https://account.windowsazure.com/ | blank (verify that menu item is hidden) | 
| classicPortal<sup>1</sup> | https://manage.windowsazure.com/	 | same  | 
| createSupportRequest | #create/Microsoft.Support | same |
| giveFeedback | [https://go.microsoft.com/fwLink/?LinkID=522329](https://go.microsoft.com/fwLink/?LinkID=522329) | [https://go.microsoft.com/fwlink/?linkid=838978](https://go.microsoft.com/fwlink/?linkid=838978) | 
| helpAndSupport | 	#blade/Microsoft_Azure_Support/HelpAndSupportBlade | same | 
| learnRelatedResources	| [https://go.microsoft.com/fwLink/?LinkID=618605](https://go.microsoft.com/fwLink/?LinkID=618605) | same  | 
| learnSharedDashboard | [https://go.microsoft.com/fwLink/?LinkID=746967](https://go.microsoft.com/fwLink/?LinkID=746967) | same  | 
| manageSupportRequests | #blade/HubsExtension/BrowseServiceBlade/        assetTypeId/Microsoft_Azure_Support_SupportRequest | same |
| privacyAndTerms | [https://go.microsoft.com/fwLink/?LinkID=522330](https://go.microsoft.com/fwLink/?LinkID=522330)	 | same | 
| resourceGroupOverview	| [https://go.microsoft.com/fwLink/?LinkID=394393](https://go.microsoft.com/fwLink/?LinkID=394393) | same  | 
| survey	| [https://go.microsoft.com/fwLink/?LinkID=733278](https://go.microsoft.com/fwLink/?LinkID=733278)  | 	??? Gauge team to follow up on this | 
| joinResearchPanel | [https://uriux.fluidsurveys.com/s/MicrosoftReseachPanel/](https://uriux.fluidsurveys.com/s/MicrosoftReseachPanel) | same |
| learnAzureCli<sup>2</sup> | 	[https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-azure-resource-manager/](https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-azure-resource-manager/)	 | same |
 
<a name="custom-domains-override-links-hubs-links"></a>
### Hubs links

| Setting name / notes  | Public Value | Extension Value |
| --------------------- | ------------ | ---------- |
| createNewSubscription     | [https://go.microsoft.com/fwLink/?LinkID=522331](https://go.microsoft.com/fwLink/?LinkID=522331)	 | same |
| manageAzureResourceHelp   | [https://go.microsoft.com/fwLink/?LinkID=394637](https://go.microsoft.com/fwLink/?LinkID=394637)	 | same |
| moveResourcesDoc<sup>2</sup>	| [https://go.microsoft.com/fwLink/?LinkID=747963](https://go.microsoft.com/fwLink/?LinkID=747963)	 | same |
| resourceGroupInstallClientLibraries (not found)	| [https://go.microsoft.com/fwLink/?LinkID=234674](https://go.microsoft.com/fwLink/?LinkID=234674)	 | same |
| resourceGroupInstallPowerShell<sup>1</sup>	| [https://go.microsoft.com/fwLink/?LinkID=https://go.microsoft.com/?linkid=9811175](https://go.microsoft.com/fwLink/?LinkID=https://go.microsoft.com/?linkid=9811175)			(download link) | same |
| resourceGroupInstallTools (page not found) | [https://go.microsoft.com/fwLink/?LinkID=94686](https://go.microsoft.com/fwLink/?LinkID=94686)	 | same |
| resourceGroupIntroduction	| [https://go.microsoft.com/fwLink/?LinkID=394393](https://go.microsoft.com/fwLink/?LinkID=394393)	 | same |
| resourceGroupIntroductionVideo	| [https://go.microsoft.com/fwLink/?LinkID=394394](https://go.microsoft.com/fwLink/?LinkID=394394)	 | same |
| resourceGroupResourceManagement	| [https://go.microsoft.com/fwLink/?LinkID=394396](https://go.microsoft.com/fwLink/?LinkID=394396)	 | same |
| resourceGroupSample	| [https://go.microsoft.com/fwLink/?LinkID=394397](https://go.microsoft.com/fwLink/?LinkID=394397)	 | same |
| resourceGroupTemplate	| [https://go.microsoft.com/fwLink/?LinkID=394395](https://go.microsoft.com/fwLink/?LinkID=394395)	 | same |
| subCerts	| [https://go.microsoft.com/fwLink/?LinkID=734721](https://go.microsoft.com/fwLink/?LinkID=734721)	 | same |
| tourHelp	| [https://go.microsoft.com/fwLink/?LinkID=626007](https://go.microsoft.com/fwLink/?LinkID=626007)	 | same |
| templateDeployment	| [https://go.microsoft.com/fwLink/?LinkID=733371](https://go.microsoft.com/fwLink/?LinkID=733371)	 | same |
| tagsHelp<sup>2</sup> | [https://go.microsoft.com/fwLink/?LinkID=822935](https://go.microsoft.com/fwLink/?LinkID=822935)  | same |
| pricingHelp<sup>2</sup> | [https://go.microsoft.com/fwLink/?LinkID=829091](https://go.microsoft.com/fwLink/?LinkID=829091)  | same |
| azureStatus<sup>2</sup> | [https://status.azure.com]()  | same |
 
<a name="custom-domains-override-links-error-page-links"></a>
### Error Page Links
	
| Setting name / notes | Public Value |	Extension Value |
| -------------------- | ------------ | ---------- |
| classicPortal<sup>1</sup>	   | https://manage.windowsazure.com/	| same |
| contactSupport |	[https://go.microsoft.com/fwLink/?LinkID=733312](https://go.microsoft.com/fwLink/?LinkID=733312)	 | same |
| html5StorageExceededHelp |	[https://go.microsoft.com/fwLink/?LinkID=522344](https://go.microsoft.com/fwLink/?LinkID=522344)	 | same |
| javaScriptDisabledHelp |	[https://go.microsoft.com/fwLink/?LinkID=530268](https://go.microsoft.com/fwLink/?LinkID=530268)	 | same |
| noHtml5StorageHelp |	[https://go.microsoft.com/fwLink/?LinkID=522344](https://go.microsoft.com/fwLink/?LinkID=522344)	 | same |
| portalVideo |	[https://go.microsoft.com/fwLink/?LinkID=394684](https://go.microsoft.com/fwLink/?LinkID=394684) |	 blank |
| supportedBrowserMatrix |	[https://go.microsoft.com/fwLink/?LinkID=394683](https://go.microsoft.com/fwLink/?LinkID=394683)	 | same |
| unsupportedLayoutHelp	 |[https://go.microsoft.com/fwLink/?LinkID=394683]()	 | same |



<a name="custom-domains-override-links-consumption-example"></a>
### Consumption example

The following three examples demonstrate how to 

* Consumption example 

    ```cs
    [ImportingConstructor]
    public MyConsumingClass(PortalContext portalContext, MyConfiguration myConfiguration)
    {
        this.portalContext = portalContext;
        this.myConfiguration = myConfiguration;
    }
    ...
    public void DoSomthing()
    {
        var settings = this.myConfiguration.Get(this.portalContext.TrustedAuthorityHost, CultureInfo.CurrentUICulture);
        if (settings.ShowPricing) {...};
        string expandedUrl = settings.GettingStarted;
    }
    ```

* Configuration and settings classes

    ```cs
    namespace Microsoft.MyExtension.Configuration
    {
        [Export]
        public class MyConfiguration : DictionaryConfiguration<MySettings>
        {
        }

        /// <summary>Configuration that can vary by the domain by which the user accesses the portal (or some other string)</summary>
        public class MySettings
        {
            [JsonProperty]
            public string LinkTemplate { get; private set; }
    
            [JsonProperty]
            public MyLinks Links { get; private set; }
    
            [JsonProperty]
            public bool ShowPricing{ get; private set; }
        }

        /// <summary>Links don't have to be a separate class, I just like to group them separately to other settings</summary>
        public class MyLinks
        {
            [JsonProperty, Link]
            public string GettingStarted { get; private set; }
    
            [JsonProperty, Link]
            public string Support { get; private set; }
    
            [JsonProperty, Link]
            public string TermsAndConditions { get; private set; }
        }
    }
    ```

* Corresponding example config settings

    This example supports a deployment that returns different run-time configuration values for the settings class. The value that is returned is dependent on whether the portal was accessed through `portal.azure.com`, `example.microsoft.com`, `fujitsu.portal.azure.com`, or `hostfileoverride.com`. The configuration block is in the following code.

    ```xml
    <add key="Microsoft.MyExtension.Configuration.MyConfiguration.Settings" value="{
        'default': {
            'linkTemplate': 'https://go.microsoft.com/fwLink/?LinkID={linkId}&amp;clcid=0x{lcid}',
            'links': {
                'gettingStarted': '111111',
                'support': '#create/Microsoft.Support',
                'termsAndConditions': 'https://microsoft.com',
            },
            'ShowPricing': true,
        },
        'fujitsu.portal.azure.com' : {
            'links': {
                'gettingStarted': '222222',
                'support': '',
                'termsAndConditions': 'https://fujitsu.com',
            },
            'ShowPricing': false,
        },
        'example.microsoft.com': {
            'ShowPricing': false,
        }
        }"/>
    ```

    The following code calls the configuration code block.

    ```cs
        var config = this.myConfiguration.Get(this.portalContext.TrustedAuthorityHost, CultureInfo.CurrentUICulture);
    ```

    The call to the code block results in the following values.

    | URL                        | User's culture | config.showPricing | gettingStarted  | support |termsAndConditions |
    | ------------------------------------ | --------------|------------------|---------------------------|--------------------|-------------------------------
    | `portal.azure.com`         | en-us   | true  | [https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x409](https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x409) | #create/Microsoft.Support | [https://microsoft.com](https://microsoft.com) |
    | `portal.azure.com`         | zh-hans | true  | [https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x4](https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x4) | #create/Microsoft.Support | [https://microsoft.com](https://microsoft.com) |
    | `fujitsu.portal.azure.com` | en-us   | false | [https://go.microsoft.com/fwLink/?LinkID=222222&clcid=0x409](https://go.microsoft.com/fwLink/?LinkID=222222&clcid=0x409) | | [https://fujitsu.com](https://fujitsu.com)
    | `example.microsoft.com`    | en-us   | false | [https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x409](https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x409) | #create/Microsoft.Support  | [https://microsoft.com](https://microsoft.com) |
    | `hostFileOverride.com`     | en-us   | true  | [https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x409](https://go.microsoft.com/fwLink/?LinkID=111111&clcid=0x409) | #create/Microsoft.Support | [https://microsoft.com](https://microsoft.com) |
 
**NOTE**: The only URL that matters is the one that the user uses to access the Portal. The URL that is used to access the extension is not relevant and does not change.

**NOTE**: `gettingStarted`, `support`, and `termsAndConditions` are members of the  `links` parameter in the `config` variable.



<a name="custom-domains-custom-domain-questionnaire-template"></a>
## Custom Domain - Questionnaire Template

The following template contains questions that your team answers previous to  the granting of the  custom domain. You may want to make a copy and fill the details.

1. Why do you need a Custom Domain?

1. What is the name of the extension in Ibiza Portal?

1. When do you expect the extension to be ready for deployment?

1. What timelines are you looking to go live? 

    | Requirement                        | Estimated Completion Date |
    | ---------------------------------- | ------------------------- |
    | Azure Portal team PM Lead approval |                           |
    | Completed Questionnaire            |                           |
    | Completed Default Dashboard Json   |                           |
    | Planning for Dev work              | 1 week                    |
    | Dev work                           | Requires 3-4 weeks after scheduling, subject to resource availability | 
    | Deployments                        | Post dev work 2-3 weeks to Prod based on Safe deployment schedule | 

1. URL

	| Setting name / notes	| Public Value	        | Extension value                  |
    | --------------------- | --------------------  | -------------------------------- |
    | Production URL        | `portal.azure.com`    | `aad.portal.azure.com`           |
    | Dogfood URL           | `df.portal.azure.com` | `df-aad.onecloud.azure-test.net` |

1. Branding and Chrome Values

Recommended extension values are located in [portalfx-extensions-bp-custom-domains.md# branding-and-chrome-values](portalfx-extensions-bp-custom-domains.md#branding-and-chrome-values). Include the values for  settings and feature flags for your extension in the following list by making a copy and adding the appropriate details.

| Setting or feature flag     | Extension value |
| ---------------------- | ---------------   |
| URL                    | The address of the extension |       
| Title                  | Azure Active Directory admin center |
| internalonly           | false | 
| Product Name           | Azure Active Directory admin center  |
| hideSearchBox          | true |
| feedback               | true |  
| hideDashboardShare     | true |
| hideCreateButton       | true |
| defaultTheme           | light |
| Description            | Azure Active Directory admin center for administrators |
| hidePartsGalleryPivots | true |
| hiddenGalleryParts     | Only include markdown, clock and video, help & support |

* Feature flags

These feature flags impact dashboard settings that are not immediately visible. The recommended values are prepopulated, although you can modify them for your extension.

| Setting                     | Recommended Value  | 
| --------------------------- | -------- |
| hidesupport                 | false |
| nps                         | false |
| hubsextension_skipeventpoll | True, unless showing all resource types from all extensions. |

 
 ## Best Practices

<a name="custom-domains-custom-domain-questionnaire-template-using-portalcontext-trustedauthorityhost-properly"></a>
### Using PortalContext.TrustedAuthorityHost properly

Do not use PortalContext.TrustedAuthorityHost directly, as in the following example. 

```cs
if (PortalContext.TrustedAuthorityHost.startsWith("contoso"))
{
    // We're in contoso.portal.azure.com, contoso-df.portal.azure.com, contoso.onestb.cloudapp.net etc
    ...
}
```

This is an anti-pattern for several reasons. The code becomes peppered with Cloud specific 'if' blocks that are hard to test, maintain, and find. This increases in complexity as the number of supported Clouds increases. Also, if another cloud that requires any of this logic comes online, the code has to be updated. Instead, use domain-based configuration to create a setting that varies by `PortalContext.TrustedAuthorityHost`, which is available at run time and returns the host name under which the extension was loaded.

<a name="custom-domains-custom-domain-questionnaire-template-dynamic-pdl-changes"></a>
### Dynamic PDL changes

Never change the PDL that your extension serves based on the host of the caller, or based on feature flags that change based on callers.

1. The Shell caches the PDL on behalf of the client, and the same PDL bundle is delivered to all community clouds.

1. When the Shell directly requests the PDL from each extension, it does so on its own behalf and not on behalf of any specific user. In such cases, `PortalContext.TrustedAuthorityHost` is a constant.

1. Changing the default feature flags that are sent to the extension requires Shell config changes and redeployment.

<a name="custom-domains-custom-domain-questionnaire-template-branding-and-chrome-values"></a>
### Branding and Chrome Values

The following values are recommended, but not required, values for  setting and feature flags for  extension branding and chrome.

| Setting or feature flag     | Extension value |
| ---------------------- | ---------------   |
| URL                    | The address of the extension |       
| Title                  | Azure Active Directory admin center |
| internalonly           | false | 
| Product Name           | Azure Active Directory admin center  |
| hideSearchBox          | true |
| feedback               | true |  
| hideDashboardShare     | true |
| hideCreateButton       | true |
| defaultTheme           | light |
| Description            | Azure Active Directory admin center for administrators |
| hidePartsGalleryPivots | true |
| hiddenGalleryParts     | Only include markdown, clock and video, help & support |

* Feature flags

These feature flags impact dashboard settings that are not immediately visible. The recommended values are prepopulated, although you can modify them for your extension.

| Setting                     | Recommended Value  | 
| --------------------------- | -------- |
| hidesupport                 | false |
| nps                         | false | 	
| hubsextension_skipeventpoll | True, unless showing all resource types from all extensions. |


 
<a name="custom-domains-frequently-asked-questions"></a>
## Frequently Asked Questions

<a name="custom-domains-frequently-asked-questions-site-is-not-accessible"></a>
### Site is not accessible

***My site is not accessible from a custom URL in DogFood***

SOLUTION:  Verify that the configuration setting  `Microsoft.Portal.Framework.FrameworkConfiguration.AllowedParentFrame` is correctly set in your extension’s configuration file. Your extension will reject calls from pages from domains and subdomains of the values listed here. For example, a setting value of `df.onecloud.azure-test.net` will NOT allow calls from pages hosted at `df-myExtension.onecloud.azure-test.net`, but a setting of `onecloud.azure-test.net` will allow calls from both `df.onecloud.azure-test.net` and `df-myExtension.onecloud.azure-test.net`.

* * * 

<a name="custom-domains-frequently-asked-questions-default-favorites-list-do-not-match-the-submitted-list"></a>
### Default favorites list do not match the submitted list

***I changed the default favorites definition, but am still seeing the old one.***

SOLUTION: Default favorites are user-configurable and stored in `User Settings`. The system generates and saves the user's default favorites only if they haven’t been previously generated for the user. To force a refresh, reset your desktop.

* * *

<a name="custom-domains-frequently-asked-questions-shared-url-for-cloud-and-production"></a>
### Shared URL for cloud and production

***Can I point my Community Cloud and Production to the same extension URL?***

Yes, if the only difference between your Community Cloud and Production is branding the hiding of other extensions UI. 

However, a major reuse restriction is that you must serve the same PDL to both Production and your Community Cloud. You can serve different domain-based configuration to the user’s browser,  as specified in `AuxDocs`, and you can review  `PortalContext.TrustedAuthorityHost` to determine the  environment from which the client is calling your extension.  However, you cannot change the behavior of server-to-server calls, and PDL is requested by servers.

