
## 📁 **FILE 4: `/docs/COMMISSION-SYSTEM.md`**

```markdown
# Commission System with Safeguards

## Database Schema

```sql
-- Enhanced commission ledger with hold periods
CREATE TABLE commission_ledger (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  from_user_id UUID REFERENCES auth.users NOT NULL,
  to_user_id UUID REFERENCES auth.users NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  commission_level INTEGER NOT NULL,
  commission_type TEXT DEFAULT 'ongoing', -- 'ongoing' or 'one_time'
  upfront_amount DECIMAL(10,2) DEFAULT 0.00,
  source_type TEXT NOT NULL, -- 'visa_sale', 'matrix', 'upgrade'
  status TEXT DEFAULT 'pending', -- 'pending', 'available', 'reversed'
  available_after TIMESTAMPTZ, -- 30-day hold for one-time commissions
  reversed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- VISA sales tracking
CREATE TABLE visa_sales (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  buyer_user_id UUID REFERENCES auth.users,
  seller_user_id UUID REFERENCES auth.users,
  visa_tier_id UUID REFERENCES visa_tiers(id),
  upfront_amount DECIMAL(10,2) NOT NULL,
  one_time_commission_paid BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

35% Commission Logic

```sql
CREATE OR REPLACE FUNCTION process_visa_sale_commission(
  buyer_id UUID,
  recruiter_id UUID,
  visa_tier_id UUID
) RETURNS DECIMAL AS $$
DECLARE
  tier_record RECORD;
  one_time_commission DECIMAL;
BEGIN
  SELECT * INTO tier_record FROM visa_tiers WHERE id = visa_tier_id;
  
  IF tier_record.level IN (1, 2) THEN
    one_time_commission := tier_record.upfront_cost * 0.35;
    
    INSERT INTO commission_ledger (
      from_user_id, to_user_id, amount, commission_level,
      commission_type, upfront_amount, source_type,
      status, available_after
    ) VALUES (
      buyer_id, recruiter_id, one_time_commission, 0,
      'one_time', tier_record.upfront_cost, 'visa_sale',
      'pending', NOW() + INTERVAL '30 days'
    );
    
    RETURN one_time_commission;
  ELSE
    RETURN 0.00;
  END IF;
END;
$$ LANGUAGE plpgsql;
```

Chargeback Protection

```sql
-- 30-day cooldown period
CREATE OR REPLACE FUNCTION get_available_balance(user_id UUID)
RETURNS DECIMAL AS $$
DECLARE
  available DECIMAL;
BEGIN
  SELECT COALESCE(SUM(amount), 0) INTO available
  FROM commission_ledger 
  WHERE to_user_id = user_id 
    AND status = 'available'
    AND (available_after IS NULL OR available_after <= NOW());
    
  RETURN available;
END;
$$ LANGUAGE plpgsql;
```

Monthly Fee Logic

· Fees only charged when: commission earnings > monthly fee amount + 10%
· Example: £575 monthly fee requires £632.50 in commissions
