# PaperFlame 🔥📄

**Plateforme SaaS de gestion financière pour freelances** - Suivez vos dépenses et factures en temps réel avec synchronisation automatique vers Google Drive.

## 🎯 Vision

Simplifier la gestion administrative des freelances avec :
- 📊 Suivi en temps réel des dépenses et factures
- ☁️ Backup automatique sur Google Drive
- 📈 Tableaux de bord et rapports
- 🔐 Architecture multi-tenant (SaaS-ready)

## 🏗️ Architecture

### Backend (NestJS)
- **Framework** : NestJS - Architecture modulaire et scalable
- **Base de données** : PostgreSQL + Prisma ORM
- **Cache & Jobs** : Redis + Bull Queue
- **API** : REST + Swagger auto-généré
- **Intégrations** : Google Drive API

### Frontend (Next.js)
- **Framework** : Next.js 14 (App Router)
- **UI** : TailwindCSS + shadcn/ui
- **State** : React Query (TanStack Query)
- **Auth** : NextAuth.js

### Infrastructure
- **Conteneurisation** : Docker + Docker Compose
- **CI/CD** : GitHub Actions
- **Sécurité** : CodeQL + Dependabot

## 🔄 Synchronisation Google Drive

```mermaid
graph LR
    A[Événement] --> B[API NestJS]
    B --> C[PostgreSQL]
    B --> D[Redis Queue]
    D --> E[Worker]
    E --> F[Google Drive]
```

Chaque création/modification de dépense ou facture déclenche :
1. Sauvegarde en base de données
2. Job ajouté à la queue Redis
3. Export automatique vers Google Drive (JSON + CSV + PDF)

## 🚀 Démarrage rapide

### Prérequis
- Node.js 20+
- Docker & Docker Compose
- Compte Google Cloud (API Drive)

### Installation

```bash
# Cloner le projet
git clone https://github.com/padubrulle/paperflame.git
cd paperflame

# Démarrer l'infrastructure (PostgreSQL + Redis)
docker-compose up -d

# Backend
cd backend
npm install
npx prisma migrate dev
npm run start:dev

# Frontend (nouveau terminal)
cd frontend
npm install
npm run dev
```

### Variables d'environnement

**Backend** (`.env`) :
```env
DATABASE_URL="postgresql://user:password@localhost:5432/paperflame"
REDIS_URL="redis://localhost:6379"
GOOGLE_CLIENT_ID="your-client-id"
GOOGLE_CLIENT_SECRET="your-client-secret"
GOOGLE_REDIRECT_URI="http://localhost:3000/auth/google/callback"
JWT_SECRET="your-secret-key"
```

**Frontend** (`.env.local`) :
```env
NEXT_PUBLIC_API_URL="http://localhost:3000"
NEXTAUTH_URL="http://localhost:3001"
NEXTAUTH_SECRET="your-nextauth-secret"
```

## 📦 Structure du projet

```
paperflame/
├── backend/              # API NestJS
│   ├── src/
│   │   ├── modules/
│   │   │   ├── expenses/
│   │   │   ├── invoices/
│   │   │   ├── google-drive/
│   │   │   └── auth/
│   │   ├── prisma/
│   │   └── main.ts
│   └── package.json
├── frontend/             # App Next.js
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── package.json
├── docker-compose.yml
└── README.md
```

## 🧪 Tests

```bash
# Backend
cd backend
npm test              # Unit tests
npm run test:e2e      # E2E tests
npm run test:cov      # Coverage

# Frontend
cd frontend
npm test              # Unit tests
npm run test:e2e      # Playwright E2E
```

## 🔒 Sécurité

- **CodeQL** : Analyse automatique du code
- **Dependabot** : Mises à jour de sécurité
- **CI/CD** : Tests automatiques sur chaque PR
- **Multi-tenant** : Isolation complète des données

## 🚢 Déploiement

Le projet est conçu pour être déployé sur :
- **Backend** : Vercel / Railway / Render
- **Frontend** : Vercel (optimal pour Next.js)
- **Base de données** : Supabase / Neon / AWS RDS
- **Redis** : Upstash / Redis Cloud

## 📝 Roadmap

- [x] Setup CI/CD
- [ ] Backend NestJS + Prisma
- [ ] Frontend Next.js + shadcn/ui
- [ ] Authentification (Google OAuth)
- [ ] Module Dépenses
- [ ] Module Factures
- [ ] Intégration Google Drive
- [ ] Tableaux de bord
- [ ] Multi-tenant
- [ ] Mode SaaS (abonnements)

## 📄 License

Voir le fichier [LICENSE](LICENSE)

---

**Fait avec ❤️ pour simplifier la vie des freelances**
