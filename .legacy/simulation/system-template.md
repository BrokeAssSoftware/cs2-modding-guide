# System Template

```csharp
public partial class VnoResidentAISystem : SystemBase
{
    private EntityQuery _citizenQuery;
    private ComponentLookup<VnoAttributes> _attributesLookup;

    protected override void OnCreate()
    {
        RequireForUpdate<VnoAttributes>();
        _citizenQuery = GetEntityQuery(ComponentType.ReadOnly<Citizen>(), ComponentType.ReadWrite<VnoAttributes>());
        _attributesLookup = GetComponentLookup<VnoAttributes>();
    }

    protected override void OnUpdate()
    {
        var deltaTime = Time.DeltaTime;
        _attributesLookup.Update(this);

        Entities
            .WithName("VnoResidentAISystem")
            .WithStoreEntityQueryInField(ref _citizenQuery)
            .ForEach((ref VnoAttributes attributes, in Citizen citizen) =>
            {
                attributes.Heat = math.clamp(attributes.Heat + deltaTime * attributes.GainRate, 0f, 100f);
            })
            .ScheduleParallel();
    }
}
```

**Highlights**
- `RequireForUpdate<T>()` prevents execution when prerequisites are missing.
- Cache `EntityQuery` and `ComponentLookup` instances in `OnCreate` and refresh them inside `OnUpdate`.
- Use `ScheduleParallel()` when there are no structural changes; switch to `Schedule()` or `Run()` only when necessary.
