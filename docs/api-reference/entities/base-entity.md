# BaseTickerEntity

Base class for all ticker entities.

## Type Definition

```csharp
public class BaseTickerEntity
{
    public Guid Id { get; set; }
    public string Function { get; set; }
    public string Description { get; set; }
    public string InitIdentifier { get; internal set; }
    public DateTime CreatedAt { get; internal set; }
    public DateTime UpdatedAt { get; internal set; }
}
```

## Properties

| Property | Type | Access | Description |
|---------|------|--------|-------------|
| `Id` | `Guid` | Read/Write | Unique identifier (auto-generated if `Guid.Empty`) |
| `Function` | `string` | Read/Write | Function name (must match `[TickerFunction]` attribute exactly) |
| `Description` | `string` | Read/Write | Optional human-readable description |
| `InitIdentifier` | `string` | Internal | Internal identifier (set automatically) |
| `CreatedAt` | `DateTime` | Internal | Timestamp when entity was created (UTC) |
| `UpdatedAt` | `DateTime` | Internal | Timestamp when entity was last updated (UTC) |

## Usage

All TickerQ entities inherit from `BaseTickerEntity`:

- `TimeTickerEntity&lt;TTimeTicker&gt;` extends `BaseTickerEntity`
- `CronTickerEntity` extends `BaseTickerEntity`

## See Also

- [TimeTickerEntity](./time-ticker-entity) - Time-based job entity
- [CronTickerEntity](./cron-ticker-entity) - Cron-based job entity

