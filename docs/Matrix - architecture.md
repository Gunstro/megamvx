# Matrix Architecture & Placement Logic

## Database Schema

```sql
-- Matrix capacity tracking
CREATE TABLE matrix_level_capacity (
  level INTEGER PRIMARY KEY,
  tier_name TEXT NOT NULL,
  max_capacity INTEGER NOT NULL,
  current_count INTEGER DEFAULT 0,
  UNIQUE(level, tier_name)
);

-- User matrix positions
CREATE TABLE user_matrix_positions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users NOT NULL,
  visa_tier_id UUID REFERENCES visa_tiers(id) NOT NULL,
  matrix_level INTEGER NOT NULL,
  position_path LTREE NOT NULL,
  upline_user_id UUID REFERENCES auth.users,
  direct_downlines_count INTEGER DEFAULT 0,
  max_downlines INTEGER NOT NULL,
  is_active BOOLEAN DEFAULT true,
  total_earnings DECIMAL(10,2) DEFAULT 0.00,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Spatial indexes for performance
CREATE INDEX user_matrix_path_idx ON user_matrix_positions USING GIST(position_path);
CREATE INDEX user_matrix_upline_idx ON user_matrix_positions(upline_user_id);
