# EuronSupport - Complete Setup Guide

##  Prerequisites Checklist

Before starting, ensure you have:

- [ ] Python 3.8 or higher installed
- [ ] PostgreSQL database access (Neon or any PostgreSQL instance)
- [ ] OpenAI API key (get from https://platform.openai.com)
- [ ] Git (optional, for version control)

##  Step-by-Step Setup

### Step 1: Navigate to Project Directory

```bash
cd d:\crewai17thjangenaicer
```

### Step 2: Create Virtual Environment (Recommended)

```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate
# On Linux/Mac:
source venv/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install -r requirements.txt
```

**Expected Output:**
```
Successfully installed crewai-1.8.1 openai-1.83.0 streamlit-1.49.1 ...
```

**If you encounter NumPy errors:**
```bash
pip install "numpy<2.0.0"
```

### Step 4: Configure Environment Variables

1. **Copy the example file:**
```bash
# Windows PowerShell
Copy-Item .env.example .env

# Linux/Mac
cp .env.example .env
```

2. **Edit `.env` file with your values:**

```env
# Required: OpenAI API Key
OPENAI_API_KEY=sk-proj-your-actual-key-here

# Required: PostgreSQL Connection String
DATABASE_URL=postgresql://neondb_owner:npg_gDh86RpYwObM@ep-empty-queen-ahxgu44w-pooler.c-3.us-east-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require

# Optional: Email Configuration (for notifications)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_FROM=your_email@gmail.com

# Optional: Admin Password
ADMIN_PASSWORD=admin123
```

**Getting PostgreSQL Connection String:**
- If using Neon: Copy connection string from Neon dashboard
- Format: `postgresql://user:password@host:port/database?sslmode=require`

**Getting OpenAI API Key:**
1. Go to https://platform.openai.com
2. Sign in or create account
3. Navigate to API Keys section
4. Create new secret key
5. Copy and paste into `.env` file

### Step 5: Initialize Database

The database will be automatically initialized on first run. No manual setup needed!

**What happens on first run:**
- Tables are created automatically
- Default managers are inserted
- Indexes are created for performance

### Step 6: Test the System

```bash
python test_euronsupport.py
```

**Expected Output:**
```
[OK] API Key loaded: sk-proj-...
[OK] Database connected and initialized successfully
[OK] Found 5 Indian managers in database
[OK] Test ticket created: EURON-...
[OK] AI processing completed successfully!
[SUCCESS] ALL TESTS PASSED!
```

### Step 7: Start Applications

**Option A: Use Launcher (Recommended)**
```bash
python launcher.py
```

**Option B: Start Manually**

**Terminal 1 - User Interface:**
```bash
streamlit run user_app.py --server.port 8501
```

**Terminal 2 - Admin Dashboard:**
```bash
streamlit run admin_app.py --server.port 8502
```

### Step 8: Access Applications

1. **User Interface**: Open browser to http://localhost:8501
2. **Admin Dashboard**: Open browser to http://localhost:8502
   - Password: `admin123` (or your configured password)

##  Verification Checklist

After setup, verify:

- [ ] Database connection works (test script passes)
- [ ] User interface loads at http://localhost:8501
- [ ] Admin dashboard loads at http://localhost:8502
- [ ] Can create a test ticket
- [ ] AI processing completes successfully
- [ ] Manager assignment works
- [ ] Metrics display in admin dashboard

##  Troubleshooting

### Issue: Database Connection Error

**Error:** `psycopg2.OperationalError: could not connect to server`

**Solutions:**
1. Check DATABASE_URL in `.env` file
2. Verify PostgreSQL server is running
3. Check firewall settings
4. Verify SSL mode in connection string

### Issue: OpenAI API Error

**Error:** `openai.error.AuthenticationError`

**Solutions:**
1. Verify OPENAI_API_KEY in `.env` file
2. Check API key is valid and not expired
3. Verify account has credits
4. Check API key format (should start with `sk-`)

### Issue: NumPy Compatibility Error

**Error:** `AttributeError: np.unicode_ was removed`

**Solution:**
```bash
pip install "numpy<2.0.0"
```

### Issue: Import Errors

**Error:** `ModuleNotFoundError: No module named 'xxx'`

**Solution:**
```bash
pip install -r requirements.txt
```

### Issue: Port Already in Use

**Error:** `Port 8501 is already in use`

**Solutions:**
1. Stop other Streamlit instances
2. Use different ports:
   ```bash
   streamlit run user_app.py --server.port 8503
   streamlit run admin_app.py --server.port 8504
   ```

##  Post-Setup Configuration

### Change Admin Password

Edit `.env` file:
```env
ADMIN_PASSWORD=your_secure_password_here
```

### Configure Email Notifications

1. For Gmail:
   - Enable 2-factor authentication
   - Generate App Password: https://myaccount.google.com/apppasswords
   - Use App Password in SMTP_PASSWORD

2. Update `.env`:
```env
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_FROM=your_email@gmail.com
```

### Add Past Incidents

To improve AI context, add past incidents to database:

```python
from database import Database
from datetime import date

db = Database()
# Past incidents are added via database directly
# See ARCHITECTURE.md for schema details
```

##  Next Steps

1. **Read Documentation**: See [ARCHITECTURE.md](ARCHITECTURE.md) for complete system details
2. **Test System**: Create test tickets and verify processing
3. **Configure Email**: Set up SMTP for notifications
4. **Customize**: Update managers, context, or configurations as needed

##  Additional Resources

- **Architecture Documentation**: [ARCHITECTURE.md](ARCHITECTURE.md)
- **CrewAI Docs**: https://docs.crewai.com
- **Streamlit Docs**: https://docs.streamlit.io
- **PostgreSQL Docs**: https://www.postgresql.org/docs

---

**Setup Complete!**  Your EuronSupport system is ready to use.
