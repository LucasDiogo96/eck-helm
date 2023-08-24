# Introduction
This document provides instructions on how to configure and integrate the Elastic Cloud on Kubernetes (ECK) stack and operator with a .NET application to send logs. The Elastic Cloud on Kubernetes allows you to run Elasticsearch, Kibana, and other Elastic Stack components on Kubernetes clusters easily.

## 📦 Repository

https://dev.azure.com/neogrid/Arquitetura/_git/ECK


## Requirements
Before proceeding with the installation and integration, ensure you have the following prerequisites:

- ✔️ A running Kubernetes cluster.
- ✔️ kubectl CLI installed and configured to connect to your Kubernetes cluster.
- ✔️ Helm v3 or later installed.

## ⚙️Configuration

### 1 – Install CSI Driver
```
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver --namespace kube-system --version v1.28.2
```

### 2 – Apply manifest

```
kubectl apply -f https://raw.githubusercontent.com/LucasDiogo96/eck-helm/main/artifacts/template.yaml
```
or 
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: elastic-stack
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elastic-storage
provisioner: disk.csi.azure.com
parameters:
  skuName: StandardSSD_LRS  # alias: storageaccounttype, available values: Standard_LRS, Premium_LRS, StandardSSD_LRS, UltraSSD_LRS
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 3 – Update the Helm chart repositories:
```
helm repo add elastic https://helm.elastic.co && helm repo update
```


### 4 – Install the Elastic Cloud on Kubernetes (ECK) operator:
```
helm upgrade --install elastic-operator elastic/eck-operator -n elastic-stack
```

### 5 – Install the Elastic Cloud on Kubernetes (ECK) stack :
```
helm upgrade --install elastic-stack elastic/eck-stack -f https://raw.githubusercontent.com/LucasDiogo96/eck-helm/main/artifacts/multi-node.values.yaml -n elastic-stack
```

### 6 – Add annotations to services for internal load balancing:
```
kubectl annotate service -n elastic-stack elasticsearch-es-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
kubectl annotate service -n elastic-stack elastic-stack-eck-kibana-kb-http service.beta.kubernetes.io/azure-load-balancer-internal="true"
```

### 7 – Authenticate using the default elastic user:

```
kubectl get secret elasticsearch-es-elastic-user -n elk -o go-template='{{.data.elastic | base64decode}}' -n elastic-stack
```

## 🔌 Integration

### 1 - Create an application api key to send logs:

- Go to Stack management -> API Keys -> Create API Key
- Provide the name
- View in JSON Format and copy 

Example:
```
- {"id":"8MTyyIkBFBK7tlyQpncP","name":"app-key","api_key":"FilNNaw8TqaYdxXvGu7JFA","encoded":"OE1UeXlJa0JGQks3dGx5UXBuY1A6RmlsTk5hdzhUcWFZZHhYdkd1N0pGQQ=="}
```
###2 - Integrate a .NET application:

#### Add packages
```
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Exceptions
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

```
helm uninstall eck-stack -n elastic-stack
helm uninstall elastic-operator -n elastic-stack
helm uninstall azuredisk-csi-driver -n  kube-system
curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/v1.28.2/deploy/uninstall-driver.sh | bash -s v1.28.2 --
```


## 📚 Reference

Install CSI driver on a Kubernetes cluster:

https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/docs/install-csi-driver-v1.28.2.md

ECK documentation:

- Github:
https://github.com/elastic/cloud-on-k8s/blob/main/README.md

- Elastic CO:
https://www.elastic.co/guide/en/cloud-on-k8s/2.8/index.html
