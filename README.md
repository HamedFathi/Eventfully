# Eventfully .NET
Lightweight Reliable Messaging Framework using Outbox Pattern / EFCore / AzureServiceBus 
>Eventfully provides a gentle way for enterprises to begin incorporating asynchronous messaging patterns, long running workflows, and eventual consistency into their projects using familiar transactional patterns.

[![NuGet version (Eventfully.Core)](https://img.shields.io/nuget/v/Eventfully.Core.svg?style=flat-square)](https://www.nuget.org/packages/Eventfully.Core/)
[![Build status](https://ci.appveyor.com/api/projects/status/38p46q88w79akoe7?svg=true)](https://ci.appveyor.com/project/cfrenzel/eventfully)



**Why**
- Dispatch Messages within a Transaction/UnitOfWork using an Outbox within your database
- Simple Configuration
- Advanced Retry Logic for all your eventual consistency needs
- No requirement for shared message classes between apps/services
- EFCore SqlServer support
- Azure Service Bus support
- Dependency Injection support 
- Delayed Dispatch and Timeouts
- Easy to customize message deserialization
- Encryption support (AES)
  - Azure Key Vault support out of the box
- Simple Sagas
- Configurable Message Processing Pipeline
- Pluggable Transports, Outboxes, MessageHandling, Encryption, Dependency Injection
- Supports Events, Command/Reply


**Events**

Events implement <code>IIntegrationEvent</code>.  A base class <code>IntegrationEvent</code> is provided.  Overriding MessageType provides a unique identifier for our new Event type.   
```csharp
 public class OrderCreated : IntegrationEvent
 {
     public override string MessageType => "Sales.OrderCreated";
     public Guid OrderId { get; private set; }
     public Decimal TotalDue { get; private set; }
     public string CurrencyCode { get; private set; }
 }
```
**Event Handlers**

Event handlers implement <code>IMessageHandler&lt;Event&gt;</code>.
  
  ```csharp
  public class OrderCreatedHandler : IMessageHandler<OrderCreated>
  {
        public Task Handle(OrderCreated ev, MessageContext context)
        {
            Console.WriteLine($"Received OrderCreated Event");
            Console.WriteLine($"\tOrderId: {ev.OrderId}");
            Console.WriteLine($"\tTotal Due: {ev.TotalDue} {ev.CurrencyCode}");
            return Task.CompletedTask;
        }
  }
  ```

**Publishing Events**

To Publish an <code>OrderCreated</code> event only if saving the <code>Order</code> Entity succeeds - inject an <code>IMessagingClient</code> into your constructor.  Use the <code>IMessagingClient</code> to publish the event before calling <code>DbContext.SaveChanges</code>.  This will save the event to the <code>Outbox</code> within the same transaction as the <code>Order</code>.  The framework will try (and retry) to publish the event to the configured Transport in the background.

```csharp
  public class OrderCreator
  {
            private readonly ApplicationDbContext _db;
            private readonly ILogger<Handler> _log;
            private readonly IMessagingClient _messagingClient;

            public OrderCreator(ApplicationDbContext db, ILogger<Handler> log, IMessagingClient messagingClient)
            {
                _db = db;
                _log = log;
                _messagingClient = messagingClient;
            }

            public async Task<Guid?> CreateOrder(CreateOrderCommand command, CancellationToken cancellationToken)
            {
                try
                {
                    Order newOrder = new Order(command.Amount, command.OrderedAtUtc);
                    _db.Add(newOrder);
                    _messagingClient.Publish(
                        new OrderCreated.Event(newOrder.Id, newOrder.Amount, "USD", null)
                     );
                     await _db.SaveChangesAsync();
                     return r.Id;
                }
                catch (Exception exc)
                {
                    _log.LogError(exc, "Error creating rate", null);
                    return null;
                }
            }
```

**Configure Messaging**

The simplest way to configure Transports (think AzureServiceBus) and Endpoints (think a specific queue or topic) is to implement a <code>Profile</code>.
  - Configure an Endpoint by providing a name: "Events" 
   
  - Specify whether the endpoint is 
    - <code>Inbound</code> - we want to receive and handle messages from it
    - <code>Outbound</code> - we want to write messages to it
    - <code>InboundOutbound</code> - we want to do both.  Useful for apps that asynchronously talk to themselves
  - For <code>Outbound</code> endpoints we can bind specific Message Types to the endpoint.  This allows us to publish Events without specifying an Endpoint
    - <code>.BindEvent<OrderCreated>()</code>
    - <code>.AsEventDefault()</code> to make the endpoint the default for all Events
    - > <strong>Convention:</strong> use Endpoint names: "Events", "Commands", "Replies" to automatically make the endpoint a default
  - Specify a Transport
    - <code>.UseAzureServiceBusTransport()</code>
    - <code>.UseLocalTransport()</code> only uses the <code>Outbox</code> and dispatches messages locally without a servicebus

```csharp
public class MessagingProfile : Profile
    {
        public MessagingProfile(Microsoft.Extensions.Configuration.IConfiguration config)
        {
            ConfigureEndpoint("Events")
                .AsInboundOutbound() //for our example will be reading and writing to this endpoint
                .BindEvent<PaymentMethodCreated>()
                .UseLocalTransport()
                ;
        }
    }
```

**Configuring Transient Dispatch for EFCore and SqlServer**

Often when you're sending a small number of messages within a Transaction, it makes sense for the messages to dispatch immediately after committing to the outbox.  This avoids any delays normally incurred by polling the outbox.  This feature is enabled by default and is referred to <code>TransientDispatch</code>.  Until we figure out a better way to detect a successful save in EFCore you'll need to help out by implementing `ISupportTransientDispatch` in your DbContext.  Simply publish a C# event when the changes ar saved.

```csharp
public class ApplicationDbContext : DbContext, ISupportTransientDispatch
{     
    public event EventHandler ChangesPersisted;
```

```csharp
 protected override void OnModelCreating(ModelBuilder builder)
 {
     base.OnModelCreating(builder);

     /*** Add Outbox Entities ***/
     builder.AddEFCoreOutbox();
 }
```

```csharp
 public override int SaveChanges()
 {
     var res = base.SaveChanges();
      _postSaveChanges();
     return res;
 }
      
 public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default(CancellationToken))
 {
      var res = await base.SaveChangesAsync(cancellationToken);
      _postSaveChanges();
      return res;
 }

 private void _postSaveChanges()
 {
      this.ChangesPersisted?.Invoke(this, null);
 }
```

**Registering with Container at Startup**

Eventfully plugs into your DI framework.  For Microsoft.DependencyInjection 

- `services.AddMessaging`
- `.WithEFCoreOutbox<ApplicationDbContext>` - configure an outbox
- `_serviceProvider.UseMessagingHost()` - to enable processing of `Inbound` endpoints and `Outbox`

```csharp
     _services.AddMessaging(
         new MessagingProfile(_config),
         typeof(Program).GetTypeInfo().Assembly
      )
      .WithEFCoreOutbox<ApplicationDbContext>(settings =>
      {
        settings.DisableTransientDispatch = false;
        settings.MaxConcurrency = 1;
        settings.SqlConnectionString = _config.GetConnectionString("ApplicationConnection");
      });
    
    //start messaging processing from outbox and inbound endpoints
    _serviceProvider = services.BuildServiceProvider();
    _serviceProvider.UseMessagingHost();
```


**Encryption for an Event Type**

```csharp
    ConfigureEndpoint("Events")
    .AsInboundOutbound()
    .BindEvent<PaymentMethodCreated>()
        .UseAesEncryption(config.GetSection("SampleAESKey").Value)
    .UseLocalTransport()
    ;            
```

**Encryption with AzureKeyVault KeyProvider**

```csharp
    .UseAesEncryption("keyName", new AzureKeyVaultKeyProvider(config.GetSection("KeyVaultUrl").Value))
```

**Configuring Messages with MessageMetaData**

- Add arbitrary data to a message
```csharp
   var meta = new MessageMetaData();
   meta.Add("CustomProp", "CustomValue");

```
- Predefined Meta Data
  - DispatchDelay
  - CorrelationId 
  - SessionId
  - MessageId
  - SkipTransient

**Delayed Dispatch**

```csharp
   await client.Publish(new OrderCreated.Event(Guid.NewGuid(), 722.99M, "USD", null),
        new MessageMetaData(delay: TimeSpan.FromSeconds(30))
   );
```

**Publishing Raw Message Data**

```csharp
    dynamic json = new ExpandoObject();
    json.OrderId = Guid.NewGuid();
    json.TotalDue = 622.99M;
    json.CurrencyCode = "USD";
    json.ShippingAddress = new
    {
        Line1 = "456 Peachtree St",
        Line2 = "Suite A",
        City = "Atlanta",
        StateCode = "GA",
        Zip = "30319"
    };

    await client.Publish(
        "Sales.OrderCreated",
        Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(json))
    ); 
```

**Custom Message Extractor**

Within your event class implement `IMessageExtractor`
```csharp
      public class Event : IntegrationEvent, IMessageExtractor
        {
            public override string MessageType => "Organization.Reputation";

            public Guid OrganizatoinId { get; set; }
            public string EventType { get; set; }
            public string Details { get; set; }

            private Event() { }
            public Event(Guid id, string type, string details)
            {
                this.OrganizatoinId = id;
                this.EventType = type;
                this.Details = details;
            }

            public IIntegrationMessage Extract(byte[] data)
            {
                var textData = Encoding.UTF8.GetString(data);
                dynamic json = JValue.Parse(textData);
                Guid id = json.OrganizationId;
                string type = json.EventType;
                string details = json.EventDetails;
                var @event = new OrganizationReputation.Event(id, type, details);
                return @event;
            }
        }
```

**Bypassing the Outbox - Non Transactional**

```csharp
 await client.PublishSynchronously(new OrderCreated.Event(Guid.NewGuid(), 722.99M, "USD", null));
```
