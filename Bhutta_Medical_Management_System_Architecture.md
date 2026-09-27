# Bhutta Medical Management System
### Technical Architecture, Database Schema & Core Implementation

---

## 1. Overview

**Application Name:** Bhutta Medical Management System
**Purpose:** A comprehensive pharmacy management platform covering inventory, staff/payroll, point-of-sale billing, and financial analytics for daily retail pharmacy operations.

---

## 2. Recommended Tech Stack

Given the mix of real-time inventory, transactional billing, and financial reporting, a **TypeScript-first, single-language stack** is recommended — it reduces context-switching for a small dev team and gives type safety from database to UI.

| Layer | Technology | Why |
|---|---|---|
| Frontend | **Next.js 14 (App Router)** + TypeScript | SSR for dashboard performance, file-based routing fits multi-module app |
| UI Kit | **Tailwind CSS + shadcn/ui** | Fast, professional, accessible components (tables, dialogs, forms) |
| Charts | **Recharts** | Clean analytics visualizations (sales trends, P&L) |
| Backend | **NestJS** (Node.js/TypeScript) | Modular architecture (Inventory, Sales, Payroll, Auth as separate modules), built-in guards/decorators for RBAC |
| ORM | **Prisma** | Type-safe queries, easy migrations, great with PostgreSQL |
| Database | **PostgreSQL** (SQLite for local/dev) | ACID compliance is essential for sales transactions and stock decrements |
| Auth | **JWT + Passport.js**, bcrypt for passwords | Stateless auth, role guards for Admin vs Cashier |
| Background Jobs | **node-cron** | Nightly job to flag expiring stock, compute daily closing reports |
| Deployment | Docker Compose (Postgres + API + Web); Vercel (frontend) + Railway/Render (backend+DB) | Simple, scalable for a single-branch retail deployment |

**Alternative:** If the team prefers Python, swap NestJS → **FastAPI** + SQLAlchemy + Pydantic — the schema and endpoint logic below map 1:1 either way. Code below is in Node/NestJS + Prisma flavor, since it pairs naturally with a Next.js frontend.

---

## 3. Database Schema

### 3.1 Entity Relationship Overview

```
Supplier (1) ──< (N) Medicine
User/Employee (1) ──< (N) Sale          [cashier who processed it]
Sale (1) ──< (N) SaleItem >── (N) Medicine
User/Employee (1) ──< (N) Payroll
User/Employee (1) ──< (N) ShiftLog
User/Employee (1) ──< (N) Advance
Expense (standalone, linked to recordedBy User)
```

### 3.2 Prisma Schema (`schema.prisma`)

```prisma
model Supplier {
  id        String     @id @default(cuid())
  name      String
  phone     String?
  address   String?
  medicines Medicine[]
  createdAt DateTime   @default(now())
}

model Medicine {
  id               String   @id @default(cuid())
  name             String
  genericName      String
  batchNo          String
  purchasePrice    Decimal  @db.Decimal(10, 2)
  salePrice        Decimal  @db.Decimal(10, 2)
  quantity         Int      @default(0)
  reorderThreshold Int      @default(10)   // configurable low-stock trigger
  expiryDate       DateTime
  category         String?
  supplierId       String
  supplier         Supplier @relation(fields: [supplierId], references: [id])
  saleItems        SaleItem[]
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@index([expiryDate])
  @@index([quantity])
}

enum Role {
  ADMIN
  CASHIER
}

model User {
  id            String     @id @default(cuid())
  name          String
  email         String     @unique
  passwordHash  String
  role          Role       @default(CASHIER)
  baseSalary    Decimal    @db.Decimal(10, 2)
  status        String     @default("active") // active | terminated
  joinDate      DateTime   @default(now())
  sales         Sale[]
  payrolls      Payroll[]
  shiftLogs     ShiftLog[]
  advances      Advance[]
}

model Sale {
  id            String     @id @default(cuid())
  invoiceNo     String     @unique
  cashierId     String
  cashier       User       @relation(fields: [cashierId], references: [id])
  items         SaleItem[]
  subtotal      Decimal    @db.Decimal(10, 2)
  discount      Decimal    @default(0) @db.Decimal(10, 2)
  totalAmount   Decimal    @db.Decimal(10, 2)
  paymentMethod String     @default("cash") // cash | card | credit
  createdAt     DateTime   @default(now())

  @@index([createdAt])
}

model SaleItem {
  id              String   @id @default(cuid())
  saleId          String
  sale            Sale     @relation(fields: [saleId], references: [id])
  medicineId      String
  medicine        Medicine @relation(fields: [medicineId], references: [id])
  quantity        Int
  unitSalePrice   Decimal  @db.Decimal(10, 2)  // snapshot at sale time
  unitCostPrice   Decimal  @db.Decimal(10, 2)  // snapshot for COGS accuracy
  lineTotal       Decimal  @db.Decimal(10, 2)
}

model Payroll {
  id          String   @id @default(cuid())
  employeeId  String
  employee    User     @relation(fields: [employeeId], references: [id])
  month       String   // "2026-09"
  baseSalary  Decimal  @db.Decimal(10, 2)
  advances    Decimal  @default(0) @db.Decimal(10, 2)
  bonuses     Decimal  @default(0) @db.Decimal(10, 2)
  deductions  Decimal  @default(0) @db.Decimal(10, 2)
  netPay      Decimal  @db.Decimal(10, 2)
  paidOn      DateTime?
  status      String   @default("pending") // pending | paid

  @@unique([employeeId, month])
}

model Advance {
  id         String   @id @default(cuid())
  employeeId String
  employee   User     @relation(fields: [employeeId], references: [id])
  amount     Decimal  @db.Decimal(10, 2)
  reason     String?
  givenOn    DateTime @default(now())
}

model ShiftLog {
  id         String    @id @default(cuid())
  employeeId String
  employee   User      @relation(fields: [employeeId], references: [id])
  clockIn    DateTime
  clockOut   DateTime?
  date       DateTime  @default(now())
}

model Expense {
  id          String   @id @default(cuid())
  category    String   // rent | utilities | maintenance | misc
  description String?
  amount      Decimal  @db.Decimal(10, 2)
  date        DateTime @default(now())
  recordedBy  String
}
```

### 3.3 Design Notes

- `SaleItem` snapshots both cost and sale price at transaction time — critical so historical P&L stays accurate even if a medicine's purchase price is later updated.
- `Decimal` type (not float) is used throughout for money fields to avoid rounding errors.
- Indexes on `expiryDate` and `quantity` make the low-stock/expiry dashboard queries fast even with large inventories.

---

## 4. UI Workflow & Page Structure

```
/login                          → Role-based redirect (Admin → full dashboard, Cashier → POS-first view)

/dashboard                      → KPI cards: Today's Sales, Stock Valuation, Low Stock Count,
                                   Expiring Soon Count, Net Profit (MTD)
                                   + Sales trend chart + Quick action buttons

/inventory
  /inventory/all                → Searchable/sortable medicine table
  /inventory/add                → Add Medicine form (Name, Formula, Batch, Prices, Qty, Expiry, Supplier)
  /inventory/low-stock           → Filtered view (qty < threshold), reorder button per row
  /inventory/expiry               → Tabs: "Expiring in 30-60 days" (yellow) | "Expired" (red badge)

/pos                            → Cashier-first fast checkout:
                                   search/barcode → add to cart → discount → payment method → print invoice
                                   (auto-decrements stock on submit)

/sales
  /sales/history                 → Transaction log, filterable by cashier/date
  /sales/reports                 → Daily / Weekly / Monthly toggle + export

/employees
  /employees/all                  → Staff list, role tags (Admin/Cashier)
  /employees/:id/payroll          → Salary history, advances, "Run Payroll" button
  /employees/:id/shifts           → Clock in/out log

/finance
  /finance/expenses               → Add/list operating expenses
  /finance/profit-loss            → P&L dashboard: Revenue - (COGS + Salaries + Expenses) = Net Profit
                                     with period selector and breakdown chart

/settings                        → Low-stock threshold config, user role management (Admin only)
```

**Access control pattern:** Admin/Owner sees everything; Cashier role is restricted to `/pos`, `/inventory/all` (read-only), and their own `/employees/:id/shifts` — enforced both by route guards and by hiding nav items client-side.

---

## 5. Core Backend Logic

### 5.1 Role-Based Guard (NestJS)

```typescript
// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<Role[]>('roles', context.getHandler());
    if (!requiredRoles) return true;
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.includes(user.role);
  }
}

// Usage on a controller route
@Roles(Role.ADMIN)
@UseGuards(JwtAuthGuard, RolesGuard)
@Post('employees/:id/payroll')
runPayroll(@Param('id') id: string, @Body() dto: RunPayrollDto) { ... }
```

### 5.2 Low Stock Endpoint

Prisma cannot compare two columns directly in a `where` clause, so this uses a raw query for correctness:

```typescript
// inventory.controller.ts
@Get('inventory/low-stock')
async getLowStock() {
  return this.prisma.$queryRaw`
    SELECT * FROM "Medicine"
    WHERE "quantity" <= "reorderThreshold"
    ORDER BY "quantity" ASC
  `;
}
```

### 5.3 Expiry Management Endpoint

```typescript
@Get('inventory/expiry')
async getExpiryStatus() {
  const now = new Date();
  const in30 = new Date(now); in30.setDate(now.getDate() + 30);
  const in60 = new Date(now); in60.setDate(now.getDate() + 60);

  const [expired, expiringSoon] = await Promise.all([
    this.prisma.medicine.findMany({ where: { expiryDate: { lt: now } } }),
    this.prisma.medicine.findMany({
      where: { expiryDate: { gte: now, lte: in60 } },
      orderBy: { expiryDate: 'asc' },
    }),
  ]);

  // Tag each item red (expired) / yellow (<30d) / amber (30-60d) for UI badges
  const tagged = expiringSoon.map(m => ({
    ...m,
    urgency: m.expiryDate <= in30 ? 'critical' : 'warning',
  }));

  return { expired, expiringSoon: tagged };
}
```

### 5.4 POS / Sale Transaction (atomic stock decrement)

```typescript
@Post('sales')
async createSale(@Body() dto: CreateSaleDto, @Req() req) {
  return this.prisma.$transaction(async (tx) => {
    let subtotal = 0;
    const itemsData = [];

    for (const item of dto.items) {
      const medicine = await tx.medicine.findUniqueOrThrow({ where: { id: item.medicineId } });
      if (medicine.quantity < item.quantity) {
        throw new BadRequestException(`Insufficient stock for ${medicine.name}`);
      }
      const lineTotal = Number(medicine.salePrice) * item.quantity;
      subtotal += lineTotal;

      itemsData.push({
        medicineId: medicine.id,
        quantity: item.quantity,
        unitSalePrice: medicine.salePrice,
        unitCostPrice: medicine.purchasePrice, // snapshot for accurate COGS later
        lineTotal,
      });

      await tx.medicine.update({
        where: { id: medicine.id },
        data: { quantity: { decrement: item.quantity } },
      });
    }

    const totalAmount = subtotal - (dto.discount ?? 0);

    return tx.sale.create({
      data: {
        invoiceNo: `INV-${Date.now()}`,
        cashierId: req.user.id,
        subtotal,
        discount: dto.discount ?? 0,
        totalAmount,
        paymentMethod: dto.paymentMethod,
        items: { create: itemsData },
      },
      include: { items: true },
    });
  });
}
```

Wrapping this in `$transaction` guarantees stock decrement and sale record creation succeed or fail together — essential for a billing system.

### 5.5 Profit & Loss Calculation

```typescript
@Get('finance/profit-loss')
async getProfitLoss(@Query('from') from: string, @Query('to') to: string) {
  const start = new Date(from), end = new Date(to);

  // 1. Revenue + COGS from sale line items (using snapshotted prices)
  const saleItems = await this.prisma.saleItem.findMany({
    where: { sale: { createdAt: { gte: start, lte: end } } },
  });
  const totalRevenue = saleItems.reduce((sum, i) => sum + Number(i.lineTotal), 0);
  const cogs = saleItems.reduce((sum, i) => sum + Number(i.unitCostPrice) * i.quantity, 0);

  // 2. Salaries paid within period
  const payrolls = await this.prisma.payroll.findMany({
    where: { paidOn: { gte: start, lte: end }, status: 'paid' },
  });
  const totalSalaries = payrolls.reduce((sum, p) => sum + Number(p.netPay), 0);

  // 3. Operating expenses
  const expenses = await this.prisma.expense.findMany({
    where: { date: { gte: start, lte: end } },
  });
  const totalExpenses = expenses.reduce((sum, e) => sum + Number(e.amount), 0);

  const netProfit = totalRevenue - (cogs + totalSalaries + totalExpenses);

  return {
    period: { from, to },
    totalRevenue,
    cogs,
    totalSalaries,
    totalExpenses,
    netProfit,
    marginPercent: totalRevenue ? ((netProfit / totalRevenue) * 100).toFixed(2) : 0,
  };
}
```

### 5.6 Total Stock Valuation (real-time metric)

```typescript
@Get('inventory/valuation')
async getStockValuation() {
  const result = await this.prisma.$queryRaw<{ total: number }[]>`
    SELECT COALESCE(SUM("purchasePrice" * "quantity"), 0) AS total FROM "Medicine"
  `;
  return { totalInvestmentValue: result[0].total };
}
```

### 5.7 Nightly Expiry-Check Job

```typescript
@Cron('0 6 * * *') // 6 AM daily
async flagExpiringStock() {
  const in30 = new Date(); in30.setDate(in30.getDate() + 30);
  const soon = await this.prisma.medicine.findMany({
    where: { expiryDate: { lte: in30 } },
  });
  if (soon.length) {
    // push to notification table / send admin email/SMS
    await this.notificationService.notifyAdmins(
      `${soon.length} medicine(s) expiring within 30 days.`
    );
  }
}
```

---

## 6. Suggested Build Order

1. **Auth + RBAC** (Admin/Cashier login, JWT guards)
2. **Inventory CRUD** + low-stock/expiry views (unblocks daily operations fastest)
3. **POS/Billing** with atomic stock decrement
4. **Sales reports** (daily/weekly/monthly aggregation)
5. **Payroll/Employee module**
6. **P&L dashboard** tying it all together

This order lets Bhutta Medical start using inventory tracking and POS within the first couple of sprints, while the payroll/analytics layer is added afterward without schema rework.
