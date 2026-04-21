# Taskboard

Gerenciador de tarefas em buckets com autenticação e painel admin.
**Plano Spark (gratuito) · Sem Cloud Functions.**

**[→ Tutorial completo](TUTORIAL.md)**

## Stack

- HTML + CSS + JS vanilla — sem build step, sem bundler
- Firebase Authentication (email/senha)
- Cloud Firestore (sync em tempo real, offline-first)
- GitHub Pages (hospedagem gratuita)

## Estrutura

```
taskboard/
├── index.html        ← app completo — cole seu firebaseConfig aqui
├── firestore.rules   ← regras de segurança
├── firebase.json     ← config do CLI
├── TUTORIAL.md       ← guia passo a passo
└── README.md
```

## Setup em 5 passos

```bash
# 1. Clone
git clone https://github.com/SEU_USUARIO/taskboard.git && cd taskboard

# 2. Cole seu firebaseConfig em index.html (seção "1. CONFIG")

# 3. Instale o CLI
npm install -g firebase-tools && firebase login && firebase init

# 4. Deploy das Security Rules
firebase deploy --only firestore:rules

# 5. Push para GitHub Pages
git add . && git commit -m "init" && git push origin main
```

Depois: adicione `SEU_USUARIO.github.io` em Firebase → Authentication → Authorized Domains.

## Funcionalidades

- Buckets de tarefas por pessoa ou projeto  
- Tarefas com checkbox, descrição e progresso  
- Login com código de convite (acesso controlado)  
- Dados isolados por usuário (Security Rules server-side)  
- Painel Admin: gerar convites, ver estatísticas, promover admins  
- Sync em tempo real e modo offline
