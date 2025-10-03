# PROMPT “AGENTE DE TIPOS SEMÂNTICOS” 

> Esse é o prompt para você usar no seu agente de programação para ele fazer o trabalho todo por você <3

**COPIE E COLE O TEXTO ABAIXO**


**Contexto**
Você é um agente de codificação encarregado de **inferir tipos semânticos atômicos** a partir de código TypeScript existente que hoje usa apenas **primitivos** (`boolean`, `number`, `string`, `Date`). Não assuma bibliotecas externas instaladas. Seu objetivo é:

1. detectar candidatos a **tipos semânticos**;
2. propor **nomes canônicos** no padrão `dominio.entidade.nome` (com ponto);
3. sugerir **regras e validações** mínimas;
4. gerar **artefatos** auto-contidos (sem dependências externas);
5. listar **até 5 novos tipos** que ainda **não existem** no repositório e que valem a pena padronizar.

> Use linguagem e comentários em **português**.

---

## Entradas que você tem

* Árvore de arquivos `.ts/.tsx` e testes.
* Nomes de variáveis, propriedades, funções, schemas, migrações, seeds.
* Strings literais, sufixos/prefixos, nomes de colunas/IDs e chamadas de API.

---

## Saídas obrigatórias (em ORDEM)

1. **Relatório** em Markdown: achados, suposições e conflitos.
2. **Manifesto** `semantic-types.manifest.json` com as inferências.
3. **Stubs** de tipos canônicos (arquivos `.ts` auto-contidos).
4. **Patches** (diff unificado) mostrando como aplicar os tipos.
5. **Roadmap** com até **5** tipos novos recomendados (ainda não mapeados).

---

## Padrão de Nomes (canônicos)

Use `kebab` no arquivo e **ponto** no brand:

* Arquivo: `domains/{Domain}/{Entity}/{name}.ts`
* Brand interno: `{domain}.{entity}.{name}`
  Ex.: `domains/ecommerce/order/total.ts` → brand `ecommerce.order.total`.

---

## Heurísticas de Inferência

### A) Booleans

Detecção por **prefixos** e contexto de uso:

* `is*`, `has*`, `can*`, `should*`, `enable*`, `allow*`, `show*`
* Exemplos canônicos:

  * `isDone` → `project.task.isDone`
  * `hasAllergies` → `health.patient.hasAllergies`
  * `showWeekViewCalendar` → `ui.calendar.showWeekView`
    Valide: coação para boolean, proibir `null/undefined` (ou explicitar `Maybe`), e **combinadores** só via funções (`and`, `or`) — nunca `&&` direto entre brands.

### B) Numbers

Use nomes, unidades, e padrões:

* Sufixos/palavras-chave: `total`, `amount`, `price`, `quantity`, `count`, `size`, `capacity`, `rate`, `percentage`, `score`, `weight`, `length`, `height`, `width`, `radius`, `duration`.
* Padrões literais: `%`, `ms`, `s`, `min`, `h`, `kg`, `g`, `m`, `cm`, `km`, `px`, `rem`, `brl`, `usd`.
* Exemplos:

  * `totalAppointments` → `clinic.schedule.totalAppointments` (inteiro ≥ 0)
  * `revenueTotal` → `ecommerce.order.revenueTotal.brl` (moeda BRL)
  * `teethExtracted` → `dentistry.procedure.teethExtracted` (inteiro 0–32)
  * `percentage` → `metrics.kpi.percentage` (0–100 ou 0–1 — explicitar escala)
  * `durationMs` → `time.duration.ms`
    Valide: **faixas** (min/max), **inteireza** quando nome implicar contagem, e **unidade** no nome canônico (ex.: `.brl`, `.usd`, `.kg`, `.ms`).

### C) Strings

Detecte **formatos**:

* `email`, `phone`, `url`, `slug`, `isoCode`, `cpf/cnpj`, `uuid`, `id` (se houver padrão).
* Regex úteis (documente no stub):

  * Email (robusto suficiente): `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
  * URL: usar `new URL()` no validador (cair no catch se inválida)
  * UUID v4: `/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i`
* Exemplos:

  * `customerEmail` → `crm.customer.email`
  * `orderId` (uuid) → `ecommerce.order.id`
  * `countryIso2` → `geo.country.iso2`

### D) Date/Tempo

Mapeie granularidade: `createdAt`, `updatedAt`, `startAt`, `endAt`, `birthday`, `scheduledFor`, `dueAt`.

* Exemplos:

  * `scheduledFor` → `calendar.event.scheduledAt` (Date)
  * `birthday` → `identity.person.birthDate` (Date sem hora — normalize)

---

## Sem biblioteca externa? Forneça **Shim** local (TS puro)

Crie **um** utilitário interno para “brand” de semântica **sem runtime**:

```ts
// src/semantic/shim.ts
declare const __brand: unique symbol;

export type Brand<T, Name extends string> = T & { readonly [__brand]: Name };

export function STAMP<Name extends string>() {
  return {
    of: <T>(v: T) => v as Brand<T, Name>,
    un: <T>(v: Brand<T, Name>) => v as unknown as T,
  };
}
```

---

## Gerar stubs auto-contidos por tipo

### Boolean (ex.: `project.task.isDone`)

```ts
// domains/project/task/is-done.ts
import { Brand, STAMP } from "../../src/semantic/shim";
export type IsDone = Brand<boolean, "project.task.isDone">;

export const IsDone = (() => {
  const f = STAMP<"project.task.isDone">();
  return {
    of: (v: unknown): IsDone => f.of(Boolean(v)),
    un: (v: IsDone): boolean => f.un(v),
    and: (a: IsDone, b: IsDone): IsDone => f.of((f.un(a) && f.un(b)) as boolean),
  };
})();
```

### Number com moeda (ex.: `ecommerce.order.revenueTotal.brl`)

```ts
// domains/ecommerce/order/revenue-total.brl.ts
import { Brand, STAMP } from "../../src/semantic/shim";
export type RevenueTotalBRL = Brand<number, "ecommerce.order.revenueTotal.brl">;

export const RevenueTotalBRL = (() => {
  const f = STAMP<"ecommerce.order.revenueTotal.brl">();
  return {
    of: (v: unknown): RevenueTotalBRL => {
      const n = Number(v);
      if (!Number.isFinite(n) || n < 0) throw new TypeError("revenue must be >= 0");
      return f.of(n);
    },
    un: (v: RevenueTotalBRL) => f.un(v),
    add: (a: RevenueTotalBRL, b: RevenueTotalBRL): RevenueTotalBRL => f.of(f.un(a) + f.un(b)),
  };
})();
```

### String formatada (ex.: `crm.customer.email`)

```ts
// domains/crm/customer/email.ts
import { Brand, STAMP } from "../../src/semantic/shim";
export type CustomerEmail = Brand<string, "crm.customer.email">;

export const CustomerEmail = (() => {
  const f = STAMP<"crm.customer.email">();
  const emailRx = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return {
    of: (v: unknown): CustomerEmail => {
      const s = String(v).trim();
      if (!emailRx.test(s)) throw new TypeError("invalid email");
      return f.of(s);
    },
    un: (v: CustomerEmail) => f.un(v),
  };
})();
```

### Date (ex.: `calendar.event.scheduledAt`)

```ts
// domains/calendar/event/scheduled-at.ts
import { Brand, STAMP } from "../../src/semantic/shim";
export type ScheduledAt = Brand<Date, "calendar.event.scheduledAt">;

export const ScheduledAt = (() => {
  const f = STAMP<"calendar.event.scheduledAt">();
  return {
    of: (v: unknown): ScheduledAt => {
      const d = v instanceof Date ? v : new Date(String(v));
      if (Number.isNaN(d.getTime())) throw new TypeError("invalid date");
      return f.of(d);
    },
    un: (v: ScheduledAt) => f.un(v),
  };
})();
```

---

## Manifesto de inferência (gere este arquivo)

`semantic-types.manifest.json`

```json
{
  "version": 1,
  "discovered": [
    {
      "from": "src/modules/orders/service.ts:42",
      "symbol": "revenueTotal",
      "primitive": "number",
      "brand": "ecommerce.order.revenueTotal.brl",
      "file": "domains/ecommerce/order/revenue-total.brl.ts",
      "validations": [">= 0", "finite"],
      "ops": ["add", "sum"],
      "confidence": 0.86
    },
    {
      "from": "src/calendar/ui/Week.tsx:18",
      "symbol": "showWeekView",
      "primitive": "boolean",
      "brand": "ui.calendar.showWeekView",
      "file": "domains/ui/calendar/show-week-view.ts",
      "validations": [],
      "ops": ["and"],
      "confidence": 0.78
    }
  ],
  "unmapped_candidates": [
    "metrics.kpi.percentage", 
    "dentistry.procedure.teethExtracted",
    "clinic.schedule.totalAppointments"
  ],
  "suggest_new_top5": [
    { "brand": "metrics.kpi.percentage", "why": "encontrado `%` em labels/testes; validação [0..100]" },
    { "brand": "time.duration.ms", "why": "sufixo Ms em várias props; normalizar" },
    { "brand": "crm.customer.phoneE164", "why": "strings com dígitos e +; padronizar E.164" },
    { "brand": "ecommerce.order.taxRate", "why": "uso de `tax` proporcional; [0..1] ou [0..100]" },
    { "brand": "geo.country.iso2", "why": "ocorrência de códigos de 2 letras" }
  ]
}
```

---

## Patches sugeridos (diff)

Gere um diff unificado propondo substituição **local** (import relativo) e uso das fábricas:

```diff
- const revenueTotal: number = order.items.reduce((sum, it) => sum + it.price, 0)
+ import { RevenueTotalBRL } from "../../domains/ecommerce/order/revenue-total.brl"
+ const revenueTotal = RevenueTotalBRL.of(
+   order.items.reduce((sum, it) => sum + it.price, 0)
+ )

- if (showWeekView && isDone) { ... }
+ import { IsDone } from "../../domains/project/task/is-done"
+ import { ShowWeekView } from "../../domains/ui/calendar/show-week-view"
+ if (IsDone.un(isDone) && ShowWeekView.un(showWeekView)) { ... }
```

> Se o projeto ainda não tem `src/semantic/shim.ts`, inclua o arquivo no patch.

---

## Regras de qualidade que o agente deve aplicar

* **Nunca** substituir primitivo por tipo semântico sem **stub** + **import**.
* Apontar ambiguidade (`rate`, `size`, `code`) e sugerir **brand** com **unidade ou domínio** explícitos.
* Sugerir **faixas**: contagens (≥ 0), porcentagens (0–1 ou 0–100 — escolha e documente).
* Não criar tipos transdomínio: `ecommerce.*` não reage com `crm.*` (a menos que o projeto já defina).
* Se detectar `any` em locais críticos, abrir **nota** para migração gradual com marcas semânticas.

---

# Como usar esse prompt (roteiro rápido)

1. Cole o prompt na sua ferramenta de agente (ou pipeline CI).
2. Rode o agente na raiz do repositório.
3. Revise o **Relatório** e o **Manifesto**.
4. Aplique os **stubs** e **patches** (commit separado).
5. Selecione até **5** tipos sugeridos para virar tarefa nas próximas semanas (backlog).

---

## Bônus: exemplos mínimos quando **não houver nada instalado**

Peça ao agente para incluir **sempre** estes 4 stubs base (como acima):

* `src/semantic/shim.ts`
* `domains/project/task/is-done.ts`
* `domains/ecommerce/order/revenue-total.brl.ts`
* `domains/crm/customer/email.ts`
* `domains/calendar/event/scheduled-at.ts`

Eles servem de **exemplos-guia** para o resto do repositório.

---

Se quiser, eu adapto esse prompt para **rodar via GitHub Action** (varrendo `.ts/.tsx`, gerando o manifesto e abrindo uma **issue** com os diffs e o “Top 5” de tipos a criar).
