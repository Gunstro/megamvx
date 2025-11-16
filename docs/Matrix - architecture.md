📁 FILE 3: /docs/MATRIX-ARCHITECTURE.md

```markdown
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
```

Placement Algorithms

Level 1 & 2 Placement (Fixed Capacity)

```sql
-- Atomic position claiming
UPDATE matrix_level_capacity 
SET current_count = current_count + 1
WHERE level = 1 
  AND tier_name = 'Gold VIP'
  AND current_count < max_capacity
RETURNING current_count;
```

Level 3+ Placement (Breadth-First Fill)

```sql
-- Find first available position for sponsor
SELECT id FROM user_matrix_positions 
WHERE upline_user_id = 'sponsor_uuid'
  AND direct_downlines_count < max_downlines
ORDER BY created_at ASC 
LIMIT 1;
```

Upgrade Mechanics

· Automatic suggestions when users hit 80% of earning cap
· Forced upgrades when cap reached (to continue earning)
· Upgrade funding can use earned balance
· No cross-grading between tiers

```
