# Playwright Fixtures: Análise Técnica

## 📚 Índice
1. [Introdução](#introdução)
2. [Como Fixtures Funcionam](#como-fixtures-funcionam)
3. [Fluxo de Execução](#fluxo-de-execução)
4. [Arquitetura do Projeto](#arquitetura-do-projeto)
5. [Vantagens](#vantagens)
6. [Desvantagens](#desvantagens)
7. [Quando Usar](#quando-usar)
8. [Referências](#referências)

---

## Introdução

**Fixtures** são um mecanismo do Playwright que permite:
- Compartilhar setup entre testes
- Injetar dependências automaticamente
- Gerenciar ciclo de vida de recursos (setup/teardown)
- Reutilizar código comum

Neste repositório, fixtures são utilizadas para centralizar o acesso aos **Page Objects** através do `PageManager`, abstraindo a complexidade de inicialização e mantendo os testes limpos e legíveis.

---

## Como Fixtures Funcionam

### Conceito Fundamental

```mermaid
graph LR
    A["Test File<br/>(login.spec.ts)"]
    B["Fixture Provider<br/>(base-fixture.ts)"]
    C["PageManager<br/>(page-manager.ts)"]
    D["Page Objects<br/>(login-page.ts)"]
    
    A -->|"Solicita: { pm }"| B
    B -->|"Fornece: PageManager"| A
    B -->|"Instancia"| C
    C -->|"Gerencia"| D
    
    style A fill:#e1f5ff,color:#000
    style B fill:#fff3e0,color:#000
    style C fill:#f3e5f5,color:#000
    style D fill:#e8f5e9,color:#000
```

### Estrutura em Camadas

```mermaid
graph TD
    A["login.spec.ts<br/><br/>test('Login...', async { pm } => {...})"]
    B["base-fixture.ts<br/><br/>test.extend<TestOptions> <br/>define fixture 'pm'"]
    C["page-manager.ts<br/><br/>Instancia todos Page Objects<br/>Fornece getters"]
    D1["login-page.ts"]
    D2["dashboard-page.ts"]
    D3["top-menu-page.ts"]
    DE["...mais Page Objects"]
    
    A -->|"Recebe pm"| B
    B -->|"Cria new PageManager"| C
    C -->|"Gerencia"| D1
    C -->|"Gerencia"| D2
    C -->|"Gerencia"| D3
    C -->|"Gerencia"| DE
    
    style A fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    style B fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    style C fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    style D1 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
    style D2 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
    style D3 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
    style DE fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
```

---

## Fluxo de Execução

### Sequência de Inicialização

```mermaid
sequenceDiagram
    participant Test as Test (login.spec.ts)
    participant Fixture as Fixture Provider<br/>(base-fixture.ts)
    participant OpenPage as openPage Fixture
    participant PageMgr as PageManager
    participant Pages as Page Objects
    
    Test->>Fixture: [1] Teste inicia, solicita { pm }
    Fixture->>OpenPage: [2] Executa openPage fixture
    OpenPage->>OpenPage: [3] page.goto("") - abre URL base
    OpenPage-->>Fixture: [4] Fixture openPage completo
    
    Fixture->>PageMgr: [5] new PageManager(page)
    PageMgr->>Pages: [6] Inicializa todos Page Objects
    Pages-->>PageMgr: [7] Page Objects prontos
    PageMgr-->>Fixture: [8] PageManager criado
    
    Fixture->>Test: [9] Injeta pm no contexto do teste
    
    Test->>Test: [10] Executa código do teste com pm
    Test->>PageMgr: pm.getLoginPage().doLogin(...)
    PageMgr->>Pages: Acessa LoginPage
    Pages-->>Test: Executa ações
    
    Test-->>Fixture: [11] Teste completa
    Fixture->>OpenPage: [12] Cleanup - fecha page
    Fixture-->>Fixture: [13] Teste finaliza
```

### Ciclo de Vida Completo

```mermaid
graph LR
    A["SETUP<br/>openPage.execute"] -->|"✓"| B["SETUP<br/>pm.execute"]
    B -->|"✓"| C["TEST<br/>Código do teste"]
    C -->|"✓ ou ✗"| D["TEARDOWN<br/>Cleanup"]
    D -->|"✓"| E["FIM"]
    
    style A fill:#c8e6c9,color:#000
    style B fill:#c8e6c9,color:#000
    style C fill:#bbdefb,color:#000
    style D fill:#ffe0b2,color:#000
    style E fill:#f0f0f0,color:#000
```

---

## Arquitetura do Projeto

### Estrutura Geral

```mermaid
graph TB
    subgraph Tests["📝 TESTES"]
        T1["login.spec.ts"]
        T2["contacts.spec.ts"]
        T3["message-batch-email.spec.ts"]
    end
    
    subgraph Setup["⚙️ SETUP"]
        F1["base-fixture.ts<br/>Fixtures Customizadas"]
        F2["base-env-setup.ts<br/>Credenciais"]
        F3["base-url.ts<br/>URLs"]
    end
    
    subgraph Manager["🎛️ MANAGER"]
        PM["PageManager<br/>Gerenciador Central"]
    end
    
    subgraph Pages["📄 PAGE OBJECTS"]
        P1["LoginPage"]
        P2["DashboardPage"]
        P3["ContactsPage"]
        P4["MessagePage"]
        P5["...mais"]
    end
    
    subgraph Lib["🎭 BIBLIOTECA"]
        LIB["Playwright Test API"]
    end
    
    T1 -.->|import| F1
    T2 -.->|import| F1
    T3 -.->|import| F1
    
    F1 -->|extends| LIB
    F1 -->|define fixtures| F2
    F1 -->|instantiate| PM
    
    PM -->|manage| P1
    PM -->|manage| P2
    PM -->|manage| P3
    PM -->|manage| P4
    PM -->|manage| P5
    
    T1 -->|usa| PM
    T2 -->|usa| PM
    T3 -->|usa| PM
    
    style Tests fill:#e3f2fd,stroke:#1976d2
    style Setup fill:#fff3e0,stroke:#f57c00
    style Manager fill:#f3e5f5,stroke:#7b1fa2
    style Pages fill:#e8f5e9,stroke:#388e3c
    style Lib fill:#fce4ec,stroke:#c2185b
```

---

## Comparação: Com vs Sem Fixtures

### SEM Fixtures (❌ Duplicação)

```mermaid
graph TD
    T1["test('Login', async { page } => {<br/>  await page.goto('')<br/>  const login = new LoginPage(page)<br/>  await login.doLogin(...)<br/>})"]
    
    T2["test('Dashboard', async { page } => {<br/>  await page.goto('')<br/>  const dashboard = new DashboardPage(page)<br/>  await dashboard.assertTitle()<br/>})"]
    
    T3["test('Contacts', async { page } => {<br/>  await page.goto('')<br/>  const contacts = new ContactsPage(page)<br/>  await contacts.addContact(...)<br/>})"]
    
    PROB["❌ PROBLEMAS:<br/>- Código duplicado<br/>- Difícil manutenção<br/>- Inconsistência"]
    
    T1 -.-> PROB
    T2 -.-> PROB
    T3 -.-> PROB
    
    style T1 fill:#ffebee,color:#000
    style T2 fill:#ffebee,color:#000
    style T3 fill:#ffebee,color:#000
    style PROB fill:#ffcdd2,stroke:#d32f2f,stroke-width:2px,color:#000
```

### COM Fixtures (✅ Centralizado)

```mermaid
graph TD
    T1["test('Login', async { pm } => {<br/>  await pm.getLoginPage().doLogin(...)<br/>})"]
    
    T2["test('Dashboard', async { pm } => {<br/>  await pm.getDashboardPage().assertTitle()<br/>})"]
    
    T3["test('Contacts', async { pm } => {<br/>  await pm.getContactsPage().addContact(...)<br/>})"]
    
    F["Fixture 'pm'<br/>- Inicializa PageManager<br/>- Uma vez por teste<br/>- Reutilizável"]
    
    T1 -.-> F
    T2 -.-> F
    T3 -.-> F
    
    SUC["✅ VANTAGENS:<br/>- Sem duplicação<br/>- Fácil manutenção<br/>- Consistência"]
    
    F -.-> SUC
    
    style T1 fill:#e8f5e9,color:#000
    style T2 fill:#e8f5e9,color:#000
    style T3 fill:#e8f5e9,color:#000
    style F fill:#c8e6c9,stroke:#388e3c,stroke-width:2px,color:#000
    style SUC fill:#a5d6a7,stroke:#2e7d32,stroke-width:2px,color:#000
```

---

## Código: Implementação Passo a Passo

### 1️⃣ Base-Fixture: Definir os Fixtures

```typescript
// tests/setup/base-fixture.ts
import { test as base } from "@playwright/test";
import { PageManager } from "@pages/page-manager";

// Tipagem: o que cada fixture fornece
export type TestOptions = {
  openPage: string;
  pm: PageManager;
};

export const test = base.extend<TestOptions>({
  // Fixture 1: openPage (setup da página)
  openPage: async ({ page }, use) => {
    const URL = await page.goto("");  // Abre a URL base
    console.log("========= Opening ", URL?.url());
    await use("");  // Injeta no próximo fixture
  },

  // Fixture 2: pm (depende de openPage)
  pm: async ({ page, openPage }, use) => {
    // ↑ Recebe page e openPage automaticamente
    const pageManager = new PageManager(page);
    await use(pageManager);  // Injeta no teste
    // Cleanup automático após o teste
  },
});

export { expect } from "@playwright/test";
```

### 2️⃣ PageManager: Centralizar Page Objects

```typescript
// tests/ui/pages/page-manager.ts
import { type Page } from "@playwright/test";
import { LoginPage } from "@pages/general/login-page";
import { DashboardPage } from "@pages/general/dashboard-page";
// ... mais imports

export class PageManager {
  readonly page: Page;
  private readonly loginPage: LoginPage;
  private readonly dashboardPage: DashboardPage;
  // ... mais Page Objects

  constructor(page: Page) {
    this.page = page;
    this.loginPage = new LoginPage(this.page);
    this.dashboardPage = new DashboardPage(this.page);
    // ... inicializa mais Page Objects
  }

  getLoginPage() {
    return this.loginPage;
  }

  getDashboardPage() {
    return this.dashboardPage;
  }
  // ... mais getters
}
```

### 3️⃣ Test: Usar o Fixture

```typescript
// tests/ui/specs/login.spec.ts
import { test } from "@setup/base-fixture";  // Nosso fixture customizado!
import userData from "@data/user-data";

test.describe("Login suite", { tag: ["@login"] }, () => {
  test("Login with valid credentials", async ({ pm }) => {
    // ↑ pm é injetado automaticamente pelo fixture!
    
    await test.step("Login and validate", async () => {
      await pm.getLoginPage().doLogin(username, password);
      await pm.getDashboardPage().assertDashboardTitleIsPresent();
    });
  });
});
```

---

## Vantagens

| # | Vantagem | Descrição | Benefício |
|---|----------|-----------|-----------|
| 1 | **Reutilização** | `openPage` e `pm` são criados uma vez e compartilhados | Menos código duplicado |
| 2 | **Injeção de Dependências** | Fixtures resolvem dependências automaticamente | Código mais limpo |
| 3 | **Isolamento** | Cada teste recebe instância limpa de `pm` | Testes independentes |
| 4 | **Centralização** | Um ponto único (`PageManager`) para acessar Page Objects | Fácil de manter |
| 5 | **Abstração** | Testes não lidam com `page` diretamente | Foco no cenário |
| 6 | **Manutenção** | Mudanças em Page Objects não afetam testes | Menos refatoração |
| 7 | **Lifecycle** | Setup/Teardown automático gerenciado pelo Playwright | Menos código boilerplate |
| 8 | **Type Safety** | TypeScript garante que `pm` tem métodos corretos | Menos erros |

### Exemplo Prático: Vantagem de Centralização

```typescript
// ❌ SEM CENTRALIZAÇÃO - Múltiplos testes, múltiplas inicializações
test("test1", async ({ page }) => {
  await page.goto("");
  const loginPage = new LoginPage(page);
  const dashboard = new DashboardPage(page);
  await loginPage.doLogin(...);
});

test("test2", async ({ page }) => {
  await page.goto("");  // ❌ DUPLICADO
  const loginPage = new LoginPage(page);  // ❌ DUPLICADO
  const contacts = new ContactsPage(page);
  await contacts.addContact(...);
});

// ✅ COM FIXTURES - Um setup, múltiplos testes
test("test1", async ({ pm }) => {
  await pm.getLoginPage().doLogin(...);  // ✅ Simples!
  await pm.getDashboardPage().assertTitle();
});

test("test2", async ({ pm }) => {
  await pm.getContactsPage().addContact(...);  // ✅ Simples!
});
```

---

## Desvantagens

| # | Desvantagem | Descrição | Impacto |
|---|-------------|-----------|--------|
| 1 | **Curva Aprendizado** | Developers precisam entender fixtures e dependências | Onboarding mais longo |
| 2 | **Overhead** | Criação de PageManager a cada teste (mesmo simples) | Performance ligeiramente reduzida |
| 3 | **Menos Controle** | Customização de setup em testes específicos é complexa | Menos flexibilidade |
| 4 | **Debugging Difícil** | Rastrear fluxo entre múltiplas fixtures é confuso | Debugging mais demorado |
| 5 | **Inicialização Completa** | Se um teste precisa de 1 Page Object, todos os 8+ são criados | Desperdício de recursos |
| 6 | **Chaining Profundo** | Fixtures que dependem de outros fixtures | Complexidade aumenta |
| 7 | **Rigidez** | Difícil fazer override de fixtures em testes específicos | Menos flexibilidade |
| 8 | **Encapsulamento** | Pode ocultar detalhes importantes do que está acontecendo | Menos transparência |

### Exemplo: Overhead de Inicialização

```typescript
// ❌ Teste simples que inicializa TUDO
test("only needs LoginPage", async ({ pm }) => {
  // pm.constructor() inicializa:
  // - LoginPage ✓ (usado)
  // - DashboardPage ✗ (não usado)
  // - ContactsPage ✗ (não usado)
  // - DataJobsPage ✗ (não usado)
  // - MessagePage ✗ (não usado)
  // - DiagramPage ✗ (não usado)
  // - KitchenSinkStepsPage ✗ (não usado)
  // - AudiencePage ✗ (não usado)
  
  await pm.getLoginPage().doLogin(...);
});

// ✅ Alternativa: sem fixture para casos simples
test("only needs LoginPage (sem fixture)", async ({ page }) => {
  await page.goto("");
  const loginPage = new LoginPage(page);
  await loginPage.doLogin(...);
});
```

---

## Quando Usar

### ✅ Use Fixtures Quando:

```mermaid
graph TD
    A["Decisão: Usar Fixtures?"]
    
    A -->|"Muitos Page Objects?"| B1["SIM ✅"]
    A -->|"Suite grande?"| B2["SIM ✅"]
    A -->|"Padrão repetitivo?"| B3["SIM ✅"]
    A -->|"Dependências claras?"| B4["SIM ✅"]
    A -->|"Equipe conhece padrão?"| B5["SIM ✅"]
    
    B1 --> C["USE FIXTURES"]
    B2 --> C
    B3 --> C
    B4 --> C
    B5 --> C
    
    style A fill:#fff9c4,stroke:#f57f17,color:#000
    style B1 fill:#c8e6c9,stroke:#388e3c,color:#000
    style B2 fill:#c8e6c9,stroke:#388e3c,color:#000
    style B3 fill:#c8e6c9,stroke:#388e3c,color:#000
    style B4 fill:#c8e6c9,stroke:#388e3c,color:#000
    style B5 fill:#c8e6c9,stroke:#388e3c,color:#000
    style C fill:#a5d6a7,stroke:#2e7d32,stroke-width:2px,color:#000
```

### ❌ Evite Fixtures Quando:

```mermaid
graph TD
    A["Decisão: NÃO usar Fixtures?"]
    
    A -->|"Projeto pequeno?"| B1["SIM ❌"]
    A -->|"Cada teste diferente?"| B2["SIM ❌"]
    A -->|"Performance crítica?"| B3["SIM ❌"]
    A -->|"Prototipagem rápida?"| B4["SIM ❌"]
    A -->|"Equipe nova com Playwright?"| B5["SIM ❌"]
    
    B1 --> C["NÃO USE FIXTURES"]
    B2 --> C
    B3 --> C
    B4 --> C
    B5 --> C
    
    style A fill:#ffccbc,stroke:#d84315,color:#000
    style B1 fill:#ffcdd2,stroke:#d32f2f,color:#000
    style B2 fill:#ffcdd2,stroke:#d32f2f,color:#000
    style B3 fill:#ffcdd2,stroke:#d32f2f,color:#000
    style B4 fill:#ffcdd2,stroke:#d32f2f,color:#000
    style B5 fill:#ffcdd2,stroke:#d32f2f,color:#000
    style C fill:#ef9a9a,stroke:#c62828,stroke-width:2px,color:#000
```

---

## Matriz de Decisão

```mermaid
graph TB
    subgraph Perfect["🎯 CENÁRIO PERFEITO<br/>(Este Projeto)"]
        P1["✅ 8+ Page Objects"]
        P2["✅ 100+ testes"]
        P3["✅ Padrão consistente"]
        P4["✅ Setup complexo"]
        P5["✅ Equipe experiente"]
    end
    
    subgraph GoodFit["👍 BOM FIT"]
        G1["✅ 3-7 Page Objects"]
        G2["✅ 20-100 testes"]
        G3["✅ Setup moderado"]
    end
    
    subgraph Maybe["🤔 TALVEZ"]
        M1["❓ 2-3 Page Objects"]
        M2["❓ 10-20 testes"]
        M3["❓ Setup simples"]
    end
    
    subgraph BadFit["👎 MÁ FIT"]
        B1["❌ 1 Page Object"]
        B2["❌ <10 testes"]
        B3["❌ Sem setup"]
        B4["❌ Testes ad-hoc"]
    end
    
    Perfect -.-> R1["Recomendação:<br/>USE FIXTURES"]
    GoodFit -.-> R1
    Maybe -.-> R2["Recomendação:<br/>TALVEZ USE"]
    BadFit -.-> R3["Recomendação:<br/>NÃO USE"]
    
    style Perfect fill:#a5d6a7,stroke:#2e7d32,stroke-width:2px,color:#000
    style GoodFit fill:#c8e6c9,stroke:#558b2f,color:#000
    style Maybe fill:#fff9c4,stroke:#f57f17,color:#000
    style BadFit fill:#ffcdd2,stroke:#d32f2f,color:#000
    style P1 fill:#a5d6a7,color:#000
    style P2 fill:#a5d6a7,color:#000
    style P3 fill:#a5d6a7,color:#000
    style P4 fill:#a5d6a7,color:#000
    style P5 fill:#a5d6a7,color:#000
    style G1 fill:#c8e6c9,color:#000
    style G2 fill:#c8e6c9,color:#000
    style G3 fill:#c8e6c9,color:#000
    style M1 fill:#fff9c4,color:#000
    style M2 fill:#fff9c4,color:#000
    style M3 fill:#fff9c4,color:#000
    style B1 fill:#ffcdd2,color:#000
    style B2 fill:#ffcdd2,color:#000
    style B3 fill:#ffcdd2,color:#000
    style B4 fill:#ffcdd2,color:#000
    style R1 fill:#66bb6a,stroke:#2e7d32,stroke-width:2px,color:#000
    style R2 fill:#fbc02d,stroke:#f57f17,stroke-width:2px,color:#000
    style R3 fill:#ef5350,stroke:#d32f2f,stroke-width:2px,color:#000
```

---

## Resumo Técnico

### Injeção de Dependência em Ação

```mermaid
graph LR
    A["Playwright<br/>Base API"]
    B["base-fixture.ts<br/>.extend<<br/>TestOptions>"]
    C["openPage<br/>Fixture"]
    D["pm<br/>Fixture"]
    E["Test Function<br/>async { pm }"]
    
    A -->|"estende"| B
    B -->|"define"| C
    B -->|"define + depende de"| D
    C -->|"inicia page"| D
    D -->|"injeta"| E
    
    style A fill:#fce4ec,stroke:#c2185b,color:#000
    style B fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#000
    style C fill:#ede7f6,stroke:#5e35b1,color:#000
    style D fill:#ede7f6,stroke:#5e35b1,stroke-width:2px,color:#000
    style E fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
```

### Ordem de Execução

| # | Etapa | Código | Status |
|---|-------|--------|--------|
| 1 | Playwright inicia | `test.extend<TestOptions>()` | ⏳ Setup |
| 2 | openPage fixture | `page.goto("")` | ⏳ Setup |
| 3 | pm fixture | `new PageManager(page)` | ⏳ Setup |
| 4 | Teste executa | `await pm.getLoginPage()...` | ⏱️ Execução |
| 5 | Cleanup | Fechamento de recursos | ⏸️ Teardown |

---

## Comparação com Outras Abordagens

```mermaid
graph TD
    subgraph Approach1["❌ Sem Setup (Pior)"]
        A1["test('test', async { page } => {<br/>  // Duplicar setup em cada teste"]
    end
    
    subgraph Approach2["👎 Helper Functions"]
        A2["async function setupPage(page) {<br/>  return new PageManager(page)<br/>}<br/>test('test', async { page } => {<br/>  const pm = await setupPage(page)"]
    end
    
    subgraph Approach3["✅ Fixtures (Melhor)"]
        A3["pm: async ({ page }, use) => {<br/>  const pm = new PageManager(page)<br/>  await use(pm)<br/>}<br/>test('test', async { pm } => { ... }"]
    end
    
    Quality["Qualidade ↑<br/>Reusabilidade ↑<br/>Manutenibilidade ↑"]
    
    Approach1 -.-> Quality
    Approach2 -.-> Quality
    Approach3 -.-> Quality
    
    style Approach1 fill:#ffcdd2,stroke:#d32f2f,color:#000
    style Approach2 fill:#ffe0b2,stroke:#f57c00,color:#000
    style Approach3 fill:#a5d6a7,stroke:#2e7d32,stroke-width:2px,color:#000
    style Quality fill:#fff9c4,stroke:#f57f17,stroke-width:2px,color:#000
```

---

## Conclusão

### 📊 Análise Final

Este projeto implementa **fixtures como um padrão de Design Pattern** com:

✅ **Pontos Positivos:**
- Centralização via `PageManager`
- Abstração da complexidade
- Consistência entre testes
- Fácil manutenção e extensão
- Type-safe com TypeScript

⚠️ **Pontos de Atenção:**
- Curva de aprendizado inicial
- Overhead de inicialização completa
- Menos flexibilidade para customizações específicas

### 🎯 Recomendação

**Este projeto faz USO ADEQUADO de fixtures porque:**

1. Possui **múltiplos Page Objects** (8+)
2. Tem **grande volume de testes** (100+)
3. Segue **padrão consistente**
4. Apresenta **dependências claras** (openPage → pm)
5. Prioriza **manutenibilidade e escalabilidade**

### 🚀 Melhorias Possíveis

```typescript
// Fixture opcional para testes simples (lazy loading)
export const test = base.extend<TestOptions>({
  pm: async ({ page, openPage }, use) => {
    // Lazy initialization
    const pageManager = new PageManager(page);
    await use(pageManager);
  },
});

// Fixture para casos específicos
test.extend<{ simpleLogin: LoginPage }>({
  simpleLogin: async ({ page }, use) => {
    await page.goto("");
    await use(new LoginPage(page));
  },
});
```

---

## Referências

- 📖 [Playwright Fixtures Documentation](https://playwright.dev/docs/test-fixtures)
- 🎯 [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- 🏗️ [Page Object Model Pattern](https://playwright.dev/docs/pom)
- 📚 [Dependency Injection Pattern](https://en.wikipedia.org/wiki/Dependency_injection)

---

**Documento gerado em:** 2025-12-06  
**Projeto:** project2 - Playwright Test Suite  
**Versão:** 1.0

