[README.md](https://github.com/user-attachments/files/26947148/README.md)
# 🔥 Firebase RBAC — Controle de Acesso por Cargo

Aplicação web demonstrando autenticação com Firebase Auth e controle de acesso baseado em cargos (**Role-Based Access Control**) usando Firebase Realtime Database e Security Rules.

---

## 📁 Estrutura do Banco de Dados

O banco é organizado como uma árvore JSON com três nós principais:

```
{
  "users": { ... },         ← perfis e cargos dos usuários
  "admin-data": { ... },    ← dados exclusivos de administradores
  "public-data": { ... }    ← dados acessíveis a todos os autenticados
}
```

### `/users/{uid}`

Armazena o perfil de cada usuário. O campo `role` é a chave do sistema de permissões — ele é consultado pelas Security Rules em tempo real a cada requisição ao banco.

```json
"users": {
  "uid_alice_123": {
    "email": "alice@example.com",
    "displayName": "Alice",
    "role": "admin",
    "createdAt": 1714000000000
  },
  "uid_bob_456": {
    "email": "bob@example.com",
    "displayName": "Bob",
    "role": "user",
    "createdAt": 1714001000000
  }
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `email` | string | Email do usuário (espelho do Firebase Auth) |
| `displayName` | string | Nome de exibição |
| `role` | string | Cargo: `"admin"` ou `"user"` |
| `createdAt` | number | Timestamp Unix em milissegundos |

### `/admin-data`

Nó restrito — somente usuários com `role === "admin"` têm acesso de leitura e escrita. Contém configurações do sistema e logs de auditoria.

```json
"admin-data": {
  "config": {
    "maintenanceMode": false,
    "maxUsersAllowed": 1000,
    "featureFlags": {
      "betaUI": true
    }
  },
  "logs": {
    "log_001": {
      "action": "user_deleted",
      "by": "uid_alice_123"
    }
  }
}
```

### `/public-data`

Nó de leitura aberta a qualquer usuário autenticado. Escrita restrita a admins.

```json
"public-data": {
  "announcements": {
    "ann_001": {
      "title": "Bem-vindo!",
      "body": "Sistema online e funcionando."
    }
  }
}
```

---

## 🔒 Security Rules

As regras são avaliadas no servidor Firebase antes de qualquer dado trafegar para o cliente.

```json
{
  "rules": {
    "users": {
      "$uid": {
        ".read":  "auth != null && (root.child('users').child(auth.uid).child('role').val() === 'admin' || auth.uid === $uid)",
        ".write": "auth != null && (root.child('users').child(auth.uid).child('role').val() === 'admin' || (auth.uid === $uid && !newData.child('role').exists()))",
        ".validate": "newData.hasChildren(['email', 'role', 'createdAt']) && (newData.child('role').val() === 'admin' || newData.child('role').val() === 'user')"
      }
    },
    "admin-data": {
      ".read":  "auth != null && root.child('users').child(auth.uid).child('role').val() === 'admin'",
      ".write": "auth != null && root.child('users').child(auth.uid).child('role').val() === 'admin'"
    },
    "public-data": {
      ".read":  "auth != null",
      ".write": "auth != null && root.child('users').child(auth.uid).child('role').val() === 'admin'"
    },
    "$other": {
      ".read":  false,
      ".write": false
    }
  }
}
```

### Proteções críticas implementadas

**Isolamento por UID** — a variável `$uid` nas Rules é comparada com `auth.uid` em tempo real. Um usuário comum só consegue ler e editar o próprio perfil em `/users/{uid}`.

**Bloqueio de escalonamento de privilégio** — a regra `!newData.child('role').exists()` impede que um usuário comum altere o próprio campo `role` para `"admin"`. Somente um admin já autenticado pode modificar esse campo.

**Nós negados por padrão** — o nó `$other` com `.read: false` e `.write: false` garante que qualquer caminho não mapeado nas Rules seja bloqueado automaticamente.

---

## 🔑 Fluxo de Autenticação

O cargo é persistido no banco imediatamente após a criação da conta:

```javascript
async function registerUser(email, password, displayName, role) {
  // 1. Cria o usuário no Firebase Auth
  const credential = await createUserWithEmailAndPassword(auth, email, password);

  // 2. Salva perfil + cargo no Realtime Database
  await set(ref(db, `users/${credential.user.uid}`), {
    email,
    displayName,
    role,           // "admin" | "user"
    createdAt: Date.now()
  });

  return credential.user;
}
```

---

## 🗂 Matriz de Permissões

| Caminho | Operação | Admin | User (próprio) | User (outros) | Anônimo |
|---|---|---|---|---|---|
| `/users/{uid}` | read | ✅ | ✅ | ❌ | ❌ |
| `/users/{uid}` | write | ✅ | ⚠️ parcial* | ❌ | ❌ |
| `/users/{uid}/role` | write | ✅ | ❌ | ❌ | ❌ |
| `/admin-data` | read | ✅ | ❌ | ❌ | ❌ |
| `/admin-data` | write | ✅ | ❌ | ❌ | ❌ |
| `/public-data` | read | ✅ | ✅ | ✅ | ❌ |
| `/public-data` | write | ✅ | ❌ | ❌ | ❌ |

*Usuário pode editar o próprio perfil, exceto o campo `role`.

---

## 🚀 Como Executar

O projeto é um arquivo HTML puro, sem dependências externas ou servidor necessário.

1. Baixe o arquivo `firebase-rbac-app.html`
2. Dê dois cliques para abrir no navegador
3. Na aba **Auth**, crie uma conta escolhendo o cargo desejado
4. Acesse a aba **Demo** para visualizar as diferenças de acesso por cargo

---

## 📦 Tecnologias

| Tecnologia | Uso |
|---|---|
| Firebase Auth | Autenticação de usuários |
| Firebase Realtime Database | Armazenamento de dados em tempo real |
| Firebase Security Rules | Controle de acesso no servidor |
| HTML / CSS / JavaScript | Interface visual (sem frameworks) |
