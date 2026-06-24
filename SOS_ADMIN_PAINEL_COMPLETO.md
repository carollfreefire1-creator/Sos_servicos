# SOS SERVIÇOS — PAINEL ADMINISTRATIVO COMPLETO

Arquivos novos. Não substitui nada existente.

---

## ESTRUTURA DE ARQUIVOS

```
sos-servicos/
├── supabase/migrations/
│   └── 0005_admin.sql                              ← NOVO
├── prisma/schema.prisma                            ← ADIÇÃO (campos admin)
├── middleware.ts                                    ← ADIÇÃO (proteção /admin)
├── lib/admin/
│   └── auth.ts                                      ← NOVO
├── actions/
│   └── admin.ts                                     ← NOVO
├── components/admin/
│   ├── AdminSidebar.tsx                             ← NOVO
│   ├── AdminTopbar.tsx                              ← NOVO
│   ├── StatCard.tsx                                 ← NOVO
│   ├── RevenueChart.tsx                             ← NOVO
│   ├── DataTable.tsx                                ← NOVO
│   ├── StatusBadge.tsx                               ← NOVO
│   └── ConfirmDialog.tsx                            ← NOVO
└── app/admin/
    ├── layout.tsx                                    ← NOVO
    ├── page.tsx                                      ← NOVO (dashboard)
    ├── users/page.tsx                                ← NOVO
    ├── professionals/page.tsx                        ← NOVO
    ├── services/page.tsx                             ← NOVO
    ├── categories/page.tsx                           ← NOVO
    ├── payments/page.tsx                             ← NOVO
    ├── reports/page.tsx                              ← NOVO
    └── settings/page.tsx                             ← NOVO
```

---

## 1. `supabase/migrations/0005_admin.sql`

```sql
-- ============================================================
-- Migration: painel administrativo
-- ============================================================

-- Papel de admin no usuário
alter table users add column if not exists role text not null default 'user';
alter table users add column if not exists blocked boolean not null default false;
alter table users add column if not exists blocked_reason text;

-- Status de aprovação dos profissionais
alter table professionals add column if not exists approval_status text not null default 'pending';
alter table professionals add column if not exists approved_at timestamptz;
alter table professionals add column if not exists rejected_reason text;

-- Categorias de serviço
create table if not exists categories (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  slug        text not null unique,
  icon        text,
  active      boolean not null default true,
  order_index integer not null default 0,
  created_at  timestamptz not null default now()
);

-- Planos de assinatura
create table if not exists plans (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  price_cents   integer not null,
  interval      text not null default 'month',
  features      jsonb not null default '[]',
  active        boolean not null default true,
  created_at    timestamptz not null default now()
);

-- Assinaturas dos profissionais
create table if not exists subscriptions (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references users(id) on delete cascade,
  plan_id      uuid not null references plans(id),
  status       text not null default 'active',
  current_period_end timestamptz,
  created_at   timestamptz not null default now()
);

-- Pagamentos
create table if not exists payments (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid not null references users(id) on delete cascade,
  request_id   uuid references service_requests(id) on delete set null,
  amount_cents integer not null,
  status       text not null default 'pending',
  method       text,
  gateway_id   text,
  created_at   timestamptz not null default now()
);

-- Log de ações administrativas
create table if not exists admin_logs (
  id          uuid primary key default gen_random_uuid(),
  admin_id    uuid not null references users(id),
  action      text not null,
  target_type text not null,
  target_id   uuid,
  details     jsonb,
  created_at  timestamptz not null default now()
);

create index if not exists idx_categories_active on categories(active);
create index if not exists idx_payments_status on payments(status);
create index if not exists idx_payments_created on payments(created_at desc);
create index if not exists idx_subscriptions_user on subscriptions(user_id);
create index if not exists idx_admin_logs_created on admin_logs(created_at desc);

alter table categories enable row level security;
alter table plans enable row level security;
alter table subscriptions enable row level security;
alter table payments enable row level security;
alter table admin_logs enable row level security;

create policy "categories_public_read" on categories for select using (true);
create policy "plans_public_read" on plans for select using (true);

create policy "admin_full_categories" on categories for all
  using (exists (select 1 from users where email = auth.email() and role = 'admin'))
  with check (exists (select 1 from users where email = auth.email() and role = 'admin'));

create policy "admin_full_plans" on plans for all
  using (exists (select 1 from users where email = auth.email() and role = 'admin'))
  with check (exists (select 1 from users where email = auth.email() and role = 'admin'));

create policy "admin_full_payments" on payments for all
  using (exists (select 1 from users where email = auth.email() and role = 'admin'))
  with check (exists (select 1 from users where email = auth.email() and role = 'admin'));

create policy "admin_full_logs" on admin_logs for all
  using (exists (select 1 from users where email = auth.email() and role = 'admin'))
  with check (exists (select 1 from users where email = auth.email() and role = 'admin'));

create policy "subscriptions_own_or_admin" on subscriptions for select
  using (
    user_id = (select id from users where email = auth.email() limit 1)
    or exists (select 1 from users where email = auth.email() and role = 'admin')
  );

create policy "admin_write_subscriptions" on subscriptions for all
  using (exists (select 1 from users where email = auth.email() and role = 'admin'))
  with check (exists (select 1 from users where email = auth.email() and role = 'admin'));

insert into categories (name, slug, icon, order_index) values
  ('Elétrica', 'eletrica', '⚡', 1),
  ('Hidráulica', 'hidraulica', '🚰', 2),
  ('Pintura', 'pintura', '🎨', 3),
  ('Limpeza', 'limpeza', '🧹', 4),
  ('Reformas', 'reformas', '🔨', 5),
  ('Jardinagem', 'jardinagem', '🌳', 6)
on conflict (slug) do nothing;

insert into plans (name, price_cents, interval, features) values
  ('Básico', 0, 'month', '["5 propostas por mês","Perfil padrão"]'),
  ('Profissional', 4990, 'month', '["Propostas ilimitadas","Selo verificado","Destaque na busca"]'),
  ('Premium', 9990, 'month', '["Tudo do Profissional","Suporte prioritário","Relatórios avançados"]')
on conflict do nothing;
```

---

## 2. `prisma/schema.prisma` — ADIÇÃO

```prisma
// ─── ADMIN ──────────────────────────────────────────────────────────────────

enum Role {
  user
  professional
  admin
}

enum ApprovalStatus {
  pending
  approved
  rejected
}

model Category {
  id         String   @id @default(uuid()) @db.Uuid
  name       String
  slug       String   @unique
  icon       String?
  active     Boolean  @default(true)
  orderIndex Int      @default(0) @map("order_index")
  createdAt  DateTime @default(now()) @map("created_at")

  @@map("categories")
}

model Plan {
  id           String         @id @default(uuid()) @db.Uuid
  name         String
  priceCents   Int            @map("price_cents")
  interval     String         @default("month")
  features     Json           @default("[]")
  active       Boolean        @default(true)
  createdAt    DateTime       @default(now()) @map("created_at")
  subscriptions Subscription[]

  @@map("plans")
}

model Subscription {
  id                String    @id @default(uuid()) @db.Uuid
  userId            String    @db.Uuid
  planId            String    @db.Uuid
  status            String    @default("active")
  currentPeriodEnd  DateTime? @map("current_period_end")
  createdAt         DateTime  @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  plan Plan @relation(fields: [planId], references: [id])

  @@index([userId])
  @@map("subscriptions")
}

model Payment {
  id          String   @id @default(uuid()) @db.Uuid
  userId      String   @db.Uuid
  requestId   String?  @db.Uuid
  amountCents Int      @map("amount_cents")
  status      String   @default("pending")
  method      String?
  gatewayId   String?  @map("gateway_id")
  createdAt   DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([status])
  @@index([createdAt(sort: Desc)])
  @@map("payments")
}

model AdminLog {
  id         String   @id @default(uuid()) @db.Uuid
  adminId    String   @db.Uuid
  action     String
  targetType String   @map("target_type")
  targetId   String?  @db.Uuid
  details    Json?
  createdAt  DateTime @default(now()) @map("created_at")

  @@index([createdAt(sort: Desc)])
  @@map("admin_logs")
}
```

No modelo `User`, adicione:
```prisma
  role         Role     @default(user)
  blocked      Boolean  @default(false)
  blockedReason String? @map("blocked_reason")
  payments      Payment[]
  subscriptions Subscription[]
```

No modelo `Professional`, adicione:
```prisma
  approvalStatus  ApprovalStatus @default(pending) @map("approval_status")
  approvedAt      DateTime?      @map("approved_at")
  rejectedReason  String?        @map("rejected_reason")
```

---

## 3. `lib/admin/auth.ts` ← NOVO

```typescript
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import { prisma } from "@/lib/prisma";

export async function requireAdmin() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  const dbUser = await prisma.user.findFirst({
    where: { email: user.email! },
    select: { id: true, name: true, email: true, avatarUrl: true, role: true },
  });

  if (!dbUser || dbUser.role !== "admin") redirect("/");

  return dbUser;
}
```

---

## 4. `middleware.ts` — ADIÇÃO

```typescript
// Dentro do middleware existente, adicione antes do retorno final:

if (request.nextUrl.pathname.startsWith("/admin")) {
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
}
```

---

## 5. `actions/admin.ts` ← NOVO

```typescript
"use server";

import { prisma } from "@/lib/prisma";
import { requireAdmin } from "@/lib/admin/auth";
import { revalidatePath } from "next/cache";

type ActionResult<T = void> =
  | { success: true; data?: T }
  | { success: false; error: string };

async function logAction(adminId: string, action: string, targetType: string, targetId?: string, details?: object) {
  await prisma.adminLog.create({
    data: { adminId, action, targetType, targetId, details: details ?? {} },
  });
}

// ── DASHBOARD ────────────────────────────────────────────────
export async function getDashboardStatsAction(): Promise<ActionResult<{
  totalUsers: number;
  totalProfessionals: number;
  monthlyRevenueCents: number;
  totalRequests: number;
  activeSubscriptions: number;
  pendingProfessionals: number;
  revenueByMonth: { month: string; cents: number }[];
  requestsByMonth: { month: string; count: number }[];
}>> {
  try {
    await requireAdmin();

    const now = new Date();
    const startOfMonth = new Date(now.getFullYear(), now.getMonth(), 1);

    const [
      totalUsers,
      totalProfessionals,
      totalRequests,
      activeSubscriptions,
      pendingProfessionals,
      monthlyPayments,
    ] = await Promise.all([
      prisma.user.count(),
      prisma.professional.count(),
      prisma.serviceRequest.count(),
      prisma.subscription.count({ where: { status: "active" } }),
      prisma.professional.count({ where: { approvalStatus: "pending" } }),
      prisma.payment.findMany({
        where: { status: "paid", createdAt: { gte: startOfMonth } },
        select: { amountCents: true },
      }),
    ]);

    const monthlyRevenueCents = monthlyPayments.reduce((sum, p) => sum + p.amountCents, 0);

    const sixMonthsAgo = new Date();
    sixMonthsAgo.setMonth(sixMonthsAgo.getMonth() - 5);

    const payments = await prisma.payment.findMany({
      where: { status: "paid", createdAt: { gte: sixMonthsAgo } },
      select: { amountCents: true, createdAt: true },
    });

    const requests = await prisma.serviceRequest.findMany({
      where: { createdAt: { gte: sixMonthsAgo } },
      select: { createdAt: true },
    });

    const monthLabels: string[] = [];
    for (let i = 5; i >= 0; i--) {
      const d = new Date();
      d.setMonth(d.getMonth() - i);
      monthLabels.push(d.toLocaleDateString("pt-BR", { month: "short" }));
    }

    const revenueByMonth = monthLabels.map((label, idx) => {
      const d = new Date();
      d.setMonth(d.getMonth() - (5 - idx));
      const cents = payments
        .filter((p) => p.createdAt.getMonth() === d.getMonth() && p.createdAt.getFullYear() === d.getFullYear())
        .reduce((s, p) => s + p.amountCents, 0);
      return { month: label, cents };
    });

    const requestsByMonth = monthLabels.map((label, idx) => {
      const d = new Date();
      d.setMonth(d.getMonth() - (5 - idx));
      const count = requests.filter(
        (r) => r.createdAt.getMonth() === d.getMonth() && r.createdAt.getFullYear() === d.getFullYear()
      ).length;
      return { month: label, count };
    });

    return {
      success: true,
      data: {
        totalUsers,
        totalProfessionals,
        monthlyRevenueCents,
        totalRequests,
        activeSubscriptions,
        pendingProfessionals,
        revenueByMonth,
        requestsByMonth,
      },
    };
  } catch (err) {
    console.error("[getDashboardStatsAction]", err);
    return { success: false, error: "Erro ao buscar estatísticas" };
  }
}

// ── USERS ────────────────────────────────────────────────────
export async function listUsersAction(params: {
  search?: string;
  role?: string;
  page?: number;
}): Promise<ActionResult<{ users: any[]; total: number }>> {
  try {
    await requireAdmin();
    const page = params.page ?? 1;
    const perPage = 20;

    const where: any = {};
    if (params.search) {
      where.OR = [
        { name: { contains: params.search, mode: "insensitive" } },
        { email: { contains: params.search, mode: "insensitive" } },
      ];
    }
    if (params.role) where.role = params.role;

    const [users, total] = await Promise.all([
      prisma.user.findMany({
        where,
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * perPage,
        take: perPage,
        select: {
          id: true, name: true, email: true, avatarUrl: true,
          role: true, blocked: true, createdAt: true, phone: true,
        },
      }),
      prisma.user.count({ where }),
    ]);

    return { success: true, data: { users, total } };
  } catch (err) {
    console.error("[listUsersAction]", err);
    return { success: false, error: "Erro ao listar usuários" };
  }
}

export async function toggleBlockUserAction(userId: string, block: boolean, reason?: string): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.user.update({
      where: { id: userId },
      data: { blocked: block, blockedReason: block ? reason ?? "Bloqueado pelo administrador" : null },
    });
    await logAction(admin.id, block ? "block_user" : "unblock_user", "user", userId);
    revalidatePath("/admin/users");
    return { success: true };
  } catch (err) {
    console.error("[toggleBlockUserAction]", err);
    return { success: false, error: "Erro ao atualizar usuário" };
  }
}

export async function changeUserRoleAction(userId: string, role: "user" | "professional" | "admin"): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.user.update({ where: { id: userId }, data: { role } });
    await logAction(admin.id, "change_role", "user", userId, { role });
    revalidatePath("/admin/users");
    return { success: true };
  } catch (err) {
    console.error("[changeUserRoleAction]", err);
    return { success: false, error: "Erro ao alterar papel" };
  }
}

// ── PROFESSIONALS ────────────────────────────────────────────
export async function listProfessionalsAction(params: {
  status?: string;
  search?: string;
  page?: number;
}): Promise<ActionResult<{ professionals: any[]; total: number }>> {
  try {
    await requireAdmin();
    const page = params.page ?? 1;
    const perPage = 20;

    const where: any = {};
    if (params.status) where.approvalStatus = params.status;
    if (params.search) {
      where.user = {
        OR: [
          { name: { contains: params.search, mode: "insensitive" } },
          { email: { contains: params.search, mode: "insensitive" } },
        ],
      };
    }

    const [professionals, total] = await Promise.all([
      prisma.professional.findMany({
        where,
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * perPage,
        take: perPage,
        include: { user: { select: { name: true, email: true, avatarUrl: true, phone: true, blocked: true } } },
      }),
      prisma.professional.count({ where }),
    ]);

    return { success: true, data: { professionals, total } };
  } catch (err) {
    console.error("[listProfessionalsAction]", err);
    return { success: false, error: "Erro ao listar profissionais" };
  }
}

export async function approveProfessionalAction(professionalId: string): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.professional.update({
      where: { id: professionalId },
      data: { approvalStatus: "approved", approvedAt: new Date(), rejectedReason: null },
    });
    await logAction(admin.id, "approve_professional", "professional", professionalId);
    revalidatePath("/admin/professionals");
    return { success: true };
  } catch (err) {
    console.error("[approveProfessionalAction]", err);
    return { success: false, error: "Erro ao aprovar profissional" };
  }
}

export async function rejectProfessionalAction(professionalId: string, reason: string): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.professional.update({
      where: { id: professionalId },
      data: { approvalStatus: "rejected", rejectedReason: reason },
    });
    await logAction(admin.id, "reject_professional", "professional", professionalId, { reason });
    revalidatePath("/admin/professionals");
    return { success: true };
  } catch (err) {
    console.error("[rejectProfessionalAction]", err);
    return { success: false, error: "Erro ao rejeitar profissional" };
  }
}

export async function blockProfessionalAction(professionalId: string, userId: string, block: boolean): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.user.update({ where: { id: userId }, data: { blocked: block } });
    await logAction(admin.id, block ? "block_professional" : "unblock_professional", "professional", professionalId);
    revalidatePath("/admin/professionals");
    return { success: true };
  } catch (err) {
    console.error("[blockProfessionalAction]", err);
    return { success: false, error: "Erro ao bloquear profissional" };
  }
}

// ── SERVICES (pedidos) ───────────────────────────────────────
export async function listServicesAction(params: {
  status?: string;
  search?: string;
  page?: number;
}): Promise<ActionResult<{ services: any[]; total: number }>> {
  try {
    await requireAdmin();
    const page = params.page ?? 1;
    const perPage = 20;

    const where: any = {};
    if (params.status) where.status = params.status;
    if (params.search) where.title = { contains: params.search, mode: "insensitive" };

    const [services, total] = await Promise.all([
      prisma.serviceRequest.findMany({
        where,
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * perPage,
        take: perPage,
        include: {
          client: { select: { name: true, email: true } },
          category: { select: { name: true } },
        },
      }),
      prisma.serviceRequest.count({ where }),
    ]);

    return { success: true, data: { services, total } };
  } catch (err) {
    console.error("[listServicesAction]", err);
    return { success: false, error: "Erro ao listar serviços" };
  }
}

// ── CATEGORIES ────────────────────────────────────────────────
export async function listCategoriesAction(): Promise<ActionResult<any[]>> {
  try {
    await requireAdmin();
    const categories = await prisma.category.findMany({ orderBy: { orderIndex: "asc" } });
    return { success: true, data: categories };
  } catch (err) {
    console.error("[listCategoriesAction]", err);
    return { success: false, error: "Erro ao listar categorias" };
  }
}

export async function createCategoryAction(data: {
  name: string; slug: string; icon?: string; orderIndex?: number;
}): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    const category = await prisma.category.create({ data });
    await logAction(admin.id, "create_category", "category", category.id, data);
    revalidatePath("/admin/categories");
    return { success: true };
  } catch (err) {
    console.error("[createCategoryAction]", err);
    return { success: false, error: "Erro ao criar categoria" };
  }
}

export async function updateCategoryAction(id: string, data: {
  name?: string; slug?: string; icon?: string; active?: boolean; orderIndex?: number;
}): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.category.update({ where: { id }, data });
    await logAction(admin.id, "update_category", "category", id, data);
    revalidatePath("/admin/categories");
    return { success: true };
  } catch (err) {
    console.error("[updateCategoryAction]", err);
    return { success: false, error: "Erro ao atualizar categoria" };
  }
}

export async function deleteCategoryAction(id: string): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.category.delete({ where: { id } });
    await logAction(admin.id, "delete_category", "category", id);
    revalidatePath("/admin/categories");
    return { success: true };
  } catch (err) {
    console.error("[deleteCategoryAction]", err);
    return { success: false, error: "Erro ao excluir categoria" };
  }
}

// ── PLANS ──────────────────────────────────────────────────────
export async function listPlansAction(): Promise<ActionResult<any[]>> {
  try {
    await requireAdmin();
    const plans = await prisma.plan.findMany({
      orderBy: { priceCents: "asc" },
      include: { _count: { select: { subscriptions: true } } },
    });
    return { success: true, data: plans };
  } catch (err) {
    console.error("[listPlansAction]", err);
    return { success: false, error: "Erro ao listar planos" };
  }
}

export async function savePlanAction(data: {
  id?: string; name: string; priceCents: number; interval: string; features: string[]; active: boolean;
}): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    if (data.id) {
      await prisma.plan.update({
        where: { id: data.id },
        data: { name: data.name, priceCents: data.priceCents, interval: data.interval, features: data.features, active: data.active },
      });
      await logAction(admin.id, "update_plan", "plan", data.id, data);
    } else {
      const plan = await prisma.plan.create({
        data: { name: data.name, priceCents: data.priceCents, interval: data.interval, features: data.features, active: data.active },
      });
      await logAction(admin.id, "create_plan", "plan", plan.id, data);
    }
    revalidatePath("/admin/settings");
    return { success: true };
  } catch (err) {
    console.error("[savePlanAction]", err);
    return { success: false, error: "Erro ao salvar plano" };
  }
}

// ── PAYMENTS ───────────────────────────────────────────────────
export async function listPaymentsAction(params: {
  status?: string;
  search?: string;
  page?: number;
}): Promise<ActionResult<{ payments: any[]; total: number; totalCents: number }>> {
  try {
    await requireAdmin();
    const page = params.page ?? 1;
    const perPage = 20;

    const where: any = {};
    if (params.status) where.status = params.status;
    if (params.search) {
      where.user = {
        OR: [
          { name: { contains: params.search, mode: "insensitive" } },
          { email: { contains: params.search, mode: "insensitive" } },
        ],
      };
    }

    const [payments, total, agg] = await Promise.all([
      prisma.payment.findMany({
        where,
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * perPage,
        take: perPage,
        include: { user: { select: { name: true, email: true } } },
      }),
      prisma.payment.count({ where }),
      prisma.payment.aggregate({ where: { ...where, status: "paid" }, _sum: { amountCents: true } }),
    ]);

    return { success: true, data: { payments, total, totalCents: agg._sum.amountCents ?? 0 } };
  } catch (err) {
    console.error("[listPaymentsAction]", err);
    return { success: false, error: "Erro ao listar pagamentos" };
  }
}

export async function refundPaymentAction(paymentId: string): Promise<ActionResult> {
  try {
    const admin = await requireAdmin();
    await prisma.payment.update({ where: { id: paymentId }, data: { status: "refunded" } });
    await logAction(admin.id, "refund_payment", "payment", paymentId);
    revalidatePath("/admin/payments");
    return { success: true };
  } catch (err) {
    console.error("[refundPaymentAction]", err);
    return { success: false, error: "Erro ao reembolsar pagamento" };
  }
}

// ── REPORTS ────────────────────────────────────────────────────
export async function getReportsAction(params: {
  from?: string; to?: string;
}): Promise<ActionResult<{
  totalRevenueCents: number;
  totalRequests: number;
  conversionRate: number;
  topCategories: { name: string; count: number }[];
  topProfessionals: { name: string; count: number }[];
}>> {
  try {
    await requireAdmin();
    const from = params.from ? new Date(params.from) : new Date(new Date().setMonth(new Date().getMonth() - 1));
    const to = params.to ? new Date(params.to) : new Date();

    const [payments, requests, completedRequests, categoryGroups, professionalGroups] = await Promise.all([
      prisma.payment.aggregate({
        where: { status: "paid", createdAt: { gte: from, lte: to } },
        _sum: { amountCents: true },
      }),
      prisma.serviceRequest.count({ where: { createdAt: { gte: from, lte: to } } }),
      prisma.serviceRequest.count({ where: { createdAt: { gte: from, lte: to }, status: "completed" } }),
      prisma.serviceRequest.groupBy({
        by: ["categoryId"],
        where: { createdAt: { gte: from, lte: to } },
        _count: true,
        orderBy: { _count: { categoryId: "desc" } },
        take: 5,
      }),
      prisma.serviceRequest.groupBy({
        by: ["professionalId"],
        where: { createdAt: { gte: from, lte: to }, professionalId: { not: null } },
        _count: true,
        orderBy: { _count: { professionalId: "desc" } },
        take: 5,
      }),
    ]);

    const categoryIds = categoryGroups.map((c) => c.categoryId).filter(Boolean) as string[];
    const categories = await prisma.category.findMany({ where: { id: { in: categoryIds } } });
    const topCategories = categoryGroups.map((g) => ({
      name: categories.find((c) => c.id === g.categoryId)?.name ?? "—",
      count: g._count as number,
    }));

    const proIds = professionalGroups.map((p) => p.professionalId).filter(Boolean) as string[];
    const pros = await prisma.professional.findMany({
      where: { id: { in: proIds } },
      include: { user: { select: { name: true } } },
    });
    const topProfessionals = professionalGroups.map((g) => ({
      name: pros.find((p) => p.id === g.professionalId)?.user.name ?? "—",
      count: g._count as number,
    }));

    return {
      success: true,
      data: {
        totalRevenueCents: payments._sum.amountCents ?? 0,
        totalRequests: requests,
        conversionRate: requests > 0 ? Math.round((completedRequests / requests) * 100) : 0,
        topCategories,
        topProfessionals,
      },
    };
  } catch (err) {
    console.error("[getReportsAction]", err);
    return { success: false, error: "Erro ao gerar relatório" };
  }
}

// ── SETTINGS ───────────────────────────────────────────────────
export async function getAdminLogsAction(page = 1): Promise<ActionResult<{ logs: any[]; total: number }>> {
  try {
    await requireAdmin();
    const perPage = 30;
    const [logs, total] = await Promise.all([
      prisma.adminLog.findMany({
        orderBy: { createdAt: "desc" },
        skip: (page - 1) * perPage,
        take: perPage,
      }),
      prisma.adminLog.count(),
    ]);
    return { success: true, data: { logs, total } };
  } catch (err) {
    console.error("[getAdminLogsAction]", err);
    return { success: false, error: "Erro ao buscar logs" };
  }
}
```

---

## 6. `components/admin/AdminSidebar.tsx` ← NOVO

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";

const NAV = [
  { href: "/admin", label: "Dashboard", icon: "📊" },
  { href: "/admin/users", label: "Usuários", icon: "👥" },
  { href: "/admin/professionals", label: "Profissionais", icon: "🛠️" },
  { href: "/admin/services", label: "Pedidos", icon: "📋" },
  { href: "/admin/categories", label: "Categorias", icon: "🗂️" },
  { href: "/admin/payments", label: "Pagamentos", icon: "💳" },
  { href: "/admin/reports", label: "Relatórios", icon: "📈" },
  { href: "/admin/settings", label: "Configurações", icon: "⚙️" },
];

export default function AdminSidebar() {
  const pathname = usePathname();

  return (
    <aside style={{
      width: 240, flexShrink: 0, height: "100vh", background: "#0F172A",
      color: "#E2E8F0", display: "flex", flexDirection: "column",
      position: "sticky", top: 0,
    }}>
      <div style={{ padding: "22px 20px", borderBottom: "1px solid #1E293B" }}>
        <div style={{ fontSize: 18, fontWeight: 800, color: "#FFFFFF", letterSpacing: 0.3 }}>
          SOS <span style={{ color: "#38BDF8" }}>Admin</span>
        </div>
      </div>

      <nav style={{ flex: 1, padding: "12px 10px", overflowY: "auto" }}>
        {NAV.map((item) => {
          const active = pathname === item.href ||
            (item.href !== "/admin" && pathname?.startsWith(item.href));
          return (
            <Link
              key={item.href}
              href={item.href}
              style={{
                display: "flex", alignItems: "center", gap: 10,
                padding: "10px 12px", borderRadius: 8, marginBottom: 2,
                fontSize: 14, fontWeight: active ? 700 : 500,
                color: active ? "#FFFFFF" : "#94A3B8",
                background: active ? "#1E293B" : "transparent",
                textDecoration: "none",
                borderLeft: active ? "3px solid #38BDF8" : "3px solid transparent",
              }}
            >
              <span style={{ fontSize: 16 }}>{item.icon}</span>
              {item.label}
            </Link>
          );
        })}
      </nav>

      <div style={{ padding: 16, borderTop: "1px solid #1E293B", fontSize: 11, color: "#64748B" }}>
        SOS Serviços © {new Date().getFullYear()}
      </div>
    </aside>
  );
}
```

---

## 7. `components/admin/AdminTopbar.tsx` ← NOVO

```tsx
"use client";

import { useRouter } from "next/navigation";

type Props = {
  title: string;
  admin: { name: string; email: string; avatarUrl: string | null };
};

export default function AdminTopbar({ title, admin }: Props) {
  const router = useRouter();

  return (
    <header style={{
      height: 64, display: "flex", alignItems: "center", justifyContent: "space-between",
      padding: "0 24px", borderBottom: "1px solid #E2E8F0", background: "#FFFFFF",
      position: "sticky", top: 0, zIndex: 10,
    }}>
      <h1 style={{ fontSize: 19, fontWeight: 700, color: "#0F172A", margin: 0 }}>{title}</h1>

      <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
        <button
          onClick={() => router.push("/")}
          style={{
            fontSize: 13, fontWeight: 600, color: "#475569", background: "#F1F5F9",
            border: "none", borderRadius: 8, padding: "8px 14px", cursor: "pointer",
          }}
        >
          ← Voltar ao site
        </button>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <div style={{
            width: 34, height: 34, borderRadius: "50%", background: "#38BDF8",
            display: "flex", alignItems: "center", justifyContent: "center",
            color: "#FFF", fontWeight: 700, fontSize: 14, overflow: "hidden",
          }}>
            {admin.avatarUrl
              ? <img src={admin.avatarUrl} alt={admin.name} style={{ width: "100%", height: "100%", objectFit: "cover" }} />
              : admin.name.charAt(0).toUpperCase()}
          </div>
          <div style={{ fontSize: 13, fontWeight: 600, color: "#0F172A" }}>{admin.name}</div>
        </div>
      </div>
    </header>
  );
}
```

---

## 8. `components/admin/StatCard.tsx` ← NOVO

```tsx
type Props = {
  label: string;
  value: string;
  icon: string;
  trend?: { value: number; positive: boolean };
  accent?: string;
};

export default function StatCard({ label, value, icon, trend, accent = "#38BDF8" }: Props) {
  return (
    <div style={{
      background: "#FFFFFF", borderRadius: 14, padding: 20, border: "1px solid #E2E8F0",
      display: "flex", flexDirection: "column", gap: 10, minWidth: 0,
    }}>
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between" }}>
        <span style={{ fontSize: 13, fontWeight: 600, color: "#64748B" }}>{label}</span>
        <div style={{
          width: 34, height: 34, borderRadius: 9, background: `${accent}1A`,
          display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16,
        }}>
          {icon}
        </div>
      </div>
      <div style={{ fontSize: 26, fontWeight: 800, color: "#0F172A" }}>{value}</div>
      {trend && (
        <div style={{ fontSize: 12, fontWeight: 600, color: trend.positive ? "#16A34A" : "#DC2626" }}>
          {trend.positive ? "▲" : "▼"} {Math.abs(trend.value)}% vs mês anterior
        </div>
      )}
    </div>
  );
}
```

---

## 9. `components/admin/RevenueChart.tsx` ← NOVO

```tsx
"use client";

type Props = {
  data: { month: string; cents: number }[];
};

export default function RevenueChart({ data }: Props) {
  const max = Math.max(...data.map((d) => d.cents), 1);

  return (
    <div style={{ background: "#FFFFFF", borderRadius: 14, border: "1px solid #E2E8F0", padding: 20 }}>
      <div style={{ fontSize: 15, fontWeight: 700, color: "#0F172A", marginBottom: 18 }}>
        Receita mensal
      </div>
      <div style={{ display: "flex", alignItems: "flex-end", gap: 14, height: 180 }}>
        {data.map((d) => (
          <div key={d.month} style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", gap: 8 }}>
            <div style={{ fontSize: 11, fontWeight: 700, color: "#0F172A" }}>
              R$ {(d.cents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 0 })}
            </div>
            <div style={{
              width: "100%", maxWidth: 36,
              height: Math.max((d.cents / max) * 130, 4),
              background: "linear-gradient(180deg, #38BDF8, #0EA5E9)",
              borderRadius: 6,
            }} />
            <div style={{ fontSize: 12, color: "#64748B", textTransform: "capitalize" }}>{d.month}</div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 10. `components/admin/StatusBadge.tsx` ← NOVO

```tsx
const COLORS: Record<string, { bg: string; fg: string; label: string }> = {
  pending:   { bg: "#FEF3C7", fg: "#92400E", label: "Pendente" },
  approved:  { bg: "#DCFCE7", fg: "#166534", label: "Aprovado" },
  rejected:  { bg: "#FEE2E2", fg: "#991B1B", label: "Rejeitado" },
  active:    { bg: "#DCFCE7", fg: "#166534", label: "Ativo" },
  blocked:   { bg: "#FEE2E2", fg: "#991B1B", label: "Bloqueado" },
  paid:      { bg: "#DCFCE7", fg: "#166534", label: "Pago" },
  refunded:  { bg: "#E0E7FF", fg: "#3730A3", label: "Reembolsado" },
  failed:    { bg: "#FEE2E2", fg: "#991B1B", label: "Falhou" },
  completed: { bg: "#DCFCE7", fg: "#166534", label: "Concluído" },
  cancelled: { bg: "#F1F5F9", fg: "#475569", label: "Cancelado" },
  in_progress: { bg: "#DBEAFE", fg: "#1E40AF", label: "Em andamento" },
  admin:     { bg: "#EDE9FE", fg: "#5B21B6", label: "Admin" },
  professional: { bg: "#DBEAFE", fg: "#1E40AF", label: "Profissional" },
  user:      { bg: "#F1F5F9", fg: "#475569", label: "Usuário" },
};

export default function StatusBadge({ status }: { status: string }) {
  const c = COLORS[status] ?? { bg: "#F1F5F9", fg: "#475569", label: status };
  return (
    <span style={{
      display: "inline-flex", alignItems: "center", padding: "4px 10px",
      borderRadius: 999, fontSize: 12, fontWeight: 700, background: c.bg, color: c.fg,
      whiteSpace: "nowrap",
    }}>
      {c.label}
    </span>
  );
}
```

---

## 11. `components/admin/DataTable.tsx` ← NOVO

```tsx
"use client";

type Column<T> = {
  key: string;
  header: string;
  render: (row: T) => React.ReactNode;
  width?: string;
};

type Props<T> = {
  columns: Column<T>[];
  rows: T[];
  emptyMessage?: string;
  page: number;
  totalPages: number;
  onPageChange: (page: number) => void;
};

export default function DataTable<T extends { id: string }>({
  columns, rows, emptyMessage = "Nenhum registro encontrado.", page, totalPages, onPageChange,
}: Props<T>) {
  return (
    <div style={{ background: "#FFFFFF", borderRadius: 14, border: "1px solid #E2E8F0", overflow: "hidden" }}>
      <div style={{ overflowX: "auto" }}>
        <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 14 }}>
          <thead>
            <tr style={{ background: "#F8FAFC", borderBottom: "1px solid #E2E8F0" }}>
              {columns.map((col) => (
                <th key={col.key} style={{
                  textAlign: "left", padding: "12px 16px", fontSize: 12, fontWeight: 700,
                  color: "#64748B", textTransform: "uppercase", letterSpacing: 0.4,
                  width: col.width,
                }}>
                  {col.header}
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {rows.length === 0 ? (
              <tr>
                <td colSpan={columns.length} style={{ padding: 40, textAlign: "center", color: "#94A3B8" }}>
                  {emptyMessage}
                </td>
              </tr>
            ) : (
              rows.map((row) => (
                <tr key={row.id} style={{ borderBottom: "1px solid #F1F5F9" }}>
                  {columns.map((col) => (
                    <td key={col.key} style={{ padding: "14px 16px", color: "#1E293B" }}>
                      {col.render(row)}
                    </td>
                  ))}
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>

      {totalPages > 1 && (
        <div style={{
          display: "flex", alignItems: "center", justifyContent: "center", gap: 6,
          padding: "14px 16px", borderTop: "1px solid #E2E8F0",
        }}>
          <button
            disabled={page <= 1}
            onClick={() => onPageChange(page - 1)}
            style={{
              padding: "6px 12px", borderRadius: 6, border: "1px solid #E2E8F0",
              background: "#FFF", cursor: page <= 1 ? "not-allowed" : "pointer",
              opacity: page <= 1 ? 0.4 : 1, fontSize: 13,
            }}
          >
            ‹
          </button>
          <span style={{ fontSize: 13, color: "#475569", padding: "0 8px" }}>
            Página {page} de {totalPages}
          </span>
          <button
            disabled={page >= totalPages}
            onClick={() => onPageChange(page + 1)}
            style={{
              padding: "6px 12px", borderRadius: 6, border: "1px solid #E2E8F0",
              background: "#FFF", cursor: page >= totalPages ? "not-allowed" : "pointer",
              opacity: page >= totalPages ? 0.4 : 1, fontSize: 13,
            }}
          >
            ›
          </button>
        </div>
      )}
    </div>
  );
}
```

---

## 12. `components/admin/ConfirmDialog.tsx` ← NOVO

```tsx
"use client";

type Props = {
  open: boolean;
  title: string;
  message: string;
  confirmLabel?: string;
  danger?: boolean;
  onConfirm: () => void;
  onCancel: () => void;
  loading?: boolean;
};

export default function ConfirmDialog({
  open, title, message, confirmLabel = "Confirmar", danger, onConfirm, onCancel, loading,
}: Props) {
  if (!open) return null;

  return (
    <div style={{
      position: "fixed", inset: 0, background: "rgba(15,23,42,0.5)",
      display: "flex", alignItems: "center", justifyContent: "center", zIndex: 100,
    }}>
      <div style={{
        background: "#FFF", borderRadius: 14, padding: 24, width: 380, maxWidth: "90vw",
        boxShadow: "0 20px 40px rgba(0,0,0,0.2)",
      }}>
        <div style={{ fontSize: 17, fontWeight: 700, color: "#0F172A", marginBottom: 8 }}>{title}</div>
        <div style={{ fontSize: 14, color: "#475569", marginBottom: 22, lineHeight: 1.5 }}>{message}</div>
        <div style={{ display: "flex", gap: 10, justifyContent: "flex-end" }}>
          <button
            onClick={onCancel}
            disabled={loading}
            style={{
              padding: "10px 16px", borderRadius: 8, border: "1px solid #E2E8F0",
              background: "#FFF", fontSize: 14, fontWeight: 600, color: "#475569", cursor: "pointer",
            }}
          >
            Cancelar
          </button>
          <button
            onClick={onConfirm}
            disabled={loading}
            style={{
              padding: "10px 16px", borderRadius: 8, border: "none",
              background: danger ? "#DC2626" : "#0F172A", color: "#FFF",
              fontSize: 14, fontWeight: 600, cursor: "pointer", opacity: loading ? 0.6 : 1,
            }}
          >
            {loading ? "Aguarde..." : confirmLabel}
          </button>
        </div>
      </div>
    </div>
  );
}
```

---

## 13. `app/admin/layout.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import AdminSidebar from "@/components/admin/AdminSidebar";

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  await requireAdmin();

  return (
    <div style={{ display: "flex", minHeight: "100vh", background: "#F8FAFC" }}>
      <AdminSidebar />
      <div style={{ flex: 1, minWidth: 0 }}>{children}</div>
    </div>
  );
}
```

---

## 14. `app/admin/page.tsx` ← NOVO (Dashboard)

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import { getDashboardStatsAction } from "@/actions/admin";
import AdminTopbar from "@/components/admin/AdminTopbar";
import StatCard from "@/components/admin/StatCard";
import RevenueChart from "@/components/admin/RevenueChart";
import Link from "next/link";

export default async function AdminDashboard() {
  const admin = await requireAdmin();
  const res = await getDashboardStatsAction();
  const stats = res.success ? res.data! : null;

  return (
    <>
      <AdminTopbar title="Dashboard" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1280 }}>
        {!stats ? (
          <div style={{ color: "#DC2626" }}>Erro ao carregar estatísticas.</div>
        ) : (
          <>
            <div style={{
              display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(200px, 1fr))",
              gap: 16, marginBottom: 24,
            }}>
              <StatCard label="Usuários" value={stats.totalUsers.toLocaleString("pt-BR")} icon="👥" accent="#38BDF8" />
              <StatCard label="Profissionais" value={stats.totalProfessionals.toLocaleString("pt-BR")} icon="🛠️" accent="#A78BFA" />
              <StatCard
                label="Receita mensal"
                value={`R$ ${(stats.monthlyRevenueCents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 2 })}`}
                icon="💰" accent="#34D399"
              />
              <StatCard label="Pedidos" value={stats.totalRequests.toLocaleString("pt-BR")} icon="📋" accent="#FB923C" />
              <StatCard label="Assinaturas ativas" value={stats.activeSubscriptions.toLocaleString("pt-BR")} icon="💳" accent="#F472B6" />
            </div>

            {stats.pendingProfessionals > 0 && (
              <Link
                href="/admin/professionals?status=pending"
                style={{
                  display: "flex", alignItems: "center", justifyContent: "space-between",
                  background: "#FEF3C7", border: "1px solid #FDE68A", borderRadius: 12,
                  padding: "14px 18px", marginBottom: 24, textDecoration: "none",
                }}
              >
                <span style={{ fontSize: 14, fontWeight: 600, color: "#92400E" }}>
                  ⚠️ {stats.pendingProfessionals} profissional(is) aguardando aprovação
                </span>
                <span style={{ fontSize: 13, fontWeight: 700, color: "#92400E" }}>Revisar →</span>
              </Link>
            )}

            <div style={{ display: "grid", gridTemplateColumns: "2fr 1fr", gap: 16 }}>
              <RevenueChart data={stats.revenueByMonth} />
              <div style={{ background: "#FFFFFF", borderRadius: 14, border: "1px solid #E2E8F0", padding: 20 }}>
                <div style={{ fontSize: 15, fontWeight: 700, color: "#0F172A", marginBottom: 18 }}>
                  Pedidos por mês
                </div>
                {stats.requestsByMonth.map((m) => (
                  <div key={m.month} style={{ display: "flex", justifyContent: "space-between", padding: "8px 0", borderBottom: "1px solid #F1F5F9" }}>
                    <span style={{ fontSize: 13, color: "#64748B", textTransform: "capitalize" }}>{m.month}</span>
                    <span style={{ fontSize: 13, fontWeight: 700, color: "#0F172A" }}>{m.count}</span>
                  </div>
                ))}
              </div>
            </div>
          </>
        )}
      </main>
    </>
  );
}
```

---

## 15. `app/admin/users/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import AdminTopbar from "@/components/admin/AdminTopbar";
import UsersClient from "./UsersClient";

export default async function UsersPage() {
  const admin = await requireAdmin();
  return (
    <>
      <AdminTopbar title="Usuários" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1280 }}>
        <UsersClient />
      </main>
    </>
  );
}
```

### `app/admin/users/UsersClient.tsx` ← NOVO

```tsx
"use client";

import { useEffect, useState, useCallback } from "react";
import { listUsersAction, toggleBlockUserAction, changeUserRoleAction } from "@/actions/admin";
import DataTable from "@/components/admin/DataTable";
import StatusBadge from "@/components/admin/StatusBadge";
import ConfirmDialog from "@/components/admin/ConfirmDialog";

type UserRow = {
  id: string; name: string; email: string; avatarUrl: string | null;
  role: string; blocked: boolean; createdAt: string; phone: string | null;
};

export default function UsersClient() {
  const [users, setUsers] = useState<UserRow[]>([]);
  const [total, setTotal] = useState(0);
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState("");
  const [role, setRole] = useState("");
  const [loading, setLoading] = useState(true);
  const [confirmTarget, setConfirmTarget] = useState<UserRow | null>(null);
  const [actionLoading, setActionLoading] = useState(false);

  const load = useCallback(async () => {
    setLoading(true);
    const res = await listUsersAction({ search, role: role || undefined, page });
    if (res.success) {
      setUsers(res.data!.users as any);
      setTotal(res.data!.total);
    }
    setLoading(false);
  }, [search, role, page]);

  useEffect(() => { load(); }, [load]);

  async function handleToggleBlock() {
    if (!confirmTarget) return;
    setActionLoading(true);
    await toggleBlockUserAction(confirmTarget.id, !confirmTarget.blocked);
    setActionLoading(false);
    setConfirmTarget(null);
    load();
  }

  async function handleRoleChange(userId: string, newRole: string) {
    await changeUserRoleAction(userId, newRole as any);
    load();
  }

  const totalPages = Math.max(1, Math.ceil(total / 20));

  return (
    <>
      <div style={{ display: "flex", gap: 10, marginBottom: 18, flexWrap: "wrap" }}>
        <input
          placeholder="Buscar por nome ou e-mail..."
          value={search}
          onChange={(e) => { setSearch(e.target.value); setPage(1); }}
          style={{
            flex: 1, minWidth: 220, padding: "10px 14px", borderRadius: 8,
            border: "1px solid #E2E8F0", fontSize: 14,
          }}
        />
        <select
          value={role}
          onChange={(e) => { setRole(e.target.value); setPage(1); }}
          style={{ padding: "10px 14px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        >
          <option value="">Todos os papéis</option>
          <option value="user">Usuário</option>
          <option value="professional">Profissional</option>
          <option value="admin">Admin</option>
        </select>
      </div>

      <DataTable
        columns={[
          {
            key: "name", header: "Usuário",
            render: (u) => (
              <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                <div style={{
                  width: 32, height: 32, borderRadius: "50%", background: "#E2E8F0",
                  display: "flex", alignItems: "center", justifyContent: "center",
                  fontWeight: 700, fontSize: 13, color: "#475569", overflow: "hidden", flexShrink: 0,
                }}>
                  {u.avatarUrl ? <img src={u.avatarUrl} style={{ width: "100%", height: "100%", objectFit: "cover" }} /> : u.name.charAt(0)}
                </div>
                <div>
                  <div style={{ fontWeight: 600, fontSize: 14 }}>{u.name}</div>
                  <div style={{ fontSize: 12, color: "#64748B" }}>{u.email}</div>
                </div>
              </div>
            ),
          },
          {
            key: "role", header: "Papel",
            render: (u) => (
              <select
                defaultValue={u.role}
                onChange={(e) => handleRoleChange(u.id, e.target.value)}
                style={{ padding: "6px 10px", borderRadius: 6, border: "1px solid #E2E8F0", fontSize: 13 }}
              >
                <option value="user">Usuário</option>
                <option value="professional">Profissional</option>
                <option value="admin">Admin</option>
              </select>
            ),
          },
          {
            key: "status", header: "Status",
            render: (u) => <StatusBadge status={u.blocked ? "blocked" : "active"} />,
          },
          {
            key: "createdAt", header: "Cadastro",
            render: (u) => new Date(u.createdAt).toLocaleDateString("pt-BR"),
          },
          {
            key: "actions", header: "",
            render: (u) => (
              <button
                onClick={() => setConfirmTarget(u)}
                style={{
                  padding: "6px 12px", borderRadius: 6, border: "none", fontSize: 12, fontWeight: 700,
                  background: u.blocked ? "#DCFCE7" : "#FEE2E2",
                  color: u.blocked ? "#166534" : "#991B1B", cursor: "pointer",
                }}
              >
                {u.blocked ? "Desbloquear" : "Bloquear"}
              </button>
            ),
          },
        ]}
        rows={loading ? [] : users}
        emptyMessage={loading ? "Carregando..." : "Nenhum usuário encontrado."}
        page={page}
        totalPages={totalPages}
        onPageChange={setPage}
      />

      <ConfirmDialog
        open={!!confirmTarget}
        title={confirmTarget?.blocked ? "Desbloquear usuário" : "Bloquear usuário"}
        message={`Tem certeza que deseja ${confirmTarget?.blocked ? "desbloquear" : "bloquear"} ${confirmTarget?.name}?`}
        confirmLabel={confirmTarget?.blocked ? "Desbloquear" : "Bloquear"}
        danger={!confirmTarget?.blocked}
        loading={actionLoading}
        onConfirm={handleToggleBlock}
        onCancel={() => setConfirmTarget(null)}
      />
    </>
  );
}
```

---

## 16. `app/admin/professionals/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import AdminTopbar from "@/components/admin/AdminTopbar";
import ProfessionalsClient from "./ProfessionalsClient";

export default async function ProfessionalsPage() {
  const admin = await requireAdmin();
  return (
    <>
      <AdminTopbar title="Profissionais" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1280 }}>
        <ProfessionalsClient />
      </main>
    </>
  );
}
```

### `app/admin/professionals/ProfessionalsClient.tsx` ← NOVO

```tsx
"use client";

import { useEffect, useState, useCallback } from "react";
import { useSearchParams } from "next/navigation";
import {
  listProfessionalsAction, approveProfessionalAction,
  rejectProfessionalAction, blockProfessionalAction,
} from "@/actions/admin";
import DataTable from "@/components/admin/DataTable";
import StatusBadge from "@/components/admin/StatusBadge";
import ConfirmDialog from "@/components/admin/ConfirmDialog";

export default function ProfessionalsClient() {
  const searchParams = useSearchParams();
  const [pros, setPros] = useState<any[]>([]);
  const [total, setTotal] = useState(0);
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState("");
  const [status, setStatus] = useState(searchParams.get("status") ?? "");
  const [loading, setLoading] = useState(true);
  const [rejectTarget, setRejectTarget] = useState<any>(null);
  const [rejectReason, setRejectReason] = useState("");
  const [actionLoading, setActionLoading] = useState(false);

  const load = useCallback(async () => {
    setLoading(true);
    const res = await listProfessionalsAction({ search, status: status || undefined, page });
    if (res.success) {
      setPros(res.data!.professionals);
      setTotal(res.data!.total);
    }
    setLoading(false);
  }, [search, status, page]);

  useEffect(() => { load(); }, [load]);

  async function handleApprove(id: string) {
    await approveProfessionalAction(id);
    load();
  }

  async function handleReject() {
    if (!rejectTarget) return;
    setActionLoading(true);
    await rejectProfessionalAction(rejectTarget.id, rejectReason || "Não atende aos requisitos");
    setActionLoading(false);
    setRejectTarget(null);
    setRejectReason("");
    load();
  }

  async function handleBlock(p: any) {
    await blockProfessionalAction(p.id, p.userId, !p.user.blocked);
    load();
  }

  const totalPages = Math.max(1, Math.ceil(total / 20));

  return (
    <>
      <div style={{ display: "flex", gap: 10, marginBottom: 18, flexWrap: "wrap" }}>
        <input
          placeholder="Buscar por nome ou e-mail..."
          value={search}
          onChange={(e) => { setSearch(e.target.value); setPage(1); }}
          style={{ flex: 1, minWidth: 220, padding: "10px 14px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        />
        <select
          value={status}
          onChange={(e) => { setStatus(e.target.value); setPage(1); }}
          style={{ padding: "10px 14px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        >
          <option value="">Todos os status</option>
          <option value="pending">Pendente</option>
          <option value="approved">Aprovado</option>
          <option value="rejected">Rejeitado</option>
        </select>
      </div>

      <DataTable
        columns={[
          {
            key: "name", header: "Profissional",
            render: (p: any) => (
              <div>
                <div style={{ fontWeight: 600, fontSize: 14 }}>{p.user.name}</div>
                <div style={{ fontSize: 12, color: "#64748B" }}>{p.user.email}</div>
              </div>
            ),
          },
          { key: "category", header: "Categoria", render: (p: any) => p.category ?? "—" },
          { key: "status", header: "Aprovação", render: (p: any) => <StatusBadge status={p.approvalStatus} /> },
          { key: "blocked", header: "Conta", render: (p: any) => <StatusBadge status={p.user.blocked ? "blocked" : "active"} /> },
          {
            key: "actions", header: "Ações",
            render: (p: any) => (
              <div style={{ display: "flex", gap: 6 }}>
                {p.approvalStatus !== "approved" && (
                  <button
                    onClick={() => handleApprove(p.id)}
                    style={{ padding: "6px 10px", borderRadius: 6, border: "none", fontSize: 12, fontWeight: 700, background: "#DCFCE7", color: "#166534", cursor: "pointer" }}
                  >
                    Aprovar
                  </button>
                )}
                {p.approvalStatus !== "rejected" && (
                  <button
                    onClick={() => setRejectTarget(p)}
                    style={{ padding: "6px 10px", borderRadius: 6, border: "none", fontSize: 12, fontWeight: 700, background: "#FEF3C7", color: "#92400E", cursor: "pointer" }}
                  >
                    Rejeitar
                  </button>
                )}
                <button
                  onClick={() => handleBlock(p)}
                  style={{ padding: "6px 10px", borderRadius: 6, border: "none", fontSize: 12, fontWeight: 700, background: p.user.blocked ? "#DBEAFE" : "#FEE2E2", color: p.user.blocked ? "#1E40AF" : "#991B1B", cursor: "pointer" }}
                >
                  {p.user.blocked ? "Desbloquear" : "Bloquear"}
                </button>
              </div>
            ),
          },
        ]}
        rows={loading ? [] : pros}
        emptyMessage={loading ? "Carregando..." : "Nenhum profissional encontrado."}
        page={page}
        totalPages={totalPages}
        onPageChange={setPage}
      />

      <ConfirmDialog
        open={!!rejectTarget}
        title="Rejeitar profissional"
        message="Informe o motivo da rejeição (opcional):"
        confirmLabel="Rejeitar"
        danger
        loading={actionLoading}
        onConfirm={handleReject}
        onCancel={() => { setRejectTarget(null); setRejectReason(""); }}
      />
    </>
  );
}
```

---

## 17. `app/admin/services/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import { listServicesAction } from "@/actions/admin";
import AdminTopbar from "@/components/admin/AdminTopbar";
import StatusBadge from "@/components/admin/StatusBadge";

export default async function ServicesPage({
  searchParams,
}: { searchParams: Promise<{ page?: string; status?: string }> }) {
  const admin = await requireAdmin();
  const sp = await searchParams;
  const page = Number(sp.page ?? 1);
  const res = await listServicesAction({ status: sp.status, page });
  const services = res.success ? res.data!.services : [];
  const total = res.success ? res.data!.total : 0;
  const totalPages = Math.max(1, Math.ceil(total / 20));

  return (
    <>
      <AdminTopbar title="Pedidos" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1280 }}>
        <div style={{ background: "#FFFFFF", borderRadius: 14, border: "1px solid #E2E8F0", overflow: "hidden" }}>
          <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 14 }}>
            <thead>
              <tr style={{ background: "#F8FAFC", borderBottom: "1px solid #E2E8F0" }}>
                <th style={{ textAlign: "left", padding: "12px 16px", fontSize: 12, color: "#64748B" }}>Título</th>
                <th style={{ textAlign: "left", padding: "12px 16px", fontSize: 12, color: "#64748B" }}>Cliente</th>
                <th style={{ textAlign: "left", padding: "12px 16px", fontSize: 12, color: "#64748B" }}>Categoria</th>
                <th style={{ textAlign: "left", padding: "12px 16px", fontSize: 12, color: "#64748B" }}>Status</th>
                <th style={{ textAlign: "left", padding: "12px 16px", fontSize: 12, color: "#64748B" }}>Data</th>
              </tr>
            </thead>
            <tbody>
              {services.length === 0 ? (
                <tr><td colSpan={5} style={{ padding: 40, textAlign: "center", color: "#94A3B8" }}>Nenhum pedido encontrado.</td></tr>
              ) : services.map((s: any) => (
                <tr key={s.id} style={{ borderBottom: "1px solid #F1F5F9" }}>
                  <td style={{ padding: "14px 16px", fontWeight: 600 }}>{s.title}</td>
                  <td style={{ padding: "14px 16px", color: "#64748B" }}>{s.client?.name}</td>
                  <td style={{ padding: "14px 16px", color: "#64748B" }}>{s.category?.name ?? "—"}</td>
                  <td style={{ padding: "14px 16px" }}><StatusBadge status={s.status} /></td>
                  <td style={{ padding: "14px 16px", color: "#64748B" }}>{new Date(s.createdAt).toLocaleDateString("pt-BR")}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
        {totalPages > 1 && (
          <div style={{ display: "flex", justifyContent: "center", gap: 8, marginTop: 16, fontSize: 13, color: "#64748B" }}>
            Página {page} de {totalPages}
          </div>
        )}
      </main>
    </>
  );
}
```

---

## 18. `app/admin/categories/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import AdminTopbar from "@/components/admin/AdminTopbar";
import CategoriesClient from "./CategoriesClient";

export default async function CategoriesPage() {
  const admin = await requireAdmin();
  return (
    <>
      <AdminTopbar title="Categorias" admin={admin} />
      <main style={{ padding: 24, maxWidth: 900 }}>
        <CategoriesClient />
      </main>
    </>
  );
}
```

### `app/admin/categories/CategoriesClient.tsx` ← NOVO

```tsx
"use client";

import { useEffect, useState } from "react";
import {
  listCategoriesAction, createCategoryAction,
  updateCategoryAction, deleteCategoryAction,
} from "@/actions/admin";
import ConfirmDialog from "@/components/admin/ConfirmDialog";

function slugify(s: string) {
  return s.toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "")
    .replace(/[^a-z0-9]+/g, "-").replace(/(^-|-$)/g, "");
}

export default function CategoriesClient() {
  const [categories, setCategories] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);
  const [name, setName] = useState("");
  const [icon, setIcon] = useState("");
  const [editing, setEditing] = useState<any>(null);
  const [deleteTarget, setDeleteTarget] = useState<any>(null);

  async function load() {
    setLoading(true);
    const res = await listCategoriesAction();
    if (res.success) setCategories(res.data!);
    setLoading(false);
  }

  useEffect(() => { load(); }, []);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!name.trim()) return;

    if (editing) {
      await updateCategoryAction(editing.id, { name, icon, slug: slugify(name) });
    } else {
      await createCategoryAction({ name, icon, slug: slugify(name), orderIndex: categories.length });
    }
    setName(""); setIcon(""); setEditing(null);
    load();
  }

  async function toggleActive(c: any) {
    await updateCategoryAction(c.id, { active: !c.active });
    load();
  }

  async function handleDelete() {
    if (!deleteTarget) return;
    await deleteCategoryAction(deleteTarget.id);
    setDeleteTarget(null);
    load();
  }

  return (
    <>
      <form onSubmit={handleSubmit} style={{
        display: "flex", gap: 10, marginBottom: 20, background: "#FFF",
        border: "1px solid #E2E8F0", borderRadius: 12, padding: 16,
      }}>
        <input
          placeholder="Ícone (emoji)"
          value={icon}
          onChange={(e) => setIcon(e.target.value)}
          style={{ width: 90, padding: "10px 12px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        />
        <input
          placeholder="Nome da categoria"
          value={name}
          onChange={(e) => setName(e.target.value)}
          style={{ flex: 1, padding: "10px 12px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        />
        <button type="submit" style={{
          padding: "10px 18px", borderRadius: 8, border: "none", background: "#0F172A",
          color: "#FFF", fontWeight: 600, fontSize: 14, cursor: "pointer",
        }}>
          {editing ? "Salvar" : "Adicionar"}
        </button>
        {editing && (
          <button type="button" onClick={() => { setEditing(null); setName(""); setIcon(""); }} style={{
            padding: "10px 14px", borderRadius: 8, border: "1px solid #E2E8F0",
            background: "#FFF", fontSize: 14, cursor: "pointer",
          }}>
            Cancelar
          </button>
        )}
      </form>

      <div style={{ background: "#FFF", border: "1px solid #E2E8F0", borderRadius: 12, overflow: "hidden" }}>
        {loading ? (
          <div style={{ padding: 30, textAlign: "center", color: "#94A3B8" }}>Carregando...</div>
        ) : categories.length === 0 ? (
          <div style={{ padding: 30, textAlign: "center", color: "#94A3B8" }}>Nenhuma categoria cadastrada.</div>
        ) : (
          categories.map((c) => (
            <div key={c.id} style={{
              display: "flex", alignItems: "center", justifyContent: "space-between",
              padding: "14px 16px", borderBottom: "1px solid #F1F5F9",
            }}>
              <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
                <span style={{ fontSize: 20 }}>{c.icon}</span>
                <span style={{ fontWeight: 600, fontSize: 14, color: c.active ? "#0F172A" : "#94A3B8" }}>{c.name}</span>
              </div>
              <div style={{ display: "flex", gap: 8 }}>
                <button
                  onClick={() => toggleActive(c)}
                  style={{
                    padding: "6px 10px", borderRadius: 6, border: "none", fontSize: 12, fontWeight: 700,
                    background: c.active ? "#DCFCE7" : "#F1F5F9", color: c.active ? "#166534" : "#64748B", cursor: "pointer",
                  }}
                >
                  {c.active ? "Ativa" : "Inativa"}
                </button>
                <button
                  onClick={() => { setEditing(c); setName(c.name); setIcon(c.icon ?? ""); }}
                  style={{ padding: "6px 10px", borderRadius: 6, border: "1px solid #E2E8F0", background: "#FFF", fontSize: 12, cursor: "pointer" }}
                >
                  Editar
                </button>
                <button
                  onClick={() => setDeleteTarget(c)}
                  style={{ padding: "6px 10px", borderRadius: 6, border: "none", background: "#FEE2E2", color: "#991B1B", fontSize: 12, fontWeight: 700, cursor: "pointer" }}
                >
                  Excluir
                </button>
              </div>
            </div>
          ))
        )}
      </div>

      <ConfirmDialog
        open={!!deleteTarget}
        title="Excluir categoria"
        message={`Excluir "${deleteTarget?.name}"? Esta ação não pode ser desfeita.`}
        confirmLabel="Excluir"
        danger
        onConfirm={handleDelete}
        onCancel={() => setDeleteTarget(null)}
      />
    </>
  );
}
```

---

## 19. `app/admin/payments/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import AdminTopbar from "@/components/admin/AdminTopbar";
import PaymentsClient from "./PaymentsClient";

export default async function PaymentsPage() {
  const admin = await requireAdmin();
  return (
    <>
      <AdminTopbar title="Pagamentos" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1280 }}>
        <PaymentsClient />
      </main>
    </>
  );
}
```

### `app/admin/payments/PaymentsClient.tsx` ← NOVO

```tsx
"use client";

import { useEffect, useState, useCallback } from "react";
import { listPaymentsAction, refundPaymentAction } from "@/actions/admin";
import DataTable from "@/components/admin/DataTable";
import StatusBadge from "@/components/admin/StatusBadge";
import StatCard from "@/components/admin/StatCard";
import ConfirmDialog from "@/components/admin/ConfirmDialog";

export default function PaymentsClient() {
  const [payments, setPayments] = useState<any[]>([]);
  const [total, setTotal] = useState(0);
  const [totalCents, setTotalCents] = useState(0);
  const [page, setPage] = useState(1);
  const [status, setStatus] = useState("");
  const [search, setSearch] = useState("");
  const [loading, setLoading] = useState(true);
  const [refundTarget, setRefundTarget] = useState<any>(null);

  const load = useCallback(async () => {
    setLoading(true);
    const res = await listPaymentsAction({ status: status || undefined, search, page });
    if (res.success) {
      setPayments(res.data!.payments);
      setTotal(res.data!.total);
      setTotalCents(res.data!.totalCents);
    }
    setLoading(false);
  }, [status, search, page]);

  useEffect(() => { load(); }, [load]);

  async function handleRefund() {
    if (!refundTarget) return;
    await refundPaymentAction(refundTarget.id);
    setRefundTarget(null);
    load();
  }

  const totalPages = Math.max(1, Math.ceil(total / 20));

  return (
    <>
      <div style={{ marginBottom: 20, maxWidth: 260 }}>
        <StatCard label="Total recebido" value={`R$ ${(totalCents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 2 })}`} icon="💰" accent="#34D399" />
      </div>

      <div style={{ display: "flex", gap: 10, marginBottom: 18, flexWrap: "wrap" }}>
        <input
          placeholder="Buscar por nome ou e-mail..."
          value={search}
          onChange={(e) => { setSearch(e.target.value); setPage(1); }}
          style={{ flex: 1, minWidth: 220, padding: "10px 14px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        />
        <select
          value={status}
          onChange={(e) => { setStatus(e.target.value); setPage(1); }}
          style={{ padding: "10px 14px", borderRadius: 8, border: "1px solid #E2E8F0", fontSize: 14 }}
        >
          <option value="">Todos os status</option>
          <option value="paid">Pago</option>
          <option value="pending">Pendente</option>
          <option value="failed">Falhou</option>
          <option value="refunded">Reembolsado</option>
        </select>
      </div>

      <DataTable
        columns={[
          {
            key: "user", header: "Cliente",
            render: (p: any) => (
              <div>
                <div style={{ fontWeight: 600, fontSize: 14 }}>{p.user.name}</div>
                <div style={{ fontSize: 12, color: "#64748B" }}>{p.user.email}</div>
              </div>
            ),
          },
          {
            key: "amount", header: "Valor",
            render: (p: any) => `R$ ${(p.amountCents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 2 })}`,
          },
          { key: "method", header: "Método", render: (p: any) => p.method ?? "—" },
          { key: "status", header: "Status", render: (p: any) => <StatusBadge status={p.status} /> },
          { key: "date", header: "Data", render: (p: any) => new Date(p.createdAt).toLocaleDateString("pt-BR") },
          {
            key: "actions", header: "",
            render: (p: any) => p.status === "paid" ? (
              <button
                onClick={() => setRefundTarget(p)}
                style={{ padding: "6px 10px", borderRadius: 6, border: "none", fontSize: 12, fontWeight: 700, background: "#FEE2E2", color: "#991B1B", cursor: "pointer" }}
              >
                Reembolsar
              </button>
            ) : null,
          },
        ]}
        rows={loading ? [] : payments}
        emptyMessage={loading ? "Carregando..." : "Nenhum pagamento encontrado."}
        page={page}
        totalPages={totalPages}
        onPageChange={setPage}
      />

      <ConfirmDialog
        open={!!refundTarget}
        title="Reembolsar pagamento"
        message={`Reembolsar R$ ${refundTarget ? (refundTarget.amountCents / 100).toFixed(2) : ""} para ${refundTarget?.user.name}?`}
        confirmLabel="Reembolsar"
        danger
        onConfirm={handleRefund}
        onCancel={() => setRefundTarget(null)}
      />
    </>
  );
}
```

---

## 20. `app/admin/reports/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import { getReportsAction } from "@/actions/admin";
import AdminTopbar from "@/components/admin/AdminTopbar";
import StatCard from "@/components/admin/StatCard";

export default async function ReportsPage({
  searchParams,
}: { searchParams: Promise<{ from?: string; to?: string }> }) {
  const admin = await requireAdmin();
  const sp = await searchParams;
  const res = await getReportsAction({ from: sp.from, to: sp.to });
  const data = res.success ? res.data! : null;

  return (
    <>
      <AdminTopbar title="Relatórios" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1100 }}>
        <form style={{ display: "flex", gap: 10, marginBottom: 24 }}>
          <input type="date" name="from" defaultValue={sp.from} style={{ padding: "10px 12px", borderRadius: 8, border: "1px solid #E2E8F0" }} />
          <input type="date" name="to" defaultValue={sp.to} style={{ padding: "10px 12px", borderRadius: 8, border: "1px solid #E2E8F0" }} />
          <button type="submit" style={{ padding: "10px 18px", borderRadius: 8, border: "none", background: "#0F172A", color: "#FFF", fontWeight: 600, cursor: "pointer" }}>
            Filtrar
          </button>
        </form>

        {!data ? (
          <div style={{ color: "#DC2626" }}>Erro ao gerar relatório.</div>
        ) : (
          <>
            <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(200px,1fr))", gap: 16, marginBottom: 24 }}>
              <StatCard label="Receita no período" value={`R$ ${(data.totalRevenueCents / 100).toLocaleString("pt-BR", { minimumFractionDigits: 2 })}`} icon="💰" />
              <StatCard label="Pedidos no período" value={String(data.totalRequests)} icon="📋" />
              <StatCard label="Taxa de conclusão" value={`${data.conversionRate}%`} icon="✅" />
            </div>

            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 16 }}>
              <div style={{ background: "#FFF", border: "1px solid #E2E8F0", borderRadius: 14, padding: 20 }}>
                <div style={{ fontWeight: 700, marginBottom: 14 }}>Categorias mais demandadas</div>
                {data.topCategories.map((c, i) => (
                  <div key={i} style={{ display: "flex", justifyContent: "space-between", padding: "8px 0", borderBottom: "1px solid #F1F5F9" }}>
                    <span style={{ color: "#64748B" }}>{c.name}</span>
                    <span style={{ fontWeight: 700 }}>{c.count}</span>
                  </div>
                ))}
              </div>
              <div style={{ background: "#FFF", border: "1px solid #E2E8F0", borderRadius: 14, padding: 20 }}>
                <div style={{ fontWeight: 700, marginBottom: 14 }}>Top profissionais</div>
                {data.topProfessionals.map((p, i) => (
                  <div key={i} style={{ display: "flex", justifyContent: "space-between", padding: "8px 0", borderBottom: "1px solid #F1F5F9" }}>
                    <span style={{ color: "#64748B" }}>{p.name}</span>
                    <span style={{ fontWeight: 700 }}>{p.count}</span>
                  </div>
                ))}
              </div>
            </div>
          </>
        )}
      </main>
    </>
  );
}
```

---

## 21. `app/admin/settings/page.tsx` ← NOVO

```tsx
import { requireAdmin } from "@/lib/admin/auth";
import AdminTopbar from "@/components/admin/AdminTopbar";
import SettingsClient from "./SettingsClient";

export default async function SettingsPage() {
  const admin = await requireAdmin();
  return (
    <>
      <AdminTopbar title="Configurações" admin={admin} />
      <main style={{ padding: 24, maxWidth: 1000 }}>
        <SettingsClient />
      </main>
    </>
  );
}
```

### `app/admin/settings/SettingsClient.tsx` ← NOVO

```tsx
"use client";

import { useEffect, useState } from "react";
import { listPlansAction, savePlanAction, getAdminLogsAction } from "@/actions/admin";

export default function SettingsClient() {
  const [plans, setPlans] = useState<any[]>([]);
  const [logs, setLogs] = useState<any[]>([]);
  const [editing, setEditing] = useState<any>(null);
  const [form, setForm] = useState({ name: "", priceCents: 0, interval: "month", features: "", active: true });
  const [tab, setTab] = useState<"plans" | "logs">("plans");

  async function load() {
    const [p, l] = await Promise.all([listPlansAction(), getAdminLogsAction()]);
    if (p.success) setPlans(p.data!);
    if (l.success) setLogs(l.data!.logs);
  }

  useEffect(() => { load(); }, []);

  function startEdit(plan?: any) {
    if (plan) {
      setEditing(plan);
      setForm({
        name: plan.name, priceCents: plan.priceCents, interval: plan.interval,
        features: (plan.features as string[]).join("\n"), active: plan.active,
      });
    } else {
      setEditing({});
      setForm({ name: "", priceCents: 0, interval: "month", features: "", active: true });
    }
  }

  async function handleSave(e: React.FormEvent) {
    e.preventDefault();
    await savePlanAction({
      id: editing?.id,
      name: form.name,
      priceCents: Number(form.priceCents),
      interval: form.interval,
      features: form.features.split("\n").filter(Boolean),
      active: form.active,
    });
    setEditing(null);
    load();
  }

  return (
    <>
      <div style={{ display: "flex", gap: 8, marginBottom: 20 }}>
        {(["plans", "logs"] as const).map((t) => (
          <button
            key={t}
            onClick={() => setTab(t)}
            style={{
              padding: "8px 16px", borderRadius: 8, border: "none", fontSize: 14, fontWeight: 600,
              background: tab === t ? "#0F172A" : "#F1F5F9", color: tab === t ? "#FFF" : "#475569", cursor: "pointer",
            }}
          >
            {t === "plans" ? "Planos" : "Logs de auditoria"}
          </button>
        ))}
      </div>

      {tab === "plans" && (
        <>
          <button onClick={() => startEdit()} style={{ marginBottom: 16, padding: "10px 16px", borderRadius: 8, border: "none", background: "#0F172A", color: "#FFF", fontWeight: 600, cursor: "pointer" }}>
            + Novo plano
          </button>

          <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(260px,1fr))", gap: 16 }}>
            {plans.map((p) => (
              <div key={p.id} style={{ background: "#FFF", border: "1px solid #E2E8F0", borderRadius: 14, padding: 20 }}>
                <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 8 }}>
                  <div style={{ fontWeight: 700, fontSize: 16 }}>{p.name}</div>
                  <span style={{ fontSize: 12, fontWeight: 700, color: p.active ? "#166534" : "#94A3B8" }}>
                    {p.active ? "Ativo" : "Inativo"}
                  </span>
                </div>
                <div style={{ fontSize: 22, fontWeight: 800, marginBottom: 10 }}>
                  R$ {(p.priceCents / 100).toFixed(2)}<span style={{ fontSize: 13, color: "#64748B" }}>/{p.interval === "month" ? "mês" : p.interval}</span>
                </div>
                <ul style={{ fontSize: 13, color: "#475569", marginBottom: 14, paddingLeft: 18 }}>
                  {(p.features as string[]).map((f, i) => <li key={i}>{f}</li>)}
                </ul>
                <div style={{ fontSize: 12, color: "#94A3B8", marginBottom: 10 }}>
                  {p._count?.subscriptions ?? 0} assinante(s)
                </div>
                <button onClick={() => startEdit(p)} style={{ padding: "8px 14px", borderRadius: 8, border: "1px solid #E2E8F0", background: "#FFF", fontSize: 13, cursor: "pointer" }}>
                  Editar
                </button>
              </div>
            ))}
          </div>

          {editing && (
            <div style={{ position: "fixed", inset: 0, background: "rgba(15,23,42,0.5)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 100 }}>
              <form onSubmit={handleSave} style={{ background: "#FFF", borderRadius: 14, padding: 24, width: 420 }}>
                <div style={{ fontWeight: 700, fontSize: 17, marginBottom: 16 }}>{editing.id ? "Editar plano" : "Novo plano"}</div>
                <input placeholder="Nome" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} style={{ width: "100%", padding: 10, marginBottom: 10, borderRadius: 8, border: "1px solid #E2E8F0" }} />
                <input type="number" placeholder="Preço em centavos" value={form.priceCents} onChange={(e) => setForm({ ...form, priceCents: Number(e.target.value) })} style={{ width: "100%", padding: 10, marginBottom: 10, borderRadius: 8, border: "1px solid #E2E8F0" }} />
                <textarea placeholder="Recursos (um por linha)" value={form.features} onChange={(e) => setForm({ ...form, features: e.target.value })} rows={4} style={{ width: "100%", padding: 10, marginBottom: 10, borderRadius: 8, border: "1px solid #E2E8F0" }} />
                <label style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 16, fontSize: 14 }}>
                  <input type="checkbox" checked={form.active} onChange={(e) => setForm({ ...form, active: e.target.checked })} />
                  Plano ativo
                </label>
                <div style={{ display: "flex", gap: 10, justifyContent: "flex-end" }}>
                  <button type="button" onClick={() => setEditing(null)} style={{ padding: "10px 16px", borderRadius: 8, border: "1px solid #E2E8F0", background: "#FFF", cursor: "pointer" }}>Cancelar</button>
                  <button type="submit" style={{ padding: "10px 16px", borderRadius: 8, border: "none", background: "#0F172A", color: "#FFF", fontWeight: 600, cursor: "pointer" }}>Salvar</button>
                </div>
              </form>
            </div>
          )}
        </>
      )}

      {tab === "logs" && (
        <div style={{ background: "#FFF", border: "1px solid #E2E8F0", borderRadius: 14, overflow: "hidden" }}>
          {logs.length === 0 ? (
            <div style={{ padding: 30, textAlign: "center", color: "#94A3B8" }}>Nenhum log registrado.</div>
          ) : logs.map((l) => (
            <div key={l.id} style={{ padding: "12px 16px", borderBottom: "1px solid #F1F5F9", fontSize: 13 }}>
              <span style={{ fontWeight: 700 }}>{l.action}</span>
              <span style={{ color: "#64748B" }}> · {l.targetType} · {new Date(l.createdAt).toLocaleString("pt-BR")}</span>
            </div>
          ))}
        </div>
      )}
    </>
  );
}
```

---

## SETUP

```bash
# 1. Rodar a migration no SQL Editor do Supabase:
#    supabase/migrations/0005_admin.sql

# 2. Atualizar prisma/schema.prisma com os modelos e campos acima
npx prisma generate
npx prisma db push

# 3. Promover seu usuário a admin no SQL Editor:
update users set role = 'admin' where email = 'seu-email@exemplo.com';

# 4. Reiniciar:
npm run dev

# 5. Acessar:
#    http://localhost:3000/admin
```

---

**FIM — SOS_ADMIN_PAINEL_COMPLETO.md**
