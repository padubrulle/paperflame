# Architecture Technique - PaperFlame

## 🎯 Principes de conception

### Scalabilité
- Architecture modulaire (NestJS modules)
- Séparation claire backend/frontend
- Queue system pour jobs asynchrones
- Multi-tenant ready avec isolation des données

### Sécurité
- Authentification JWT + OAuth2 (Google)
- Row Level Security (RLS) avec Prisma
- Validation des inputs (class-validator)
- Rate limiting sur les endpoints
- CORS configuré strictement

### Performance
- Cache Redis pour les requêtes fréquentes
- Pagination sur toutes les listes
- Index PostgreSQL optimisés
- CDN pour les assets statiques (Next.js)
- ISR (Incremental Static Regeneration) sur Next.js

## 🗄️ Schéma de base de données

### Tables principales

```prisma
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  googleId      String?   @unique
  tenantId      String?   // Pour le multi-tenant
  
  expenses      Expense[]
  invoices      Invoice[]
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

model Expense {
  id            String    @id @default(cuid())
  userId        String
  user          User      @relation(fields: [userId], references: [id])
  
  amount        Decimal   @db.Decimal(10, 2)
  category      String
  description   String?
  date          DateTime
  attachment    String?   // URL Google Drive
  
  syncedToDrive Boolean   @default(false)
  driveFileId   String?
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  @@index([userId, date])
}

model Invoice {
  id            String    @id @default(cuid())
  userId        String
  user          User      @relation(fields: [userId], references: [id])
  
  number        String    @unique
  clientName    String
  amount        Decimal   @db.Decimal(10, 2)
  status        InvoiceStatus
  issueDate     DateTime
  dueDate       DateTime
  paidDate      DateTime?
  
  syncedToDrive Boolean   @default(false)
  driveFileId   String?
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  @@index([userId, status])
}

enum InvoiceStatus {
  DRAFT
  SENT
  PAID
  OVERDUE
  CANCELLED
}
```

## 🔄 Flux de synchronisation Google Drive

### 1. Création d'une dépense/facture

```typescript
// Controller
@Post('expenses')
async createExpense(@Body() dto: CreateExpenseDto, @User() user) {
  const expense = await this.expensesService.create(dto, user.id);
  
  // Ajoute un job à la queue pour sync Google Drive
  await this.driveQueue.add('sync-expense', {
    expenseId: expense.id,
    userId: user.id
  });
  
  return expense;
}

// Worker (processor)
@Process('sync-expense')
async handleExpenseSync(job: Job<{ expenseId: string, userId: string }>) {
  const expense = await this.expensesService.findOne(job.data.expenseId);
  
  // Export vers Google Drive
  const file = await this.googleDriveService.exportExpense(expense);
  
  // Mise à jour de l'enregistrement
  await this.expensesService.update(expense.id, {
    syncedToDrive: true,
    driveFileId: file.id
  });
}
```

### 2. Structure dans Google Drive

```
📁 PaperFlame/
  📁 2024/
    📁 Dépenses/
      📄 2024-01_depenses.csv
      📄 2024-02_depenses.csv
    📁 Factures/
      📄 FAC-2024-001.pdf
      📄 FAC-2024-002.pdf
  📁 2025/
    ...
  📁 Exports/
    📄 export_complet_2024.json
```

## 🏗️ Modules NestJS

```
backend/src/
├── modules/
│   ├── auth/              # Authentification (JWT + OAuth)
│   │   ├── strategies/
│   │   ├── guards/
│   │   └── decorators/
│   ├── users/             # Gestion utilisateurs
│   ├── expenses/          # Module dépenses
│   │   ├── dto/
│   │   ├── entities/
│   │   └── expenses.service.ts
│   ├── invoices/          # Module factures
│   ├── google-drive/      # Intégration Google Drive
│   │   ├── google-drive.service.ts
│   │   └── google-drive.processor.ts
│   ├── tenants/           # Multi-tenant (futur)
│   └── analytics/         # Tableaux de bord
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── interceptors/
│   └── pipes/
├── config/
└── prisma/
```

## 🎨 Frontend Next.js

### Structure

```
frontend/
├── app/
│   ├── (auth)/           # Routes authentification
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/      # Routes protégées
│   │   ├── expenses/
│   │   ├── invoices/
│   │   ├── analytics/
│   │   └── settings/
│   └── api/              # API routes (proxy)
├── components/
│   ├── ui/               # shadcn/ui components
│   ├── forms/
│   ├── tables/
│   └── charts/
├── lib/
│   ├── api-client.ts     # React Query hooks
│   ├── auth.ts           # NextAuth config
│   └── utils.ts
└── types/
```

### State Management

```typescript
// React Query pour le cache et sync serveur
const { data: expenses } = useQuery({
  queryKey: ['expenses', { month: currentMonth }],
  queryFn: () => api.expenses.list({ month: currentMonth }),
  staleTime: 30000, // Cache 30s
});

// Mutations avec optimistic updates
const createExpense = useMutation({
  mutationFn: api.expenses.create,
  onMutate: async (newExpense) => {
    // Optimistic update
    await queryClient.cancelQueries(['expenses']);
    const previous = queryClient.getQueryData(['expenses']);
    
    queryClient.setQueryData(['expenses'], (old) => ({
      ...old,
      data: [...old.data, newExpense]
    }));
    
    return { previous };
  },
  onSuccess: () => {
    queryClient.invalidateQueries(['expenses']);
  },
});
```

## 🚀 Stratégie de déploiement

### Option 1 : Full Vercel (recommandé pour MVP)
- **Frontend** : Vercel (Next.js natif)
- **Backend** : Vercel Serverless Functions
- **DB** : Neon (PostgreSQL serverless)
- **Redis** : Upstash

### Option 2 : Hybride (plus de contrôle)
- **Frontend** : Vercel
- **Backend** : Railway / Render (Docker)
- **DB** : Supabase
- **Redis** : Redis Cloud

### Option 3 : Cloud Provider (production scale)
- **Infrastructure** : AWS / GCP
- **Container** : ECS / Cloud Run
- **DB** : RDS / Cloud SQL
- **Redis** : ElastiCache / Memorystore

## 🔐 Sécurité Multi-tenant

```typescript
// Middleware Prisma pour isolation automatique
export const tenantMiddleware: Prisma.Middleware = async (params, next) => {
  if (params.model && TENANT_MODELS.includes(params.model)) {
    if (params.action === 'findMany' || params.action === 'findFirst') {
      params.args.where = {
        ...params.args.where,
        tenantId: getCurrentTenantId()
      };
    }
  }
  return next(params);
};
```

## 📊 Monitoring & Observabilité

- **Logs** : Winston + CloudWatch / DataDog
- **APM** : Sentry pour les erreurs
- **Métriques** : Prometheus + Grafana (optionnel)
- **Alertes** : PagerDuty / OpsGenie

---

Cette architecture permet de démarrer simple et de scaler progressivement vers un SaaS complet.
