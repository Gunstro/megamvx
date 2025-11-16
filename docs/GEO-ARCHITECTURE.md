
# Geographic Architecture

## Boundary Hierarchy Levels
- **Level 0**: Countries (Admin 0)
- **Level 1**: Provinces/States
- **Level 2**: Municipalities  
- **Level 3**: Wards
- **Level 4**: MegatonCities (>2 million population)

## Data Specifications
- **Format**: Simplified GeoJSON
- **Coordinate System**: WGS84 (EPSG:4326)
- **Source**: GS provided datasets
- **Optimization**: Pre-simplified for web performance

## Spatial Database Schema
```sql
-- Core boundaries table
CREATE TABLE geo_boundaries (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  level INTEGER NOT NULL,
  parent_id UUID REFERENCES geo_boundaries(id),
  boundary GEOGRAPHY(POLYGON) NOT NULL,
  population INTEGER,
  properties JSONB
);

-- Spatial indexes
CREATE INDEX geo_boundaries_boundary_idx ON geo_boundaries USING GIST(boundary);
CREATE INDEX geo_boundaries_level_idx ON geo_boundaries(level);
