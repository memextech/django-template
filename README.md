# Django Web Application

A beautiful Django web application

## Features

- User authentication with magic-links via django-sesame
- Data management (create, read, update, delete)
- Use Unsplash image urls instead of local images storage for image management
- Search functionality
- Responsive beautiful design

## Technology Stack

- Django web framework
- PostgreSQL database (SQLite for development)
- Bootstrap 5 for frontend
- Configured for deployment to Render.com

## Local Development Setup

1. Clone the repository
   ```
   git clone <repository-url>
   cd django_app
   ```

2. Create and activate a virtual environment
   ```
   python -m venv .venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```

3. Install dependencies
   ```
   pip install -r requirements.txt
   ```

4. Create a `.env` file in the project root with the following variables:
   ```
   DEBUG=True
   SECRET_KEY=your-secret-key
   ALLOWED_HOSTS=localhost,127.0.0.1
   # MailerSend configuration for direct API usage
   EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
   # Use SQLite for local testing
   # to connect to external render postgres 
   # DATABASE_URL=postgresql://<external-url-to-render-postgres>
   MAILERSEND_API_KEY=
   DEFAULT_FROM_EMAIL=noreply@<mailersend-test-domain>

   ```

5. Run the development server
   ```
   python manage.py runserver
   ```

6. Access the application at http://localhost:8000
   - Default admin user: admin
   - Default password: admin12345

## Deployment to Render.com (Recommended)

1. Create a free account on Render.com

2. Connect your GitHub repository

3. Create a new Web Service and select your repository

4. Use the following settings:
   - Build Command: `sh build.sh`
   - Start Command: `gunicorn django_template_website.wsgi:application`
   - Add the following environment variables:
     - `DEBUG=False`
     - `SECRET_KEY=your-secret-key`
     - `ALLOWED_HOSTS=.onrender.com`
     - `EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend`

5. Create a PostgreSQL database service on Render.com and link it to your app

6. Deploy the service

## License

This project is licensed under the MIT License - see the LICENSE file for details.