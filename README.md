# Kentico Cloud Content Management .NET SDK

[![Build status](https://ci.appveyor.com/api/projects/status/6b3tt1kc3ogmcav3/branch/master?svg=true)](https://ci.appveyor.com/project/kentico/content-management-sdk-net/branch/master)
[![NuGet](https://img.shields.io/nuget/v/KenticoCloud.ContentManagement.svg)](https://www.nuget.org/packages/KenticoCloud.ContentManagement)
[![NuGet](https://img.shields.io/nuget/dt/kenticocloud.ContentManagement.svg)](https://www.nuget.org/packages/KenticoCloud.ContentManagement)
[![Forums](https://img.shields.io/badge/chat-on%20forums-orange.svg)](https://forums.kenticocloud.com)

## Summary

The Kentico Cloud Content Management .NET SDK is a client library used for managing content in Kentico Cloud. It provides read/write access to your Kentico Cloud projects.  
You can use the SDK to migrate existing content into your Kentico Cloud project or update content in your content items. You can import content items, their language variants, and assets.

The Content Management SDK does not provide any content filtering options and is not optimized for content delivery. If you need to deliver larger amounts of content we recommend using the [Delivery SDK](https://github.com/Kentico/delivery-sdk-net) instead.

You can head over to our Developer Hub for the complete [Content Management API Reference](https://developer.kenticocloud.com/v1/reference#content-management-api). 

## Prerequisites

To manage content in a Kentico Cloud project via the Content Management API, you first need to activate the API for the project. See our documentation on how you can [activate the Content Management API](https://developer.kenticocloud.com/v1/docs/importing-to-kentico-cloud#section-enabling-the-api-for-your-project).

You also need to prepare the structure of your Kentico Cloud project before importing your content. At minimum, that means defining the [Content types](https://help.kenticocloud.com/define-content-structure/structure/creating-and-deleting-content-types) of the items you want to impoort. You might also need to set up your [Languages](https://help.kenticocloud.com/multilingual-content#managing-languages), [Taxonomy](https://help.kenticocloud.com/categorize-content/working-with-taxonomy) or [Sitemap locations](https://help.kenticocloud.com/categorize-content/using-a-sitemap) (if you are using them). 

## Using the ContentManagementClient

The `ContentManagementClient` class is the main class of the SDK. Using this class, you can import, update, view and delete content items, language variants, and assets in your Kentico Cloud projects. 

To create an instance of the class, you need to provide a [project ID](https://developer.kenticocloud.com/docs/using-delivery-api#section-getting-project-id) and a valid [Content Management API Key](https://developer.kenticocloud.com/v1/docs/importing-to-kentico-cloud#importing-content-items).

```csharp
ContentManagementOptions options = new ContentManagementOptions() { 
    ProjectId = "bb6882a0-3088-405c-a6ac-4a0da46810b0",
    ApiKey = "ew0...1eo" 
}; 
// Initialize an instance of the ContentManagementClient client
ContentManagementClient client = new ContentManagementClient(options);
```

Once you create a `ContentManagementClient`, you can start managing content in your project by calling methods on the client instance. See [Importing content items](#importing-content-items) for details.

### Strongly-typed responses

The `ContentManagementClient` also supports working with strongly-typed models.
```csharp

 // Elements to update     
CafeeModel stronglyTypedElements = new CafeModel
{
    Title = "On Roasts"
};

// Upsert a language variant of a content item
ContentItemVariantModel<CafeModel> responseVariant = await client.UpsertContentItemVariantAsync<CafeModel>(identifier, stronglyTypedElements); 
```
For more code examples using strongly-typed models, see our [CM API reference](https://developer.kenticocloud.com/v1/reference#content-management-api).

### Codename vs. ID vs. External ID

Most methods of the SDK accept an *Identifier* object that specifies which content item, language variant or asset you want to perform the given operation on. There are 3 types of identification you can use to create the identifier: 

```csharp
ContentItemIdentifier identifier = ContentItemIdentifier.ByCodename("on_roasts");
ContentItemIdentifier identifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));
ContentItemIdentifier identifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");
```

* **Codenames** are generated automatically by Kentico Cloud based on the object's name. They can make your code more readable but are not guaranteed to be unique. They should only be used in circumstances with no chance of naming conflicts. 
* (internal) **IDs** are random [GUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier) assigned to objects by Kentico Cloud at the moment of import/creation. They are unique, but only objects that are already in the system have them. You can't use them to refer to content that hasn't yet been imported. 
* **External IDs** are string-based custom identifiers defined by you. This is useful when importing a batch of cross-referencing content. See [Importing Modular and linked content](#importing-modular-and-linked-content). 

## Quick start

### Importing content items

Importing content items is a 2 step process, using 2 separate methods:

1. Creating an empty content item which serves as a wrapper for your content.
2. Adding content inside a language variant of the content item.

Each content item can consist of several localized variants. **The content itself is always part of a specific language variant, even if your project only uses one language**. See our [Importing to Kentico Cloud](https://developer.kenticocloud.com/v1/docs/importing-to-kentico-cloud#section-importing-your-content) tutorial for a more detailed explanation. 

#### 1. Creating a content item

```csharp
// Create an instance of the Content Management client
ContentManagementClient client = new ContentManagementClient(options);

// Define the imported content item
ContentItemCreateModel item = new ContentItemCreateModel() { 
    Name = "On Roasts",
    Type = ContentTypeIdentifier.ByCodename("article"),
    SitemapLocations = new[] { SitemapNodeIdentifier.ByCodename("articles") }
};

// Add your content item to your project in Kentico Cloud
ContentItemModel response = await client.CreateContentItemAsync(item);
```

Kentico Cloud will generate an internal ID and codename for the (new and empty) content item and include it in the response. In the next step, we will add the actual (localized) content.


#### 2. Adding language variants

To add localized content, you have to specify: 

* The content item you are importing into.
* The language variant of the content item.
* The content elements of the language variant you want to insert or update. Omitted elements will remain unchanged. 

```csharp
var elements = new {
    title = "On Roasts",
    post_date = new DateTime(2017, 7, 4),
    body_copy = @"
        <h1>Light Roasts</h1>
        <p>Usually roasted for 6 - 8 minutes or simply until achieving a light brown color. This method
        is used for milder coffee  varieties and for coffee tasting. This type of roasting allows the natural
        characteristics of each coffee to show. The aroma of coffees produced from light roasts is usually
        more intense.The cup itself is more acidic and the concentration of caffeine is higher.</p>
    ",
    related_articles = new [] { ContentItemIdentifier.ByCodename("which_brewing_fits_you_") },
    url_pattern = "on-roasts",
    personas = new [] { TaxonomyTermIdentifier.ByCodename("barista") }
};
ContentItemVariantUpsertModel upsertModel = new ContentItemVariantUpsertModel() { Elements = elements };

// Specify the content item and the language varaint 
ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByCodename("on_roasts");
LanguageIdentifier languageIdentifier = LanguageIdentifier.ByLanguageCodename("en-US");
ContentItemVariantIdentifier identifier = new ContentItemVariantIdentifier(itemIdentifier, languageIdentifier);

// Upsert a language variant of your content item
ContentItemVariantModel response = await client.UpsertContentItemVariantAsync(identifier, upsertModel);
```

### Importing assets

Importing assets using Content Management API is usually a 2-step process:

1. Uploading a file to Kentico Cloud.
2. Creating a new asset using the given file reference.

This SDK, however, simplifies the process and allows you to upload assets using a single method: 

```csharp 
AssetDescription assetDescription = new AssetDescription { 
    Description = "Description of the asset in English Language",
    Language = LanguageIdentifier.ByCodename("en-US") 
};
AssetDescription[] descriptions = new [] { assetDescription };

string filePath = "‪C:\Users\Kentico\Desktop\puppies.png";
string contentType = "image/png";

AssetModel response = await client.CreateAssetAsync(new FileContentSource(filePath, contentType), descriptions);
```

### Importing Modular and linked content

The content you are importing will often contain references to other pieces of imported content. A content item can reference assets or point to other content items using modular content elements or links. To avoid having to import objects in a specific order (and solve problems with cyclical dependencies), you can use **external IDs** to reference non-existent (not-yet-imported) content: 

1. Define external IDs for all content items and assets you want to import in advance. 
2. When referencing another content item ar asset, use its external ID. 
3. Import your content (use upsert methods with external ID). All references will resolve in the end.

This way, you can import your content in any order and run the import process repeatedly to keep your project up to date. In the example below we import an asset and a content item that uses it: 

```csharp
// Upsert an asset, assuming you already have the fileResult reference to the uploaded file
AssetUpsertModel asset = new AssetUpsertModel {
    FileReference = fileResult
};
string assetExternalId = "Ext-Asset-123-png";
AssetModel assetResponse = await client.UpsertAssetByExternalIdAsync(assetExternalId, asset);

// Upsert a content item
ContentItemUpsertModel item = new ContentItemUpsertModel() { 
    Name = "Brno", 
    Type = ContentTypeIdentifier.ByCodename("cafe")
};
string itemExternalId = "Ext-Item-456-Brno";
ContentItemModel itemResponse = await client.UpsertContentItemByExternalIdAsync(itemExternalId, item);

// Upsert a language variant which references the asset using external ID
var elements = {
    picture = AssetIdentifier.ByExternalId(assetExternalId),
    city = "Brno",
    country = "Czech Republic"
};
ContentItemVariantUpsertModel upsertModel = new ContentItemVariantUpsertModel() { Elements = elements };

ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByExternalId(itemExternalId);
LanguageIdentifier languageIdentifier = LanguageIdentifier.ByCodename("en-US");
ContentItemVariantIdentifier identifier = new ContentItemVariantIdentifier(itemIdentifier, languageIdentifier);

ContentItemVariantModel variantResponse = await client.UpsertContentItemVariantAsync(identifier, upsertModel);
```

### Content item methods

#### Upserting a content item by external ID

```csharp
ContentItemUpsertModel item = new ContentItemUpsertModel() { 
    Name = "New or updated name",
    Type = ContentTypeIdentifier.ByCodename("cafe"),
    SitemapLocations = new[] { SitemapNodeIdentifier.ByCodename("cafes") }   
};
string itemExternalId = "Ext-Item-456-Brno";

ContentItemModel response = await client.UpsertContentItemByExternalIdAsync(itemExternalId, item);
```

#### Adding a content item 

```csharp
ContentItemCreateModel item = new ContentItemCreateModel() { 
    Name = "Brno",
    Type = ContentTypeIdentifier.ByCodename("cafe"),
    SitemapLocations = new[] { SitemapNodeIdentifier.ByCodename("cafes") }
};

ContentItemModel response = await client.CreateContentItemAsync(item);
```

#### Updating a content item

```csharp
ContentItemIdentifier identifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier identifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));

ContentItemUpdateModel item = new ContentItemUpdateModel() { 
    Name = "New name",
    SitemapLocations = new[] { 
        SitemapNodeIdentifier.ByCodename("cafes"),
        SitemapNodeIdentifier.ByCodename("europe")
    }
};

ContentItemModel reponse = await client.UpdateContentItemAsync(identifier, item);
```

#### Viewing a content item

```csharp
ContentItemIdentifier identifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier identifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");
// ContentItemIdentifier identifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));

ContentItemModel reponse = await client.GetContentItemAsync(identifier);
```

#### Listing content items

```csharp
ListingResponseModel<ContentItemModel> response = await client.ListContentItemsAsync();
while (true)
{
    foreach (var item in response)
    {
        // use your content item
    }
    if (!response.HasNextPage())
    {
        break;
    }
    response = await response.GetNextPage();
}
 ```

#### Deleting a content item

```csharp
ContentItemIdentifier identifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier identifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");
// ContentItemIdentifier identifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));

client.DeleteContentItemAsync(identifier);
```
 
### Language variant methods

#### Upserting a language variant

```csharp
var elements = new {
    street = "Nove Sady 25",
    city = "Brno",
    country = "Czech Republic"
};
ContentItemVariantUpsertModel upsertModel = new ContentItemVariantUpsertModel() { Elements = elements };
ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));
// ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");

LanguageIdentifier languageIdentifier = LanguageIdentifier.ByCodename("en-US");
// LanguageIdentifier languageIdentifier = LanguageIdentifier.ById(Guid.Parse("00000000-0000-0000-0000-000000000000"));

ContentItemVariantIdentifier identifier = new ContentItemVariantIdentifier(itemIdentifier, languageIdentifier);

ContentItemVariantModel response = await client.UpsertContentItemVariantAsync(identifier, upsertModel);
```

#### Viewing a language variant

```csharp
ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));
// ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");

LanguageIdentifier languageIdentifier = LanguageIdentifier.ByCodename("en-US");
// LanguageIdentifier languageIdentifier = LanguageIdentifier.ById(Guid.Parse("00000000-0000-0000-0000-000000000000"));

ContentItemVariantIdentifier identifier = new ContentItemVariantIdentifier(itemIdentifier, languageIdentifier);

ContentItemVariantModel response = await client.GetContentItemVariantAsync(identifier);
```

#### Listing language variants

```csharp
ContentItemIdentifier identifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier identifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79"));
// ContentItemIdentifier identifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");

List<ContentItemVariantModel> response = await client.ListContentItemVariantsAsync(identifier);
```

#### Deleting a language variant

```csharp
ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByCodename("brno");
// ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ById(Guid.Parse("8ceeb2d8-9676-48ae-887d-47ccb0f54a79")));
// ContentItemIdentifier itemIdentifier = ContentItemIdentifier.ByExternalId("Ext-Item-456-Brno");

LanguageIdentifier languageIdentifier = LanguageIdentifier.ByCodename("en-US");
// LanguageIdentifier languageIdentifier = LanguageIdentifier.ById(Guid.Parse("00000000-0000-0000-0000-000000000000"));

await client.DeleteContentItemVariantAsync(identifier);
```


### Asset methods

#### Uploading a file 

Uploads a file to Kentico Cloud. Returns a `fileResult` reference that can later be used to create or update an asset. 

```csharp
MemoryStream stream = new MemoryStream(Encoding.UTF8.GetBytes("Hello world from CM API .NET SDK"));
string fileName = "Hello.txt";
string contentType = "text/plain";

FileReference fileResult = await client.UploadFileAsync(new FileContentSource(stream, fileName, contentType));
```

#### Upserting an asset using external ID

Updates or creates an asset using a `fileResult` reference to a previously uploaded file. You can specify an asset description for each language in your Kentico Cloud project.  

```csharp
AssetDescription assetDescription = new AssetDescription { 
    Description = "Description of the asset in English Language", 
    Language = LanguageIdentifier.ByCodename("en-US")
};
AssetDescription[] descriptions = new [] { assetDescription };

AssetUpsertModel asset = new AssetUpsertModel {
    FileReference = fileResult,
    Descriptions = descriptions;
};
string externalId = "Ext-Asset-123-png";

AssetModel response = await client.UpsertAssetByExternalIdAsync(externalId, asset);
```

#### Uploading an asset from a file system in a single step

Import the asset file and its descriptions using a single method. 

```csharp 
AssetDescription assetDescription = new AssetDescription { 
    Description = "Description of the asset in English Language", 
    Language = LanguageIdentifier.ByCodename("en-US")
};
AssetDescription[] descriptions = new [] { assetDescription };

string filePath = "‪C:\Users\Kentico\Desktop\puppies.png";
string contentType = "image/png";

AssetModel response = await client.CreateAssetAsync(new FileContentSource(filePath, contentType), descriptions);
```

#### Viewing an asset

```csharp
AssetIdentifier identifier = AssetIdentifier.ByExternalId("Ext-Asset-123-png");
// AssetIdentifier identifier = AssetIdentifier.ById(Guid.Parse("fcbb12e6-66a3-4672-85d9-d502d16b8d9c"));

AssetModel response = await client.GetAssetAsync(identifier);
```

#### Listing assets

```csharp
ListingResponseModel<AssetModel> response = await client.ListAssetsAsync();

while (true)
{
    foreach (var asset in response)
    {
        // Use your asset
    }
    if (!response.HasNextPage())
    {
        break;
    }
    response = await response.GetNextPage();
}
```

#### Deleting an asset

```csharp
AssetIdentifier identifier = AssetIdentifier.ByExternalId("Ext-Asset-123-png");
// AssetIdentifier identifier = AssetIdentifier.ById(Guid.Parse("fcbb12e6-66a3-4672-85d9-d502d16b8d9c"));

client.DeleteAssetAsync(identifier);
```

## Further information

For more developer resources, visit the Kentico Cloud Developer Hub at <https://developer.kenticocloud.com>.

### Building the sources

Prerequisites:

**Required:**
[.NET Core SDK](https://www.microsoft.com/net/download/core).

Optional:
* [Visual Studio 2017](https://www.visualstudio.com/vs/) for full experience
* or [Visual Studio Code](https://code.visualstudio.com/)

## Feedback & Contributing

Check out the [contributing](https://github.com/Kentico/content-management-sdk-net/blob/master/CONTRIBUTING.md) page to see the best places to file issues, start discussions, and begin contributing.
