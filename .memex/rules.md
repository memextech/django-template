# Django Website Development Guide

This document provides guidelines for developing Django websites with postgres database, magic-link authentication + email integration, and image handling using Unsplash. All deployed to Render.com.

## Project Structure

```
django_app/
├── .env                      # Environment variables (not in Git)
├── .gitignore                # Git ignore file
├── .memex/                   # Memex rules and guidelines
│   └── rules.md              # This file
├── build.sh                  # Deployment build script
├── implementation_plan.md    # Project implementation steps
├── manage.py                 # Django management script
├── DEPLOYMENT.md             # Deployment documentation
└── README.md                 # Project documentation
```

## Initial Setup

### 1. Create Project Directory

```bash
mkdir -p django_app
cd django_app
```

### 2. Set Up Virtual Environment 

```bash
python -m venv .venv
source .venv/bin/activate
pip install django-environ django django-sesame psycopg2-binary pillow gunicorn whitenoise
```

### 3. Initialize Django Project

```bash
django-admin startproject django-template-website .
python manage.py startapp django-template
```

### 4. Configure Git

```bash
git init
touch .gitignore
# Add standard Python/Django patterns to .gitignore
git add .
git commit -m "Initial commit"
```

## Configuration Guidelines

### Environment Variables

Create a `.env` file in the project root with these variables:

```
DEBUG=True
SECRET_KEY=your-secret-key-for-development
ALLOWED_HOSTS=localhost,127.0.0.1
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
# Use SQLite for local testing
# to connect to external render postgres 
# DATABASE_URL=postgresql://<external-url-to-render-postgres>
ALLOWED_HOSTS=localhost,127.0.0.1

# MailerSend configuration for direct API usage
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
MAILERSEND_API_KEY=
DEFAULT_FROM_EMAIL=noreply@<mailersend-test-domain>
```

### Settings.py Configuration

1. **Configure environment variables**:
   ```python
   import environ
   import os
   
   # Build paths
   BASE_DIR = Path(__file__).resolve().parent.parent
   
   # Initialize environment variables
   env = environ.Env()
   environ.Env.read_env(os.path.join(BASE_DIR, '.env'))
   
   # Get settings from environment
   SECRET_KEY = env('SECRET_KEY', default='django-insecure-dev-key')
   DEBUG = env.bool('DEBUG', default=False)
   ```

2. **Configure databases**:
   ```python
   # Use SQLite for development by default
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.sqlite3',
           'NAME': BASE_DIR / 'db.sqlite3',
       }
   }
   
   # If DATABASE_URL is defined (e.g., on Render), use that
   if 'DATABASE_URL' in os.environ:
       DATABASES = {
           'default': env.db('DATABASE_URL')
       }
   ```

3. **Configure media and static files**:
   ```python
   STATIC_URL = 'static/'
   STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
   STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
   
   MEDIA_URL = '/media/'
   MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
   
   # Create static and media directories
   os.makedirs(os.path.join(BASE_DIR, 'static'), exist_ok=True)
   os.makedirs(os.path.join(BASE_DIR, 'media'), exist_ok=True)
   ```

## Authentication Implementation

### Magic Link Authentication

1. **Install dependencies**:
   ```
   pip install django-sesame
   ```

2. **Configure settings.py**:
   ```python
   INSTALLED_APPS = [
       # ... other apps
       'sesame',
   ]
   
   MIDDLEWARE = [
       # ... other middleware
       'django.contrib.auth.middleware.AuthenticationMiddleware',
       'sesame.middleware.AuthenticationMiddleware',  # Must come after Django's auth middleware
   ]
   
   AUTHENTICATION_BACKENDS = [
       'django.contrib.auth.backends.ModelBackend',
       'sesame.backends.ModelBackend',
   ]
   ```

3. **Create a view to send magic links**:
   ```python
   from django.contrib.auth.models import User
   from sesame.utils import get_query_string
   
   def send_magic_link(request):
       if request.method == 'POST':
           email = request.POST.get('email')
           # Create user if doesn't exist (for easy onboarding)
           user, created = User.objects.get_or_create(
               email=email,
               defaults={
                   'username': email.split('@')[0],
                   'is_active': True
               }
           )
           
           # Generate login URL with magic link
           login_url = request.build_absolute_uri(
               reverse('home') + get_query_string(user)
           )
           
           # Send email with login_url
           send_magic_link_email(user, login_url)
   ```

## Email Configuration

### Development

Use Django's console backend for local development:
```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

### Production with MailerSend

1. **Create a robust direct mailer implementation**:
   ```python
   """
   Direct MailerSend integration for email sending.
   """
   from django.conf import settings
   from django.template.loader import render_to_string
   import datetime
   
   def send_mailersend_email(to_email, subject, html_content, text_content=None):
       """
       Send an email using MailerSend API directly.
       Falls back to console output if MAILERSEND_API_KEY is not set.
       
       Args:
           to_email: Recipient email
           subject: Email subject
           html_content: HTML body content
           text_content: Plain text body (optional)
       """
       # Console output for development
       if settings.EMAIL_BACKEND == 'django.core.mail.backends.console.EmailBackend':
           print("\n----- EMAIL -----")
           print(f"To: {to_email}")
           print(f"Subject: {subject}")
           print(f"HTML Content: {html_content}")
           print(f"Text Content: {text_content}")
           print("----- END EMAIL -----\n")
           return True
       
       # Use MailerSend for actual sending
       try:
           from mailersend import emails
           
           # Check if we have an API key
           if not hasattr(settings, 'MAILERSEND_API_KEY') or not settings.MAILERSEND_API_KEY:
               print("No MAILERSEND_API_KEY found in settings. Email not sent.")
               return False
           
           # Initialize the API client
           mailer = emails.NewEmail(settings.MAILERSEND_API_KEY)
           
           # Create the message structure
           message = {
               "from": {
                   "email": settings.DEFAULT_FROM_EMAIL,
                   "name": "Your App Name"
               },
               "to": [
                   {
                       "email": to_email
                   }
               ],
               "subject": subject,
               "html": html_content
           }
           
           # Add plain text version if provided
           if text_content:
               message["text"] = text_content
           
           # Send the email
           mailer.send(message)
           return True
           
       except ImportError:
           print("MailerSend package not installed. Please run 'pip install mailersend'")
           return False
       except Exception as e:
           print(f"Error sending email: {str(e)}")
           return False
   ```

2. **Create comprehensive email helper functions with fallbacks**:
   ```python
   def send_magic_link_email(user, login_url):
       """Send email with magic link for login"""
       subject = "Your Login Link"
       
       # Use template if available, otherwise fall back to inline HTML
       try:
           html_content = render_to_string('emails/magic_link.html', {
               'user': user,
               'login_url': login_url,
               'year': datetime.datetime.now().year
           })
       except:
           # Fallback HTML content
           html_content = f"""
           <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; border: 1px solid #eee; border-radius: 5px;">
               <div style="text-align: center; padding-bottom: 20px; border-bottom: 1px solid #eee;">
                   <h1 style="color: #007bff; margin: 0;">Your App Name</h1>
               </div>
               
               <div style="padding: 20px 0;">
                   <p>Hello {user.username},</p>
                   
                   <p>You requested a magic login link. Click the button below to securely log in:</p>
                   
                   <div style="text-align: center; margin: 25px 0;">
                       <a href="{login_url}" 
                          style="display: inline-block; background-color: #007bff; color: white; text-decoration: none; 
                                padding: 12px 24px; border-radius: 4px; font-weight: bold;">
                           Log In Securely
                       </a>
                   </div>
                   
                   <div style="background-color: #f8f9fa; border-left: 4px solid #007bff; padding: 15px; margin: 20px 0;">
                       <strong>Note:</strong> This link will expire soon and can only be used once.
                   </div>
                   
                   <p>If you didn't request this link, you can safely ignore this email.</p>
               </div>
               
               <div style="text-align: center; color: #999; font-size: 12px; margin-top: 30px; padding-top: 20px; border-top: 1px solid #eee;">
                   <p>&copy; {datetime.datetime.now().year} Your App Name. All rights reserved.</p>
               </div>
           </div>
           """
       
       # Plain text version
       text_content = f"""
       Hello {user.username},
       
       You requested a magic login link.
       
       To log in securely, please visit the following link:
       {login_url}
       
       Note: This link will expire soon and can only be used once.
       
       If you didn't request this link, you can safely ignore this email.
       
       © {datetime.datetime.now().year} Your App Name. All rights reserved.
       """
       
       return send_mailersend_email(user.email, subject, html_content, text_content)
   ```

3. **Configure environment variables with proper fallbacks**:
   ```
   # Development
   EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
   DEFAULT_FROM_EMAIL=noreply@example.com
   
   # Production
   # EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend  # Keep console for testing in production
   # MAILERSEND_API_KEY=your-api-key
   # DEFAULT_FROM_EMAIL=noreply@yourdomain.com  # Must be verified in MailerSend
   ```

4. **Add to settings.py**:
   ```python
   # Email configuration
   EMAIL_BACKEND = env('EMAIL_BACKEND', default='django.core.mail.backends.console.EmailBackend')
   DEFAULT_FROM_EMAIL = env('DEFAULT_FROM_EMAIL', default='noreply@example.com')
   MAILERSEND_API_KEY = env('MAILERSEND_API_KEY', default='')
   ```

## Image Handling

### Using Unsplash URLs (Recommended)

1. **Add image_url field to models**:
   ```python
   class Sample(models.Model):
       # ... other fields
       image = models.ImageField(upload_to='sample_images/', blank=True, null=True)
       image_url = models.URLField(max_length=500, blank=True, null=True)
   ```

2. **Update templates to prioritize URLs**:
   ```html
   {% if sample.image_url %}
       <img src="{{ sample.image_url }}" alt="{{ sample.title }}">
   {% elif sample.image %}
       <img src="{{ sample.image.url }}" alt="{{ sample.title }}">
   {% else %}
       <img src="https://images.unsplash.com/photo-default" alt="Default">
   {% endif %}
   ```

3. **Create forms with URL toggle**:
   ```python
   class SampleForm(forms.ModelForm):
       use_image_url = forms.BooleanField(required=False, initial=True)
       
       class Meta:
           model = Sample
           fields = ['title', 'description', 'image', 'image_url', ...]
   ```

## Deployment Guidelines

### Render.com Deployment

1. **Create build.sh**:
   ```bash
   #!/usr/bin/env bash
   pip install -r requirements.txt
   python manage.py collectstatic --noinput
   python manage.py migrate
   python manage.py setup_demo_data
   ```

2. **Create render.yaml**:
   ```yaml
   services:
     - type: web
       name: django-template-app
       env: python
       buildCommand: ./build.sh
       startCommand: gunicorn django-template_website.wsgi:application
   ```

3. **Add production settings**:
   ```python
   # Security settings for production
   if not DEBUG:
       SECURE_HSTS_SECONDS = 3600
       SECURE_SSL_REDIRECT = True
       SESSION_COOKIE_SECURE = True
       CSRF_COOKIE_SECURE = True
       SECURE_HSTS_INCLUDE_SUBDOMAINS = True
       SECURE_HSTS_PRELOAD = True
   ```

### Remote Data Management

Since the free tier of Render.com doesn't provide shell access, you should manage data by connecting to the production database from your local environment:

1. **Create a management command** for your data operations:
   ```python
   # data/management/commands/create_sample_data_with_urls.py
   from django.core.management.base import BaseCommand
   from sample.models import Sample
   from django.contrib.auth.models import User
   
   class Command(BaseCommand):
       help = 'Creates sample template using image URLs'
       
       def handle(self, *args, **options):
           # Samples creation logic here
           self.stdout.write(self.style.SUCCESS('Created sample data'))
   ```

2. **Access production data locally**:
   ```bash
   # Update .env file with Render's external database URL
   DATABASE_URL=postgres://username:password@host:port/database_name
   
   # Run your command locally
   python manage.py create_sample_template_with_urls
   ```

3. **Reset to local database** when done:
   ```bash
   # Comment out the DATABASE_URL line in .env
   # DATABASE_URL=postgres://username:password@host:port/database_name
   ```

## Common Pitfalls to Avoid

1. **Email Configuration**:
   - ✅ DO use direct MailerSend API integration instead of Django email backend
   - ❌ DON'T use custom email backend implementation unless you're certain it works
   - ✅ DO test email sending locally with console backend

2. **Image Handling**:
   - ✅ DO use direct Unsplash URLs when possible
   - ❌ DON'T rely on local file storage in production on Render (ephemeral)
   - ✅ DO provide both URL and file upload options for flexibility

3. **Database Connection**:
   - ✅ DO use a fallback to SQLite when DATABASE_URL is not defined
   - ❌ DON'T hardcode database credentials in settings.py
   - ✅ DO use environment variables for all sensitive information

4. **Authentication Flow**:
   - ✅ DO create automatic user registration with magic links
   - ❌ DON'T expose sensitive user information in URL parameters
   - ✅ DO set appropriate token expiry for magic links

5. **Performance**:
   - ✅ DO use WhiteNoise for static file handling
   - ❌ DON'T load large images without optimization
   - ✅ DO use separate URLs for different image sizes when needed

## IMPORTANT ADDITIONS

### Email Configuration - DO NOT USE ALTERNATIVES

ALWAYS use MailerSend for production email delivery with Direct API integration. No alternatives like SMTP services, SendGrid, or others should be implemented or suggested. The project specifically requires MailerSend integration.

Remind user that DEFAULT_FROM_EMAIL is correct domain from MailerSend configuration

### Email Implementation Best Practices

1. **Direct API Integration Only**:
   - Always use the MailerSend Python SDK directly (`from mailersend import emails`)
   - Never use Django's built-in email system or email backends for production
   - Implement a development fallback to console output

2. **Robust Error Handling**:
   - Add try/except blocks around API calls
   - Provide fallbacks for missing API keys
   - Include package import error handling
   - Log errors appropriately without exposing sensitive information

3. **Template Fallbacks**:
   - Always implement inline HTML fallback if template rendering fails
   - Use consistent styling between template and fallback HTML
   - Provide plain text alternatives for all emails
   - Include current year in copyright notices using datetime

4. **Configuration Pattern**:
   - Use console backend by default in development
   - Include explicit checks for EMAIL_BACKEND type
   - Support both settings methods (environment variables and settings attributes)
   - Never hard-code API keys or credentials



### Deployment - ALWAYS USE RENDERER BLUEPRINTS

When deploying to Render.com, follow the blueprints deployment process outlined in the guide. Do not use or suggest using manual feature. The deployment should be done by:

1. Creating a PostgreSQL database in render.yaml (or using existing one)
2. Creating a web service in render.yaml with the correct environment variables
3. Connecting the web service to the database using the DATABASE_URL environment variable

Follow the render.yaml configuration provided in this guide exactly, without suggesting alternative deployment methods.

### No SMTP Configuration

Do not include or suggest SMTP configuration options. The email system should exclusively use MailerSend's API integration as described in the guide.

---

_This guide was created by Memex to help developers avoid common pitfalls when building Django template websites with magic-link authentication, email integration, and Unsplash image handling._