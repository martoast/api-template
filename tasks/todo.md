# Laravel API Boilerplate Setup Plan

This document outlines the exact steps to recreate this Laravel API architecture for new projects.

---

## Phase 1: Initial Laravel Project Setup

- [x] Create new Laravel project: `composer create-project laravel/laravel project-name`
- [x] Initialize git repository: `git init`
- [x] Create `.gitignore` with Laravel defaults

---

## Phase 2: Docker & Laravel Sail Configuration

- [x] Install Laravel Sail: `composer require laravel/sail --dev`
- [x] Publish Sail files: `php artisan sail:install` (select MySQL)
- [x] Create custom `Dockerfile` with:
  - Ubuntu 22.04 base
  - PHP 8.3 with extensions (bcmath, curl, gd, intl, mbstring, mysql, pgsql, redis, soap, xml, zip, etc.)
  - Node.js 20 with npm/pnpm/bun/yarn
  - Composer
  - Supervisor for process management
  - Custom `sail` user (UID 1337)
- [x] Create `supervisord.conf` with three processes:
  - PHP artisan serve (port 80)
  - Queue worker (`queue:work --queue=high,default`)
  - Scheduler (`schedule:run` every 60 seconds)
- [x] Create `start.sh` script that:
  - Runs migrations before starting supervisor
  - Starts supervisord in foreground
- [x] Update `docker-compose.yml`:
  - Configure laravel.test service with custom Dockerfile
  - Set up MySQL 8.0 service with health check
  - Configure networking (sail bridge network)
  - Set port mappings (APP_PORT for app, 3306 for MySQL)

---

## Phase 3: Authentication Setup (Laravel Fortify + Sanctum)

- [x] Install Fortify: `composer require laravel/fortify`
- [x] Install Sanctum: `composer require laravel/sanctum`
- [x] Publish Fortify config: `php artisan vendor:publish --provider="Laravel\Fortify\FortifyServiceProvider"`
- [x] Publish Sanctum config: `php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"`
- [x] Create `FortifyServiceProvider` with:
  - User creation action registration
  - Password update/reset actions
  - Rate limiting for login (5/min by email+IP)
  - Rate limiting for two-factor (5/min)
- [x] Configure `config/fortify.php`:
  - Set guard to `web`
  - Enable features: registration, password reset, email verification, profile updates
  - Set routes prefix to `/auth`
  - Disable views (API-only mode)
- [x] Configure `config/sanctum.php`:
  - Set stateful domains (localhost, localhost:3000, app URL variants)
  - Configure guard as `web`
- [x] Create Fortify Actions in `app/Actions/Fortify/`:
  - `CreateNewUser.php`
  - `UpdateUserProfileInformation.php`
  - `UpdateUserPassword.php`
  - `ResetUserPassword.php`
  - `PasswordValidationRules.php` (trait)
- [x] Register `FortifyServiceProvider` in `bootstrap/providers.php`

---

## Phase 4: Application Bootstrap Configuration

- [x] Configure `bootstrap/app.php`:
  - Set API routes from `routes/api.php` with `/` prefix (no `/api/` prefix)
  - Add health check at `/up`
  - Disable CSRF for webhook routes (`webhooks/*`, `stripe/*`)
  - Add global `JsonResponse` middleware
  - Enable stateful API (Sanctum)
  - Register custom middleware aliases (e.g., `admin`)
- [x] Update `AppServiceProvider`:
  - Use `CarbonImmutable` for date handling
  - Customize password reset email URLs for SPA

---

## Phase 5: Custom Middleware

- [x] Create `app/Http/Middleware/JsonResponse.php`:
  - Forces `Accept: application/json` header on all requests
- [x] Create `app/Http/Middleware/AdminMiddleware.php`:
  - Checks user `isAdmin()` method
  - Returns 403 if not admin
- [x] Register middleware in `bootstrap/app.php`

---

## Phase 6: User Model Configuration

- [x] Update `app/Models/User.php` with:
  - Fillable fields: name, email, password, phone, preferred_language, address fields, provider, role, user_type, registration_source
  - Casts: email_verified_at (datetime), password (hashed), registration_source (array)
  - Helper methods: `isAdmin()`, user type checks
  - Address helper: `hasCompleteAddress()`, `getAddressAttribute()`
- [x] Create user migration with all required fields

---

## Phase 7: Database Configuration

- [x] Configure `config/database.php`:
  - Default: SQLite for local dev fallback
  - MySQL connection for production
- [x] Create base migrations:
  - `users` table with extended fields
  - `password_reset_tokens` table
  - `cache` table
  - `jobs` table (for queue)
- [x] Set up `.env` with database configuration:
  ```
  DB_CONNECTION=mysql
  DB_HOST=mysql
  DB_PORT=3306
  DB_DATABASE=laravel
  DB_USERNAME=sail
  DB_PASSWORD=password
  ```

---

## Phase 8: Queue System Setup

- [x] Configure `config/queue.php`:
  - Default driver: database
  - Table: jobs
  - Retry after: 90 seconds
- [x] Run migration: `php artisan queue:table && php artisan migrate`
- [x] Create base job class template in `app/Jobs/`
- [x] Ensure supervisor handles queue worker in Docker

---

## Phase 9: CORS Configuration

- [x] Configure `config/cors.php`:
  - Allowed paths: `auth/*`, API endpoints pattern
  - Allowed origins: `*` (or specific frontend URLs)
  - Allowed methods: `*`
  - Allowed headers: `*`
  - Supports credentials: true

---

## Phase 10: Environment Configuration Template

- [x] Create comprehensive `.env.example` with:
  ```
  APP_NAME=YourApp
  APP_ENV=local
  APP_DEBUG=true
  APP_URL=http://localhost:8001
  APP_FRONTEND_URL=http://localhost:3000
  SPA_URL=http://localhost:3000
  APP_PORT=8001

  DB_CONNECTION=mysql
  DB_HOST=mysql
  DB_PORT=3306
  DB_DATABASE=laravel
  DB_USERNAME=sail
  DB_PASSWORD=password

  SESSION_DRIVER=cookie
  SESSION_LIFETIME=120
  SESSION_DOMAIN=localhost

  QUEUE_CONNECTION=database

  MAIL_MAILER=log

  # Add placeholders for common integrations:
  # STRIPE_KEY=
  # STRIPE_SECRET=
  # STRIPE_WEBHOOK_SECRET=
  # MAILGUN_DOMAIN=
  # MAILGUN_SECRET=
  ```

---

## Phase 11: API Routes Structure

- [x] Create `routes/api.php` with organized sections:
  - Public routes (webhooks, health check)
  - Guest routes (auth flows)
  - Authenticated routes (protected by `auth:sanctum`)
  - Admin routes (protected by `admin` middleware)
- [x] Remove default `routes/web.php` routes (API-only)

---

## Phase 12: Base Controllers & Services

- [x] Create `app/Http/Controllers/Controller.php` base controller
- [x] Create `app/Http/Controllers/ProfileController.php` for user profile CRUD
- [x] Create `app/Services/` directory for business logic separation
- [x] Create `app/Http/Requests/` directory with base form request classes

---

## Phase 13: Optional Packages (Common Integrations)

- [ ] Stripe (if needed): `composer require laravel/cashier`
- [ ] Social Auth (if needed): `composer require laravel/socialite`
- [ ] S3/Spaces Storage (if needed): `composer require league/flysystem-aws-s3-v3`
- [ ] Mailgun (if needed): `composer require symfony/mailgun-mailer symfony/http-client`

---

## Phase 14: Development Tools

- [ ] Install Pint for code formatting: `composer require laravel/pint --dev`
- [ ] Install Pail for log monitoring: `composer require laravel/pail --dev`
- [ ] Ensure PHPUnit is configured in `phpunit.xml`

---

## Phase 15: Final Project Structure

```
project-name/
├── app/
│   ├── Actions/Fortify/          # Auth action handlers
│   ├── Http/
│   │   ├── Controllers/          # API controllers
│   │   ├── Middleware/           # Custom middleware
│   │   └── Requests/             # Form request validation
│   ├── Models/                   # Eloquent models
│   ├── Jobs/                     # Queue jobs
│   ├── Mail/                     # Mailable classes
│   ├── Services/                 # Business logic
│   └── Providers/                # Service providers
├── bootstrap/
│   ├── app.php                   # Main bootstrap
│   └── providers.php             # Provider list
├── config/                       # Configuration files
├── database/
│   ├── migrations/               # Database migrations
│   ├── factories/                # Model factories
│   └── seeders/                  # Database seeders
├── routes/
│   ├── api.php                   # API routes
│   └── console.php               # Console commands
├── docker-compose.yml            # Docker services
├── Dockerfile                    # Custom image
├── supervisord.conf              # Process supervision
├── start.sh                      # Container startup script
├── composer.json                 # Dependencies
└── .env.example                  # Environment template
```

---

## Phase 16: First Run Checklist

- [ ] Copy `.env.example` to `.env`
- [ ] Generate app key: `php artisan key:generate`
- [ ] Start Docker: `docker-compose up -d` or `./vendor/bin/sail up -d`
- [ ] Run migrations: `php artisan migrate`
- [ ] Test health endpoint: `curl http://localhost:8001/up`
- [ ] Test auth flow: Register user, login, access protected route

---

## Quick Reference: Key Files to Copy/Adapt

1. `Dockerfile` - Custom PHP 8.3 + Node.js + Supervisor
2. `docker-compose.yml` - Service orchestration
3. `supervisord.conf` - Process management
4. `start.sh` - Container startup script
5. `bootstrap/app.php` - Application bootstrap
6. `app/Providers/FortifyServiceProvider.php` - Auth setup
7. `app/Actions/Fortify/*` - Auth action handlers
8. `app/Http/Middleware/JsonResponse.php` - Force JSON responses
9. `app/Http/Middleware/AdminMiddleware.php` - Admin protection
10. `config/fortify.php` - Fortify configuration
11. `config/sanctum.php` - Sanctum configuration
12. `config/cors.php` - CORS settings

---

## Review

This plan provides a complete blueprint for creating a Laravel API with:
- **Docker + Laravel Sail** for containerized development
- **PHP 8.3** with all necessary extensions
- **Supervisor** for managing PHP server, queue workers, and scheduler
- **Laravel Fortify** for authentication (registration, login, password reset)
- **Laravel Sanctum** for API token + session-based authentication
- **MySQL 8.0** as the database
- **Queue system** with database driver
- **Custom middleware** for JSON responses and admin protection
- **CORS** configured for SPA frontend integration
- **Organized project structure** with clear separation of concerns

The setup is API-only (no blade views) and designed to work with a separate SPA frontend.
