# Supabase Setup Guide for Bin There

## 🚀 Quick Start (Demo Mode)

**For immediate testing, use the demo version:**
- Open `auth-demo.html` in your browser
- Use demo credentials: `demo@binthere.com` / `demo123`
- This works without any setup!

## 🔧 Full Supabase Setup

### 1. Create a Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign up/login
2. Click "New Project"
3. Choose your organization
4. Enter project details:
   - **Name**: `bin-there`
   - **Database Password**: Choose a strong password
   - **Region**: Choose the closest to your users
5. Click "Create new project"

### 2. Get Your Project Credentials

1. Go to **Settings** → **API**
2. Copy your:
   - **Project URL** (looks like: `https://your-project.supabase.co`)
   - **anon public** key (starts with `eyJ...`)

### 3. Update the Auth Page

In `auth.html`, replace these lines:

```javascript
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-key';
```

With your actual credentials:

```javascript
const SUPABASE_URL = 'https://your-actual-project.supabase.co';
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';
```

## 4. Set Up Database Tables

Run this SQL in your Supabase SQL Editor:

```sql
-- Create profiles table
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  full_name TEXT,
  email TEXT,
  phone TEXT,
  role TEXT CHECK (role IN ('citizen', 'vendor', 'municipal')) DEFAULT 'citizen',
  bin_coins INTEGER DEFAULT 0,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Enable Row Level Security
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Create policies
CREATE POLICY "Users can view own profile" ON profiles
  FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can update own profile" ON profiles
  FOR UPDATE USING (auth.uid() = id);

CREATE POLICY "Users can insert own profile" ON profiles
  FOR INSERT WITH CHECK (auth.uid() = id);

-- Create function to handle new user signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, full_name, email, role, bin_coins)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data->>'full_name', NEW.email),
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'role', 'citizen'),
    50
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create trigger for new user signup
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

## 5. Configure Authentication Providers (Optional)

### Google OAuth
1. Go to **Authentication** → **Providers**
2. Enable **Google**
3. Add your Google OAuth credentials:
   - **Client ID**: From Google Cloud Console
   - **Client Secret**: From Google Cloud Console

### GitHub OAuth
1. Go to **Authentication** → **Providers**
2. Enable **GitHub**
3. Add your GitHub OAuth credentials:
   - **Client ID**: From GitHub Developer Settings
   - **Client Secret**: From GitHub Developer Settings

## 6. Configure Email Templates (Optional)

1. Go to **Authentication** → **Email Templates**
2. Customize the email templates for:
   - Confirm signup
   - Reset password
   - Magic link

## 7. Test Your Setup

1. Open `auth.html` in your browser
2. Try creating a new account
3. Check your Supabase dashboard to see the new user
4. Try signing in with the new account

## 8. Environment Variables (Production)

For production, consider using environment variables:

```javascript
const SUPABASE_URL = process.env.SUPABASE_URL || 'https://your-project.supabase.co';
const SUPABASE_ANON_KEY = process.env.SUPABASE_ANON_KEY || 'your-anon-key';
```

## Troubleshooting

### Common Issues:

1. **CORS Errors**: Make sure your domain is added to the allowed origins in Supabase settings
2. **Email Confirmation**: Check if email confirmation is required in Auth settings
3. **Database Errors**: Ensure the profiles table and policies are set up correctly
4. **OAuth Issues**: Verify your OAuth provider credentials and redirect URLs

### Useful Commands:

```sql
-- Check if profiles table exists
SELECT * FROM information_schema.tables WHERE table_name = 'profiles';

-- View all users
SELECT * FROM auth.users;

-- View all profiles
SELECT * FROM profiles;

-- Delete a user (if needed)
DELETE FROM auth.users WHERE email = 'user@example.com';
```

## Security Notes

- Never expose your service role key in client-side code
- Use Row Level Security (RLS) policies to protect user data
- Regularly rotate your API keys
- Monitor your Supabase usage and set up billing alerts
