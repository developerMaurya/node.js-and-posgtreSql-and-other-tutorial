# Geographic Information Systems (GIS)

## PostGIS Setup
### Installation
```sql
-- Install PostGIS extension
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION postgis_raster;

-- Check version
SELECT PostGIS_Version();
```

### Basic Geometry Types
```sql
-- Point
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location GEOMETRY(Point, 4326)
);

-- LineString
CREATE TABLE routes (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    path GEOMETRY(LineString, 4326)
);

-- Polygon
CREATE TABLE areas (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    boundary GEOMETRY(Polygon, 4326)
);
```

## Spatial Operations
### Point Operations
```sql
-- Create points
INSERT INTO locations (name, location)
VALUES (
    'Central Park',
    ST_SetSRID(ST_MakePoint(-73.965355, 40.782865), 4326)
);

-- Calculate distance
SELECT 
    a.name as location1,
    b.name as location2,
    ST_Distance(
        a.location::geography,
        b.location::geography
    ) as distance_meters
FROM locations a
CROSS JOIN locations b
WHERE a.id < b.id;

-- Find nearby points
SELECT name
FROM locations
WHERE ST_DWithin(
    location::geography,
    ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326)::geography,
    5000  -- 5km radius
);
```

### Area Operations
```sql
-- Create polygon
INSERT INTO areas (name, boundary)
VALUES (
    'Manhattan',
    ST_GeomFromText('POLYGON((-74.02 40.70, -73.97 40.70, -73.97 40.77, -74.02 40.77, -74.02 40.70))', 4326)
);

-- Calculate area
SELECT 
    name,
    ST_Area(boundary::geography) as area_sq_meters
FROM areas;

-- Check if point is within polygon
SELECT name
FROM areas
WHERE ST_Contains(
    boundary,
    ST_SetSRID(ST_MakePoint(-73.985428, 40.748817), 4326)
);
```

## Spatial Indexing
### GIST Index
```sql
-- Create spatial index
CREATE INDEX idx_locations_location 
ON locations USING GIST (location);

CREATE INDEX idx_areas_boundary 
ON areas USING GIST (boundary);

-- Analyze index usage
EXPLAIN ANALYZE
SELECT name
FROM locations
WHERE ST_DWithin(
    location::geography,
    ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326)::geography,
    5000
);
```

## Advanced GIS Operations
### Geocoding
```sql
-- Install Tiger Geocoder
CREATE EXTENSION postgis_tiger_geocoder;
CREATE EXTENSION fuzzystrmatch;

-- Geocode address
SELECT g.rating, ST_X(g.geomout) As lon, ST_Y(g.geomout) As lat,
    (addy).address As street, (addy).streetname As street_name,
    (addy).streettypeabbrev As street_type, (addy).location As city,
    (addy).stateabbrev As state, (addy).zip
FROM geocode('42 Broadway, New York, NY 10004') As g;
```

### Routing
```sql
-- Install pgRouting
CREATE EXTENSION pgrouting;

-- Create routing network
CREATE TABLE street_network (
    id SERIAL PRIMARY KEY,
    source INTEGER,
    target INTEGER,
    cost FLOAT,
    reverse_cost FLOAT,
    geom GEOMETRY(LineString, 4326)
);

-- Find shortest path
SELECT * FROM pgr_dijkstra(
    'SELECT id, source, target, cost FROM street_network',
    1,  -- start vertex
    2,  -- end vertex
    directed := true
);
```

## Common Use Cases
### Location-Based Services
```sql
-- Find nearest points of interest
CREATE OR REPLACE FUNCTION find_nearest_poi(
    lat DOUBLE PRECISION,
    lon DOUBLE PRECISION,
    radius_meters INTEGER
)
RETURNS TABLE (
    name VARCHAR,
    distance DOUBLE PRECISION,
    direction TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        p.name,
        ST_Distance(p.location::geography, ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography) as distance,
        degrees(ST_Azimuth(
            ST_SetSRID(ST_MakePoint(lon, lat), 4326),
            p.location
        ))::text as direction
    FROM points_of_interest p
    WHERE ST_DWithin(
        p.location::geography,
        ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography,
        radius_meters
    )
    ORDER BY distance;
END;
$$ LANGUAGE plpgsql;
```

### Geofencing
```sql
-- Create geofence table
CREATE TABLE geofences (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    boundary GEOMETRY(Polygon, 4326),
    notification_radius INTEGER  -- meters
);

-- Check if point enters geofence
CREATE OR REPLACE FUNCTION check_geofence_entry(
    lat DOUBLE PRECISION,
    lon DOUBLE PRECISION
)
RETURNS TABLE (
    fence_name VARCHAR,
    distance DOUBLE PRECISION
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        g.name,
        ST_Distance(
            g.boundary::geography,
            ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography
        ) as distance
    FROM geofences g
    WHERE ST_DWithin(
        g.boundary::geography,
        ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography,
        g.notification_radius
    );
END;
$$ LANGUAGE plpgsql;
```

## Performance Optimization
### Index Optimization
```sql
-- Cluster table based on spatial index
CLUSTER locations USING idx_locations_location;

-- Analyze table statistics
ANALYZE locations;

-- Monitor index usage
SELECT 
    schemaname, tablename, indexname,
    idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexname LIKE 'idx_%_location';
```

### Query Optimization
```sql
-- Use ST_DWithin instead of ST_Distance
-- Bad
SELECT *
FROM locations
WHERE ST_Distance(location, point) < radius;

-- Good
SELECT *
FROM locations
WHERE ST_DWithin(location, point, radius);

-- Use spatial index hints if needed
SET enable_seqscan = false;
```

## Best Practices
1. Spatial Indexing
   - Always use spatial indexes
   - Regular ANALYZE
   - Monitor index usage

2. Data Organization
   - Appropriate SRID selection
   - Consistent coordinate systems
   - Regular maintenance

3. Query Performance
   - Use spatial functions wisely
   - Leverage index capabilities
   - Monitor query plans

## Related Topics
- [[Extensions and Plugins]]
- [[Performance Tuning]]
- [[Query Optimization]]
- [[Real-World Examples]]
