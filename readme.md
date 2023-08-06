
## Configuration

1 - Create a Kubernetes namespace if it does not exist:
```
kubectl create namespace elastic-stack
```
2 - Update the Helm chart repositories:
```
helm repo add elastic https://helm.elastic.co && helm repo update
```
3 - Install the Elastic Cloud on Kubernetes (ECK) stack and operator:
```
helm upgrade --install elastic-operator elastic/eck-operator -n elastic-stack
helm upgrade --install eck-stack elastic/eck-stack -f https://raw.githubusercontent.com/LucasDiogo96/eck-helm/main/values.yaml -n elastic-stack
```
4 - Add annotations to services for internal load balancing:
```
kubectl annotate service -n elastic-stack elasticsearch-es-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
kubectl annotate service -n elastic-stack eck-stack-eck-kibana-kb-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
```

5 - Authenticate using the default elastic user:

```
kubectl get secret elasticsearch-es-elastic-user -n elk -o go-template='{{.data.elastic | base64decode}}' -n elastic-stack
```

6 - Create an application user to send logs:

- Go to Stack management -> API Keys -> Create API Key
- Provide the name
- View in JSON Format and copy 

Example:
```
- {"id":"8MTyyIkBFBK7tlyQpncP","name":"app-key","api_key":"FilNNaw8TqaYdxXvGu7JFA","encoded":"OE1UeXlJa0JGQks3dGx5UXBuY1A6RmlsTk5hdzhUcWFZZHhYdkd1N0pGQQ=="}
```
6 - Integrate a .NET application:

#### Add packages
```
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Exceptions
dotnet add package Serilog.Sinks.Debug
dotnet add package Serilog.Sinks.Elasticsearch
```

#### Configure application
```cs
using Microsoft.AspNetCore.HttpLogging;
using Serilog;
using Serilog.Exceptions;
using Serilog.Sinks.Elasticsearch;

var builder = WebApplication.CreateBuilder(args);
Serilog.Debugging.SelfLog.Enable(Console.WriteLine);

ConfigureLogging();
builder.Host.UseSerilog();

builder.Services.AddControllers();

builder.Services.AddHttpLogging(logging =>
{
    logging.LoggingFields = HttpLoggingFields.All;
    logging.RequestBodyLogLimit = 4096;
    logging.ResponseBodyLogLimit = 4096;
});

var app = builder.Build();

app.UseHttpLogging();

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();

void ConfigureLogging()
{
    var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
    var configuration = new ConfigurationBuilder()
        .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
        .AddJsonFile(
            $"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json",
            optional: true)
        .Build();

    var teste = Log.Logger = new LoggerConfiguration()
         .Enrich.FromLogContext()
         .Enrich.WithExceptionDetails()
         .WriteTo.Debug()
         .WriteTo.Console()
         .WriteTo.Elasticsearch(ConfigureElasticSink(configuration, environment))
         .Enrich.WithProperty("Environment", environment)
         .CreateLogger();


    ElasticsearchSinkOptions ConfigureElasticSink(IConfigurationRoot configuration, string environment)
    {
        var options = new ElasticsearchSinkOptions(new Uri("http://[ELASTIC-IP]:9200"))
        {
            FailureCallback = e => Console.WriteLine("Unable to submit event " + e.MessageTemplate),
            EmitEventFailure = EmitEventFailureHandling.WriteToSelfLog |
                                       EmitEventFailureHandling.WriteToFailureSink |
                                       EmitEventFailureHandling.RaiseCallback,
            AutoRegisterTemplate = true,
            IndexFormat = $"app-{environment?.ToLower().Replace(".", "-")}-{DateTime.UtcNow:yyyy-MM}",
            
        };

        options.ModifyConnectionSettings = x => x.ApiKeyAuthentication("[ID]", "[APIKEY]");
        return options;
    }

    builder.Services.AddSingleton<Serilog.ILogger>(teste);
}
```

Make sure to replace [ELASTIC-IP] with the actual Elastic Search Load Balancer IP, and [ID] with your API Key ID, and [APIKEY] with the respective API Key.

## Uninstall

To uninstall the Elastic Cloud on Kubernetes (ECK) stack and operator, use the following commands:
```
helm uninstall eck-stack -n elastic-stack
helm uninstall elastic-operator -n elastic-stack
```



