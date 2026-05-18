# Seat Classes Architecture

## Overview
Galaxium Travels implements a three-tier seat class system for interplanetary flights:
- **Economy Class** (60% of seats) - Standard travel experience
- **Business Class** (30% of seats) - Premium comfort and service
- **Galaxium Class** (10% of seats) - Luxury space travel experience

## Current Implementation Status

### ✅ Backend Implementation (COMPLETE)

#### Database Schema
**Flight Model** ([`models.py:19-29`](booking_system_backend/models.py:19))
```python
- base_price: Integer (Economy price, 1x multiplier)
- economy_seats_available: Integer (60% of total)
- business_seats_available: Integer (30% of total)
- galaxium_seats_available: Integer (10% of total)
```

**Booking Model** ([`models.py:31-39`](booking_system_backend/models.py:31))
```python
- seat_class: String (economy/business/galaxium)
- price_paid: Integer (Actual price at booking time)
```

#### Pricing Logic
**Service Layer** ([`services/booking.py:8-12`](booking_system_backend/services/booking.py:8))
```python
SEAT_CLASS_MULTIPLIERS = {
    'economy': 1.0,    # Base price
    'business': 2.5,   # 2.5x base price
    'galaxium': 5.0    # 5x base price
}
```

#### API Schemas
**Type Definitions** ([`schemas.py:4-5`](booking_system_backend/schemas.py:4))
```python
SeatClass = Literal['economy', 'business', 'galaxium']
```

**Flight Response** ([`schemas.py:32-45`](booking_system_backend/schemas.py:32))
- Returns seat availability for all three classes
- Includes computed prices for each class

**Booking Request** ([`schemas.py:51-55`](booking_system_backend/schemas.py:51))
- Accepts seat_class parameter (defaults to economy)

### ✅ Frontend Implementation (COMPLETE)

#### Type System
**TypeScript Types** ([`types/index.ts:3`](booking_system_frontend/src/types/index.ts:3))
```typescript
export type SeatClass = 'economy' | 'business' | 'galaxium';
```

#### UI Components

**FlightCard** ([`components/flights/FlightCard.tsx:16-47`](booking_system_frontend/src/components/flights/FlightCard.tsx:16))
- Displays all three seat classes with:
  - Class-specific icons (Plane, Crown, Rocket)
  - Color-coded pricing
  - Seat availability indicators
  - Visual styling per class

**BookingModal** ([`components/bookings/BookingModal.tsx:66-100`](booking_system_frontend/src/components/bookings/BookingModal.tsx:66))
- Three-step booking flow:
  1. Seat class selection with features
  2. Quote review with price breakdown
  3. Hold confirmation with countdown timer
- Class-specific features displayed:
  - Economy: Standard seating, entertainment, snacks
  - Business: Premium seating, priority boarding, gourmet meals, extra legroom
  - Galaxium: Luxury pods, VIP lounge, concierge, zero-G experience

### ✅ Business Logic (COMPLETE)

#### Seat Management
**Booking Service** ([`services/booking.py:15-89`](booking_system_backend/services/booking.py:15))
- Validates seat class on booking
- Checks class-specific availability
- Decrements correct seat counter
- Calculates price based on multiplier

**Cancellation Service** ([`services/booking.py:92-122`](booking_system_backend/services/booking.py:92))
- Restores seat to correct class counter
- Maintains seat class integrity

#### Flight Filtering
**Flight Service** ([`services/flight.py:17-256`](booking_system_backend/services/flight.py:17))
- Supports filtering by seat class availability
- Returns computed prices for all classes
- Handles both main and feature branch filter styles

### ✅ Data Seeding (COMPLETE)

**Seed Script** ([`seed.py:34-73`](booking_system_backend/seed.py:34))
- Creates flights with 60/30/10 distribution
- Ensures minimum 1 seat per class (if total ≥ 3)
- Seeds demo bookings across all classes

## Architecture Patterns

### 1. Dual Protocol Design
- FastAPI REST API for frontend
- MCP protocol for AI agents
- Single server combining both lifespans

### 2. Service Layer Isolation
- Business logic returns Union types (Success | ErrorResponse)
- Never raises exceptions
- Clean separation of concerns

### 3. Independent Seat Tracking
- Each class has its own counter
- No shared pool calculations
- Prevents race conditions

### 4. Price Calculation Strategy
- Base price stored in Flight model
- Multipliers in service layer
- Computed prices in API responses
- Actual price stored in Booking

## Design Decisions

### Why Three Classes?
1. **Market Segmentation**: Appeals to different customer segments
2. **Revenue Optimization**: Premium classes increase average ticket price
3. **Capacity Management**: Balanced distribution (60/30/10)

### Why Independent Counters?
1. **Simplicity**: No complex calculations needed
2. **Performance**: Direct counter decrements
3. **Reliability**: No race conditions between classes
4. **Clarity**: Explicit seat availability per class

### Why Service-Layer Multipliers?
1. **Flexibility**: Easy to adjust pricing strategy
2. **Consistency**: Single source of truth
3. **Testability**: Business logic isolated
4. **Maintainability**: Centralized pricing rules

## Integration Points

### Backend → Frontend
- REST API returns all class prices
- Frontend displays class-specific data
- Booking request includes seat_class

### Frontend → User
- Visual class differentiation (icons, colors)
- Feature comparison
- Real-time availability
- Price transparency

### Database → Application
- SQLite with ephemeral data
- No migrations (create_all pattern)
- Re-seeds on deployment

## Non-Standard Patterns

1. **Name + user_id verification**: Bookings validate both fields (unusual security)
2. **Seat class multipliers in service**: Pricing logic not in models/config
3. **Error detection by field**: Frontend checks `success: false`, not HTTP status

## Testing Strategy

### Backend Tests
- Service layer tests for all classes
- Seat availability validation
- Price calculation verification
- Cancellation restoration

### Frontend Tests
- Component rendering per class
- Booking flow for each class
- Price display accuracy
- Availability indicators

## Future Considerations

### Potential Enhancements
1. **Seat Selection**: Choose specific seats within class
2. **Class Upgrades**: Allow upgrading during booking
3. **Dynamic Pricing**: Adjust multipliers based on demand
4. **Loyalty Tiers**: Additional benefits per class
5. **Seat Maps**: Visual representation of cabin layout
6. **Amenity Details**: Expanded class-specific features
7. **Waitlist**: Queue for sold-out classes

### Scalability Notes
- Current design supports easy addition of new classes
- Multiplier system allows flexible pricing strategies
- Independent counters scale well with high concurrency
- Frontend components are class-agnostic (data-driven)

## Conclusion

The seat class system is **fully implemented and production-ready**. All three classes (Economy, Business, Galaxium) are integrated throughout the entire stack with proper validation, pricing, availability tracking, and user experience.