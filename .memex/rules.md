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

1. **Create a direct mailer implementation**:
   ```python
   # django-template-app/mailer.py
   from django.conf import settings
   from mailersend import emails
   
   def send_mailersend_email(to_email, subject, html_content, text_content=None):
       mailer = emails.NewEmail(settings.MAILERSEND_API_KEY)
       
       message = {
           "from": {
               "email": settings.DEFAULT_FROM_EMAIL,
               "name": "Your App Name"
           },
           "to": [{"email": to_email}],
           "subject": subject,
           "html": html_content,
           "text": text_content or html_content,
       }
       
       mailer.send(message)
   ```

2. **Create email helper functions**:
   ```python
   def send_magic_link_email(user, login_url):
       subject = "Your Login Link"
       html_content = render_to_string('emails/magic_link.html', {
           'user': user,
           'login_url': login_url
       })
       text_content = f"Click here to log in: {login_url}"
       
       send_mailersend_email(user.email, subject, html_content, text_content)
   ```

3. **Configure environment variables**:
   ```
   MAILERSEND_API_KEY=your-api-key
   DEFAULT_FROM_EMAIL=noreply@yourdomain.com
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



---

_This guide was created by Memex to help developers avoid common pitfalls when building Django template websites with magic-link authentication, email integration, and Unsplash image handling._