# Quick Setup Guide

## Prerequisites Check

1. **Docker Desktop** - Must be running before starting services
2. **Node.js** >= 18.0.0
3. **pnpm** >= 8.0.0 (already installed)

## Setup Steps

### 1. Start Docker Desktop
Make sure Docker Desktop is running before proceeding.

### 2. Start PostgreSQL Database
```bash
docker-compose up -d
```

Verify it's running:
```bash
docker-compose ps
```

### 3. Build Shared Package
```bash
pnpm build:shared
```
Required so the API and dashboard use the latest Event & State Layer schemas and types.

### 4. Run Database Migrations
```bash
pnpm db:migrate
```

### 5. Start Development Servers
```bash
pnpm dev
```

This will start:
- **Dashboard**: http://localhost:3000
- **Landing**: http://localhost:3002  
- **API**: http://localhost:3001
- **Worker**: Running in background

## Troubleshooting

### Port Already in Use
If you see "port already in use" errors:
- Check what's running on ports 3000, 3001, 3002
- Kill the process or change ports in package.json files

### Database Connection Errors
- Ensure Docker Desktop is running
- Verify PostgreSQL container is up: `docker-compose ps`
- Check DATABASE_URL in `services/api/.env`

### Module Resolution Errors
If you see "Cannot find module" errors:
- Rebuild shared package: `pnpm build:shared`
- Reinstall dependencies: `pnpm install`

### Next.js Build Errors
- Clear Next.js cache: `rm -rf apps/*/.next`
- Rebuild: `pnpm build`

## First Time Setup Checklist

- [x] pnpm installed
- [x] Dependencies installed (`pnpm install`)
- [x] Prisma client generated (`pnpm db:generate`)
- [ ] Docker Desktop running
- [ ] PostgreSQL container started (`docker-compose up -d`)
- [ ] Shared package built (`pnpm build:shared`)
- [ ] Database migrations run (`pnpm db:migrate`)
- [ ] Dev servers started (`pnpm dev`)
