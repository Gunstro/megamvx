Excellent! This is incredibly detailed and gives us everything we need. Now I'll translate this into a complete technical specification.

🎯 Complete Database Schema for Supabase

1. VISA Tiers Table

```sql
CREATE TABLE visa_tiers (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  category TEXT NOT NULL, -- 'Super', 'Intermediate', 'Standard'
  name TEXT NOT NULL UNIQUE, -- 'Gold VIP', 'Gold Premiere', etc.
  level INTEGER NOT NULL, -- 1, 2, 3
  upfront_cost DECIMAL(10,2) DEFAULT 0.00,
  monthly_fee DECIMAL(10,2) DEFAULT 0.00,
  earning_cap DECIMAL(10,2) NOT NULL,
  max_positions INTEGER, -- 11750, 8250, etc.
  ads_required BOOLEAN DEFAULT false,
  connects_to_lower_level INTEGER NOT NULL, -- 20, 10, 5
  upgrade_only BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

2. Matrix Structure Table

```sql
CREATE TABLE matrix_structure (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  level INTEGER NOT NULL UNIQUE, -- 1 through 7
  level_name TEXT NOT NULL, -- 'Super', 'Intermediate', 'Standard'
  positions_count BIGINT NOT NULL, -- 20000, 400000, etc.
  commission_rates JSONB NOT NULL, -- Store {L1: 0.20, L2: 0.03, ...}
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

3. User Matrix Positions Table

```sql
CREATE TABLE user_matrix_positions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users NOT NULL,
  visa_tier_id UUID REFERENCES visa_tiers(id) NOT NULL,
  matrix_level INTEGER NOT NULL, -- 1, 2, 3
  position_path LTREE NOT NULL, -- Hierarchical path (e.g., '1.5.23.456')
  upline_user_id UUID REFERENCES auth.users, -- Immediate sponsor
  direct_downlines_count INTEGER DEFAULT 0,
  max_downlines INTEGER NOT NULL, -- From visa_tier.connects_to_lower_level
  is_active BOOLEAN DEFAULT true,
  total_earnings DECIMAL(10,2) DEFAULT 0.00,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Critical performance indexes
CREATE INDEX user_matrix_path_idx ON user_matrix_positions USING GIST(position_path);
CREATE INDEX user_matrix_upline_idx ON user_matrix_positions(upline_user_id);
```

4. Commission Ledger Table

```sql
CREATE TABLE commission_ledger (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  from_user_id UUID REFERENCES auth.users NOT NULL, -- Who generated commission
  to_user_id UUID REFERENCES auth.users NOT NULL, -- Who earns commission
  amount DECIMAL(10,2) NOT NULL,
  commission_level INTEGER NOT NULL, -- Which level in matrix (1-5)
  source_type TEXT NOT NULL, -- 'signup', 'purchase', 'subscription'
  matrix_position_id UUID REFERENCES user_matrix_positions(id),
  visa_tier_id UUID REFERENCES visa_tiers(id),
  calculated_rates JSONB, -- Store exact breakdown for audit
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

5. User Profiles with Financial Tracking

```sql
CREATE TABLE user_profiles (
  id UUID REFERENCES auth.users PRIMARY KEY,
  username TEXT UNIQUE,
  current_visa_tier_id UUID REFERENCES visa_tiers(id),
  home_location GEOGRAPHY(POINT),
  total_earnings DECIMAL(10,2) DEFAULT 0.00,
  current_balance DECIMAL(10,2) DEFAULT 0.00,
  earning_cap DECIMAL(10,2) DEFAULT 0.00,
  monthly_fee_status TEXT DEFAULT 'pending', -- pending, active, suspended
  last_commission_calc TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

🔄 Commission Calculation Logic

```sql
-- This function handles the complex commission distribution
CREATE OR REPLACE FUNCTION calculate_commission_flow(
  from_user_id UUID,
  source_amount DECIMAL,
  source_type TEXT
) RETURNS VOID AS $$
DECLARE
  upline_record RECORD;
  current_level INTEGER;
  commission_rates JSONB;
  level_amount DECIMAL;
BEGIN
  -- Get upline chain and apply commission rates
  FOR upline_record IN 
    WITH RECURSIVE upline_chain AS (
      SELECT ump.*, 1 as depth
      FROM user_matrix_positions ump 
      WHERE ump.user_id = from_user_id
      UNION ALL
      SELECT ump.*, uc.depth + 1
      FROM user_matrix_positions ump
      INNER JOIN upline_chain uc ON ump.user_id = uc.upline_user_id
      WHERE uc.depth < 5 -- Only levels 1-5 get commissions
    )
    SELECT * FROM upline_chain
  LOOP
    -- Get commission rate for this level
    commission_rates := (SELECT commission_rates FROM matrix_structure WHERE level = upline_record.matrix_level);
    level_amount := source_amount * (commission_rates->>('L' || upline_record.depth))::DECIMAL;
    
    -- Insert into commission ledger
    INSERT INTO commission_ledger (...)
    VALUES (from_user_id, upline_record.user_id, level_amount, upline_record.depth, ...);
    
    -- Update user earnings
    UPDATE user_profiles 
    SET total_earnings = total_earnings + level_amount,
        current_balance = current_balance + level_amount
    WHERE id = upline_record.user_id;
  END LOOP;
  
  -- Handle L0 company allocation (59% balance)
  -- Handle MB allocation (20%)
  -- Handle matrix bottom sustainability (20%)
END;
$$ LANGUAGE plpgsql;
```

🛡️ Critical Security Policies

```sql
-- Users can only see their own matrix position
CREATE POLICY "Users can view own matrix" ON user_matrix_positions
FOR SELECT USING (auth.uid() = user_id);

-- Users can only see commissions they earned
CREATE POLICY "Users can view own commissions" ON commission_ledger  
FOR SELECT USING (auth.uid() = to_user_id);

-- Only system can insert commissions (prevent fraud)
CREATE POLICY "System inserts commissions" ON commission_ledger
FOR INSERT WITH CHECK (auth.uid() = 'system_user_uuid');
```

🚀 Next Steps

1. I'll create the complete SQL migration file with all tables, functions, and security policies
2. We'll set up the initial data (VISA tiers, matrix levels, commission rates)
3. We'll build the commission calculation engine
4. Then we'll integrate with user authentication

This specification now completely encodes your business logic. The next developer can implement this exactly as specified.

Should I proceed with the complete SQL migration file, or do you need any clarifications on the schema design?

