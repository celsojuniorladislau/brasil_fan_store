# Brasil Fan Store – Resumo das conversas e plano para produção

Este documento reúne o que foi discutido sobre o projeto e o plano para levar a loja até a produção.

---

## 1. O que foi conversado

### 1.1 Análise da base de código

- O **Brasil Fan Store** é uma loja de camisas da Seleção Brasileira em um único arquivo: `index.html` (~1400 linhas).
- **Stack atual:** HTML + React 18 (via Babel CDN) + Tailwind CSS (CDN) + QRCode.js. Sem backend; tudo no cliente.
- **Fluxo:** 6 passos em estado local (passo 0 = landing, 1 = seleção de produto, 2 = carrinho, 3 = checkout, 4 = pagamento, 5 = confirmação). Três produtos fixos, carrinho em memória, pagamento simulado.
- **Problemas identificados:** classe CSS `btn-secondary` usada mas não definida; código monolítico; sem persistência; pagamento apenas mock.

### 1.2 Backend: MedusaJS

- Decisão de usar **MedusaJS** como backend da loja.
- Medusa é headless: expõe Store API (produtos, carrinho, checkout) e Admin. Requer PostgreSQL e Redis.
- Frontend se conecta via **Store API** (REST ou JS SDK com Publishable Key). CORS deve permitir a URL do frontend.
- Plano de produção inclui: instalação com `create-medusa-app`, região Brasil (BRL), cadastro dos 3 produtos com variantes (P/M/G/GG/XG), criação de Publishable API Key e configuração de variáveis de ambiente (incluindo secrets e CORS).

### 1.3 Frontend e backend em lugares diferentes

- **Como funciona:** o frontend (HTML/JS/CSS) é servido por um host (ex.: Vercel, Netlify); o backend (Medusa) roda em outro (ex.: Railway, VPS). O frontend faz requisições HTTP/HTTPS para a API do backend.
- **Por quê:** escalar e custear cada parte separadamente; deploy independente; headless commerce (mesmo backend para site, app, etc.); manter dados e segredos no servidor.

### 1.4 Melhorias no frontend (SEO, velocidade, segurança)

- **SEO:** meta tags (title, description, keywords), Open Graph, URL canônica, JSON-LD (Schema.org Product), `robots.txt` e `sitemap.xml`. Conteúdo importante no HTML inicial (SSR/SSG ajudam).
- **Velocidade:** Core Web Vitals (LCP &lt; 2,5s, FCP &lt; 1,8s, CLS &lt; 0,1); imagens com `width`/`height` e `loading="lazy"`; fontes com `font-display: swap`; `preconnect` para recursos externos; CDN e cache.
- **Segurança:** headers (X-Content-Type-Options: nosniff, X-Frame-Options, Referrer-Policy, Content-Security-Policy); HTTPS; não expor secrets no frontend.

### 1.5 SPA vs outras abordagens

- **Problemas da SPA atual:** HTML quase vazio; conteúdo depende do JS; SEO e primeiro paint prejudicados; uma única URL.
- **Opções:** SSR (HTML por requisição), SSG (HTML no build), ISR, pré-render, MPA.
- **Decisão:** migrar para **SSR com Next.js**, mantendo React e Tailwind, para melhor SEO, velocidade e URLs por página.

### 1.6 Migração para Next.js (SSR, Tailwind, React)

- Novo frontend: **Next.js** (App Router) + **Tailwind** + **React**.
- Passos atuais viram **rotas:** `/` (home), `/produtos`, `/produtos/[slug]`, `/carrinho`, `/checkout`, `/confirmacao`.
- Dados iniciais em `lib/data.ts` (mock), depois troca por Medusa; carrinho com Context + localStorage, depois por Store API; componentes extraídos (Header, Footer, ProductCard, Modal, CartDrawer, etc.).

### 1.7 Mercado Pago (PIX)

- Uso de **Mercado Pago** para pagamentos PIX.
- Referência: [Como integrar Mercado Pago Pix no Next.js 15 (App Router)](https://www.tabnews.com.br/zilvodev/como-integrar-mercado-pago-pix-no-next-js-15-app-router).
- Regra importante: **valor do pagamento sempre calculado no servidor** (total do carrinho/pedido); o frontend envia apenas identificador (ex.: `cartId` ou `orderId`). API Route cria o PIX e retorna QR code; webhook confirma pagamento e atualiza pedido.

---

## 2. Plano para levar a loja à produção

Cronograma em **6 semanas** (Dia 1 a Dia 42). Ajuste o “Dia 1” à sua data de início.

### 2.1 Visão geral por semana

| Semana | Foco | Entregas principais |
|--------|------|---------------------|
| **1** | Backend + Frontend base | Medusa rodando; Next.js criado, layout e dados mock |
| **2** | Backend admin + Frontend páginas | Região BRL, produtos no admin; Home, produtos, detalhe |
| **3** | Frontend checkout + Backend API key | Carrinho, checkout, confirmação (mock); Publishable Key |
| **4** | Integração | SDK no Next.js; produtos e carrinho via Medusa |
| **5** | Integração checkout + Mercado Pago | Checkout/complete cart com Medusa; PIX (API Route + webhook); testes E2E |
| **6** | Produção | SEO, segurança, deploy front e back, webhook MP, CNAME/DNS |

---

### 2.2 FASE 1 – Backend Medusa (Dias 1–12)

**Objetivo:** Medusa instalado, região Brasil, produtos cadastrados e Publishable Key criada.

| Entrega | Dias | Responsabilidade |
|---------|------|------------------|
| 1.1 Instalação Medusa | 1–3 | `npx create-medusa-app@latest`; PostgreSQL e Redis configurados |
| 1.2 Variáveis de ambiente | 3–4 | `.env` com `DATABASE_URL`, `REDIS_URL`, `COOKIE_SECRET`, `JWT_SECRET` |
| 1.3 Região Brasil | 4–5 | Admin: nova região BR, moeda BRL |
| 1.4 Cadastro de produtos | 5–9 | 3 produtos (Amarela, Azul, Feminina) com variantes P/M/G/GG/XG e imagens |
| 1.5 Publishable API Key | 9–10 | Admin > API Keys > tipo publishable; guardar token `pk_...` |
| 1.6 CORS | 10–12 | `STORE_CORS` com URL do frontend (dev e produção) |

**Critério de conclusão:** GET `/store/products` retorna os 3 produtos; POST `/store/carts` cria carrinho.

#### Pré-requisitos antes de começar o Medusa

- **Node.js** v20+ (LTS)
- **PostgreSQL** instalado e rodando (local ou serviço: Neon, Railway, Supabase)
- **Redis** instalado e rodando (local ou Upstash, Redis Cloud)
- **Git** instalado

#### Ordem sugerida para implementar o Medusa

1. Instalar e subir o Medusa (`create-medusa-app` + `.env`).
2. Abrir o Admin, criar o primeiro usuário e fazer login.
3. Criar a região Brasil (BRL).
4. Cadastrar os 3 produtos com variantes e preços.
5. Criar a Publishable API Key e anotar o token.
6. Ajustar CORS para a URL do frontend (ex.: `http://localhost:3000` em dev).

Depois disso o frontend Next.js pode consumir a **Store API** (REST ou `@medusajs/js-sdk` com o `publishableKey`) para listar produtos, criar/atualizar carrinho e completar o checkout.

---

### 2.3 FASE 2 – Frontend Next.js com dados mock (Dias 1–21)

**Objetivo:** Loja Next.js completa com fluxo de compra usando dados locais (sem Medusa).

| Entrega | Dias | Responsabilidade |
|---------|------|------------------|
| 2.1 Projeto e estilos | 1–5 | `create-next-app` (TS, Tailwind, App Router); imagens em `public/`; CSS do `index.html` em `globals.css`; fontes Poppins/Playfair |
| 2.2 Layout e componentes base | 5–9 | `layout.tsx`, `Header`, `Footer`, `lib/data.ts`, `CartProvider` |
| 2.3 Páginas de catálogo | 9–14 | Home, `/produtos`, `/produtos/[slug]` com `generateStaticParams` |
| 2.4 Carrinho e checkout | 14–19 | `/carrinho`, `/checkout`, `/confirmacao`; `Modal`, `CartDrawer`, `QRCodeDisplay` |
| 2.5 SEO base | 19–21 | `metadata` em layout e páginas; `robots.ts`, `sitemap.ts` |

**Critério de conclusão:** Navegação completa com carrinho em memória/localStorage; confirmação de pedido mock.

---

### 2.4 FASE 3 – Integração Frontend–Backend (Dias 22–34)

**Objetivo:** Next.js consome Medusa para produtos e carrinho; checkout usa Store API; PIX via Mercado Pago.

| Entrega | Dias | Responsabilidade |
|---------|------|------------------|
| 3.1 Cliente Medusa no frontend | 22–24 | `lib/medusa.ts` com `@medusajs/js-sdk`; variáveis de ambiente |
| 3.2 Produtos da API | 24–26 | `/produtos` e `/produtos/[slug]` buscam dados do Medusa |
| 3.3 Carrinho Medusa | 26–29 | `CartProvider` com Store API; createLineItem, updateLineItem, deleteLineItem |
| 3.4 Checkout e complete | 29–32 | Endereço/email no cart; payment session; `POST /store/carts/:id/complete`; redirecionar para `/confirmacao` |
| 3.5 Mercado Pago PIX | 31–33 | API Route PIX, componente QR Code, webhook; valor sempre no servidor |
| 3.6 Ajustes e testes | 33–34 | Fluxo E2E: produto → carrinho → checkout → PIX → confirmação |

**Critério de conclusão:** Pedido concluído no Medusa; PIX via Mercado Pago funcionando; tela de confirmação com dados reais.

#### Mercado Pago (PIX) – detalhes

- **SDK:** `npm install mercadopago`; uso apenas no servidor (Route Handlers).
- **Env:** `MERCADOPAGO_ACCESS_TOKEN` (secreto); `NEXT_PUBLIC_MERCADOPAGO_PUBLIC_KEY`.
- **API Route:** `app/api/pix/route.ts` – recebe só `cartId`/`orderId`; calcula total no servidor; retorna `qr_code`, `qr_code_base64`, `id`; usa `idempotencyKey`.
- **Webhook:** `app/api/webhook/mercadopago/route.ts` – trata notificação de pagamento; busca status com `payment.get()`; se aprovado, atualiza pedido; sempre responde 200; em produção validar HMAC.
- **Painel MP:** Webhooks → URL `https://seudominio.com.br/api/webhook/mercadopago`; evento Payments. Em dev: ngrok ou Cloudflare Tunnel.

---

### 2.5 FASE 4 – Produção (Dias 34–42)

**Objetivo:** Deploy seguro, SEO e segurança revisados, domínio configurado.

| Entrega | Dias | Responsabilidade |
|---------|------|------------------|
| 4.1 SEO e performance | 34–36 | Open Graph, JSON-LD produtos; `next/image`; Core Web Vitals |
| 4.2 Segurança | 36–38 | Headers (X-Content-Type-Options, X-Frame-Options, CSP); HTTPS; secrets só em env |
| 4.3 Deploy backend | 38–39 | Hospedar Medusa (Railway/Render/VPS); env de produção; CORS |
| 4.4 Deploy frontend | 39–40 | Vercel/Netlify com variáveis de ambiente; build e domínio |
| 4.5 DNS e CORS final | 40–41 | CNAME/DNS (ex.: brasilfanstore.dpdns.org); `STORE_CORS` com URL final |
| 4.6 Go-live e checagem | 41–42 | Smoke test; uma compra completa em produção |

**Critério de conclusão:** Loja acessível em produção; compra de teste concluída de ponta a ponta (incluindo PIX em sandbox se aplicável).

---

### 2.6 Resumo de datas (exemplo)

| Fase | Início | Fim |
|------|--------|-----|
| Fase 1 – Backend Medusa | Dia 1 | Dia 12 |
| Fase 2 – Frontend Next.js mock | Dia 1 | Dia 21 |
| Fase 3 – Integração | Dia 22 | Dia 34 |
| Fase 4 – Produção | Dia 35 | Dia 42 |

**Produção estimada:** fim da Semana 6 (ex.: se Dia 1 = 17/02, produção em ~30/03).

---

### 2.7 Checklist final antes de produção

- [ ] Medusa: região BR/BRL; 3 produtos com variantes; Publishable Key; CORS de produção
- [ ] Next.js: todas as rotas funcionando; carrinho e checkout via Medusa; Header com menu hambúrguer no mobile; login/cadastro do cliente (e regra de checkout guest vs logado)
- [ ] Mercado Pago: Access Token e Public Key em env; API Route PIX (valor só no servidor); webhook configurado; em produção, validação HMAC do webhook
- [ ] SEO: metadata, OG, JSON-LD, robots, sitemap
- [ ] Segurança: env sem secrets no repo; HTTPS; headers de segurança
- [ ] DNS: CNAME ou A apontando para o frontend
- [ ] Teste: uma compra completa em produção (incluindo PIX em sandbox)

---

## 3. Estrutura Frontend e Backend

### 3.1 Backend (MedusaJS)

Criado com `npx create-medusa-app@latest` (pode ficar em pasta `backend/` ou `brasil-fan-store-backend/` na raiz do projeto).

```
backend/                          # nome da pasta do app Medusa
├── .medusa/
│   └── server/                   # servidor Node do Medusa
├── medusa-config.js              # config (DB, Redis, CORS, etc.)
├── .env                          # DATABASE_URL, REDIS_URL, COOKIE_SECRET, JWT_SECRET, STORE_CORS
└── (demais arquivos gerados pelo create-medusa-app)
```

- **APIs:** Store em `/store/*` (produtos, carrinho, checkout); Admin em `/admin/*`.
- **Dados:** PostgreSQL (principal), Redis (cache/filas).
- **Admin:** interface em `https://seu-backend.com/app` para regiões, produtos, variantes e API Keys.

A árvore completa é gerada pelo Medusa; o que se configura é o `.env`, região Brasil (BRL), produtos no admin e Publishable Key.

---

### 3.2 Frontend (Next.js)

Pode ficar em `brasil_fan_store/` (substituindo o HTML atual) ou em uma pasta dedicada como `storefront/`.

```
brasil_fan_store/                 # ou storefront/
├── app/
│   ├── layout.tsx               # layout global, fontes, metadata
│   ├── page.tsx                 # home
│   ├── globals.css
│   ├── produtos/
│   │   ├── page.tsx             # lista de produtos
│   │   └── [slug]/page.tsx      # detalhe do produto
│   ├── carrinho/
│   │   └── page.tsx
│   ├── checkout/
│   │   └── page.tsx
│   ├── confirmacao/
│   │   └── page.tsx
│   ├── login/
│   │   └── page.tsx             # login do cliente
│   ├── cadastro/
│   │   └── page.tsx             # registro de cliente
│   ├── conta/
│   │   └── page.tsx             # minha conta (opcional)
│   └── api/                     # Route Handlers Next.js
│       ├── pix/
│       │   └── route.ts         # criar pagamento PIX (Mercado Pago)
│       └── webhook/
│           └── mercadopago/
│               └── route.ts     # webhook Mercado Pago
├── components/
│   ├── Header.tsx
│   ├── Footer.tsx
│   ├── ProductCard.tsx
│   ├── HeroSection.tsx
│   ├── FeaturesSection.tsx
│   ├── Modal.tsx
│   ├── CartDrawer.tsx
│   ├── QRCodeDisplay.tsx
│   └── CartProvider.tsx
├── lib/
│   ├── data.ts                  # produtos/tamanhos mock (até integrar Medusa)
│   └── medusa.ts                # cliente @medusajs/js-sdk
├── public/
│   └── *.jpg                    # imagens (hero, camisas)
├── tailwind.config.ts
├── next.config.js
├── package.json
└── .env.local                   # NEXT_PUBLIC_MEDUSA_BACKEND_URL, NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY,
                                  # MERCADOPAGO_ACCESS_TOKEN, NEXT_PUBLIC_MERCADOPAGO_PUBLIC_KEY
```

---

### 3.3 Onde fica cada responsabilidade

| Responsabilidade | Onde fica |
|------------------|-----------|
| Produtos, regiões, estoque | Backend (Medusa) |
| Carrinho, checkout, pedidos | Backend (Medusa Store API) |
| Telas da loja, navegação, SEO | Frontend (Next.js) |
| Gerar PIX e receber webhook do Mercado Pago | Frontend (Next.js `app/api/`) |
| Autenticação do admin | Backend (Medusa) |
| Autenticação do cliente (login/cadastro) | Backend (Medusa Store API) + Frontend (páginas e estado) |

Resumo: **Backend = Medusa** (API + admin + dados). **Frontend = Next.js** (telas + componentes + API Routes do PIX).

---

## 4. Funcionalidades adicionais do frontend

### 4.1 Menu hambúrguer (Header no mobile)

O Header do Next.js deve incluir **menu hambúrguer em telas pequenas**:

- **Desktop:** logo, links (Início, Produtos, Carrinho) e ícone do carrinho visíveis na barra.
- **Mobile:** botão hambúrguer (ícone de três listras) que abre um drawer ou dropdown com os mesmos links (e, no futuro, categorias).
- Implementação: estado `isMenuOpen` no Header (Client Component); botão hambúrguer visível só em mobile (ex.: `md:hidden`); drawer com links para `/`, `/produtos`, `/carrinho`; em telas maiores os links aparecem na barra (ex.: `hidden md:flex`).

### 4.2 Login do cliente final

O cliente poderá **criar conta e fazer login** para comprar (e opcionalmente ver histórico e endereços).

**Backend (Medusa):** A Store API já oferece fluxo de customer (registro, login por session ou JWT). Verificar se o módulo de customer/auth está habilitado.

**Frontend (Next.js):**

- **Páginas:** `/login` (email + senha), `/cadastro` (registro), e opcionalmente `/conta` (dados, endereços, histórico de pedidos via APIs do Medusa).
- **Header:** se não logado → links "Entrar" e "Cadastrar"; se logado → "Minha conta" e "Sair". Atualizar estado de autenticação (session cookie ou JWT) ao carregar a página e após login/logout.
- **Checkout:** decidir uma das opções:
  - **A)** Permitir compra como visitante (guest) sem login.
  - **B)** Exigir login antes ou durante o checkout.
  - **C)** Oferecer "Criar conta" após a compra e associar o pedido ao cliente.

Incluir no plano de implementação: rotas de login/cadastro/conta; integração com a API de auth do Medusa; exibição condicional no Header; regra escolhida para checkout (guest vs login obrigatório).

---

## 5. Segurança da aplicação

### 5.1 Estado atual (SPA em `index.html`)

| Aspecto | Situação |
|--------|----------|
| Headers de segurança | Nenhum definido no HTML (CSP, X-Frame-Options, etc.); dependem do servidor/hosting. |
| HTTPS | Depende do hosting; não há redirecionamento no código. |
| Secrets | Não há backend; não há tokens no front. |
| Scripts externos (CDN) | Sem SRI (Subresource Integrity); risco se CDN for comprometido. |
| XSS | React escapa por padrão; dados dos produtos são estáticos. Com dados dinâmicos/API, manter escape/sanitização. |
| Dados sensíveis | Checkout e pagamento são mock; nada real trafegado. |

Conclusão: hoje não há camada forte de segurança; o risco é limitado por não haver backend nem pagamento real.

### 5.2 O que o plano já cobre

- **Headers:** X-Content-Type-Options: nosniff, X-Frame-Options, Referrer-Policy, Content-Security-Policy (Fase 4, dias 36–38).
- **HTTPS** e **secrets só em variáveis de ambiente** (nunca no repositório).
- **Medusa:** COOKIE_SECRET e JWT_SECRET no `.env`; CORS (`STORE_CORS`) com URL explícita do frontend.
- **Mercado Pago:** valor do PIX sempre calculado no servidor; Access Token só em Route Handlers; em produção, validação HMAC no webhook.

### 5.3 Itens a implementar / conferir

| Item | Onde | Descrição |
|------|------|------------|
| Headers de segurança | Next.js (`next.config.js`) ou servidor/hosting | CSP, X-Content-Type-Options, X-Frame-Options, Referrer-Policy. |
| HTTPS | Hosting (Vercel/Netlify, etc.) | Forçar HTTPS e redirecionar HTTP → HTTPS. |
| Env e secrets | Backend e Frontend | `.env` / `.env.local` fora do Git; nunca commitar Access Token, JWT_SECRET, COOKIE_SECRET, MERCADOPAGO_ACCESS_TOKEN. |
| Login do cliente | Medusa + Next.js | Session em cookie seguro (HttpOnly, Secure, SameSite) ou JWT em armazenamento seguro; não guardar senha no frontend. |
| Validação de entrada | Backend e API Routes | Validar e sanitizar formulários (login, cadastro, checkout) no servidor; evitar XSS e injection em conteúdo dinâmico e JSON-LD. |
| Webhook Mercado Pago | `app/api/webhook/mercadopago/route.ts` | Além do HMAC em produção: obter status com `payment.get()` na API do MP, não confiar só no corpo da notificação. |
| Dependências | Antes do deploy | Rodar `npm audit` (e corrigir vulnerabilidades críticas/altas) no backend Medusa e no frontend Next.js. |

Checklist de segurança está alinhado com a seção 2.7; esta seção detalha o que cada item significa na prática.

---

## 6. SEO da aplicação

### 6.1 Estado atual (SPA em `index.html`)

| Aspecto | Situação |
|--------|----------|
| Title | Existe: "Brasil Fan Store - Seleção Oficial 🇧🇷". |
| Meta description | Não existe. |
| Meta keywords | Não existe. |
| Viewport | Existe: `width=device-width, initial-scale=1.0`. |
| Lang | Existe: `lang="pt-BR"`. |
| Open Graph | Não existe (og:title, og:description, og:image, og:url, etc.). |
| URL canônica | Não existe (`rel="canonical"`). |
| Robots / sitemap | Não existe (meta robots, `robots.txt`, `sitemap.xml`). |
| JSON-LD / Schema.org | Não existe (Product, Organization, etc.). |
| Conteúdo no HTML | Quase todo o conteúdo é injetado pelo React; o `<body>` tem apenas `<div id="root">`. Para crawlers que não executam JS, a página fica praticamente vazia. |

Conclusão: hoje o SEO está fraco (apenas title e viewport). Faltam descrição, Open Graph, canonical, robots, sitemap, dados estruturados e conteúdo relevante no HTML inicial.

### 6.2 O que o plano já cobre

- **Meta tags:** title, description e keywords em layout e por página (seção 1.4; Fase 2.5 – dias 19–21).
- **Open Graph e URL canônica:** citados na seção 1.4; implementação na Fase 4.1 (dias 34–36).
- **JSON-LD (Schema.org Product):** para produtos; Fase 4.1.
- **Robots e sitemap:** `robots.ts` e `sitemap.ts` no Next.js (Fase 2.5).
- **Conteúdo no HTML inicial:** migração para SSR com Next.js (seção 1.5) garante HTML por rota e conteúdo importante no primeiro paint.

O checklist da seção 2.7 inclui: "SEO: metadata, OG, JSON-LD, robots, sitemap".

### 6.3 Itens a implementar / conferir

| Item | Onde | Descrição |
|------|------|-----------|
| Metadata por página | Cada rota em `app/` | Definir `metadata` (title e description) específicos para home, lista de produtos, página do produto, carrinho, checkout, confirmação, login e cadastro. |
| Open Graph | Layout e páginas dinâmicas | og:title, og:description, og:image, og:url; imagem padrão para compartilhamento (ex.: 1200×630). |
| Twitter Card | Layout (opcional) | twitter:card, twitter:title, twitter:description, twitter:image para melhor exibição no Twitter/X. |
| URL canônica | Páginas | Garantir URL canônica por página (Next.js Metadata API ou `alternates.canonical`). |
| JSON-LD | Página do produto e home | Schema.org Product em `/produtos/[slug]`; Organization ou WebSite na home se desejado. |
| Imagens | Componentes e OG | Uso de `next/image` com `alt` em todas as imagens; dimensões e URL absoluta para og:image. |
| H1 por página | Componentes | Um único `<h1>` por rota, coerente com o title da página. |
| Core Web Vitals | Fase 4.1 e antes do go-live | Medir LCP, FCP e CLS (ex.: Lighthouse ou PageSpeed Insights) e corrigir gargalos. |

Checklist de SEO está alinhado com a seção 2.7; esta seção detalha o que implementar em cada item.

---

## 7. Referências

- [MedusaJS – Documentação](https://docs.medusajs.com/)
- [Next.js – App Router](https://nextjs.org/docs/app)
- [Mercado Pago PIX no Next.js 15 (TabNews)](https://www.tabnews.com.br/zilvodev/como-integrar-mercado-pago-pix-no-next-js-15-app-router)
- Domínio atual (CNAME): `brasilfanstore.dpdns.org` (arquivo `CNAME` na raiz do projeto)
