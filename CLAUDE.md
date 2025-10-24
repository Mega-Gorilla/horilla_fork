# Horilla HRMS - Architecture Documentation

## Overview

**Horilla** is an open-source Human Resource Management System (HRMS) built with Django 4.2+. It's a comprehensive platform for managing HR processes including employee lifecycle, recruitment, payroll, attendance, leave management, performance reviews, and more.

**Key Stats:**
- **Framework:** Django 4.2.23
- **Database:** SQLite (dev), PostgreSQL (recommended)
- **Frontend:** Alpine.js, jQuery, Select2
- **API:** Django REST Framework with JWT authentication
- **Multi-tenancy:** Company-based multi-tenancy support
- **Internationalization:** 8+ languages supported

---

## Core Architecture

### 1. App Structure & Responsibilities

Horilla is organized as a modular Django project with 28+ apps, each handling specific domain concerns:

#### **Core/Base Apps**
- **base** - Foundation layer: companies, departments, job positions, shifts, work types, announcements, email configuration, multi-approval conditions, holidays, penalties
- **employee** - Employee master data, work information, bank details, disciplinary actions, family members, education/experience history
- **auth** - Authentication module (custom authentication backends)

#### **HR Process Apps**
- **recruitment** - Recruitment pipeline, candidates, stages, job openings, interviews, surveys, job offers
- **leave** - Leave types, requests, restrictions, compensatory leave allocation, available leave tracking
- **attendance** - Attendance logging, late/early tracking, overtime, attendance activities, work records
- **onboarding** - Employee onboarding tasks, checklists, feedback collection
- **offboarding** - Employee exit management, offboarding tasks, knowledge transfer
- **pms** - Performance Management: objectives, KPIs, feedback, appraisals
- **payroll** - Payslips, contracts, loans, allowances, deductions, reimbursements, salary structures
- **project** - Project management and allocation
- **helpdesk** - Ticketing system, FAQs, support management

#### **Extension/Plugin Apps**
- **horilla_api** - REST API layer with DRF serializers and viewsets
- **horilla_automations** - Workflow automation engine: mail automation, triggers, conditions
- **horilla_views** - Reusable class-based views, mixins, and view utilities
- **horilla_widgets** - Custom widgets and UI components
- **dynamic_fields** - Dynamic field injection system (add fields to any model at runtime)
- **horilla_audit** - Audit logging and change tracking (via django-auditlog)
- **horilla_documents** - Document management and requests
- **horilla_backup** - Backup/restore functionality
- **horilla_crumbs** - Breadcrumb navigation
- **horilla_ldap** - LDAP/Active Directory integration
- **accessibility** - Accessibility features for users with disabilities

#### **Utility Apps**
- **asset** - Asset management and assignment tracking
- **biometric** - Biometric device integration
- **facedetection** - Face recognition for attendance
- **geofencing** - Location-based attendance verification
- **notifications** - Real-time notifications system
- **outlook_auth** - Microsoft Outlook/Microsoft 365 integration
- **report** - Custom reporting and analytics

---

## Key Architectural Patterns

### 2. Base Model System (Inherited Functionality)

#### **HorillaModel** (`horilla/models.py`)
All business-logic models inherit from `HorillaModel`, which provides:

```python
class HorillaModel(models.Model):
    # Timestamp tracking
    created_at = DateTimeField(auto_now_add=True)
    created_by = ForeignKey(User, editable=False)
    modified_by = ForeignKey(User, editable=False)
    
    # Audit logging integration
    horilla_history = AuditlogHistoryField()
    
    # Active/inactive status
    is_active = BooleanField(default=True)
    
    # Automatic audit logging on save
    def save(self, *args, **kwargs):
        # Auto-set created_by and modified_by from request context
        request = getattr(_thread_locals, "request", None)
        if request and not self.pk:
            self.created_by = request.user
        if request:
            self.modified_by = request.user
        super().save(*args, **kwargs)
    
    # XSS protection
    def clean_fields(self, exclude=None):
        # Detects XSS patterns in text fields
        pass
    
    # Helper methods
    @classmethod
    def find(cls, object_id)
    @classmethod
    def activate_deactivate(cls, object_id)
    @classmethod
    def get_verbose_name_related_field(cls, field_path)
```

**Key Features:**
- **Automatic audit trails** via `django-auditlog`
- **XSS detection** for text/char fields with exemptions
- **User tracking** (created_by/modified_by via request middleware)
- **Soft delete support** (is_active flag)

### 3. Multi-Company/Multi-Tenancy Pattern

#### **HorillaCompanyManager** (`base/horilla_company_manager.py`)

A custom QuerySet manager that filters all queries based on selected company from session:

```python
class HorillaCompanyManager(models.Manager):
    def __init__(self, related_company_field=None):
        # Allows custom related field path: 
        # "employee_id__employee_work_info__company_id"
        self.related_company_field = related_company_field
    
    def get_queryset(self):
        # Auto-filters by selected company from request.session
        # Can be set to "all" for super-admin view
        request = getattr(_thread_locals, "request", None)
        selected_company = request.session.get("selected_company")
        queryset = queryset.filter(model.company_filter)
        return queryset
    
    def all(self):
        # Extra logic: filters by is_active for Employee model
        # Filters by related ForeignKey is_active
        pass
    
    def entire(self):
        # Bypasses company filtering - returns ALL data
        pass
```

**Implementation:**
- Models declare: `objects = HorillaCompanyManager(related_company_field="path")`
- Middleware (`CompanyMiddleware`) adds `company_filter` Q object to models dynamically
- All queries automatically scoped to user's company unless superuser selects "all"

#### **CompanyMiddleware** (`base/middleware.py`)

Three middleware layers:
1. **CompanyMiddleware** - Manages multi-company filtering
2. **ForcePasswordChangeMiddleware** - Redirects new employees to password change
3. **TwoFactorAuthMiddleware** - Enforces 2FA if enabled

### 4. Request Context & Thread-Local Storage

#### **Thread-Local Storage** (`horilla/horilla_middlewares.py`)

```python
_thread_locals = threading.local()

class HorillaRequestMiddleware:
    def __call__(self, request):
        _thread_locals.request = request
        # Available in any model save/signal without passing request
```

**Usage:**
- `request = getattr(_thread_locals, "request", None)`
- Used in model saves, signals, and methods to access current user/company
- Avoids passing request through multiple function layers

### 5. Custom Authentication & Authorization

#### **Email-based Authentication** (`base/backends.py`)

- Custom authentication backend supporting email login (not just username)
- Dynamic email configuration per company
- Support for LDAP/Microsoft authentication integration
- Two-factor authentication (OTP via email)

#### **Permission & Role System**

- Uses Django's built-in `User.groups` and permissions
- Custom decorators for permission checking:
  - `@shift_request_change_permission` - Checks shift request authorization
  - `@work_type_request_change_permission` - Checks work type request authorization
- Reporting hierarchy respected: managers can approve subordinate requests

### 6. Email Automation System

#### **Dynamic Email Configuration** (`base/models.py` - DynamicEmailConfiguration)

- Per-company SMTP settings (host, port, username, password, TLS/SSL)
- Falls back to primary configuration if company-specific not set
- Email logging (subject, body, to, status)
- Support for HTML email templates with dynamic variables

#### **Configured Email Backend** (`base/backends.py` - ConfiguredEmailBackend`)

```python
class ConfiguredEmailBackend(BACKEND_CLASS):
    def send_messages(self, email_messages):
        # Sends email via configured SMTP
        # Logs all sent/failed emails to EmailLog model
        # Supports dynamic display name from triggering user
```

### 7. Workflow Automation Engine

#### **MailAutomation** (`horilla_automations/models.py`)

Trigger-based automation system:

```python
class MailAutomation(HorillaModel):
    title = CharField(unique=True)
    model = CharField()  # Which model to monitor
    trigger = CharField(choices=['on_create', 'on_update', 'on_delete'])
    condition = TextField()  # Python eval'd condition
    condition_querystring = TextField()  # Query-based filtering
    mail_template = ForeignKey(HorillaMailTemplate)
    delivery_channel = CharField(choices=['email', 'notification', 'both'])
    also_sent_to = ManyToMany(Employee)  # CC recipients
```

**Flow:**
1. Create automation rule with trigger, model, and conditions
2. Signals monitor database changes (post_save/post_delete)
3. On match, send email + notification
4. Support for templating: `{{ instance.field_name }}`, `{{ self }}` (triggering user)

### 8. Dynamic Fields System

#### **DynamicField** (`dynamic_fields/models.py`)

Allows adding fields to models at runtime without migrations:

```python
class DynamicField(models.Model):
    model = CharField()  # Target model name
    field_name = CharField()  # Generated from verbose_name
    type = CharField(choices=['CharField', 'IntegerField', 'TextField', 'DateField', 'FileField'])
    verbose_name = CharField()
    choices = ManyToMany(Choice)
    is_required = BooleanField()
```

**Implementation:**
- Uses Django's migration system to add columns
- Fields mapped to Django field types
- Support for choice fields
- Can be added/removed dynamically

---

## REST API Layer

### 9. DRF Integration (`horilla_api/`)

#### **Directory Structure**
```
horilla_api/
├── api_serializers/     # ModelSerializers organized by app
│   ├── auth/
│   ├── employee/        # EmployeeSerializer, EmployeeWorkInfoSerializer, etc.
│   ├── payroll/         # PayslipSerializer, ContractSerializer, etc.
│   ├── leave/
│   ├── attendance/
│   └── ...
├── api_views/           # ViewSets organized by app
│   ├── auth/
│   ├── employee/        # EmployeeViewSet with CRUD + filters
│   └── ...
├── api_urls/            # Router registrations
├── api_filters/         # Custom DRF FilterSet classes
├── api_decorators/      # Custom decorators (permission checks, logging)
└── api_methods/         # Helper methods for serialization
```

#### **ViewSet Pattern**

```python
class EmployeeViewSet(ModelViewSet):
    queryset = Employee.objects.all()
    serializer_class = EmployeeSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = EmployeeFilter
    search_fields = ['employee_first_name', 'employee_last_name', 'email']
    ordering_fields = ['employee_first_name', 'created_at']
    
    # Auto-handles GET /employees/, POST, GET /employees/{id}/, PUT, DELETE
    # Company filtering applied via queryset
```

#### **Authentication**
- JWT tokens (via `djangorestframework-simplejwt`)
- Company-scoped access (only see data from selected company)
- Same permission system as web interface

---

## View Layer

### 10. Class-Based Views (CBV) & Mixins

#### **horilla_views/cbv_methods.py**

Provides reusable CBV utilities:

```python
# Field widget mapping
FIELD_WIDGET_MAP = {
    CharField: TextInput(attrs={"class": "oh-input w-100"}),
    DateField: DateInput(attrs={"type": "date"}),
    ForeignKey: Select(attrs={"class": "oh-select oh-select-2"}),
    # ... etc
}

# Form field mapping
MODEL_FORM_FIELD_MAP = {
    CharField: forms.CharField,
    IntegerField: forms.IntegerField,
    # ... etc
}

# Helpers
def render_template(template_path, context):
    """Render HTML template to string (used in AJAX responses)"""
    
def login_required(view_func):
    """Permission check decorator"""
    
# ... More helper functions
```

#### **Generic Views** (`horilla_views/generic/`)

Pre-built views for common patterns:
- List views with filtering and pagination
- Detail views with audit history
- Create/Update forms with dynamic field handling
- Bulk operations (export, delete, update)
- Async views for long-running operations

---

## Frontend Stack

### 11. Frontend Technologies

#### **Core Libraries** (`package.json`)
- **Alpine.js 3.10+** - Lightweight reactive framework (replacing Vue/React)
- **jQuery 3.6+** - DOM manipulation and AJAX
- **Select2 4.1+** - Advanced select/multi-select dropdowns
- **JS Datepicker 5.18+** - Calendar date picker
- **jQuery UI** - UI interactions (draggable, sortable, etc.)
- **Ionicons 7.1+** - Icon library

#### **Build Tool**
- **Laravel Mix 6** - Node-based asset compilation (CSS/JS bundling)

#### **CSS Framework**
- Custom "Horilla" CSS classes: `oh-input`, `oh-select`, `oh-alert`, `oh-btn`
- Responsive grid system
- Dark mode support

#### **Key UI Patterns**
- Modal dialogs with Alpine.js `x-show`
- AJAX form submissions with inline validation
- Real-time search with debouncing
- Dynamic table rendering with sorting/filtering
- Drag-and-drop for assignments

### 12. Template System

#### **Template Organization**
```
templates/
├── base.html            # Main layout (navigation, sidebar)
├── includes/            # Reusable snippets
│   ├── pagination.html
│   ├── filters.html
│   └── modals.html
├── [app_name]/
│   ├── [model_name]_list.html
│   ├── [model_name]_detail.html
│   └── [model_name]_form.html
└── ...
```

#### **Custom Template Tags** (`horilla_views/templatetags/`)
- `render_badge(instance)` - Renders status badges
- `getattribute(object, attr_name)` - Safely access nested attributes
- `paginate(queryset)` - Apply pagination with custom count

---

## Database & ORM

### 13. Model Relationships

#### **Key Models & Dependencies**

```
Company (HorillaModel)
├── Department (M2M)
│   └── JobPosition (FK to Department)
│       └── JobRole (FK to JobPosition)
├── Employee (HorillaModel)
│   ├── EmployeeWorkInformation
│   │   ├── Company (FK)
│   │   ├── Department (FK)
│   │   ├── JobPosition (FK)
│   │   ├── ReportingManager (FK to Employee)
│   │   └── WorkType (FK)
│   ├── EmployeeShift (M2M)
│   ├── AvailableLeave (FK to LeaveType)
│   ├── Discipline Actions (FK)
│   └── PenaltyAccounts (FK)
├── EmployeeShift (HorillaModel)
│   └── EmployeeShiftSchedule (FK + day schedule)
└── LeaveRequest (HorillaModel)
    ├── Employee (FK)
    ├── LeaveType (FK)
    └── LeaveAllocationRequest (FK)
```

#### **Query Optimization**
- Uses `select_related()` for FK
- Uses `prefetch_related()` for M2M and reverse relations
- Caches company models in `CACHE_KEY = "horilla_company_models_cache_key"`
- `HorillaCompanyManager.entire()` bypasses filtering when needed

### 14. Audit Logging

**Django-auditlog Integration:**
- Every `HorillaModel` subclass automatically tracked
- Change history stored in `AuditLog` table
- Fields tracked: what changed, who changed, when
- Used for compliance and debugging

---

## Security Features

### 15. Built-in Security Layers

#### **XSS Protection**
- `HorillaModel.clean_fields()` detects XSS patterns
- Regex patterns match: `<script>`, `javascript:`, event handlers, dangerous HTML
- Configurable exempt fields via `xss_exempt_fields`

#### **CSRF Protection**
- Django's CSRF middleware active
- CSRF tokens in all forms
- Trusted origins configured via env

#### **SQL Injection**
- Django ORM parameterized queries (no raw SQL except in migrations)

#### **Authentication**
- Password hashing (Django's default PBKDF2)
- Permission-based authorization on all views
- Custom permission system for approval workflows
- Two-factor authentication support (OTP via email)

#### **Rate Limiting / Fail2Ban**
```python
class Fail2BanMiddleware:
    # Tracks failed login attempts per session
    # Bans IP/session after N attempts for M seconds
    # Configurable via settings
```

#### **IP Whitelisting for Attendance**
```python
class AttendanceAllowedIP:
    # Optional: restrict attendance marking to specific IPs/networks
    is_enabled = BooleanField()
    additional_data = JSONField(default={"allowed_ips": []})
```

---

## Signal System & Hooks

### 16. Event-Driven Architecture

#### **Key Signals** (`base/signals.py`)

1. **Post-Save Signals**
   ```python
   @receiver(post_save, sender=PenaltyAccounts)
   def create_deduction_cutleave_from_penalty(sender, instance, created, **kwargs):
       # Auto-create payroll Deduction from penalty
       # Deduct from available leave
   ```

2. **M2M Changed Signals**
   ```python
   @receiver(m2m_changed, sender=Announcement.employees.through)
   def filtered_employees(sender, instance, action, **kwargs):
       # Update Announcement.filtered_employees when employees/departments/positions change
   ```

3. **Login Failure Signals**
   ```python
   @receiver(user_login_failed)
   def log_login_failed(sender, credentials, request, **kwargs):
       # Log failed login attempts
       # Implement Fail2Ban banning logic
   ```

#### **Bulk Update Signals** (`base/horilla_company_manager.py`)
```python
pre_bulk_update.send(sender=self.model, queryset=self, args=args, kwargs=kwargs)
# ... bulk update ...
post_bulk_update.send(sender=self.model, queryset=self, args=args, kwargs=kwargs)
```

#### **Other Signals**
- `onboarding/signals.py` - Onboarding task automation
- `payroll/signals.py` - Payslip finalization, salary calculation
- `leave/signals.py` - Leave allocation, restriction enforcement
- `attendance/signals.py` - Attendance validation, late/early detection
- `dynamic_fields/signals.py` - Dynamic field creation/removal

---

## Configuration & Settings

### 17. Django Settings Structure

#### **Main Settings** (`horilla/settings.py`)
```python
INSTALLED_APPS = [
    # Django core
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    # Third-party
    'rest_framework',
    'django_filters',
    'simple_history',
    'corsheaders',
    'django_apscheduler',
    # Horilla apps
    'base',
    'employee',
    'recruitment',
    'leave',
    'pms',
    'onboarding',
    'asset',
    'attendance',
    'payroll',
    'horilla_api',
    'horilla_automations',
    'horilla_views',
    'horilla_widgets',
    'dynamic_fields',
    'horilla_audit',
    'horilla_documents',
    # ... more
]

MIDDLEWARE = [
    'base.middleware.CompanyMiddleware',
    'base.middleware.ForcePasswordChangeMiddleware',
    'base.middleware.TwoFactorAuthMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    'simple_history.middleware.HistoryRequestMiddleware',
    # ... more
]
```

#### **Horilla Settings** (`horilla/horilla_settings.py`)
```python
HORILLA_DATE_FORMATS = {
    "DD/MM/YYYY": "%d/%m/%Y",
    "YYYY-MM-DD": "%Y-%m-%d",
    # ... 10+ formats
}

HORILLA_TIME_FORMATS = {
    "hh:mm A": "%I:%M %p",  # 12-hour
    "HH:mm": "%H:%M",        # 24-hour
}

APPS = ["base", "employee", "horilla_documents", "horilla_automations"]

NO_PERMISSION_MODALS = [
    "historicalbonuspoint",  # Models that skip permission checks
    "attachment",
    # ... many more
]
```

#### **Environment Variables** (`.env`)
```
DEBUG=True
SECRET_KEY=...
DB_ENGINE=django.db.backends.postgresql
DB_NAME=horilla_main
DB_USER=horilla
DB_PASSWORD=...
DB_HOST=localhost
DB_PORT=5432

# Email config
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=...
EMAIL_HOST_PASSWORD=...
DEFAULT_FROM_EMAIL=...

# 2FA
TWO_FACTORS_AUTHENTICATION=True

# AWS S3 (optional)
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_STORAGE_BUCKET_NAME=...

# LDAP (optional)
LDAP_SERVER=ldap://127.0.0.1:389
BIND_DN=cn=admin,dc=horilla,dc=com
BIND_PASSWORD=...
```

---

## Data Flow & Request Lifecycle

### 18. Typical Request Journey

**Web Request (HTML Form):**
```
1. Request arrives → Django routing
2. CompanyMiddleware:
   - Extracts selected_company from session
   - Adds company_filter Q object to models
3. View processing:
   - Access request.user.employee_get for current employee
   - Access request.session['selected_company'] for company scoping
   - Queries auto-filtered by HorillaCompanyManager
4. Template rendering:
   - Alpine.js hydration for interactive components
   - Select2 initialization for dropdowns
5. Response with CSRF token for next form
```

**API Request (REST):**
```
1. JWT token validation
2. Authenticate user and set _thread_locals.request
3. Viewset processes request:
   - Queryset auto-filtered by company
   - DRF serializer validates data
4. Signal fires (post_save, post_delete)
5. JSON response with proper status codes
```

**Automation Trigger:**
```
1. Model instance saved → post_save signal
2. MailAutomation records checked:
   - Does trigger match (on_create/on_update/on_delete)?
   - Do conditions evaluate True?
3. If match:
   - Render mail template with instance context
   - Send email via ConfiguredEmailBackend
   - Create notification
   - Log to EmailLog
```

---

## Internationalization (i18n)

### 19. Multi-Language Support

**Supported Languages:**
- English (US)
- German (Deutsch)
- Spanish (Español)
- French (Français)
- Arabic (عربى)
- Portuguese (Brasil)
- Chinese (Simplified & Traditional)
- Italian

**Implementation:**
- `USE_I18N = True` in settings
- Strings wrapped with `gettext_lazy()` or `_()` function
- Django's `django.middleware.locale.LocaleMiddleware` detects user language
- Locale files in `horilla/locale/[lang_code]/`
- Frontend uses `django-i18n-js` for JavaScript translations

---

## Testing Approach

### 20. Test Structure

Each app has `tests.py` with test cases:
```
[app_name]/tests.py
├── Test models (validation, save logic)
├── Test views (permissions, filters, pagination)
├── Test forms (field validation)
├── Test signals (automation triggers)
└── Test API (serialization, endpoints)
```

**Testing Tools:**
- Django's built-in TestCase (with database rollback)
- Factory Boy (if used) for test data generation
- Mock/patch for external dependencies

**Test Execution:**
```bash
python manage.py test                    # All tests
python manage.py test employee.tests     # Specific app
python manage.py test employee.tests.EmployeeTestCase  # Specific test class
```

---

## File Upload & Media Management

### 21. File Handling

#### **Upload Paths**
```python
def upload_path(instance, filename):
    """Generates: {app}/{model}/{field}/{unique-name}.ext"""
    return f"{app_label}/{model_name}/{field_name}/{slugify(base)}-{uuid4().hex[:8]}.{ext}"
```

#### **Storage Options**
- **Local:** FileSystemStorage → `media/` directory
- **AWS S3:** Via `storages` package (configurable)
- **Google Cloud:** Via Google Cloud Storage backend

#### **Custom FieldFile.url Property**
```python
# Extended with custom property:
FieldFile.url  # Returns reverse('404') if file not found
```

---

## Performance Optimization

### 22. Performance Patterns

#### **Query Optimization**
- `HorillaCompanyManager` adds distinct() when duplicates detected
- Select/prefetch related used throughout models
- Pagination default: 20-50 items per page (user-configurable)
- DynamicPagination model stores user's preference

#### **Caching**
- Company models cached: `CACHE_KEY = "horilla_company_models_cache_key"`
- Email configuration cached per request
- Display name for emails cached in Django cache

#### **Async Operations**
- Task scheduler via `django-apscheduler`
- Long-running operations (payroll, report generation) async
- Thread-based operations for biometric device polling

#### **Static Files**
- WhiteNoise middleware for efficient static serving
- CSS/JS bundling via Laravel Mix
- Compressed static files storage

---

## Deployment Considerations

### 23. Production Setup

#### **Security Checklist**
```python
# Recommended settings (automatic when DEBUG=False)
SECURE_BROWSER_XSS_FILTER = True
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

#### **Database**
- PostgreSQL recommended over SQLite
- Connection pooling via psycopg2
- Backup strategy for attached media

#### **Email**
- Configure `DynamicEmailConfiguration` per company
- Use application passwords (not main account password)
- Implement SPF/DKIM/DMARC records

#### **LDAP/OAuth Integration**
- `horilla_ldap` for Active Directory
- `django-microsoft-auth` for Azure AD
- `outlook_auth` for Outlook calendar integration

#### **Monitoring & Logging**
- Django error logging to file/email
- Fail2Ban security logging
- EmailLog table for audit trail
- Auditlog history for data changes

---

## Common Development Patterns

### 24. Adding a New Feature

**Step 1: Create Model**
```python
# myapp/models.py
class MyModel(HorillaModel):
    name = CharField(max_length=100)
    company_id = ForeignKey(Company, on_delete=models.PROTECT)
    objects = HorillaCompanyManager()
```

**Step 2: Create Forms**
```python
# myapp/forms.py
from django import forms
from .models import MyModel

class MyModelForm(forms.ModelForm):
    class Meta:
        model = MyModel
        fields = '__all__'
```

**Step 3: Create Views**
```python
# myapp/views.py
from horilla_views.generic import ListView, CreateView

class MyModelListView(ListView):
    model = MyModel
    paginate_by = 20
    template_name = 'myapp/mymodel_list.html'
    filter_class = MyModelFilter

class MyModelCreateView(CreateView):
    model = MyModel
    form_class = MyModelForm
    template_name = 'myapp/mymodel_form.html'
    success_url = reverse_lazy('mymodel-list')
```

**Step 4: Add URLs**
```python
# myapp/urls.py
from django.urls import path
from .views import MyModelListView, MyModelCreateView

urlpatterns = [
    path('', MyModelListView.as_view(), name='mymodel-list'),
    path('create/', MyModelCreateView.as_view(), name='mymodel-create'),
]
```

**Step 5: Create Templates**
```html
<!-- myapp/templates/mymodel_list.html -->
{% extends 'base.html' %}
{% block content %}
<div class="oh-card">
    <table>
        {% for obj in object_list %}
        <tr>
            <td>{{ obj.name }}</td>
            <td>{{ obj.created_at }}</td>
        </tr>
        {% endfor %}
    </table>
</div>
{% endblock %}
```

**Step 6: Create Serializer (for API)**
```python
# horilla_api/api_serializers/myapp/serializers.py
from rest_framework import serializers
from myapp.models import MyModel

class MyModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = MyModel
        fields = '__all__'
```

**Step 7: Create ViewSet (for API)**
```python
# horilla_api/api_views/myapp/views.py
from rest_framework.viewsets import ModelViewSet
from .serializers import MyModelSerializer
from myapp.models import MyModel

class MyModelViewSet(ModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
```

**Step 8: Register ViewSet**
```python
# horilla_api/api_urls/myapp/urls.py
from rest_framework.routers import DefaultRouter
from ..views import MyModelViewSet

router = DefaultRouter()
router.register('mymodel', MyModelViewSet)
```

---

## Troubleshooting Common Issues

### 25. Common Problems & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "No employee related to this user" | Employee not linked to User | Create Employee with employee_user_id |
| Company filter not working | Not using HorillaCompanyManager | Add: `objects = HorillaCompanyManager()` |
| Email not sending | DynamicEmailConfiguration not set | Create in admin or set EMAIL_HOST env |
| Permissions not enforcing | No permission check in view | Add `@permission_required()` or check in form |
| Slow queries | N+1 problem | Add `select_related()` / `prefetch_related()` |
| XSS detected | Text field contains HTML/script | Set `xss_exempt_fields = ['field_name']` |
| Dynamic fields not showing | Not registered in forms | Call `dynamic_fields.reload_queryset(form.fields)` |

---

## Resources & Entry Points

### 26. Key Files to Understand First

**For Backend Developers:**
1. `/horilla/settings.py` - Configuration and installed apps
2. `/horilla/models.py` - HorillaModel base class
3. `/base/models.py` - Company, Department, Employee, etc.
4. `/base/middleware.py` - Multi-tenancy enforcement
5. `/base/horilla_company_manager.py` - Query filtering
6. `/base/methods.py` - Utility functions (helpers for views)
7. `[app_name]/models.py` - Domain models
8. `[app_name]/views.py` - HTTP request handlers
9. `[app_name]/urls.py` - URL routing

**For Frontend Developers:**
1. `/templates/base.html` - Main layout
2. `/static/js/` - Alpine.js components, jQuery plugins
3. `[app_name]/templates/[model_name]_list.html` - List views
4. `/static/css/` - Styling and theme

**For API Developers:**
1. `/horilla_api/api_serializers/` - Data serialization
2. `/horilla_api/api_views/` - API endpoints
3. `/horilla_api/api_filters/` - Query filtering
4. `/horilla/rest_conf.py` - DRF configuration

**For Automation/Integration:**
1. `/horilla_automations/models.py` - Automation rules
2. `/horilla_automations/signals.py` - Automation triggers
3. `/base/backends.py` - Email backend configuration
4. `/dynamic_fields/models.py` - Dynamic field system

---

## Summary of Key Takeaways

1. **Multi-Tenancy:** Every query is auto-filtered by company via HorillaCompanyManager
2. **Audit Trail:** All changes tracked automatically via HorillaModel + django-auditlog
3. **Request Context:** Thread-local storage avoids passing request through layers
4. **Automation:** Event-driven via signals (post_save, post_delete, m2m_changed)
5. **API First:** REST API via DRF follows same patterns as web views
6. **Security:** XSS protection, CSRF tokens, permission system, 2FA support
7. **Extensibility:** Dynamic fields, custom widgets, plugin apps, template tags
8. **Modularity:** 28+ independent Django apps, each handling domain concerns
9. **Frontend:** Alpine.js for reactivity + jQuery for DOM manipulation + Select2 for advanced selects
10. **Performance:** Caching, query optimization, pagination, async operations

This architecture scales from single company to large enterprises with multiple companies, departments, and complex approval workflows.

