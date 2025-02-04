# Gerenciador de Catálogos da Netflix com Azure Functions e Banco de Dados

Neste tutorial, vamos criar um gerenciador de catálogos da Netflix utilizando **Azure Functions** e **CosmosDB**. Este sistema será capaz de armazenar, filtrar e listar registros de catálogos de filmes, utilizando a infraestrutura de nuvem do **Azure**.

## Requisitos

- **Conta no Azure** (Crie sua conta em [https://azure.microsoft.com](https://azure.microsoft.com))
- **Azure CLI** (Instale o Azure CLI em [https://learn.microsoft.com/en-us/cli/azure/install-azure-cli](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli))
- **.NET 8** (Instale o SDK do .NET 8 em [https://dotnet.microsoft.com/download/dotnet](https://dotnet.microsoft.com/download/dotnet))
- **IDE recomendada**: Visual Studio ou VS Code com as extensões para Azure Functions

## Arquitetura

Usaremos a arquitetura **serverless** do **Azure Functions** e o **Azure CosmosDB** como banco de dados NoSQL para armazenar e consultar os registros dos catálogos. A solução terá as seguintes funcionalidades:

1. **Salvar arquivos no Azure Storage Account**.
2. **Salvar registros no Azure CosmosDB**.
3. **Filtrar registros no CosmosDB**.
4. **Listar registros do CosmosDB**.

## Passo 1: Criar a Infraestrutura no Azure

1. **Criar uma conta no Azure** e fazer login com o comando:

    ```bash
    az login
    ```

2. **Criar um grupo de recursos** no Azure (substitua `<nome-do-grupo-de-recursos>` por um nome único):

    ```bash
    az group create --name <nome-do-grupo-de-recursos> --location eastus
    ```

3. **Criar uma conta do CosmosDB** para armazenar os dados (substitua `<nome-da-conta-cosmosdb>`):

    ```bash
    az cosmosdb create --name <nome-da-conta-cosmosdb> --resource-group <nome-do-grupo-de-recursos> --kind MongoDB
    ```

4. **Criar um Storage Account** para armazenar os arquivos:

    ```bash
    az storage account create --name <nome-da-conta-storage> --resource-group <nome-do-grupo-de-recursos> --location eastus --sku Standard_LRS
    ```

## Passo 2: Criar a Azure Function para Salvar Arquivos no Storage Account

1. Crie um novo projeto de **Azure Function** no seu terminal:

    ```bash
    dotnet new func -n CatalogManager --template "Function" --runtime dotnet
    ```

2. Navegue até o diretório do projeto:

    ```bash
    cd CatalogManager
    ```

3. Abra o arquivo `Function1.cs` e crie a função para salvar arquivos no **Storage Account**. Aqui está um exemplo básico:

    ```csharp
    using Microsoft.Azure.WebJobs;
    using Microsoft.Extensions.Logging;
    using Microsoft.WindowsAzure.Storage.Blob;
    using System.IO;
    using System.Threading.Tasks;

    public static class UploadFileFunction
    {
        [FunctionName("UploadFile")]
        public static async Task Run(
            [BlobTrigger("uploads/{name}", Connection = "AzureWebJobsStorage")] Stream myBlob,
            string name,
            [Blob("catalogs/{name}", FileAccess.Write, Connection = "AzureWebJobsStorage")] CloudBlockBlob outputBlob,
            ILogger log)
        {
            log.LogInformation($"Iniciando o upload do arquivo {name}");
            await outputBlob.UploadFromStreamAsync(myBlob);
            log.LogInformation($"Arquivo {name} carregado com sucesso para o Storage");
        }
    }
    ```

4. Esta função será acionada quando um arquivo for carregado na pasta `uploads`. O arquivo será movido para a pasta `catalogs`.

## Passo 3: Criar a Azure Function para Salvar Registros no CosmosDB

1. Adicione o **NuGet Package** para integração com o CosmosDB:

    ```bash
    dotnet add package Microsoft.Azure.Cosmos
    ```

2. Crie uma nova função para salvar registros no CosmosDB. Aqui está um exemplo básico de uma função para adicionar um filme ao catálogo:

    ```csharp
    using Microsoft.Azure.WebJobs;
    using Microsoft.Extensions.Logging;
    using Microsoft.Azure.Cosmos;
    using System.Threading.Tasks;

    public static class SaveCatalogFunction
    {
        private static CosmosClient cosmosClient = new CosmosClient("<your-cosmosdb-endpoint>", "<your-cosmosdb-key>");
        private static Database database = cosmosClient.GetDatabase("<your-database-name>");
        private static Container container = database.GetContainer("<your-container-name>");

        [FunctionName("SaveCatalogItem")]
        public static async Task Run(
            [HttpTrigger(AuthorizationLevel.Function, "post")] Movie movie,
            ILogger log)
        {
            log.LogInformation($"Adicionando filme ao catálogo: {movie.Title}");
            await container.CreateItemAsync(movie);
            log.LogInformation($"Filme {movie.Title} adicionado com sucesso ao CosmosDB");
        }
    }

    public class Movie
    {
        public string Id { get; set; }
        public string Title { get; set; }
        public string Genre { get; set; }
        public int Year { get; set; }
    }
    ```

3. A função `SaveCatalogItem` será chamada via um **POST request** e armazenará um novo filme no banco de dados CosmosDB.

## Passo 4: Criar a Azure Function para Filtrar Registros no CosmosDB

1. Crie uma nova função para filtrar filmes no banco de dados CosmosDB com base no gênero ou outro critério:

    ```csharp
    [FunctionName("FilterCatalogItems")]
    public static async Task<IActionResult> FilterCatalog(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "filter/{genre}")] string genre,
        ILogger log)
    {
        var sqlQueryText = $"SELECT * FROM c WHERE c.Genre = '{genre}'";
        var queryDefinition = new QueryDefinition(sqlQueryText);
        var resultSetIterator = container.GetItemQueryIterator<Movie>(queryDefinition);

        List<Movie> movies = new List<Movie>();
        while (resultSetIterator.HasMoreResults)
        {
            var response = await resultSetIterator.ReadNextAsync();
            movies.AddRange(response);
        }

        return new OkObjectResult(movies);
    }
    ```

2. A função `FilterCatalogItems` será chamada via um **GET request** e retornará os filmes filtrados por gênero.

## Passo 5: Criar a Azure Function para Listar Registros no CosmosDB

1. Crie uma função para listar todos os filmes armazenados no banco de dados CosmosDB:

    ```csharp
    [FunctionName("ListCatalogItems")]
    public static async Task<IActionResult> ListCatalog(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "list")] HttpRequestMessage req,
        ILogger log)
    {
        var sqlQueryText = "SELECT * FROM c";
        var queryDefinition = new QueryDefinition(sqlQueryText);
        var resultSetIterator = container.GetItemQueryIterator<Movie>(queryDefinition);

        List<Movie> movies = new List<Movie>();
        while (resultSetIterator.HasMoreResults)
        {
            var response = await resultSetIterator.ReadNextAsync();
            movies.AddRange(response);
        }

        return new OkObjectResult(movies);
    }
    ```

2. A função `ListCatalogItems` será chamada via um **GET request** e retornará todos os filmes armazenados no catálogo.

## Passo 6: Testar e Publicar

1. **Testar localmente**:

    Execute o seguinte comando para testar suas funções localmente:

    ```bash
    func start
    ```

2. **Publicar no Azure**:

    Para publicar sua função no Azure, execute:

    ```bash
    func azure functionapp publish <nome-da-sua-function-app>
    ```

    A URL pública para suas funções estará disponível no console de saída.

## Conclusão

Agora você tem um gerenciador de catálogos da Netflix simples usando **Azure Functions** e **CosmosDB**. Você pode adicionar novos filmes ao catálogo, filtrar filmes por gênero e listar todos os filmes de maneira eficiente e escalável. As funções são completamente serverless e podem escalar conforme a demanda.

Para mais informações sobre Azure Functions e CosmosDB, consulte a [documentação oficial](https://learn.microsoft.com/en-us/azure/azure-functions/).
