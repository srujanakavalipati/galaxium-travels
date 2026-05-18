# Traveller Filter Implementation Plan

## Overview
Add a "Traveller" filter to the Flights page that allows users to specify the number of Adults, Children, and Infants needed for their trip. The system will filter flights to show only those with sufficient total seats available across all seat classes.

## Requirements
- Filter flights based on total traveller count (Adults + Children + Infants)
- Display input fields for each traveller category
- Show only flights with enough total seats to accommodate all travellers
- Maintain consistency with existing filter architecture

## Architecture Decisions

### Traveller Categories
- **Adults**: Standard passengers (age 12+)
- **Children**: Young passengers (age 2-11)
- **Infants**: Very young passengers (age 0-1)

**Note**: For this implementation, all categories count equally toward seat requirements (1 seat per person). In a real system, infants might not require separate seats, but for simplicity we'll count all travellers equally.

### Seat Calculation Logic
Total seats needed = `num_adults + num_children + num_infants`

Filter shows flights where:
```
(economy_seats_available + business_seats_available + galaxium_seats_available) >= total_travellers
```

## Implementation Steps

### 1. Backend Changes

#### 1.1 Update [`schemas.py`](booking_system_backend/schemas.py)
Add traveller filter parameters to [`FlightQueryParams`](booking_system_backend/schemas.py:8):

```python
class FlightQueryParams(BaseModel):
    # ... existing fields ...
    
    # Traveller filters
    num_adults: Optional[int] = None
    num_children: Optional[int] = None
    num_infants: Optional[int] = None
```

#### 1.2 Update [`services/flight.py`](booking_system_backend/services/flight.py)
Modify [`list_flights()`](booking_system_backend/services/flight.py:17) function:

**Add parameters** (after line 41):
```python
def list_flights(
    db: Session,
    # ... existing parameters ...
    # Traveller filters
    num_adults: Optional[int] = None,
    num_children: Optional[int] = None,
    num_infants: Optional[int] = None
) -> list[FlightOut] | ErrorResponse:
```

**Add filtering logic** (after line 168, in the min_seats_available section):
```python
# Traveller count filter
if num_adults is not None or num_children is not None or num_infants is not None:
    total_travellers = (num_adults or 0) + (num_children or 0) + (num_infants or 0)
    if total_travellers > 0:
        total_seats = (
            Flight.economy_seats_available +
            Flight.business_seats_available +
            Flight.galaxium_seats_available
        )
        query = query.filter(total_seats >= total_travellers)
```

**Update docstring** to document new parameters:
```python
"""List flights with optional filtering and sorting.
    
    Args:
        # ... existing args ...
        num_adults: Number of adult travellers (age 12+)
        num_children: Number of child travellers (age 2-11)
        num_infants: Number of infant travellers (age 0-1)
    
    Returns:
        List of FlightOut objects with computed prices for all seat classes
    """
```

#### 1.3 Update [`server.py`](booking_system_backend/server.py)
Modify the REST API endpoint to accept new query parameters (around line 120-140):

```python
@app.get("/flights", response_model=list[FlightOut])
def get_flights(
    # ... existing parameters ...
    num_adults: Optional[int] = None,
    num_children: Optional[int] = None,
    num_infants: Optional[int] = None,
    db: Session = Depends(get_db)
):
    """Get all flights with optional filtering and sorting."""
    result = flight.list_flights(
        db,
        # ... existing parameters ...
        num_adults=num_adults,
        num_children=num_children,
        num_infants=num_infants
    )
    # ... rest of function ...
```

### 2. Frontend Changes

#### 2.1 Update [`types/index.ts`](booking_system_frontend/src/types/index.ts)
Add traveller fields to [`FlightFilters`](booking_system_frontend/src/types/index.ts:61) interface:

```typescript
export interface FlightFilters {
  // ... existing fields ...
  
  // Traveller filters
  num_adults?: number;
  num_children?: number;
  num_infants?: number;
}
```

#### 2.2 Update [`services/api.ts`](booking_system_frontend/src/services/api.ts)
Add traveller fields to [`FlightFilters`](booking_system_frontend/src/services/api.ts:45) interface:

```typescript
export interface FlightFilters {
  // ... existing fields ...
  
  // Traveller filters
  num_adults?: number;
  num_children?: number;
  num_infants?: number;
}
```

**Note**: The [`getFlights()`](booking_system_frontend/src/services/api.ts:74) function already handles all filter parameters dynamically, so no changes needed there.

#### 2.3 Update [`components/flights/FlightFilters.tsx`](booking_system_frontend/src/components/flights/FlightFilters.tsx)
Add traveller filter UI section after the "Minimum Seats" filter (after line 219):

```tsx
{/* Traveller Filter */}
<div className="space-y-2">
  <label className="text-sm font-medium text-star-white">Travellers</label>
  <div className="grid grid-cols-3 gap-2">
    <div>
      <input
        type="number"
        min="0"
        placeholder="Adults"
        value={filters.num_adults || ''}
        onChange={(e) =>
          updateFilter('num_adults', e.target.value ? parseInt(e.target.value) : undefined)
        }
        className="w-full px-3 py-2 bg-white/5 border border-white/10 rounded-lg text-star-white text-sm focus:outline-none focus:ring-2 focus:ring-cosmic-purple"
      />
      <span className="text-xs text-star-white/50 mt-1">Adults (12+)</span>
    </div>
    <div>
      <input
        type="number"
        min="0"
        placeholder="Children"
        value={filters.num_children || ''}
        onChange={(e) =>
          updateFilter('num_children', e.target.value ? parseInt(e.target.value) : undefined)
        }
        className="w-full px-3 py-2 bg-white/5 border border-white/10 rounded-lg text-star-white text-sm focus:outline-none focus:ring-2 focus:ring-cosmic-purple"
      />
      <span className="text-xs text-star-white/50 mt-1">Children (2-11)</span>
    </div>
    <div>
      <input
        type="number"
        min="0"
        placeholder="Infants"
        value={filters.num_infants || ''}
        onChange={(e) =>
          updateFilter('num_infants', e.target.value ? parseInt(e.target.value) : undefined)
        }
        className="w-full px-3 py-2 bg-white/5 border border-white/10 rounded-lg text-star-white text-sm focus:outline-none focus:ring-2 focus:ring-cosmic-purple"
      />
      <span className="text-xs text-star-white/50 mt-1">Infants (0-1)</span>
    </div>
  </div>
  {(filters.num_adults || filters.num_children || filters.num_infants) && (
    <p className="text-xs text-star-white/70 mt-1">
      Total: {(filters.num_adults || 0) + (filters.num_children || 0) + (filters.num_infants || 0)} traveller(s)
    </p>
  )}
</div>
```

### 3. Testing Strategy

#### 3.1 Backend Tests
Add test cases to [`tests/test_services.py`](booking_system_backend/tests/test_services.py):

```python
def test_list_flights_with_traveller_filter(db_session):
    """Test filtering flights by traveller count"""
    # Test with 2 adults
    flights = flight.list_flights(db_session, num_adults=2)
    assert all(
        (f.economy_seats_available + f.business_seats_available + f.galaxium_seats_available) >= 2
        for f in flights
    )
    
    # Test with mixed travellers (2 adults + 1 child + 1 infant = 4 total)
    flights = flight.list_flights(db_session, num_adults=2, num_children=1, num_infants=1)
    assert all(
        (f.economy_seats_available + f.business_seats_available + f.galaxium_seats_available) >= 4
        for f in flights
    )
```

#### 3.2 Manual Testing Checklist
- [ ] Open Flights page and expand filters
- [ ] Enter "2" in Adults field - verify flights update
- [ ] Add "1" in Children field - verify flights update with total of 3
- [ ] Add "1" in Infants field - verify flights update with total of 4
- [ ] Verify total traveller count displays correctly
- [ ] Clear filters and verify all flights show again
- [ ] Test with very high numbers (e.g., 100) - should show no results
- [ ] Verify filter appears in active filters section
- [ ] Test removing individual traveller filters via X button

### 4. UI/UX Considerations

#### Visual Design
- Use 3-column grid layout for compact display
- Show age ranges below each input for clarity
- Display total traveller count when any value is entered
- Match existing filter styling (glass-card effect, cosmic theme)

#### User Experience
- Default values are empty (no filtering)
- Minimum value is 0 for all fields
- Only filter when at least one traveller type has a value > 0
- Clear indication of how many total travellers are selected

#### Accessibility
- Proper labels for screen readers
- Number input type for mobile keyboard optimization
- Clear placeholder text
- Visible focus states

## Implementation Order

1. **Backend Schema** - Add parameters to [`FlightQueryParams`](booking_system_backend/schemas.py:8)
2. **Backend Service** - Update [`list_flights()`](booking_system_backend/services/flight.py:17) with filtering logic
3. **Backend API** - Update REST endpoint in [`server.py`](booking_system_backend/server.py)
4. **Frontend Types** - Add fields to both type definitions
5. **Frontend UI** - Add traveller filter component to [`FlightFilters.tsx`](booking_system_frontend/src/components/flights/FlightFilters.tsx)
6. **Testing** - Add backend tests and perform manual testing

## Edge Cases to Handle

1. **Zero travellers**: If all fields are 0 or empty, don't apply filter
2. **Negative numbers**: Input type="number" with min="0" prevents this
3. **Very large numbers**: Backend will naturally filter to empty results
4. **Decimal numbers**: parseInt() will handle conversion
5. **Non-numeric input**: parseInt() returns NaN, which is handled by the undefined check

## Future Enhancements (Out of Scope)

- Different seat requirements for infants (lap infants vs. seat infants)
- Age-based pricing tiers
- Group booking discounts
- Seat assignment preferences
- Family seating arrangements

## Success Criteria

✅ Users can specify number of Adults, Children, and Infants
✅ Flights are filtered to show only those with sufficient total seats
✅ Filter integrates seamlessly with existing filter system
✅ Total traveller count is clearly displayed
✅ Filter can be cleared individually or with "Reset All"
✅ Backend tests pass
✅ Manual testing checklist completed