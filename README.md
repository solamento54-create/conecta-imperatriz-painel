# Conecta Imperatriz — Painel Web (v3)

Painel administrativo para funcionários da Prefeitura de Imperatriz/MA, **conectado à API REST** (FastAPI).

---

## 🆕 O que mudou na v3

A versão anterior (v2) era um **mockup com dados fixos** no próprio JavaScript. A v3 trocou tudo isso por chamadas reais à API:

| Antes (v2) | Agora (v3) |
|------------|------------|
| Arrays `OCORRENCIAS`, `SECRETARIAS`, `USERS` no próprio HTML | Tudo carregado do backend via `fetch` |
| Login fake (qualquer senha entrava) | Login real com **JWT**, senha em `bcrypt` |
| Recarregar a página apagava tudo o que foi "criado" | Dados persistem no PostgreSQL |
| Sem controle de acesso | Cada perfil (admin/fiscal/secretaria) vê só o que pode |
| Nenhum erro ao desconectar | Toast, mensagens de erro, redirect ao login |

---

## 🚀 Como rodar

### 1. Suba o backend primeiro

```bash
cd backend
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # configure DATABASE_URL e JWT_SECRET_KEY
uvicorn app.main:app --reload
```

Backend fica em **http://localhost:8000**. Confira `/docs`.

### 2. Abra o painel

**Opção A — Mais simples:** dê duplo-clique no arquivo `index.html` (abre no navegador via `file://`). O backend já está configurado para aceitar essa origem.

**Opção B — Servidor local:** use qualquer servidor estático. Por exemplo, com Python:

```bash
cd painel-web
python3 -m http.server 5500
```

Acesse: `http://127.0.0.1:5500`

### 3. Login

Use as credenciais do usuário admin criado pelo SQL inicial:

- **E-mail:** `admin@imperatriz.ma.gov.br`
- **Senha:** a que você definiu ao inserir o usuário (ou rode o script abaixo para criar uma)

#### Criar um admin de teste rapidamente

Se o admin do SQL tem senha em hash fake, crie um novo direto via API. Abra um terminal Python:

```python
from passlib.hash import bcrypt  # ou use: pip install bcrypt
import psycopg2

conn = psycopg2.connect("postgresql://postgres:senha@localhost:5432/conecta_imperatriz")
cur = conn.cursor()
senha_hash = bcrypt.hash("admin123")
cur.execute("UPDATE usuarios SET senha_hash = %s WHERE email = 'admin@imperatriz.ma.gov.br'", (senha_hash,))
conn.commit()
print("Senha redefinida: admin123")
```

Ou mais simples — cadastre via Swagger (`http://localhost:8000/docs` → `/auth/cadastrar-cidadao`).

---

## ⚙️ Configurar URL da API

Por padrão, o painel chama `http://localhost:8000/api/v1`. Para mudar (ex.: API em produção):

1. Abra o navegador
2. F12 → Console
3. Digite: `localStorage.setItem('api_base', 'https://api.imperatriz.ma.gov.br/api/v1')`
4. Recarregue

---

## 🎨 Como o painel está organizado

```
painel-web/
├── index.html       # Tudo num arquivo: HTML + CSS + JS
│   ├─ Tela de login (centralizada, com brasão)
│   ├─ Sidebar com navegação
│   └─ 7 páginas: Dashboard, Ocorrências, Mapa, Secretarias, Sec-Detail, Usuários, Configurações
└── README.md        # Este arquivo
```

### Arquitetura do JavaScript (resumo)

- `api` → wrapper sobre `fetch` que adiciona o token JWT em todas as chamadas
- `store` → cache local dos dados carregados (secretarias, categorias, etc)
- `doLogin/doLogout/restaurarSessao` → controle de sessão
- `carregarOcorrencias/carregarUsuarios/...` → buscam dados da API e renderizam

### Tratamento de erros

- **Backend offline:** mensagem clara na tela de login
- **Token expirado (401):** logout automático + toast vermelho
- **Sem permissão (403):** toast com motivo
- **Erros de validação:** mostrados no toast com a mensagem do FastAPI

---

## 🔐 Permissões por perfil

| Perfil | O que vê e faz |
|--------|----------------|
| **admin** | Tudo: gerenciar usuários, ver todas ocorrências, dashboard global |
| **fiscal** | Ver e editar todas ocorrências (campo) |
| **secretaria** | Vê SOMENTE as ocorrências da sua secretaria. Não vê página de usuários. |

O painel respeita automaticamente: se um usuário "secretaria" tentar abrir a página de usuários, o backend retorna 403 e o painel mostra uma mensagem.

---

## 🐛 Problemas comuns

**"Não foi possível conectar à API"**
→ Backend não está rodando. Suba com `uvicorn app.main:app --reload`.

**Login não funciona**
→ Verifique se o usuário existe no banco (`SELECT email FROM usuarios;`) e se a senha está correta. Hashes precisam ser gerados via `bcrypt.hash()` real, não o exemplo do SQL.

**CORS error**
→ O backend já permite `file://` (origin `null`) e `localhost`. Se mudar a porta, ajuste `ALLOWED_ORIGINS` no `.env`.

**Tabela vazia ao logar**
→ Normal! O banco está vazio. Use o Swagger (`/docs`) para criar uma ocorrência de teste:
1. Cadastrar cidadão via `/auth/cadastrar-cidadao`
2. Login cidadão via `/auth/login-cidadao` → copiar o token
3. Criar ocorrência via `/ocorrencias` (clicar no cadeado e colar o token)
4. Voltar pro painel e ver na lista!

---

**Conecta Imperatriz · Painel v3 · Conectado à API REST**
