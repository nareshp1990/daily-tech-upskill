# Day 32 — System Design Case Study: Ride-Sharing Platform
**Date:** 2026-04-08
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**System Design Case Study: Ride-Sharing Platform — Real-Time Location & Matching**

Uber/Lyft/Grab are among the most technically complex consumer apps. The core problems: tracking millions of moving drivers in real-time, matching riders to nearby drivers in milliseconds, and coordinating the entire trip lifecycle. This design introduces geospatial indexing, WebSockets for real-time communication, and the state machine at the heart of a trip.

---

## 2. Requirements

**Functional:**
- Rider requests a ride (origin + destination)
- System finds nearby available drivers
- Match rider to best driver
- Real-time location tracking (driver moves on rider's map)
- Fare estimation and trip completion payment
- Driver app: accept/decline rides, navigate, complete trips

**Non-Functional:**
- 10M trips/day in one city (~100 concurrent trips at 2am, 50,000 at rush hour)
- Driver location updates: every 5 seconds
- Matching latency: < 2 seconds (rider should see driver in < 2s)
- Location accuracy: ~10 meters
- 99.99% availability for matching (lost ride = revenue loss)

---

## 3. Core Challenge: Geospatial Indexing

Finding "drivers within 5km of this point" is not a standard database query. You need a spatial index.

### 3.1 Geohash

```
Geohash divides the world into a grid of cells.
Each cell is identified by a string: longer = smaller area.

"9q5cs" = San Francisco area (~5km × 5km)
"9q5csg" = 1 specific block (~300m × 300m)
"9q5csgg" = one building (~40m × 40m)

Property: nearby cells share a common prefix.
  "9q5csg" and "9q5csh" are adjacent.
  To find nearby drivers, search 9 geohash cells (current + 8 neighbors).
```

```java
// Geohash library: com.github.davidmoten:geo
public class DriverLocationService {

    private final RedisTemplate<String, String> redis;

    // Driver reports location
    public void updateDriverLocation(Long driverId, double lat, double lng) {
        String geohash = GeoHash.encodeHash(lat, lng, 7); // 7-char ≈ 150m precision
        String key = "drivers:geohash:" + geohash;
        redis.opsForSet().add(key, driverId.toString());
        redis.expire(key, Duration.ofSeconds(30));   // Auto-expire if driver goes offline
    }

    // Find nearby drivers
    public List<Long> findNearbyDrivers(double lat, double lng, int radiusKm) {
        String centerHash = GeoHash.encodeHash(lat, lng, 7);
        List<String> neighborHashes = GeoHash.neighbours(centerHash);
        neighborHashes.add(centerHash);

        Set<Long> driverIds = new HashSet<>();
        for (String hash : neighborHashes) {
            Set<String> ids = redis.opsForSet().members("drivers:geohash:" + hash);
            ids.forEach(id -> driverIds.add(Long.parseLong(id)));
        }
        return new ArrayList<>(driverIds);
    }
}
```

### 3.2 Redis GEO Commands (Production Alternative)

```java
// Simpler: Redis native GEO data structure
public class DriverLocationServiceRedisGeo {

    private final RedisTemplate<String, String> redis;
    private static final String KEY = "active_drivers";

    public void updateLocation(Long driverId, double lng, double lat) {
        redis.opsForGeo().add(KEY,
            new Point(lng, lat),   // Note: Redis GEO uses lng, lat order
            driverId.toString());
    }

    public List<Long> findNearbyDrivers(double lng, double lat, double radiusKm) {
        GeoResults<RedisGeoCommands.GeoLocation<String>> results =
            redis.opsForGeo().radius(KEY,
                new Circle(new Point(lng, lat),
                           new Distance(radiusKm, Metrics.KILOMETERS)));

        return results.getContent().stream()
            .map(r -> Long.parseLong(r.getContent().getName()))
            .collect(Collectors.toList());
    }
}
```

---

## 4. Real-Time Communication (WebSockets)

```
Driver App ←──── WebSocket ────→ Location Service
Rider App  ←──── WebSocket ────→ Trip Service

Why WebSockets over polling?
  Polling: client sends request every 2s → 500ms average staleness + wasted requests
  WebSocket: server pushes when location changes → 0ms latency, no wasted requests
```

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(locationWebSocketHandler(), "/ws/location")
                .addHandler(tripWebSocketHandler(), "/ws/trip")
                .setAllowedOrigins("*");
    }
}

@Component
public class LocationWebSocketHandler extends TextWebSocketHandler {

    // Map of driverId → WebSocket session (for pushing to specific driver)
    private final ConcurrentHashMap<Long, WebSocketSession> driverSessions = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        Long driverId = extractDriverId(session);
        driverSessions.put(driverId, session);
        log.info("Driver {} connected", driverId);
    }

    // Called when driver app sends location update
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        LocationUpdate update = parseLocationUpdate(message.getPayload());
        locationService.updateDriverLocation(update.getDriverId(), update.getLat(), update.getLng());

        // If driver has an active trip, push location to rider
        if (tripService.hasActiveTrip(update.getDriverId())) {
            Long riderId = tripService.getRiderForDriver(update.getDriverId());
            pushLocationToRider(riderId, update);
        }
    }
}
```

---

## 5. Trip State Machine

```
REQUESTED → DRIVER_ACCEPTED → DRIVER_ARRIVED → TRIP_STARTED → TRIP_COMPLETED
     │              │
     │         DRIVER_CANCELLED
     │
  NO_DRIVER_FOUND (timeout 60s)
  RIDER_CANCELLED

State transitions trigger:
  REQUESTED → DRIVER_ACCEPTED: driver accepts → notify rider
  DRIVER_ARRIVED → TRIP_STARTED: driver taps "Start trip"
  TRIP_STARTED → TRIP_COMPLETED: driver taps "End trip" → trigger payment
```

```java
@Entity
public class Trip {
    @Id private Long id;
    private Long riderId;
    private Long driverId;

    @Enumerated(EnumType.STRING)
    private TripStatus status;

    private Double originLat, originLng;
    private Double destLat, destLng;
    private BigDecimal estimatedFare;
    private BigDecimal finalFare;
    private Instant createdAt;
    private Instant completedAt;

    // State machine transitions with validation
    public void acceptByDriver(Long driverId) {
        if (this.status != TripStatus.REQUESTED) throw new InvalidStateException(...);
        this.driverId = driverId;
        this.status = TripStatus.DRIVER_ACCEPTED;
    }

    public void complete(BigDecimal finalFare) {
        if (this.status != TripStatus.TRIP_STARTED) throw new InvalidStateException(...);
        this.status = TripStatus.TRIP_COMPLETED;
        this.finalFare = finalFare;
        this.completedAt = Instant.now();
    }
}
```

---

## 6. Matching Algorithm

```
On ride request:
1. Find all available drivers within 5km (Redis GEO)
2. Filter: is driver available? (status = AVAILABLE)
3. Rank by: distance + driver rating + estimated pick-up ETA
4. Send request to top 3 drivers simultaneously
5. First driver to accept wins
6. If no acceptance in 30s → try next 3 drivers
7. If no acceptance in 60s → notify rider "no cars available"

ETA calculation:
  Simple: straight-line distance / average speed
  Production: call routing service (Google Maps, OSRM) for road-network ETA
```

---

## 7. Full Architecture

```
Driver App ←──WebSocket──→ Location Service → Redis GEO + Geohash
                                    │
                                    ▼
Rider App  ←──WebSocket──→ Trip Service ──→ Matching Service
                                    │
                                    ▼
                             MySQL (trips, users)
                                    │
                                    ▼ TRIP_COMPLETED event
                                 Kafka
                               ┌────┤
                               ▼    ▼
                          Payment  Analytics
                          Service  Service
                            │
                            ▼
                      Stripe/PayPal
```

---

## 8. Key Takeaways

1. **Geospatial indexing is non-negotiable** — standard database indexes can't efficiently query "within 5km of point."
2. **WebSockets for real-time push, not polling** — location updates every 5 seconds × 10M drivers = polling doesn't scale.
3. **Model the domain as a state machine** — trips have well-defined states and transitions; enforce them in code.
4. **Separate location tracking from trip management** — different scaling requirements, different failure modes.
5. **Async payment processing** — never block the trip completion on payment; fire event and confirm asynchronously.

---

## 9. Resources

- **[Uber Engineering Blog](https://www.uber.com/en-US/blog/engineering/)** — Real architecture decisions at Uber.
- **[Redis GEO commands](https://redis.io/docs/manual/data-types/geospatial/)** — Official docs for geospatial queries.
- **[H3: Uber's Hexagonal Hierarchical Spatial Index](https://eng.uber.com/h3/)** — Production alternative to Geohash.

---

*Day 32. When millions of cars move every second, the map isn't just a feature — it's the entire system.*
