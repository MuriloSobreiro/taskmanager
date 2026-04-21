# Taskboard — Tutorial de Deploy
### Plano Spark (gratuito) · Sem Cloud Functions · GitHub Pages ou Firebase Hosting

---

## Índice

1. [Como funciona a segurança sem Functions](#1-como-funciona-a-segurança-sem-functions)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Criar o projeto Firebase](#3-criar-o-projeto-firebase)
4. [Configurar o index.html](#4-configurar-o-indexhtml)
5. [Instalar o Firebase CLI](#5-instalar-o-firebase-cli)
6. [Deploy das Firestore Security Rules](#6-deploy-das-firestore-security-rules)
7. [Publicar no GitHub Pages](#7-publicar-no-github-pages)
8. [Criar o primeiro administrador](#8-criar-o-primeiro-administrador)
9. [Gerar códigos de convite](#9-gerar-códigos-de-convite)
10. [Fluxo do usuário final](#10-fluxo-do-usuário-final)
11. [Promover outros administradores](#11-promover-outros-administradores)
12. [Alternativa: Firebase Hosting](#12-alternativa-firebase-hosting)
13. [Operação e manutenção](#13-operação-e-manutenção)
14. [Estimativa de custos](#14-estimativa-de-custos)
15. [Solução de problemas](#15-solução-de-problemas)

---

## 1. Como funciona a segurança sem Functions

Sem Cloud Functions, a autorização usa as **Firestore Security Rules** com leituras cross-documento (`get()`):

```
Cada operação em /buckets verifica:

 Firestore Rule                  Firestore DB
 ─────────────                  ────────────
 isApproved()  ──── get() ────▶  /users/{uid}
                                   approved: true ✅
                                   admin: true    ✅

 Se ambos OK → operação permitida
 Se falhar    → operação bloqueada no servidor
```

**O front-end não tem papel na autorização.** Mesmo que alguém inspecione o JavaScript, não há como burlar as regras — elas rodam no servidor do Firestore, nunca no browser.

| Responsabilidade | Onde fica |
|---|---|
| Autenticação (quem é você) | Firebase Auth |
| Aprovação (tem convite?) | Firestore `/users/{uid}.approved` via Security Rule |
| Permissões (é admin?) | Firestore `/users/{uid}.admin` via Security Rule |
| Dados da aplicação | Firestore `/users/{uid}/buckets` |
| Convites (geração/consumo) | Firestore `/inviteCodes` via Security Rule + transação client-side |

---

## 2. Pré-requisitos

| Ferramenta | Como obter |
|---|---|
| Node.js 18+ | https://nodejs.org |
| Git | https://git-scm.com |
| Conta Google | https://accounts.google.com |
| Conta GitHub | https://github.com |

---

## 3. Criar o projeto Firebase

### 3.1 — Novo projeto

1. Acesse [https://console.firebase.google.com](https://console.firebase.google.com)
2. **Adicionar projeto** → nome (ex: `meu-taskboard`) → desative Analytics → **Criar**

### 3.2 — Ativar Authentication

1. Menu: **Build → Authentication → Começar**
2. Aba **Sign-in method → E-mail/senha → Ativar → Salvar**

### 3.3 — Criar o Firestore

1. Menu: **Build → Firestore Database → Criar banco de dados**
2. Selecione **modo de produção** *(as rules corretas serão aplicadas no passo 6)*
3. Região recomendada: `southamerica-east1 (São Paulo)`

### 3.4 — Obter a configuração SDK

1. Engrenagem ⚙️ → **Configurações do projeto → Seus apps → `</>`** (Web)
2. Registre um apelido (ex: `taskboard-web`) → **Registrar app**
3. Copie o objeto `firebaseConfig`:

```javascript
const firebaseConfig = {
  apiKey:            "AIzaSy...",
  authDomain:        "meu-taskboard.firebaseapp.com",
  projectId:         "meu-taskboard",
  storageBucket:     "meu-taskboard.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123:web:abc"
};
```

---

## 4. Configurar o index.html

Localize a seção `// 1. CONFIG` no `index.html` e cole seus valores:

```javascript
const FIREBASE_CONFIG = {
  apiKey:            "AIzaSy...",       // ← seus dados aqui
  authDomain:        "meu-taskboard.firebaseapp.com",
  projectId:         "meu-taskboard",
  storageBucket:     "meu-taskboard.appspot.com",
  messagingSenderId: "123456789",
  appId:             "1:123:web:abc"
};
```

Salve o arquivo.

> **A `apiKey` é pública por design** — a segurança real está nas Security Rules.

---

## 5. Instalar o Firebase CLI

```bash
npm install -g firebase-tools
firebase login

# Na pasta do projeto:
firebase init
```

Na inicialização, selecione apenas:
- ✅ **Firestore** (rules)
- ✅ **Hosting** (opcional, veja seção 12)

> Não selecione **Functions** — não são necessárias neste setup.

Quando perguntado:
- **Use an existing project** → selecione seu projeto
- **Firestore Rules file**: Enter (mantém `firestore.rules`)
- **Public directory**: `.` (ponto)
- **Configure as a SPA**: Yes
- **Overwrite index.html**: **No**

---

## 6. Deploy das Firestore Security Rules

```bash
firebase deploy --only firestore:rules
```

Verifique em **Firebase Console → Firestore → Regras** se as regras estão ativas.

> ⚠️ Este é o passo mais importante. Sem as rules corretas, os dados ficam totalmente bloqueados.

---

## 7. Publicar no GitHub Pages

### 7.1 — Criar repositório

1. [https://github.com/new](https://github.com/new) → nome `taskboard` → **Create repository**

### 7.2 — Push inicial

```bash
git init
git add .
git commit -m "feat: taskboard inicial"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/taskboard.git
git push -u origin main
```

### 7.3 — Ativar Pages

1. Repositório → **Settings → Pages**
2. **Source**: Deploy from a branch → Branch: `main` / `/ (root)` → **Save**

Site disponível em ~2 minutos em:
```
https://SEU_USUARIO.github.io/taskboard/
```

### 7.4 — Autorizar o domínio no Firebase ⚠️

Sem este passo, o login retorna erro `auth/unauthorized-domain`:

1. **Firebase Console → Authentication → Settings → Authorized domains**
2. **Add domain** → `SEU_USUARIO.github.io` → **Add**

---

## 8. Criar o primeiro administrador

Sem Functions, o primeiro admin é configurado diretamente no Firebase Console. **Faça isso uma vez, logo após criar sua conta no app.**

### 8.1 — Criar o primeiro código de convite manualmente

No **Firebase Console → Firestore → Dados**:

1. Crie a coleção `inviteCodes`
2. Documento ID: `ADMIN-0001`
3. Adicione os campos:

| Campo | Tipo | Valor |
|---|---|---|
| `used` | boolean | `false` |
| `createdBy` | string | `"manual"` |
| `expiresAt` | timestamp | data futura (ex: 30 dias) |

### 8.2 — Criar sua conta no app

1. Acesse o app → **Criar conta**
2. Use o código `ADMIN-0001`
3. Sua conta é criada e aprovada

### 8.3 — Promover a conta a admin no Firestore Console

1. **Firebase Console → Firestore → Dados → users → [seu UID]**
2. Clique em **Editar documento** (ícone de lápis)
3. Altere o campo `admin` de `false` para `true`
4. Salve

> O UID está disponível em **Authentication → Users → copiar UID**.

### 8.4 — Atualizar a sessão

No app, faça **logout** e **login** novamente. A aba **🛡️ Admin** aparecerá no header.

---

## 9. Gerar códigos de convite

Com acesso ao painel Admin:

1. Clique em **🛡️ Admin** no header
2. Configure:
   - **Quantidade**: 1 a 50 códigos
   - **Validade**: de 7 a 90 dias
3. Clique **✦ Gerar**
4. Os códigos aparecem em chips copiáveis
5. Use **📋 Copiar todos** para copiar a lista e enviar por e-mail, WhatsApp etc.

---

## 10. Fluxo do usuário final

```
1. Recebe código do administrador (ex: ABCD-EFGH)

2. Acessa o app → aba "Criar conta"
   ┌──────────────────────────────────┐
   │ E-mail  [usuario@email.com     ] │
   │ Senha   [••••••••              ] │
   │ Convite [ABCD-EFGH             ] │
   │         [      Criar conta     ] │
   └──────────────────────────────────┘

3. O app executa:
   a) Cria conta no Firebase Auth
   b) Cria /users/{uid} com approved=false
   c) Transação Firestore: marca código como used=true
   d) Atualiza /users/{uid} com approved=true
   e) Firestore Rule valida o passo (d) server-side
   f) App carrega normalmente

4. Próximas sessões: login normal, sem código
```

---

## 11. Promover outros administradores

No painel Admin:

1. Abra **Firebase Console → Authentication → Users**
2. Copie o UID do usuário que deseja promover
3. No painel Admin do app → seção **Promover usuário**
4. Cole o UID e clique **🛡️ Promover**
5. A Security Rule valida que apenas admins podem executar esta ação

O usuário promovido precisa fazer **logout e login** para ver a aba Admin.

---

## 12. Alternativa: Firebase Hosting

Se preferir hospedar no Firebase ao invés do GitHub Pages:

```bash
firebase deploy --only hosting
```

URL: `https://SEU_PROJETO.web.app`

Vantagens sobre GitHub Pages:
- Não precisa adicionar domínio em Authorized Domains (já é autorizado automaticamente)
- CDN global
- Preview channels para testar antes de publicar

---

## 13. Operação e manutenção

### Atualizar o app
```bash
# Edite index.html, depois:
git add index.html
git commit -m "feat: minha alteração"
git push origin main
# GitHub Pages atualiza automaticamente
```

### Atualizar as Security Rules
```bash
# Edite firestore.rules, depois:
firebase deploy --only firestore:rules
```

### Revogar um código não utilizado
**Firestore Console → inviteCodes → [código] → Excluir documento**

### Desativar um usuário
**Firebase Console → Authentication → Users → selecionar → Desativar**

A desativação bloqueia o login, mas não apaga os dados do Firestore.

### Remover privilégio de admin
**Firestore Console → users → [uid] → editar campo `admin` → false**

### Ver quem usou qual código
**Firestore Console → inviteCodes → filtrar por `used == true`** — o campo `usedBy` contém o UID.

---

## 14. Estimativa de custos

Todo o projeto opera no **plano Spark (100% gratuito)**:

| Serviço | Limite Spark | Uso estimado (30 usuários ativos/dia) |
|---|---|---|
| Authentication | 10.000 logins/mês | ~300/mês ✅ |
| Firestore reads | 50.000/dia | ~5.000/dia* ✅ |
| Firestore writes | 20.000/dia | ~800/dia ✅ |
| Firestore storage | 1 GB | < 5 MB ✅ |

*Inclui o extra de 1 `get()` por operação de bucket (verificação de aprovação nas Rules).

> **Sem Cloud Functions = sem necessidade do plano Blaze.** O app inteiro funciona no Spark.

---

## 15. Solução de problemas

### "auth/unauthorized-domain"
→ Adicione `SEU_USUARIO.github.io` em **Authentication → Settings → Authorized domains**.

### Aba Admin não aparece após editar o Firestore
→ Faça logout e login. O app lê o documento do usuário no boot, então uma nova sessão é necessária.

### "Missing or insufficient permissions"
→ As Security Rules não foram deployadas ou o usuário não tem `approved: true`. Verifique:
```bash
firebase deploy --only firestore:rules
```

### Código de convite retorna "não encontrado"
→ Verifique se o documento existe em **Firestore → inviteCodes** e se o `expiresAt` é uma data futura.

### "The query requires an index"
→ Clique no link do erro no console do browser — ele abre o Firebase Console para criar o índice automaticamente.

### Usuário criado mas não consegue logar depois
→ Verifique no **Firestore → users → [uid]** se `approved == true`. Se não, o código não foi consumido corretamente.

### GitHub Pages mostra 404
→ Aguarde 2–3 min após o push. Se persistir, verifique **Settings → Pages**.
