# DocuViagem ✈️ 🔒 — Sistema Seguro de Coleta de Documentos (Showcase)

> **Aviso:** Este é um repositório de demonstração (*showcase*). O código-fonte original é mantido em repositório privado por questões de segurança e confidencialidade da agência cliente.

---

## 🎯 O Problema

Você mandaria a foto do seu cartão de crédito pelo WhatsApp?

Durante o desenvolvimento de um projeto para o meu portfólio, um empreendedor — dono de uma agência de viagens — me relatou um problema real: ele estava perdendo vendas porque clientes se recusavam a enviar documentos sensíveis (RG, CPF, Passaporte) e dados de pagamento por aplicativos de mensagens comuns. Parei os projetos paralelos para resolver essa dor.

## 💡 A Solução

O **DocuViagem** é uma plataforma web *white label* que permite agências de viagens coletarem documentos de clientes e dados de cartão de crédito com criptografia ponta a ponta e total conformidade com a LGPD.

[▶️ **CLIQUE AQUI PARA VER O VÍDEO DO SISTEMA FUNCIONANDO**](https://www.linkedin.com/feed/update/urn:li:activity:7481316393901207552/)

---

## 🛠️ Stack Tecnológica

| Camada | Tecnologia | Decisão |
|--------|-----------|---------|
| Backend | Node.js 18+ + Express 4 | Ecossistema maduro, excelente suporte a I/O assíncrono para operações de arquivo e banco |
| Banco de dados | MySQL 5.7 com pool de conexões | Requisito do cliente (infraestrutura Hostgator existente) |
| Criptografia | AES-256-CBC via módulo `crypto` nativo | Sem dependências externas para operações críticas de segurança |
| Autenticação | JWT (Access + Refresh Token) + Google OAuth 2.0 | Stateless, escalável, com suporte a renovação silenciosa de sessão |
| Frontend | HTML5 + CSS3 + JavaScript puro (SPA via Fetch API) | Zero dependências de framework — bundle size mínimo, carregamento instantâneo |
| Upload | Multer com storage local | MVP pragmático; arquitetura preparada para migração para S3 |
| E-mail | Gmail API via Conta de Serviço Google | Contorna bloqueios de SMTP em produção; sem senhas de app expostas |
| Hospedagem | Railway | Deploy contínuo via GitHub, variáveis de ambiente gerenciadas pelo painel |

---

## 🏗️ Arquitetura do Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                        RAILWAY                              │
│                                                             │
│  ┌─────────────────────┐    ┌──────────────────────────┐   │
│  │   Express (API)     │    │   Static Files           │   │
│  │   /api/*            │◄──►│   index.html (cliente)   │   │
│  │                     │    │   admin-*.html (admin)   │   │
│  └──────────┬──────────┘    └──────────────────────────┘   │
│             │                                               │
└─────────────┼───────────────────────────────────────────────┘
              │
              ├──► MySQL 5.7 (Hostgator) — metadados + logs
              ├──► /uploads — arquivos físicos (exclusão automática)
              └──► Gmail API — notificações transacionais
```

### Separação de contextos

O sistema opera com **dois fluxos completamente isolados**, sem compartilhar rotas, sessões ou lógica de autenticação:

```
CLIENTE                              ADMIN
────────────────────                 ────────────────────
/index.html                          /admin-login.html
/register.html          ≠            /admin-dashboard.html
/terms.html                          /admin-register.html
/app.html                            /admin-*.html

JWT role: 'client'                   JWT role: 'admin'
Sem senha própria                    bcrypt hash (custo 12)
Login por email/Google               Login por email+senha
```

---

## 🔐 Destaques Técnicos

### 1. Criptografia de Ponta a Ponta com Arquitetura de Chaves por Envio

O ponto mais crítico do projeto. A solução adotada vai além de criptografar com uma chave mestre do servidor — o que criaria um ponto único de falha.

**O fluxo:**

```
[Cliente adiciona cartão]
        │
        ▼
encrypt(AES-256-CBC, chave do servidor) ──► banco

[Cliente clica em Enviar]
        │
        ▼
Gera senha de 6 chars ──► exibida na tela ──► enviada por e-mail
        │                  NUNCA salva no banco
        ▼
decrypt(chave servidor) ──► dados em claro
        │
        ▼
encrypt(AES-256-CBC, senha do cliente) ──► banco

[Admin tenta acessar]
        │
        ▼
Digita senha do cliente ──► decrypt tenta rodar
        │
   ┌────┴────┐
  OK        Falha silenciosa (dados ilegíveis)
   │
   ▼
Dados visíveis apenas nessa sessão
```

Cada envio gera uma nova senha, criptografando os dados daquele envio de forma independente. Se o cliente enviar documentos uma segunda vez, os documentos anteriores permanecem acessíveis separadamente — **arquitetura de submissions**, não de usuários.

### 2. Arquitetura de Submissions

Em vez de associar documentos diretamente ao usuário, cada envio cria um `submission` independente com sua própria chave de acesso. Isso resolve um problema real:

> *"E se o cliente enviar documentos duas vezes antes do admin abrir?"*

```sql
submissions
├── id
├── user_id
├── submitted_at
├── first_admin_access_at  -- registra quando admin acessou pela 1ª vez
├── delete_scheduled_at    -- T+24h do primeiro acesso
└── deleted

documents ──► submission_id (FK)
card_data ──► submission_id (FK)
```

### 3. Refresh Token com Rotação e Hash no Banco

O refresh token nunca é armazenado em texto claro — apenas seu hash SHA-256. A cada renovação, o token anterior é revogado automaticamente.

```javascript
// Nunca armazenamos o token cru
const tokenHash = crypto
  .createHash('sha256')
  .update(rawToken)
  .digest('hex');

// Cookie httpOnly — inacessível por JavaScript
res.cookie('refresh_token', rawToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'none',
  maxAge: 7 * 24 * 60 * 60 * 1000
});
```

### 4. Validação de CPF com Algoritmo da Receita Federal

Máscaras de input são UX, não validação. O formulário de cartão implementa o algoritmo dos dois dígitos verificadores antes de aceitar qualquer submissão — tanto no frontend (feedback imediato) quanto no backend (segurança real).

```javascript
function validarCPF(cpf) {
  const nums = cpf.replace(/\D/g, '');
  if (nums.length !== 11) return false;
  if (/^(\d)\1{10}$/.test(nums)) return false;

  let soma = 0;
  for (let i = 0; i < 9; i++) soma += parseInt(nums[i]) * (10 - i);
  let resto = (soma * 10) % 11;
  if (resto === 10 || resto === 11) resto = 0;
  if (resto !== parseInt(nums[9])) return false;

  soma = 0;
  for (let i = 0; i < 10; i++) soma += parseInt(nums[i]) * (11 - i);
  resto = (soma * 10) % 11;
  if (resto === 10 || resto === 11) resto = 0;
  return resto === parseInt(nums[10]);
}
```

### 5. Conformidade com a LGPD

Não é um checkbox — está no core do sistema:

| Princípio LGPD | Implementação |
|----------------|--------------|
| Consentimento | Aceite registrado com IP, User-Agent e timestamp antes de qualquer coleta |
| Minimização | Apenas dados necessários para a prestação do serviço são coletados |
| Segurança | AES-256-CBC em repouso, TLS em trânsito, acesso por senha única |
| Exclusão | Arquivos físicos deletados após visualização pelo admin |
| Rastreabilidade | Log de auditoria completo de todos os acessos, downloads e tentativas falhas |
| Temporalidade | Exclusão automática programada 24h após primeiro acesso do admin |

### 6. Log de Auditoria

Toda ação relevante é registrada com contexto completo:

```
audit_logs
├── user_id       -- quem fez
├── action        -- o que fez (LOGIN, UPLOAD, ADMIN_VIEW, DOWNLOAD, DELETE...)
├── entity_type   -- sobre o que (document, submission, user)
├── entity_id     -- qual registro específico
├── description   -- descrição legível
├── ip_address    -- de onde (IPv4/IPv6)
├── user_agent    -- com qual dispositivo/browser
└── created_at    -- quando
```

### 7. White Label via config.json

O sistema foi projetado para ser replicado para múltiplas agências. Toda identidade visual é externalizada:

```json
{
  "agency_name": "Epic Travels",
  "primary_color": "#0a1f6b",
  "accent_color": "#c9a84c",
  "logo_url": "/assets/logo.png",
  "term_title": "Termo de Uso — Epic Travels",
  "term_body": ["..."]
}
```

O `theme-loader.js` aplica CSS variables, substitui textos e carrega o logo antes da primeira renderização — sem reescrever HTML ou CSS para cada cliente.

---

## 📁 Estrutura do Projeto

```
docuviagem/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── audit.js        # Log de auditoria
│   │   │   ├── crypto.js       # AES-256 + geração de senha por envio
│   │   │   ├── database.js     # Pool MySQL
│   │   │   ├── email.js        # Gmail API + templates HTML
│   │   │   ├── jwt.js          # Access/Refresh tokens com rotação
│   │   │   └── passport.js     # Google OAuth 2.0
│   │   ├── controllers/        # Lógica de negócio
│   │   ├── middlewares/        # Auth, Admin, Upload
│   │   ├── routes/             # Separação cliente/admin
│   │   └── app.js
│   └── uploads/                # Arquivos temporários (gitignored)
├── frontend/
│   ├── js/
│   │   ├── api.js              # Camada HTTP centralizada
│   │   ├── auth.js             # Gerenciamento de sessão
│   │   ├── app.js              # Fluxo do cliente
│   │   ├── admin-dashboard.js  # Fluxo do admin
│   │   └── theme-loader.js     # White label
│   ├── config.json             # Identidade visual da instância
│   └── *.html                  # Páginas do cliente e admin
└── database/
    └── schema.sql
```

---

## 🗄️ Modelo de Dados (simplificado)

```
users
  id, name, email, phone, google_id, role, is_admin
  auth_provider, submitted_at, admin_password_hash

submissions
  id, user_id → users
  submitted_at, first_admin_access_at, delete_scheduled_at, deleted

documents
  id, user_id → users, submission_id → submissions
  doc_type, original_name, stored_name, mime_type, file_size, status

card_data
  id, user_id → users, submission_id → submissions
  card_label, last_four, card_brand
  encrypted_data, iv  ← AES-256-CBC com senha do cliente
  has_photo_front, has_photo_back

term_acceptances
  id, user_id → users, term_version, accepted_at, ip_address, user_agent

audit_logs
  id, user_id → users, action, entity_type, entity_id
  description, ip_address, user_agent, created_at

refresh_tokens
  id, user_id → users, token_hash (SHA-256), expires_at, revoked

admin_invites
  id, email, token_hash, invited_by → users, expires_at, used
```

---

## 🌊 Fluxos Principais

### Fluxo do Cliente

```
Link da agência → index.html
    │
    ├─ Email já cadastrado ──────────────────► app.html
    │
    └─ Email novo → register.html → terms.html → app.html
                         │
                   nome + telefone
                   (ou Google OAuth)
                         │
                    app.html
                    ├── Upload de documentos (câmera ou galeria)
                    └── Dados do cartão (formulário com validação CPF)
                         │
                    [Enviar]
                         │
                    Senha de 6 chars ──► tela + e-mail
                    Docs ocultados do dashboard
                    Cliente repassa a senha ao agente
```

### Fluxo do Admin

```
admin-login.html → admin-dashboard.html
    │
    ├── Card: Total de clientes
    ├── Card: Aguardando envio        (clicável → filtra)
    └── Card: Aguardando visualização (clicável → filtra)
         │
    [Ver docs] → modal: digita senha do cliente
         │
    decrypt(AES-256, senha) → dados visíveis
         │
    ├── Baixar documentos (nome formatado: "João Silva - CNH.pdf")
    ├── Ver cartão completo (número, CVV, validade, CPF)
    └── [Confirmo que baixei — Excluir arquivos]
              │
         Arquivos físicos deletados
         Linha do banco mantida para auditoria
         Timer de expiração removido
         Cliente liberado para novo envio
```

---

## 🔒 Camadas de Segurança

```
Requisição HTTP
      │
      ▼
[Helmet]        → headers de segurança (CSP, HSTS, X-Frame-Options...)
      │
      ▼
[Rate Limiter]  → máx. 50 req/15min em /api/auth/*
      │
      ▼
[CORS]          → whitelist de origens
      │
      ▼
[requireAuth]   → valida JWT, rejeita token expirado com código específico
      │
      ▼
[requireAdmin]  → verifica role='admin' no payload do token
      │
      ▼
Controller      → sanitização de inputs + validações de negócio
      │
      ▼
[Multer]        → whitelist de MIME types, limite de 10MB, nome aleatório no disco
```

---

## 👨‍💻 Autor

**Antonio Kerr**

- [LinkedIn](https://www.linkedin.com/in/antoniokerr/)
- [GitHub](https://github.com/tonykerr07)
