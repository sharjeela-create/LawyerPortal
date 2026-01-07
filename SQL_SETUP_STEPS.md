# Attorney Profiles - Step-by-Step SQL Setup Guide

## Overview
This guide provides step-by-step SQL queries to execute in your Supabase SQL Editor to set up the attorney profiles feature.

---

## STEP 1: Create the attorney_profiles Table

Copy and paste this entire query into your Supabase SQL Editor and click "Run":

```sql
-- Create the main attorney_profiles table
CREATE TABLE IF NOT EXISTS attorney_profiles (
  -- Primary identification
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  
  -- Tab 1: General Information - Core Identity
  profile_photo_url TEXT,
  full_name TEXT NOT NULL,
  firm_name TEXT NOT NULL,
  bar_association_number TEXT NOT NULL,
  professional_bio TEXT,
  years_experience INTEGER,
  languages_spoken TEXT[] NOT NULL DEFAULT '{}',
  
  -- Tab 1: General Information - Contact Details
  primary_email TEXT NOT NULL,
  direct_phone TEXT NOT NULL,
  office_address TEXT NOT NULL,
  website_url TEXT,
  preferred_contact_method TEXT CHECK (preferred_contact_method IN ('email', 'phone', 'text')),
  
  -- Tab 1: General Information - Support Staff
  assistant_name TEXT,
  assistant_email TEXT,
  
  -- Tab 2: Expertise & Jurisdiction - Geographic Coverage
  licensed_states TEXT[] NOT NULL DEFAULT '{}',
  primary_city TEXT NOT NULL,
  counties_covered TEXT[] DEFAULT '{}',
  federal_court_admissions TEXT,
  
  -- Tab 2: Expertise & Jurisdiction - Case Specialization
  primary_practice_focus TEXT NOT NULL,
  injury_categories TEXT[] NOT NULL DEFAULT '{}',
  exclusionary_criteria TEXT[] DEFAULT '{}',
  minimum_case_value DECIMAL(12, 2),
  
  -- Tab 3: Capacity & Performance - Real-Time Status
  availability_status TEXT NOT NULL DEFAULT 'accepting' 
    CHECK (availability_status IN ('accepting', 'at_capacity', 'on_leave')),
  firm_size TEXT CHECK (firm_size IN ('solo', 'small', 'medium', 'large')),
  case_management_software TEXT,
  
  -- Tab 3: Capacity & Performance - Performance Metrics
  insurance_carriers_handled TEXT[] DEFAULT '{}',
  litigation_style INTEGER CHECK (litigation_style BETWEEN 1 AND 5),
  largest_settlement_amount DECIMAL(15, 2),
  avg_time_to_close TEXT,
  
  -- Metadata
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  -- Constraints
  UNIQUE(user_id),
  UNIQUE(bar_association_number)
);
```

**Expected Result:** Table created successfully

---

## STEP 2: Create Indexes for Performance

Run this query to create indexes for faster searches:

```sql
-- Create indexes for common query patterns
CREATE INDEX IF NOT EXISTS idx_attorney_profiles_user_id 
  ON attorney_profiles(user_id);

CREATE INDEX IF NOT EXISTS idx_attorney_profiles_availability 
  ON attorney_profiles(availability_status);

CREATE INDEX IF NOT EXISTS idx_attorney_profiles_licensed_states 
  ON attorney_profiles USING GIN(licensed_states);

CREATE INDEX IF NOT EXISTS idx_attorney_profiles_injury_categories 
  ON attorney_profiles USING GIN(injury_categories);

CREATE INDEX IF NOT EXISTS idx_attorney_profiles_primary_city 
  ON attorney_profiles(primary_city);

CREATE INDEX IF NOT EXISTS idx_attorney_profiles_practice_focus 
  ON attorney_profiles(primary_practice_focus);
```

**Expected Result:** 6 indexes created successfully

---

## STEP 3: Create Updated Timestamp Trigger

Run this query to automatically update the `updated_at` field:

```sql
-- Create function to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_attorney_profiles_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER attorney_profiles_updated_at
  BEFORE UPDATE ON attorney_profiles
  FOR EACH ROW
  EXECUTE FUNCTION update_attorney_profiles_updated_at();
```

**Expected Result:** Function and trigger created successfully

---

## STEP 4: Enable Row Level Security (RLS)

Run this query to enable RLS on the table:

```sql
-- Enable Row Level Security
ALTER TABLE attorney_profiles ENABLE ROW LEVEL SECURITY;
```

**Expected Result:** RLS enabled

---

## STEP 5: Create RLS Policies

Run these queries one by one to create security policies:

### Policy 1: Users can view their own profile
```sql
CREATE POLICY attorney_profiles_select_own
  ON attorney_profiles
  FOR SELECT
  USING (auth.uid() = user_id);
```

### Policy 2: Users can insert their own profile
```sql
CREATE POLICY attorney_profiles_insert_own
  ON attorney_profiles
  FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

### Policy 3: Users can update their own profile
```sql
CREATE POLICY attorney_profiles_update_own
  ON attorney_profiles
  FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

### Policy 4: Admins can view all profiles
```sql
CREATE POLICY attorney_profiles_select_admin
  ON attorney_profiles
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM app_users
      WHERE app_users.user_id = auth.uid()
      AND app_users.role = 'admin'
    )
  );
```

### Policy 5: Public can view active attorney profiles (for directory)
```sql
CREATE POLICY attorney_profiles_select_public
  ON attorney_profiles
  FOR SELECT
  USING (availability_status = 'accepting');
```

**Expected Result:** 5 policies created successfully

---

## STEP 6: Create Helper Views (Optional but Recommended)

### View 1: Active Attorneys
```sql
CREATE OR REPLACE VIEW active_attorneys AS
SELECT 
  id,
  user_id,
  full_name,
  firm_name,
  primary_city,
  licensed_states,
  primary_practice_focus,
  injury_categories,
  minimum_case_value,
  years_experience,
  languages_spoken,
  created_at,
  updated_at
FROM attorney_profiles
WHERE availability_status = 'accepting';
```

### View 2: Attorney Directory (Public)
```sql
CREATE OR REPLACE VIEW attorney_directory AS
SELECT 
  id,
  full_name,
  firm_name,
  professional_bio,
  years_experience,
  primary_city,
  licensed_states,
  primary_practice_focus,
  injury_categories,
  languages_spoken,
  website_url,
  created_at
FROM attorney_profiles
WHERE availability_status = 'accepting'
ORDER BY created_at DESC;
```

**Expected Result:** 2 views created successfully

---

## STEP 7: Verify Setup

Run this query to verify everything is set up correctly:

```sql
-- Check if table exists
SELECT EXISTS (
  SELECT FROM information_schema.tables 
  WHERE table_schema = 'public' 
  AND table_name = 'attorney_profiles'
) AS table_exists;

-- Check if indexes exist
SELECT indexname 
FROM pg_indexes 
WHERE tablename = 'attorney_profiles';

-- Check if RLS is enabled
SELECT relname, relrowsecurity 
FROM pg_class 
WHERE relname = 'attorney_profiles';

-- Check policies
SELECT policyname 
FROM pg_policies 
WHERE tablename = 'attorney_profiles';
```

**Expected Results:**
- `table_exists`: true
- 6 indexes listed
- `relrowsecurity`: true
- 5 policies listed

---

## Testing Queries

### Test 1: Insert a Sample Attorney Profile

```sql
-- Replace 'YOUR_USER_ID' with an actual user_id from auth.users
INSERT INTO attorney_profiles (
  user_id,
  full_name,
  firm_name,
  bar_association_number,
  professional_bio,
  years_experience,
  languages_spoken,
  primary_email,
  direct_phone,
  office_address,
  licensed_states,
  primary_city,
  primary_practice_focus,
  injury_categories,
  availability_status
) VALUES (
  'YOUR_USER_ID',
  'John Doe',
  'Doe & Associates Law Firm',
  'BAR123456',
  'Experienced personal injury attorney with over 15 years of practice.',
  15,
  ARRAY['english', 'spanish'],
  'john.doe@lawfirm.com',
  '+1 (555) 123-4567',
  '123 Main Street, Suite 100, Los Angeles, CA 90001',
  ARRAY['CA', 'NV'],
  'Los Angeles',
  'personal_injury',
  ARRAY['auto_accidents', 'truck_accidents', 'slip_and_fall'],
  'accepting'
);
```

### Test 2: Query Attorney Profiles

```sql
-- Get all attorney profiles
SELECT * FROM attorney_profiles;

-- Get active attorneys only
SELECT * FROM active_attorneys;

-- Search for attorneys by state and practice area
SELECT 
  full_name,
  firm_name,
  primary_city,
  years_experience
FROM attorney_profiles
WHERE availability_status = 'accepting'
  AND 'CA' = ANY(licensed_states)
  AND 'auto_accidents' = ANY(injury_categories)
ORDER BY years_experience DESC;
```

### Test 3: Update Attorney Profile

```sql
-- Update availability status
UPDATE attorney_profiles
SET availability_status = 'at_capacity'
WHERE user_id = 'YOUR_USER_ID';

-- Update multiple fields
UPDATE attorney_profiles
SET 
  years_experience = 16,
  professional_bio = 'Updated bio text',
  website_url = 'https://www.example.com'
WHERE user_id = 'YOUR_USER_ID';
```

---

## Common Queries for Application Use

### Query 1: Get Profile for Current User
```sql
SELECT * FROM attorney_profiles
WHERE user_id = auth.uid();
```

### Query 2: Upsert (Insert or Update) Profile
```sql
INSERT INTO attorney_profiles (
  user_id,
  full_name,
  firm_name,
  bar_association_number,
  -- ... other fields
) VALUES (
  auth.uid(),
  'John Doe',
  'Doe Law Firm',
  'BAR123456'
  -- ... other values
)
ON CONFLICT (user_id)
DO UPDATE SET
  full_name = EXCLUDED.full_name,
  firm_name = EXCLUDED.firm_name,
  -- ... other fields
RETURNING *;
```

### Query 3: Search Attorneys for Case Matching
```sql
SELECT 
  full_name,
  firm_name,
  primary_city,
  direct_phone,
  primary_email,
  years_experience
FROM attorney_profiles
WHERE availability_status = 'accepting'
  AND 'CA' = ANY(licensed_states)
  AND 'auto_accidents' = ANY(injury_categories)
  AND (minimum_case_value IS NULL OR minimum_case_value <= 50000)
ORDER BY years_experience DESC
LIMIT 10;
```

### Query 4: Get Attorney Directory Listing
```sql
SELECT * FROM attorney_directory
LIMIT 20 OFFSET 0;
```

---

## Troubleshooting

### Issue: "relation attorney_profiles does not exist"
**Solution:** Run STEP 1 again to create the table.

### Issue: "permission denied for table attorney_profiles"
**Solution:** 
1. Check if RLS is enabled (STEP 4)
2. Verify policies are created (STEP 5)
3. Ensure you're authenticated when testing

### Issue: "duplicate key value violates unique constraint"
**Solution:** 
- For `user_id`: Each user can only have one profile. Use UPDATE or UPSERT instead.
- For `bar_association_number`: This number must be unique. Use a different bar number.

### Issue: "column does not exist"
**Solution:** Verify column names match exactly (use snake_case like `full_name`, not camelCase).

---

## Summary

After completing all steps, you will have:

✅ **1 Table**: `attorney_profiles` with 29 columns
✅ **6 Indexes**: For optimized queries
✅ **1 Trigger**: Auto-update `updated_at` timestamp
✅ **5 RLS Policies**: Secure access control
✅ **2 Views**: Helper views for common queries

Your attorney profiles feature is now ready to use! The frontend will automatically save and load data from this table.
