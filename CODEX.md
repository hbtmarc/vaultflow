# VaultFlow (Dev) — Contexto para Codex

## Objetivo
Aplicação web (SaaS/MVP) de controle financeiro com:
- Autenticação Firebase (Email/Senha e Google)
- Firestore como banco
- Wallets por usuário (pessoais) + Wallet Casal (agregado)
- UI simples e direta (fintech-like), com foco em performance e consistência
- Auto-sync do Casal: ao adicionar/excluir transações nas wallets pessoais, atualiza automaticamente `reports` e `transactions_view` do Casal

## Restrições técnicas (importante)
- Projeto “single-file” no frontend: manter o app principal em `docs/index.html`.
- Sem build step obrigatório (sem bundlers). Firebase via ES Modules CDN.
- Publicação via GitHub Pages (pasta `docs/`).
- Evitar criar múltiplos arquivos JS/CSS quando não for estritamente necessário.

## Ambientes
- Firebase Project (DEV): VaultFlow-Dev
- Project ID: vaultflow-dev-marc35
- Auth: Email/Senha e Google habilitados
- Firestore habilitado
- GitHub Pages: usa `docs/index.html`

## Usuários (DEV / allowlist)
Somente estes e-mails devem conseguir acessar o app (também reforçado nas Rules):
- hbmarc35@gmail.com  (Marcelino)
- lzmarc25@gmail.com  (Luiza)
- herbertmarck@gmail.com (Conta Casal / Viewer)

---

# Firestore Schema (canônico)

## Visão geral (árvore)
- `/users/{uid}`
  - **Subcoleção**: `/users/{uid}/memberships/{walletId}`  (atalho para listagem)
- `/wallets/{walletId}`
  - **Subcoleção**: `/wallets/{walletId}/members/{uid}` (ACL: permissões oficiais)
  - **Subcoleção**: `/wallets/{walletId}/categories/{catId}` (somente wallets pessoais)
  - **Subcoleção**: `/wallets/{walletId}/transactions/{txnId}` (somente wallets pessoais)
  - **Subcoleção**: `/wallets/{walletId}/reports/{yyyy-MM}` (somente wallet agregada: w_casal)
  - **Subcoleção**: `/wallets/{walletId}/transactions_view/{viewId}` (somente wallet agregada: w_casal)

## 1) users

### `/users/{uid}` (documento)
**ID**: `uid` do Firebase Auth  
**Campos sugeridos**
- `displayName` (string)
- `email` (string, lower-case)
- `createdAt` (timestamp)

**Observação**
- Este documento é “perfil básico”. Cada usuário só acessa o próprio.

### `/users/{uid}/memberships/{walletId}` (subcoleção — atalho)
**ID do doc**: `walletId` (ex.: `w_marcelino`, `w_luiza`, `w_casal`)  
**Campos**
- `role` (string: `owner | editor | viewer`)
- `createdAt` (timestamp)

**Ponto-chave**
- Isso NÃO define permissão. É apenas um “índice” para o front listar wallets rapidamente.
- A permissão real é `wallets/{walletId}/members/{uid}`.

---

## 2) wallets

### `/wallets/{walletId}` (documento)
**IDs padrão do projeto**
- `w_marcelino` (personal)
- `w_luiza` (personal)
- `w_casal` (aggregate)

**Campos**
- `name` (string) — ex.: "Marcelino (Pessoal)"
- `type` (string) — `personal` ou `aggregate`
- `createdBy` (uid)
- `createdAt` (timestamp)

**Somente para `aggregate`**
- `sources` (array de DocumentReference)
  - ex.: `[ /wallets/w_marcelino, /wallets/w_luiza ]`

---

## 3) ACL (permissões oficiais)

### `/wallets/{walletId}/members/{uid}` (subcoleção — ACL canônica)
**ID do doc**: `uid`  
**Campos**
- `role` (string: `owner | editor | viewer`)
- `createdAt` (timestamp)
- `createdBy` (uid) (opcional)

**Regras de negócio**
- `owner`: pode tudo na wallet (inclusive gerenciar membros)
- `editor`: pode criar/editar/excluir categorias e transações (mas não gerencia membros)
- `viewer`: só lê (no casal: vê totais + detalhes; nas pessoais: pode ser bloqueado no front por UX, mas Rules permitem leitura se for membro)

---

## 4) categories (somente wallets pessoais)

### `/wallets/{walletId}/categories/{catId}`
**Campos**
- `name` (string) — ex.: “Alimentação”
- `kind` (string) — `expense | income | both`
- `createdAt` (timestamp)
- `createdBy` (uid)

---

## 5) transactions (somente wallets pessoais)

### `/wallets/{walletId}/transactions/{txnId}`
**Campos**
- `date` (timestamp) — data base da transação
- `amountCents` (number) — valor em centavos (inteiro)
- `type` (string) — `income | expense`
- `categoryId` (string)
- `categoryName` (string) — denormalizado para performance/consistência do histórico
- `note` (string) — observação
- `createdAt` (timestamp)
- `createdBy` (uid)

---

## 6) wallet agregada (Casal)

### `w_casal` — Totais do mês
#### `/wallets/w_casal/reports/{yyyy-MM}`
**ID do doc**: `YYYY-MM` (ex.: `2025-12`)  
**Campos**
- `month` (string) — `YYYY-MM`
- `createdAt`, `createdBy`
- `updatedAt`, `updatedBy`
- `sources` (map)
  - `sources.w_marcelino.incomeCents` (number)
  - `sources.w_marcelino.expenseCents` (number)
  - `sources.w_luiza.incomeCents` (number)
  - `sources.w_luiza.expenseCents` (number)
  - cada fonte pode ter `updatedAt/updatedBy`

**Semântica**
- Totais do casal = soma das fontes no `sources`.

### `w_casal` — Detalhes consolidados
#### `/wallets/w_casal/transactions_view/{viewId}`
**ID do doc**: `${sourceWalletId}_${sourceTxnId}`
- Ex.: `w_marcelino_abc123` ou `w_luiza_def456`

**Campos**
- `sourceWalletId` (string) — `w_marcelino` ou `w_luiza`
- `sourceTxnId` (string)
- `sourceUid` (uid)
- `date` (timestamp)
- `amountCents` (number)
- `type` (string) — `income | expense`
- `categoryId` (string)
- `categoryName` (string)
- `note` (string)
- `syncedAt` (timestamp)

**Semântica**
- É um “espelho” para listar detalhes do casal rapidamente, sem query cross-wallet.

---

# Fluxo de Auto-sync do Casal (padrão atual)
1) Usuário adiciona/exclui transação em wallet pessoal (`/wallets/w_marcelino|w_luiza/transactions`)
2) No mesmo `runTransaction()`:
   - escreve/atualiza `w_casal/reports/{yyyy-MM}` (totais por fonte)
   - escreve/remove `w_casal/transactions_view/{sourceWalletId}_{sourceTxnId}` (detalhes)
3) Conta viewer (`herbertmarck@gmail.com`) lê `reports` e `transactions_view`, mas não escreve.

Regra importante de implementação:
- Em `runTransaction()`, **todas as leituras devem acontecer antes de qualquer escrita**, senão ocorre:
  "Firestore transactions require all reads to be executed before all writes."

---

# Firestore Rules (cole no Console)
Atenção:
- Estas rules incluem allowlist por e-mail (DEV), além de membership por wallet.
- Viewer tem leitura do casal; owners/editors escrevem.

```rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ===== Helpers =====

    function signedIn() {
      return request.auth != null;
    }

    // DEV allowlist (reforço de segurança)
    function allowedDevUser() {
      return signedIn()
        && request.auth.token.email != null
        && request.auth.token.email in [
          "hbmarc35@gmail.com",
          "lzmarc25@gmail.com",
          "herbertmarck@gmail.com"
        ];
    }

    function isMember(walletId) {
      return allowedDevUser()
        && exists(/databases/$(database)/documents/wallets/$(walletId)/members/$(request.auth.uid));
    }

    function memberRole(walletId) {
      return get(/databases/$(database)/documents/wallets/$(walletId)/members/$(request.auth.uid)).data.role;
    }

    function canWriteWallet(walletId) {
      return isMember(walletId)
        && (memberRole(walletId) == 'owner' || memberRole(walletId) == 'editor');
    }

    // ===== Users (perfil) =====
    match /users/{uid} {
      allow read, write: if allowedDevUser() && request.auth.uid == uid;

      // Atalho de listagem (cada usuário só acessa o próprio)
      match /memberships/{walletId} {
        allow read, write: if allowedDevUser() && request.auth.uid == uid;
      }
    }

    // ===== Wallets =====
    match /wallets/{walletId} {

      // Ler metadata da wallet: precisa ser membro
      allow read: if isMember(walletId);

      // Escrever metadata da wallet: só owner/editor
      allow write: if canWriteWallet(walletId);

      // ACL (membros)
      match /members/{uid} {
        allow read: if isMember(walletId);

        // Só owner pode gerenciar membros (criar/alterar/excluir)
        allow write: if isMember(walletId) && memberRole(walletId) == 'owner';
      }

      // Categorias
      match /categories/{catId} {
        allow read: if isMember(walletId);
        allow write: if canWriteWallet(walletId);
      }

      // Transações
      match /transactions/{txnId} {
        allow read: if isMember(walletId);
        allow write: if canWriteWallet(walletId);
      }

      // Reports (usado no casal agregado, mas genérico aqui)
      match /reports/{monthId} {
        allow read: if isMember(walletId);
        allow write: if canWriteWallet(walletId);
      }

      // View consolidada (detalhes do casal)
      match /transactions_view/{viewId} {
        allow read: if isMember(walletId);
        allow write: if canWriteWallet(walletId);
      }
    }

    // ===== Default deny =====
    match /{document=**} {
      allow read, write: if false;
    }
  }
}


---

Checklist de dados mínimos (DEV)

Wallets (documentos em /wallets):

w_marcelino (type: personal)

w_luiza (type: personal)

w_casal (type: aggregate, sources: [/wallets/w_marcelino, /wallets/w_luiza])

Members (ACL canônica):

/wallets/w_marcelino/members/{uidMarcelino} role: owner

/wallets/w_luiza/members/{uidLuiza} role: owner

/wallets/w_casal/members/{uidMarcelino} role: owner

/wallets/w_casal/members/{uidLuiza} role: owner

/wallets/w_casal/members/{uidViewerCasal} role: viewer

Memberships (atalho por usuário — opcional, mas recomendado para UX):

/users/{uidMarcelino}/memberships/w_marcelino role: owner

/users/{uidMarcelino}/memberships/w_casal role: owner

/users/{uidLuiza}/memberships/w_luiza role: owner

/users/{uidLuiza}/memberships/w_casal role: owner

/users/{uidViewerCasal}/memberships/w_casal role: viewer


---


