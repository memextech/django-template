# Deployment Instructions

This document provides detailed instructions for deploying the Django Website to Render.com and configuring all necessary services.

## Deployment to Render.com

1. **Create a Render Account**:
   - Sign up at [Render.com](https://render.com) (free tier available)
   - Connect your GitHub repository

2. **Create a PostgreSQL Database**:
   - In the Render dashboard, go to "New" → "PostgreSQL"
   - Choose a name (e.g., "django-template-app-db")
   - Select the Free plan
   - Note the "External Database URL" for the next step

3. **Create a Web Service**:
   - Go to "New" → "Web Service"
   - Connect to your GitHub repository
   - Configure settings:
     - Name: "django-template-app" (or your preferred name)
     - Environment: Python
     - Build Command: `sh build.sh`
     - Start Command: `gunicorn django_template_website.wsgi:application`
     - Plan: Free

4. **Configure Environment Variables**:
   - `DEBUG=False`
   - `SECRET_KEY=your-secure-random-string`
   - `ALLOWED_HOSTS=.onrender.com,your-custom-domain.com`
   - `DATABASE_URL=your-postgres-external-url`
   - `MAILERSEND_API_KEY=your-mailersend-api-key`
   - `DEFAULT_FROM_EMAIL=noreply@your-mailersend-domain.com`

5. **Deploy Your Application**:
   - Click "Create Web Service"
   - Your app will be deployed automatically

## Email Configuration

### Development Environment (Local)
- The application uses Django's console email backend for development:
  ```
  EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
  ```
- Magic links will be printed to the console instead of being sent

### Production Environment (Render.com)
1. **Set Up MailerSend**:
   - Create an account at [MailerSend](https://www.mailersend.com/) (free Hobby tier)
   - Create a sending domain or use the test domain (format: test-xxx.mlsender.net)
   - Generate an API key with sending permissions

2. **Configure Environment Variables on Render**:
   - `MAILERSEND_API_KEY=your-mailersend-api-key`
   - `DEFAULT_FROM_EMAIL=noreply@your-mailersend-domain.com`

3. **Email Features**:
   - Beautiful HTML emails for magic links
   - Direct API integration (no middleware)
   - Plain text fallback for all emails

## Image Handling

The application supports two methods for images:

1. **Unsplash Image URLs (Recommended)**:
   - Directly use images from Unsplash CDN
   - Format: `https://images.unsplash.com/photo-ID?w=800&auto=format&fit=crop`
   - No storage requirements
   - Better performance
   - User interface prioritizes this option

2. **File Uploads (Legacy)**:
   - Traditional file uploads stored on the server
   - Limited by Render's ephemeral filesystem
   - Not recommended for production use without cloud storage

## Managing Data in Production

### Important: Free Tier Limitations

The Render.com free tier does not provide shell access to your application. Instead, you can connect to the production database from your local machine:

1. **Get Database Connection String**:
   - In the Render dashboard, go to your PostgreSQL service 
   - Copy the "External Database URL" (contains username, password, host)

2. **Update Local Environment**:
   - Temporarily edit your local `.env` file:
   ```
   DATABASE_URL=postgres://your_render_postgres_external_url_here
   ```

3. **Run Management Commands Locally**:
   - Activate your virtual environment
   - Run commands against the production database:
   ```bash
   # Delete existing data
   python manage.py shell -c "from demo.models import Demo; Demo.objects.all().delete()"
   ```

4. **Reset Local Environment**:
   - After completing your tasks, comment out the DATABASE_URL line in your `.env` file
   - Return to local development with SQLite

### Warning

Be extremely careful when connecting to your production database. Incorrect commands can result in permanent data loss. Always verify what you're running and consider making a database backup first.

## Updating Your Deployment

1. Push changes to your GitHub repository
2. Render will automatically detect changes and redeploy
3. You can also trigger manual deploys in the Render dashboard

## Monitoring and Maintenance

1. **Database Backups**:
   - Render automatically backs up your PostgreSQL database daily
   - You can create manual backups from the database dashboard

2. **Logs and Monitoring**:
   - Access logs from the Render dashboard
   - Set up health checks for automatic monitoring

3. **Custom Domain**:
   - In your Render dashboard, go to your web service
   - Click "Settings" → "Custom Domain"
   - Follow instructions to set up your domain and SSL