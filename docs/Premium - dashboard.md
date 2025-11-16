## 📁 **FILE 5: `/docs/PREMIUM-DASHBOARD.md`**

```markdown
# Premium Recruitment Dashboard

## Real-Time Position Tracking

```sql
CREATE VIEW premium_position_availability AS
SELECT 
  vt.name as visa_tier,
  vt.level,
  vt.upfront_cost,
  mc.max_capacity,
  mc.current_count,
  (mc.max_capacity - mc.current_count) as remaining_slots,
  ROUND((mc.current_count::DECIMAL / mc.max_capacity) * 100, 1) as fill_percentage,
  (vt.upfront_cost * 0.35) as potential_commission
FROM matrix_level_capacity mc
JOIN visa_tiers vt ON mc.level = vt.level AND mc.tier_name = vt.name
WHERE vt.level IN (1, 2)
ORDER BY vt.level, vt.upfront_cost DESC;
```

Recruitment Performance Metrics

```sql
CREATE VIEW recruiter_performance AS
SELECT 
  u.id as recruiter_id,
  u.username,
  COUNT(DISTINCT CASE WHEN vt.level IN (1,2) THEN vs.id END) as premium_recruits,
  SUM(CASE WHEN vt.level IN (1,2) THEN cl.amount ELSE 0 END) as premium_commissions_earned,
  (SELECT COUNT(*) FROM premium_position_availability WHERE remaining_slots > 0) as premium_slots_available
FROM user_profiles u
LEFT JOIN visa_sales vs ON vs.seller_user_id = u.id
LEFT JOIN visa_tiers vt ON vs.visa_tier_id = vt.id
LEFT JOIN commission_ledger cl ON cl.to_user_id = u.id AND cl.source_type = 'visa_sale'
GROUP BY u.id, u.username;
```

Vue.js Dashboard Components

Position Availability Widget

· Real-time slot counts
· Fill percentages
· Potential commissions per sale

Recruitment Calculator

· Maximum earnings potential
· Tier-specific referral links
· Progress tracking

Performance Leaderboard

· Top recruiter rankings
· Premium conversion rates
· Commission earnings

```
