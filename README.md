
DATABASE_URL="postgresql://user:password@localhost:5432/sutsa_crm"

// ===============================
// SUTSA CRM - FULL BACKEND SETUP
// Next.js + Prisma + tRPC
// ===============================

import { PrismaClient } from "@prisma/client";
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const prisma = new PrismaClient();
const t = initTRPC.create();

// ===============================
// 🧠 DATABASE MODELS (PRISMA)
// ===============================

/*
model Company {
  id            String   @id @default(uuid())
  name          String
  country       String
  city          String?
  address       String?
  phone         String?
  email         String?
  importVolume  Float?
  verified      Boolean  @default(false)
  createdAt     DateTime @default(now())
}

model Contact {
  id        String @id @default(uuid())
  firstName String
  lastName  String
  email     String?
  phone     String?
  companyId String
  company   Company @relation(fields: [companyId], references: [id])
}

model Product {
  id          String @id @default(uuid())
  name        String
  category    String
  price       Float?
  hsCode      String
  description String?
}

model Lead {
  id            String @id @default(uuid())
  companyId     String
  status        String
  lastContact   DateTime?
  opportunity   String
  createdAt     DateTime @default(now())
}

model Deal {
  id        String @id @default(uuid())
  leadId    String
  value     Float
  stage     String
  createdAt DateTime @default(now())
}
*/

// ===============================
// 🔌 VERITRADE API (MOCK)
// ===============================

async function fetchVeritradeImporters(country: string) {
  // 🔥 REEMPLAZAR CON API REAL
  return [
    {
      name: "PLASTEXTIL S.A.S",
      country,
      importVolume: 15598381,
      verified: true,
    },
    {
      name: "CALIPLASTICOS LTDA",
      country,
      importVolume: 55236246,
      verified: true,
    },
  ];
}

// ===============================
// 📡 tRPC ROUTERS
// ===============================

export const appRouter = t.router({

  // ===============================
  // 📊 IMPORTADORES
  // ===============================
  importers: t.procedure
    .input(z.object({ country: z.string() }))
    .query(async ({ input }) => {
      return fetchVeritradeImporters(input.country);
    }),

  // ===============================
  // 👥 PROSPECTOS
  // ===============================
  createLead: t.procedure
    .input(z.object({
      companyId: z.string(),
      status: z.string(),
      opportunity: z.string()
    }))
    .mutation(async ({ input }) => {
      return prisma.lead.create({ data: input });
    }),

  getLeads: t.procedure.query(async () => {
    return prisma.lead.findMany({
      include: { }
    });
  }),

  updateLeadStatus: t.procedure
    .input(z.object({
      id: z.string(),
      status: z.string()
    }))
    .mutation(async ({ input }) => {
      return prisma.lead.update({
        where: { id: input.id },
        data: { status: input.status }
      });
    }),

  // ===============================
  // 📦 PRODUCTOS
  // ===============================
  createProduct: t.procedure
    .input(z.object({
      name: z.string(),
      category: z.string(),
      hsCode: z.string(),
      price: z.number().optional()
    }))
    .mutation(async ({ input }) => {
      return prisma.product.create({ data: input });
    }),

  getProducts: t.procedure.query(async () => {
    return prisma.product.findMany();
  }),

  // ===============================
  // 💰 PIPELINE
  // ===============================
  createDeal: t.procedure
    .input(z.object({
      leadId: z.string(),
      value: z.number(),
      stage: z.string()
    }))
    .mutation(async ({ input }) => {
      return prisma.deal.create({ data: input });
    }),

  getPipeline: t.procedure.query(async () => {
    return prisma.deal.findMany();
  }),

  // ===============================
  // 📈 REPORTES
  // ===============================
  dashboard: t.procedure.query(async () => {

    const totalLeads = await prisma.lead.count();
    const deals = await prisma.deal.findMany();

    const totalRevenue = deals.reduce((acc, d) => acc + d.value, 0);

    return {
      totalLeads,
      totalRevenue,
      deals
    };
  }),

});

// ===============================
// 🧩 EXPORT
// ===============================
export type AppRouter = typeof appRouter;
