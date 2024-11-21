### Az időszak során a napi rendelések száma / ebből hány visszatérő vásárló által leadott

```csharp
var startDateValue = searchModel.StartDate.HasValue
    ? (DateTime?)_dateTimeHelper.ConvertToUtcTime(searchModel.StartDate.Value, await _dateTimeHelper.GetCurrentTimeZoneAsync())
    : null;
var endDateValue = searchModel.EndDate.HasValue
    ? (DateTime?)_dateTimeHelper.ConvertToUtcTime(searchModel.EndDate.Value, await _dateTimeHelper.GetCurrentTimeZoneAsync()).AddDays(1)
    : null;

// Rendelések lekérdezése a megadott időszakra
var orders = await _orderService.SearchOrdersShoperiaAsync(
    createdFromUtc: startDateValue,
    createdToUtc: endDateValue,
    pageIndex: searchModel.Page - 1,
    pageSize: searchModel.PageSize
);

// Csoportosítás dátum alapján
var groupedOrders = orders
    .GroupBy(o => o.CreatedOnUtc.Date)
    .Select(g => new DailyOrdersReportModel
    {
        Date = g.Key,
        Count = g.Count(),
        ReturningCustomerCount = g.Count(o => _customReportService.IsReturningCustomerAsync(o.CustomerId, o.OrderGuid).Result)
    })
    .ToList();

return groupedOrders;
```

```csharp
public async Task<bool> IsReturningCustomerAsync(int customerId, Guid orderGuid)
{
    // Lekérdezés
    var query = from o in _orderRepository.Table
                where o.CustomerId == customerId && o.OrderGuid != orderGuid
                select o;

    // Van-e volt rendelése, vagy nem
    var hasOtherOrders = await query.AnyAsync();

    return hasOtherOrders;
}
```

### Az időszak során felhasznált kuponok típusa, az egyes típusok totál felhasználási száma és totál kedvezménye

```csharp
// Lekérdezzük a DiscountUsageHistory és Order adatokat, majd csoportosítjuk a DiscountTypeId alapján
var query = from duh in _discountUsageHistoryRepository.Table
            join o in _orderRepository.Table on duh.OrderId equals o.Id
            join d in _discountRepository.Table on duh.DiscountId equals d.Id
            where (!createdFromUtc.HasValue || duh.CreatedOnUtc >= createdFromUtc.Value) &&
                  (!createdToUtc.HasValue || duh.CreatedOnUtc <= createdToUtc.Value)
            group new { duh, o, d } by d.DiscountTypeId into g
            select new DiscountReportModel
            {
                DiscountTypeName = ((DiscountType)g.Key).ToString(),  // A kupon típus azonosítója
                UsageCount = g.Count(),  // Felhasználási számok összege
                TotalDiscountAmount = g.Sum(x => x.o.OrderSubTotalDiscountInclTax) + g.Sum(x => x.d.DiscountAmount) // Kedvezmény összegzése
            };

var result = await query.ToListAsync();

return result;
```

### Regisztrált vásárlók száma: jelenlegi totál és az időszak alatt regisztráltak száma

```csharp
// Az összes regisztrált customer száma
var totalRegisteredCustomers = await _customerRepository.Table
    .CountAsync(c => true);  // Minden regisztrált vásárlót lekérünk

// Az időszakban regisztrált vásárlók száma (CreatedOnUtc alapján)
var registeredCustomersInPeriod = await _customerRepository.Table
    .CountAsync(c =>
                     (!createdFromUtc.HasValue || c.CreatedOnUtc >= createdFromUtc.Value) &&
                     (!createdToUtc.HasValue || c.CreatedOnUtc <= createdToUtc.Value));

// Visszaadunk egy tuple-t, ami tartalmazza a teljes számot és az időszakon belüliek számát
return (totalRegisteredCustomers, registeredCustomersInPeriod);
```

### Shoperia+: jelenlegi totál és az időszak alatt feliratkozó száma

```csharp
// Az összes customer száma, akiknél a Husegprogram mező igaz
var totalShoperiaPlusSubscriptions = await _customerRepository.Table
    .CountAsync(c => c.Husegprogram);

// Az időszakban feliratkozott hűségprogramos vásárlók száma (HusegCsatlakozasDatum alapján)
var shoperiaPlusSubscriptionsInPeriod = await _customerRepository.Table
    .CountAsync(c => c.Husegprogram &&
                     c.HusegCsatlakozasDatum.HasValue &&
                     (!createdFromUtc.HasValue || c.HusegCsatlakozasDatum >= createdFromUtc.Value) &&
                     (!createdToUtc.HasValue || c.HusegCsatlakozasDatum <= createdToUtc.Value));

// Visszaadunk egy tuple-t, ami tartalmazza a teljes számot és az időszakon belüliek számát
return (totalShoperiaPlusSubscriptions, shoperiaPlusSubscriptionsInPeriod);
```

### Időszak alatt át „visszaérkezett”, azaz kiszállított, majd törölt státuszba átkerült csomagok száma.

```csharp
var query = from order in _orderRepository.Table
            join shipment in _shipmentRepository.Table on order.Id equals shipment.OrderId
            where order.OrderStatusId == 50 // Csak a törölt státuszú rendelések
                  && shipment.DeliveryDateUtc != null // Csak a szállítással rendelkező rendelések
                  && (!createdFromUtc.HasValue || shipment.DeliveryDateUtc >= createdFromUtc.Value)
                  && (!createdToUtc.HasValue || shipment.DeliveryDateUtc <= createdToUtc.Value)
            select order.Id; // Csak az érintett rendelés ID-k

// Visszaadjuk az érintett rendelések számát
return await query.Distinct().CountAsync();
```
