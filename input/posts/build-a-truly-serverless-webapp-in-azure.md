Title: Build a truly serverless web app in Azure
Lead: (And the step where we show Cosmos DB the door...)
Image: ../images/build-a-truly-serverless-webapp-in-azure.png
Published: 7/27/2018
Tags:

- web
- Azure
- Azure Storage
- Azure Table Storage
- Azure Functions
- Cosmos DB
- cloud
- serverless

---

We review serverless computing and then make a Microsoft tutorial even better.

Topics:

- [What is serverless computing?](#what-is-serverless-computing)
- [Serverless?](#committing-is-sharing)
- [Replacing Cosmos DB with Azure Table Storage](#replacing-cosmos-db-with-azure-table-storage)

## What is serverless computing?

Serverless computing is the concept of storing and processing data without the _consideration_ of a server. One of the great appeals of the cloud is that we pay for what you use. If we need another server, we can spin up another server. If we want to decrease resources, we can change to a smaller server. Serverless takes these things to the extreme.

Instead of having to figure out what size or kind of server we (might) need and how many servers we (might) need, we can just use serverless computing to just give us whatever resources we need when we need them. When we don't need them, we don't use them.

So, in my opinion, a true serverless app should cost us absolutely nothing to run if nothing is actually running. We _might_ pay for storage, since we are always storing our code, but smaller apps can often get away without paying anything or paying a few cents per month. If someone makes a single request to our app, the app immediately spins up, executes the request and charges us for whatever happened in that one request.

## Serverless?

So now that we've gotten that out of the way, I would like to complain about a Microsoft tutorial  titled [Build a serverless web app in Azure](https://docs.microsoft.com/en-us/azure/functions/tutorial-static-website-serverless-api-with-database). It's actually a great tutorial and totally worth going through, but the one thing that bugs me is that the tutorial makes us use Cosmos DB. Why does this bother me? Because I don't consider Cosmos DB to be serverless.

Sure, we don't have to manage or spin up a web server or a database server, but we do have to spin up "Reserved Request Units" (4 at a minimum). This violates the concept of only paying for what we need and what we use. This could potentially be excused if the cost was negligible, but it's not. The minimum required units leads to a required minimum cost of [$23.32](https://azure.microsoft.com/en-us/pricing/calculator/#cosmos-dbe511f151-eda8-4041-96c9-8f3d2d9c6940)! No thank you.

## Replacing Cosmos DB with Azure Table Storage

The best serverless alternative is Azure Table Storage. So let's see what it would take to fix Microsoft's [serverless tutorial](https://docs.microsoft.com/en-us/azure/functions/tutorial-static-website-serverless-api-with-database). Step 1: Do the tutorial. Step 2: Come back here. Step 3: Follow the rest of this post.

### Create a table in Azure Storage

1. In the portal, go to the Azure Storage account you previously created
2. Create a table named "images".

### Update ResizeImage

1. In the portal, go to the serverless function you previously created.
2. Under "ResizeImage", click "Integrate", then select "Azure Cosmos DB ($return)" under "Outputs"
3. Click "delete" to make room for our new output.
4. Click on "New Output", "Azure Table Storage", "Select"
5. Populate the following fields:
   - Use function return value: `checked`
   - Table parameter name: `$return`
   - Table name: `images`
   - Storage account connection: `AZURE_STORAGE_CONNECTION_STRING`
6. Save.
7. Click on "ResizeImage" and wait for the code to load.
8. Add the following code:

```csharp
#r "Microsoft.WindowsAzure.Storage"
using Microsoft.WindowsAzure.Storage.Table;
```

9. Replace the `return` statement with:

```csharp
var content = new {
    id = name,
    imgPath = "/images/" + name,
    thumbnailPath = "/thumbnails/" + name,
    description = result.description
};

return new Document
{
    PartitionKey = "images",
    RowKey = name,
    Content = JsonConvert.SerializeObject(content)
};
```

10. Add this class to the bottom:

```csharp
public class Document : TableEntity
{
    public string Content { get; set; }
}
```

11. Go to your web page and upload a file.
12. Verify that the file made it to the `images` and `thumbnails` containers.

### Update GetImages

1. In the portal, go to the serverless function you previously created.
2. Under "GetImages", click "Integrate", then select "Azure Cosmos DB (documents)" under "Inputs"
3. Click "delete" to make room for our new input.
4. Click on "New Input", "Azure Table Storage", "Select"
5. Populate the following fields:
   - Table parameter name: `docs`
   - Table name: `images`
   - Storage account connection: `AZURE_STORAGE_CONNECTION_STRING`
6. Save.
7. Click on "GetImages" again and wait for the code to load.
8. Replace all the code with the following:

```csharp
#r "Microsoft.WindowsAzure.Storage"
#r "Newtonsoft.Json"

using Microsoft.WindowsAzure.Storage.Table;
using Newtonsoft.Json;
using System.Net;

public static IEnumerable<object> Run(HttpRequestMessage req, CloudTable documents)
{
    var query = new TableQuery<Document>()
        .Where(TableQuery.GenerateFilterCondition("PartitionKey", QueryComparisons.Equal, "images"));
    return documents
        .ExecuteQuery(query)
        .Select(d => JsonConvert.DeserializeObject<dynamic>(d.Content));
}

public class Document : TableEntity
{
    public string Content { get; set; }
}
```

10. Go to your web page and refresh the page. You should now see the new image. (Images uploaded to Cosmos DB won't show up here.)

### Delete Cosmos DB

1. In the portal, go to the Cosmos DB account you previously created.
2. Click "Delete Account"

Now we have a true serverless app!