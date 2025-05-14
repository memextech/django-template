# Django Web Application

A beautiful Django web application with Postgres database integration, magic-links user authentication and deployment to Render.com

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

1. Run the development server
   ```
   python manage.py runserver
   ```

2. Access the application at http://localhost:8000

## Deployment to Render.com

1. Create a free account on Render.com

2. Connect your GitHub repository

3. Create a new Blueprint and select your repository

4. Create a free account on MailerSend.com

4. Set MAILERSEND_API_KEY & DEFAULT_FROM_EMAIL env variables from MailerSend

## License

This project is licensed under the MIT License - see the LICENSE file for details.