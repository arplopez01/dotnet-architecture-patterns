# .NET Architecture Patterns

Practical implementations of the patterns I reach for most in enterprise .NET work. Not textbook examples — these are shaped by real production constraints: legacy systems that can't go down, teams that need to understand the code six months later, and the kind of requirements that change three times before you ship.

All examples are .NET 10, C#, MediatR where it fits.

---

## Patterns

- [CQRS with MediatR](#cqrs-with-mediatr)
- [Strangler Fig — incremental extraction](#strangler-fig)
- [Saga — distributed transaction coordination](#saga-pattern)
- [Event Sourcing — the lightweight version](#event-sourcing)

---

## CQRS with MediatR

The core idea is simple: reads and writes are different problems, so stop treating them the same. Commands change state and return minimal data. Queries return data and change nothing.

Where this pays off in practice: once you separate them, you can optimise each side independently. The write side stays clean and domain-focused. The read side can be as denormalised and performance-tuned as you need without polluting your domain model.

```csharp
// Commands — change state, return only what the caller needs to proceed
public record PlaceOrderCommand(Guid CustomerId, List<OrderItem> Items) 
    : IRequest<Guid>;

public sealed class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _repo;
    private readonly IEventBus _bus;

    public PlaceOrderHandler(IOrderRepository repo, IEventBus bus)
    {
        _repo = repo;
        _bus = bus;
    }

    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items);

        _repo.Add(order);
        await _repo.SaveAsync(ct);

        // Publish after save — if the save fails, nothing gets published
        await _bus.PublishAsync(new OrderPlacedEvent(order.Id, order.Total), ct);

        return order.Id;
    }
}
```

```csharp
// Queries — read-optimised, no domain model overhead
public record GetOrderSummaryQuery(Guid OrderId) : IRequest<OrderSummaryDto>;

public sealed class GetOrderSummaryHandler 
    : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto>
{
    private readonly IReadDbContext _db;

    public async Task<OrderSummaryDto> Handle(
        GetOrderSummaryQuery query, CancellationToken ct)
    {
        // Query directly against a read model — no ORM ceremony, no domain objects
        return await _db.OrderSummaries
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderSummaryDto(o.Id, o.Status, o.Total, o.PlacedAt))
            .FirstOrDefaultAsync(ct)
            ?? throw new OrderNotFoundException(query.OrderId);
    }
}
```

The read model (`OrderSummaries`) is a separate table updated by event handlers. It can be structured exactly how the UI needs it, without any compromise to keep the domain model happy.

---

## Strangler Fig

Named after the fig tree that grows around a host tree until it replaces it entirely. The idea is that you never stop the old system to build the new one — you build the new one alongside it, route traffic incrementally, and let the legacy code die of neglect rather than deletion.

This is how I approach most legacy .NET modernisations. The alternative — a full rewrite with a hard cutover — has a poor track record at enterprise scale.

```csharp
// The facade sits in front of both systems.
// New requests go to the new service. Legacy requests fall through.
// Over time, you extend what the new service handles until the fallback is never hit.

public sealed class OrderFacade
{
    private readonly IModernOrderService _modern;
    private readonly ILegacyOrderService _legacy;
    private readonly IFeatureFlags _flags;

    public async Task<OrderResult> PlaceOrderAsync(PlaceOrderRequest request)
    {
        // Feature flag controls the cutover per order type, customer segment, 
        // or whatever granularity makes sense. This lets you roll back instantly
        // if something breaks in production without a redeploy.
        if (_flags.IsEnabled("use-modern-order-service", request.CustomerId))
        {
            return await _modern.PlaceOrderAsync(request);
        }

        // Legacy path — untouched, still running, still trusted
        return await _legacy.PlaceOrderAsync(request);
    }
}
```

```csharp
// The modern service is a clean .NET 10 implementation.
// It doesn't know the legacy system exists.
public sealed class ModernOrderService : IModernOrderService
{
    public async Task<OrderResult> PlaceOrderAsync(PlaceOrderRequest request)
    {
        var cmd = new PlaceOrderCommand(request.CustomerId, request.Items);
        var orderId = await _mediator.Send(cmd);
        return OrderResult.Success(orderId);
    }
}
```

The real work is in the data sync layer. If both systems need to read the same data during the migration period, you need a reliable sync mechanism — CDC (Change Data Capture) works well here because it captures every change at the database level without touching application code.

---

## Saga Pattern

When a business process spans multiple services, you can't use a database transaction. A saga is a sequence of local transactions, each publishing an event that triggers the next step. If any step fails, compensating transactions undo the previous steps.

```csharp
// Each step in the saga is a local transaction with a corresponding compensation
public sealed class OrderFulfillmentSaga :
    IEventHandler<OrderPlacedEvent>,
    IEventHandler<PaymentProcessedEvent>,
    IEventHandler<PaymentFailedEvent>
{
    public async Task Handle(OrderPlacedEvent evt, CancellationToken ct)
    {
        // Step 1 complete. Trigger step 2.
        await _bus.PublishAsync(new ProcessPaymentCommand(evt.OrderId, evt.Total), ct);
    }

    public async Task Handle(PaymentProcessedEvent evt, CancellationToken ct)
    {
        // Step 2 complete. Trigger step 3.
        await _bus.PublishAsync(new ReserveInventoryCommand(evt.OrderId), ct);
    }

    public async Task Handle(PaymentFailedEvent evt, CancellationToken ct)
    {
        // Compensate — release anything already reserved
        await _bus.PublishAsync(new CancelOrderCommand(evt.OrderId, evt.Reason), ct);
    }
}
```

The key thing sagas get right that two-phase commit gets wrong: each service only needs to know about its own state. The saga coordinator handles the sequence. Services stay decoupled.

---

## Event Sourcing

Instead of storing current state, you store the sequence of events that produced that state. Current state is derived by replaying the events.

This is genuinely useful when audit history matters (financial systems, compliance) or when you need to reconstruct state at a point in time. It's overkill for most CRUD-heavy domains.

```csharp
public abstract class AggregateRoot
{
    private readonly List<DomainEvent> _uncommittedEvents = new();

    public IReadOnlyList<DomainEvent> UncommittedEvents => _uncommittedEvents;

    protected void RaiseEvent(DomainEvent evt)
    {
        Apply(evt);                    // mutate state
        _uncommittedEvents.Add(evt);   // record for persistence
    }

    protected abstract void Apply(DomainEvent evt);
}

public sealed class Order : AggregateRoot
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; private set; }

    // Reconstitute from event history (used by the repository on load)
    public static Order Rehydrate(IEnumerable<DomainEvent> history)
    {
        var order = new Order();
        foreach (var evt in history) order.Apply(evt);
        return order;
    }

    public static Order Create(Guid customerId, List<OrderItem> items)
    {
        var order = new Order();
        order.RaiseEvent(new OrderCreatedEvent(
            Guid.NewGuid(), customerId, items, DateTime.UtcNow));
        return order;
    }

    protected override void Apply(DomainEvent evt)
    {
        switch (evt)
        {
            case OrderCreatedEvent e:
                Id = e.OrderId;
                Total = e.Items.Sum(i => i.Price * i.Quantity);
                Status = OrderStatus.Pending;
                break;

            case OrderConfirmedEvent:
                Status = OrderStatus.Confirmed;
                break;

            case OrderCancelledEvent:
                Status = OrderStatus.Cancelled;
                break;
        }
    }
}
```

One practical note: event sourcing adds real complexity. The query side becomes a separate concern entirely (you need projections). Don't reach for it unless the audit trail or temporal query requirements genuinely justify it.

---

## Notes

These aren't runnable projects — they're reference implementations showing the structure and intent of each pattern. Production versions have more error handling, logging, and domain complexity, but the bones are the same.

Questions or want to discuss a specific implementation detail: inquire@ryanlopez.dev
