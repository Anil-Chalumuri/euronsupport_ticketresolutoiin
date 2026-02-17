# EuronSupport - System Architecture & Documentation

##  Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Database Schema](#database-schema)
4. [File System Structure](#file-system-structure)
5. [Workflow & Process Flow](#workflow--process-flow)
6. [Configuration Guide](#configuration-guide)
7. [Setup Instructions](#setup-instructions)
8. [Component Details](#component-details)
9. [API & Data Flow](#api--data-flow)

---

##  System Overview

**EuronSupport** is an AI-powered ticket resolution system that automatically processes customer support tickets, classifies them, and assigns them to the appropriate manager using CrewAI agents.

### Key Features
- **AI-Powered Processing**: 5 specialized CrewAI agents analyze tickets
- **Automatic Assignment**: Intelligent manager assignment based on ticket analysis
- **PostgreSQL Database**: Production-ready database on Neon
- **Real-time Analytics**: Comprehensive metrics and visualizations
- **Past Incidents Learning**: Uses historical data for better resolution
- **Email Notifications**: Automatic notifications to managers and users

---

##  Architecture Diagram

```

                        EuronSupport System                      


                  
  User App               Admin App              Launcher    
 (Streamlit)            (Streamlit)              Script     
  Port 8501              Port 8502                          
                  
                                                       
       
                                
                    
                       Application Layer  
                      (Python Modules)    
                    
                                
        
                                                      
    
  Database Layer       AI Processing         Email Service 
  (PostgreSQL)         (CrewAI Agents)       (SMTP)        
                                                           
  - Users              - Triage Lead         - Notifications
  - Tickets            - Support Analyst     - Updates      
  - Managers           - SRE Analyst                       
  - Logs               - Backend Analyst                   
  - Agent Results      - Tech Lead                         
  - Past Incidents                                         
    
        
        

          PostgreSQL Database (Neon)                 
  - Production-ready cloud database                   
  - ACID compliance                                    
  - Scalable architecture                             

```

---

##  Database Schema

### Tables Overview

#### 1. **users** Table
Stores user information for ticket submission.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `name`: User's full name
- `email`: Unique email address (used for ticket lookup)
- `phone`: Optional phone number
- `created_at`: Timestamp of account creation

#### 2. **managers** Table
Stores manager information for ticket assignment.

```sql
CREATE TABLE managers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    role VARCHAR(100) NOT NULL,
    department VARCHAR(100),
    expertise TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `name`: Manager's name
- `email`: Manager's email (for notifications)
- `role`: Manager role (SRE Lead, Backend Lead, etc.)
- `department`: Department name
- `expertise`: Areas of expertise
- `is_active`: Whether manager is currently active
- `created_at`: Timestamp of record creation

**Default Managers:**
- Rajesh Kumar - SRE Lead (Infrastructure)
- Priya Sharma - Backend Lead (Engineering)
- Amit Patel - Support Manager (Customer Support)
- Anjali Singh - QA Lead (Quality Assurance)
- Vikram Reddy - Tech Lead (Engineering)

#### 3. **tickets** Table
Main table storing all ticket information.

```sql
CREATE TABLE tickets (
    id SERIAL PRIMARY KEY,
    ticket_number VARCHAR(100) UNIQUE NOT NULL,
    user_id INTEGER REFERENCES users(id),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    category VARCHAR(100),
    severity VARCHAR(10),
    status VARCHAR(50) DEFAULT 'open',
    priority VARCHAR(20),
    assigned_manager_id INTEGER REFERENCES managers(id),
    assigned_manager_email VARCHAR(255),
    assignment_reason TEXT,
    resolution TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP
);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `ticket_number`: Unique ticket identifier (format: EURON-YYYYMMDD-timestamp)
- `user_id`: Foreign key to users table
- `title`: Ticket title/summary
- `description`: Detailed ticket description
- `category`: Ticket category (payment, video, auth, performance, crash, other)
- `severity`: Severity level (P0, P1, P2)
- `status`: Current status (open, assigned, in_progress, resolved, closed)
- `priority`: Priority level (high, medium, low)
- `assigned_manager_id`: Foreign key to managers table
- `assigned_manager_email`: Manager email (denormalized for quick access)
- `assignment_reason`: AI-generated reason for assignment
- `resolution`: Resolution details (when resolved)
- `created_at`: Ticket creation timestamp
- `updated_at`: Last update timestamp
- `resolved_at`: Resolution timestamp

**Status Flow:**
```
open → assigned → in_progress → resolved → closed
```

#### 4. **ticket_logs** Table
Audit trail of all ticket activities.

```sql
CREATE TABLE ticket_logs (
    id SERIAL PRIMARY KEY,
    ticket_id INTEGER NOT NULL REFERENCES tickets(id),
    action VARCHAR(100) NOT NULL,
    performed_by VARCHAR(255),
    details TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `ticket_id`: Foreign key to tickets table
- `action`: Action type (created, assigned, resolved, closed, etc.)
- `performed_by`: Who performed the action (system, admin, manager)
- `details`: Action details/description
- `metadata`: Additional JSON metadata
- `created_at`: Timestamp of action

**Common Actions:**
- `created`: Ticket created
- `processing_started`: AI processing began
- `assigned`: Ticket assigned to manager
- `processing_completed`: AI processing finished
- `resolved`: Ticket resolved
- `closed`: Ticket closed

#### 5. **agent_results** Table
Stores AI agent processing results.

```sql
CREATE TABLE agent_results (
    id SERIAL PRIMARY KEY,
    ticket_id INTEGER NOT NULL REFERENCES tickets(id),
    agent_name VARCHAR(255) NOT NULL,
    result_text TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `ticket_id`: Foreign key to tickets table
- `agent_name`: Name of the agent (Triage Lead, Support Analyst, etc.)
- `result_text`: Agent's analysis output
- `metadata`: Additional JSON metadata
- `created_at`: Timestamp of result

#### 6. **past_incidents** Table
Historical incidents for AI context and pattern matching.

```sql
CREATE TABLE past_incidents (
    id SERIAL PRIMARY KEY,
    incident_date DATE NOT NULL,
    summary TEXT NOT NULL,
    category VARCHAR(100),
    severity VARCHAR(10),
    resolution TEXT,
    mitigation_steps TEXT,
    related_tickets TEXT[],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields:**
- `id`: Auto-incrementing primary key
- `incident_date`: Date of incident
- `summary`: Incident summary
- `category`: Incident category
- `severity`: Incident severity
- `resolution`: How it was resolved
- `mitigation_steps`: Steps taken to mitigate
- `related_tickets`: Array of related ticket numbers
- `created_at`: Record creation timestamp

### Indexes

For performance optimization:

```sql
CREATE INDEX idx_tickets_status ON tickets(status);
CREATE INDEX idx_tickets_user_id ON tickets(user_id);
CREATE INDEX idx_tickets_created_at ON tickets(created_at);
CREATE INDEX idx_ticket_logs_ticket_id ON ticket_logs(ticket_id);
```

### Relationships

```
users (1) → (N) tickets
managers (1) → (N) tickets
tickets (1) → (N) ticket_logs
tickets (1) → (N) agent_results
```

---

##  File System Structure

```
euronsupport/

 Core Application Files
    database.py              # PostgreSQL database operations
    config.py                # Configuration and environment variables
    agents.py                # CrewAI agent definitions
    ticket_processor.py      # Ticket processing logic
    email_service.py         # Email notification service

 User Interface
    user_app.py              # Streamlit user interface (Port 8501)

 Admin Interface
    admin_app.py             # Streamlit admin dashboard (Port 8502)

 Utilities
    launcher.py              # Launches both apps
    test_euronsupport.py     # System test script

 Configuration
    requirements.txt         # Python dependencies
    .env.example             # Environment variables template
    .gitignore               # Git ignore rules

 Documentation
    README.md                # Quick start guide
    ARCHITECTURE.md          # This file - comprehensive documentation

 Reference (Optional)
     crewai.ipynb             # Original demo notebook
```

### File Descriptions

#### Core Application Files

**database.py** (448 lines)
- PostgreSQL database connection and operations
- Table creation and initialization
- CRUD operations for all entities
- Metrics and analytics queries
- Past incidents retrieval

**config.py** (72 lines)
- Environment variable loading
- Configuration constants
- Organization context (ORG_CONTEXT)
- Database URL configuration
- Email settings

**agents.py** (275 lines)
- CrewAI agent definitions (5 agents)
- Agent creation function
- Crew creation for ticket processing
- JSON extraction utilities
- Past incidents integration

**ticket_processor.py** (232 lines)
- Main ticket processing orchestration
- CrewAI crew execution
- Classification extraction
- Manager assignment logic
- Email notification triggering

**email_service.py**
- SMTP email configuration
- Manager assignment notifications
- User status update emails
- Error handling for email failures

#### User Interface

**user_app.py** (200+ lines)
- Streamlit user interface
- Ticket submission form
- User ticket tracking
- Status display with refresh
- Activity log viewing

#### Admin Interface

**admin_app.py** (373 lines)
- Streamlit admin dashboard
- Password authentication
- Metrics and analytics
- Interactive charts (Plotly)
- Ticket management
- AI results viewing
- Refresh functionality

#### Utilities

**launcher.py**
- Launches both Streamlit apps
- Background process management

**test_euronsupport.py**
- Comprehensive system tests
- Database connection test
- Ticket creation test
- AI processing test
- Metrics test

---

##  Workflow & Process Flow

### Ticket Submission Flow

```
1. User submits ticket via user_app.py
   
   → Validate input fields
   
2. Create/Get user in database
   
3. Create ticket record (status: 'open')
   
4. Log ticket creation
   
5. Trigger AI processing (ticket_processor.py)
   
   → Create CrewAI crew with 5 agents
   
   → Agent 1: Triage Lead
      → Analyze context and past incidents
   
   → Agent 2: Support Analyst
      → Classify ticket (severity, priority, category)
   
   → Agent 3: SRE Analyst
      → Analyze infrastructure aspects
   
   → Agent 4: Backend Analyst
      → Analyze backend/data aspects
   
   → Agent 5: Tech Lead
      → Synthesize findings and recommend manager
   
6. Extract classification from agent results
   
7. Assign manager based on recommendation
   
8. Update ticket with:
      - Severity, Priority, Category
      - Assigned Manager
      - Assignment Reason
      - Status: 'assigned'
   
9. Send email notification to manager
   
10. Send confirmation email to user
    
    → Ticket ready for manager action
```

### AI Processing Flow

```
Ticket Input
    
    → Triage Lead Agent
       → Context Analysis
           - Org context review
           - Past incidents matching
           - Risk hypothesis generation
    
    → Support Analyst Agent
       → Ticket Classification
           - User impact assessment
           - Severity classification (P0/P1/P2)
           - Priority classification (high/medium/low)
           - Category classification
           - JSON output generation
    
    → SRE Analyst Agent
       → Infrastructure Analysis
           - Metrics to check
           - Log evidence
           - Infra causes
           - Mitigation steps
           - Manager role recommendation
    
    → Backend Analyst Agent
       → Backend Analysis
           - Service/component identification
           - Data consistency risks
           - Root cause hypotheses
           - Code-level fixes
           - Idempotency recommendations
    
    → Tech Lead Agent
        → Synthesis & Assignment
            - Final severity/priority
            - Manager role recommendation
            - Assignment reason
            - Action items
            - JSON output generation
```

### Manager Assignment Logic

```
AI Recommendation
    
    → Extract recommended_manager_role from JSON
    
    → Map role keywords:
       - "SRE" / "Infra" → SRE Lead (Rajesh Kumar)
       - "Backend" / "API" → Backend Lead (Priya Sharma)
       - "Support" / "Customer" → Support Manager (Amit Patel)
       - "QA" / "Test" → QA Lead (Anjali Singh)
       - "Tech" / "Engineering" → Tech Lead (Vikram Reddy)
    
    → Query managers table by role
    
    → Assign manager to ticket
    
    → Send email notification
```

### Status Transition Flow

```
open
  
  → [AI Processing] → assigned
                         
                         → [Manager Action] → in_progress
                                                
                                                → [Resolution] → resolved
                                                                    
                                                                    → [Admin Action] → closed
  
  → [Direct Admin Action] → closed
```

---

##  Configuration Guide

### Environment Variables

Create a `.env` file in the project root:

```env
# OpenAI/CrewAI Configuration
OPENAI_API_KEY=sk-proj-...your-key-here...
LLM_MODEL=gpt-4o-mini

# Database Configuration - PostgreSQL (Neon)
DATABASE_URL=postgresql://user:password@host:port/database?sslmode=require

# Email Configuration (Optional - for notifications)
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email@gmail.com
SMTP_PASSWORD=your_app_password
EMAIL_FROM=your_email@gmail.com

# Streamlit Configuration
STREAMLIT_PORT=8501
ADMIN_PASSWORD=admin123
```

### Configuration Files

**config.py** - Main configuration
- Loads environment variables
- Defines ORG_CONTEXT (organization context for AI)
- Sets default values
- Configures database and email settings

**ORG_CONTEXT Structure:**
```python
{
    "product": "EuronSupport - AI-Powered Ticket Resolution System",
    "stack": {
        "backend": "FastAPI + PostgreSQL + Redis",
        "streaming": "HLS via CDN + DRM",
        "auth": "OTP provider + fallback provider",
        "payments": "Payment gateway + webhook processor",
        "observability": "Sentry + Datadog + CloudWatch"
    },
    "recent_changes": [...],
    "known_incidents": [...],
    "sla_targets": {...}
}
```

---

##  Setup Instructions

### Prerequisites

1. **Python 3.8+** installed
2. **PostgreSQL Database** (Neon or any PostgreSQL instance)
3. **OpenAI API Key**

### Step-by-Step Setup

#### 1. Clone/Navigate to Project
```bash
cd d:\crewai17thjangenaicer
```

#### 2. Install Dependencies
```bash
pip install -r requirements.txt
```

**Key Dependencies:**
- `crewai==1.8.1` - AI agent framework
- `openai~=1.83.0` - OpenAI API client
- `streamlit>=1.28.0` - Web UI framework
- `psycopg2-binary>=2.9.0` - PostgreSQL adapter
- `plotly>=5.0.0` - Interactive charts
- `pandas>=2.0.0` - Data manipulation
- `numpy<2.0.0` - Numerical computing (compatibility)
- `python-dotenv~=1.1.1` - Environment variable management

#### 3. Configure Environment
```bash
# Copy example file
cp .env.example .env

# Edit .env with your values
# - Add OpenAI API key
# - Add PostgreSQL connection string
# - (Optional) Add email credentials
```

#### 4. Initialize Database
The database will be automatically initialized on first run. Tables will be created if they don't exist.

#### 5. Start Applications

**Option A: Use Launcher**
```bash
python launcher.py
```

**Option B: Start Manually**
```bash
# Terminal 1 - User Interface
streamlit run user_app.py --server.port 8501

# Terminal 2 - Admin Dashboard
streamlit run admin_app.py --server.port 8502
```

#### 6. Access Applications
- **User Interface**: http://localhost:8501
- **Admin Dashboard**: http://localhost:8502 (Password: `admin123`)

---

##  Component Details

### Database Layer (database.py)

**Class: Database**

**Key Methods:**
- `init_database()`: Creates all tables and indexes
- `create_user()`: Creates new user
- `get_user_by_email()`: Retrieves user by email
- `get_user_tickets()`: Gets all tickets for a user
- `create_ticket()`: Creates new ticket
- `update_ticket()`: Updates ticket fields
- `get_all_tickets()`: Retrieves tickets with filters
- `get_managers()`: Gets all active managers
- `get_manager_by_role()`: Finds manager by role keyword
- `add_ticket_log()`: Adds activity log entry
- `save_agent_result()`: Saves AI agent output
- `get_past_incidents()`: Retrieves historical incidents
- `get_ticket_metrics()`: Calculates dashboard metrics

**Connection Management:**
- Uses `psycopg2` for PostgreSQL
- Context manager for connection handling
- Automatic transaction management
- RealDictCursor for dictionary-like results

### AI Processing Layer (agents.py + ticket_processor.py)

**agents.py - Agent Definitions**

**5 Specialized Agents:**

1. **Triage Lead** (Incident Triage Lead)
   - Role: Turn raw complaints into structured incidents
   - Analyzes context and past incidents
   - Generates risk hypotheses

2. **Support Analyst** (Customer Support Analyst)
   - Role: Summarize customer complaints
   - Classifies severity, priority, category
   - Outputs structured JSON

3. **SRE Analyst** (SRE / Infra Analyst)
   - Role: Analyze infra-level signals
   - Identifies metrics to check
   - Proposes mitigations

4. **Backend Analyst** (Backend & Data Analyst)
   - Role: Investigate backend/data causes
   - Identifies service components
   - Proposes code-level fixes

5. **Tech Lead** (Engineering Tech Lead)
   - Role: Synthesize findings
   - Recommends manager assignment
   - Provides action plan

**ticket_processor.py - Processing Orchestration**

**Class: TicketProcessor**

**Key Methods:**
- `process_ticket()`: Main processing function
- `_extract_classification()`: Extracts severity/priority/category
- `_extract_assignment_info()`: Extracts manager recommendation
- `_assign_manager()`: Assigns manager based on role

**Processing Steps:**
1. Create CrewAI crew with 5 agents
2. Execute crew with ticket details
3. Extract classification from results
4. Extract manager assignment recommendation
5. Assign appropriate manager
6. Update ticket with all information
7. Send email notifications

### User Interface (user_app.py)

**Features:**
- Ticket submission form
- User ticket tracking (all tickets for user)
- Real-time status updates
- Refresh button for resolved/closed tickets
- Activity log viewing
- Status indicators with emojis

**Tabs:**
1. **Raise New Ticket**: Submit new support tickets
2. **My Tickets**: View and track all user tickets

### Admin Interface (admin_app.py)

**Features:**
- Password-protected access
- Comprehensive metrics dashboard
- Interactive charts:
  - Pie chart: Status distribution
  - Bar chart: Severity distribution
  - Line chart: Daily ticket trends
- Ticket filtering (status, severity, manager)
- Ticket management actions
- AI processing results viewing
- Activity logs
- Refresh functionality

**Metrics Displayed:**
- Total tickets
- Tickets by status
- Tickets by severity
- Average resolution time
- Daily ticket creation trends

---

##  API & Data Flow

### Data Flow Diagram

```
User Input
    
    → user_app.py (Streamlit)
           
           → Database.create_ticket()
                   
                   → PostgreSQL (tickets table)
    
    → ticket_processor.py
           
           → agents.py (CrewAI)
                  
                  → OpenAI API (GPT-4o-mini)
                  
                  → Agent Results
           
           → Database.save_agent_result()
           
           → Extract Classification
           
           → Database.get_manager_by_role()
           
           → Database.update_ticket()
           
           → email_service.py
                   
                   → SMTP Server
    
    → admin_app.py (Streamlit)
            
            → Database.get_all_tickets()
            
            → Database.get_ticket_metrics()
            
            → Display with Plotly charts
```

### Key Data Flows

**1. Ticket Creation Flow**
```
User Form → user_app.py → Database.create_ticket() → PostgreSQL
```

**2. AI Processing Flow**
```
Ticket → ticket_processor.py → agents.py → CrewAI → OpenAI API
    → Agent Results → Database.save_agent_result() → PostgreSQL
```

**3. Manager Assignment Flow**
```
AI Recommendation → ticket_processor._assign_manager()
    → Database.get_manager_by_role() → Database.update_ticket()
    → email_service.send_ticket_assignment_notification()
```

**4. Metrics Retrieval Flow**
```
admin_app.py → Database.get_ticket_metrics()
    → PostgreSQL Queries → Aggregate Data → Plotly Charts
```

---

##  Security Considerations

1. **Environment Variables**: Sensitive data in `.env` file
2. **Password Protection**: Admin dashboard requires password
3. **SQL Injection Prevention**: Parameterized queries throughout
4. **Email Security**: SMTP with TLS/SSL
5. **Database Security**: Connection string with SSL mode

---

##  Performance Optimizations

1. **Database Indexes**: Created on frequently queried columns
2. **Connection Pooling**: PostgreSQL connection management
3. **Caching**: Streamlit session state for user data
4. **Lazy Loading**: Metrics calculated on-demand
5. **Efficient Queries**: Optimized SQL with proper joins

---

## 🧪 Testing

Run comprehensive tests:
```bash
python test_euronsupport.py
```

**Test Coverage:**
- Database connection
- Ticket creation
- AI processing
- Manager assignment
- Metrics retrieval
- User ticket retrieval

---

##  Troubleshooting

### Common Issues

1. **Database Connection Error**
   - Check DATABASE_URL in .env
   - Verify PostgreSQL is accessible
   - Check SSL mode settings

2. **OpenAI API Error**
   - Verify OPENAI_API_KEY in .env
   - Check API key validity
   - Verify account has credits

3. **Import Errors**
   - Run: `pip install -r requirements.txt`
   - Check Python version (3.8+)
   - Verify all dependencies installed

4. **NumPy Compatibility**
   - Ensure numpy<2.0.0 (for Plotly compatibility)
   - Run: `pip install "numpy<2.0.0"`

---

##  Maintenance

### Regular Tasks

1. **Monitor Database Size**: Clean old logs if needed
2. **Update Past Incidents**: Add resolved incidents for better AI context
3. **Review Metrics**: Check resolution times and trends
4. **Update Managers**: Keep manager list current
5. **Backup Database**: Regular PostgreSQL backups

### Adding New Managers

Edit `database.py` → `_insert_default_managers()` method:
```python
default_managers = [
    ("Name", "email@euronsupport.com", "Role", "Department", "Expertise"),
    # Add more managers here
]
```

---

##  Deployment Considerations

### Production Checklist

- [ ] Change ADMIN_PASSWORD in .env
- [ ] Configure proper SMTP credentials
- [ ] Set up database backups
- [ ] Configure SSL/TLS for database
- [ ] Set up monitoring and alerts
- [ ] Configure log rotation
- [ ] Set up reverse proxy (nginx/Apache)
- [ ] Configure firewall rules
- [ ] Set up CI/CD pipeline (optional)
- [ ] Document runbooks

### Scaling Considerations

- **Database**: PostgreSQL can scale with read replicas
- **Application**: Run multiple Streamlit instances behind load balancer
- **AI Processing**: Consider async processing queue for high volume
- **Caching**: Add Redis for frequently accessed data

---

##  Additional Resources

- **CrewAI Documentation**: https://docs.crewai.com
- **Streamlit Documentation**: https://docs.streamlit.io
- **PostgreSQL Documentation**: https://www.postgresql.org/docs
- **Neon Database**: https://neon.tech

---

**EuronSupport** - Built with  using CrewAI, Streamlit, and PostgreSQL
