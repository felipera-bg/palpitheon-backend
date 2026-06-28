
# Palpitheon — Contexto do Projeto

## Visão Geral

<!-- O que é o projeto? Qual problema resolve? -->
O Palpitheon é um sistema que permite usuários cadastrem seus palpites para cada categoria do The Game Awards e sejam rankeados com base em seus acertos.

## Stack

### Backend
<!-- Ex: .NET 9, Entity Framework, PostgreSQL, JWT -->
.NET 10, Dapper, PostgreSQL, JWT

### Frontend
<!-- Ex: React, Next.js, TailwindCSS -->
A stack do frontend não foi definida ainda. Para caso o projeto seja:
- Web: Vue.js
- PWA: React
- Mobile: React Native

### Infraestrutura
<!-- Ex: Docker, Railway, Vercel, S3 -->
As imagens dos indicados serão salvas no Firebase Storage. Quanto ao resto, não foi definido.

## Arquitetura da API

<!-- Descreva o padrão arquitetural (ex: Clean Architecture, minimal APIs, REST),
     autenticação, versionamento, etc. Cole ou referencie o diagrama se tiver. -->
```
src/
├── Palpitheon. Core 
│   ├── Entities/ 
│   ├── Interfaces/
|		|		├── Data
|		|		├── Services 
|		|		└── Notifications 
|		|		
│   └── Services/
│
├── Palpitheon. Infrastructure
│   ├── Data/
│   └── Repositories/
│
└── Palpitheon. API
   ├── Controllers/
   ├── Hubs/
   └── Program.cs
```


## Diagrama Entidade-Relacionamento

<!-- Cole o DER em texto, Mermaid, ou descreva as entidades principais e seus relacionamentos -->
```dbml
Table usuarios {
  id integer [primary key, increment]
  username varchar(50) [unique, not null]
  email varchar(255) [unique, not null]
  password_hash varchar(255) [not null]
  role usuarios_roles [default: 'user', not null]
  created_at timestamp [default: `now()`]
}

Enum usuarios_roles {
  user
  admin
}

Table categorias {
  id integer [primary key, increment]
  nome varchar(100) [not null]
  descricao text
  indicado_vencedor_id integer [null]
  status categorias_status [default: 'votacao_aberta', not null]
  created_at timestamp [default: `now()`]
}

Enum categorias_status {
  votacao_aberta
  votacao_fechada
}

Table indicados {
  id integer [primary key, increment]
  nome varchar(150) [not null]
  descricao text
  imagem_url varchar(500)
}

Table categoria_indicados {
  id integer [primary key, increment]
  categoria_id integer [not null]
  indicado_id integer [not null]

  Indexes {
    (categoria_id, indicado_id) [unique] // Garante que um indicado não seja duplicado na mesma categoria
  }
}

Table palpites {
  id integer [primary key, increment]
  usuario_id integer [not null]
  categoria_id integer [not null]
  indicado_id integer [not null]
  created_at timestamp [default: `now()`]
  updated_at timestamp [default: `now()`]

  Indexes {
    (usuario_id, categoria_id) [unique] // Regra de negócio: 1 palpite por categoria por usuário
  }
}

// --- RELAÇÕES (Chaves Estrangeiras) ---

// Relacionamentos da Tabela Pivô (Muitos para Muitos)
Ref: categoria_indicados.categoria_id > categorias.id [delete: cascade]
Ref: categoria_indicados.indicado_id > indicados.id [delete: cascade]

// Relacionamentos de Palpites
Ref: palpites.usuario_id > usuarios.id [delete: cascade]
// Vincula o palpite à tabela pivô usando uma chave composta (Garante que o usuário só vote em um indicado que REALMENTE pertence àquela categoria)
Ref: palpites.(categoria_id, indicado_id) > categoria_indicados.(categoria_id, indicado_id)

// Relacionamento do Vencedor da Categoria
Ref: categorias.indicado_vencedor_id > indicados.id

Table refresh_tokens {
  id integer [primary key, increment]
  usuario_id integer [not null]
  token varchar(500) [unique, not null]
  expires_at timestamp [not null]
  created_at timestamp [default: `now()`]
}

Ref: refresh_tokens.usuario_id > usuarios.id [delete: cascade]
```

## User Flows Principais

<!-- Descreva os fluxos mais importantes do ponto de vista do usuário.
     Ex:
     1. Cadastro → login → dashboard
     2. Criação de palpite → confirmação → resultado -->

### 1. Fluxo de Autenticação
* **Tela de Login (Entrada)**
    * `SE` o usuário já possui conta ➔ Preenche os dados ➔ Clica em "Entrar" ➔ Vai para a tela **Palpites do Usuário** (no modo Meu Palpite).
    * `SE NOT` possui conta ➔ Clica em "Se Cadastrar" ➔ Vai para **Tela de Cadastro**
* **Tela de Cadastro**
    * Preenche os dados ➔ Clica em "Concluir" ➔ Retorna para a **Tela de Login**

---

### 2. Fluxo do Usuário Comum

#### Aba 1: Lista de Palpites
*(Nota técnica: Esta tela é um componente reaproveitável que exibe a listagem de cards de categorias. Ela pode ser renderizada no modo **Meus Palpites** ou no modo **Palpites de Outro Usuário**)*

* **Modo A: Meus Palpites (Home)**
    * Exibe as categorias com base nas escolhas do usuário logado.
    * `SE` status é `votacao_aberta` e já foi votada ➔ O card exibe a foto e o nome do indicado escolhido.
    * `SE` status é `votacao_fechada` e `indicado_vencedor_id` está preenchido:
        * `SE` (Meu Palpite == Vencedor) ➔ Aplicar **Efeito Visual de Acerto** (ex: borda verde/selo de check).
        * `SE NOT` (Meu Palpite == Vencedor) ➔ Aplicar **Efeito Visual de Erro** (ex: card opaco/cinza).
    * O usuário clica em uma Categoria ➔ Vai para **Tela de Categoria Única**

* **Modo B: Palpites de Outro Usuário (Acessado via Ranking)**
    * Exibe as categorias com base nas escolhas do usuário que foi selecionado.
    * `SE` status é `votacao_aberta` ➔ O card oculta a escolha (Exibe: "Votação em andamento").
    * `SE` status é `votacao_fechada` e sem vencedor ➔ O card exibe o palpite dele, bloqueado para edição.
    * `SE` status é `votacao_fechada` e com vencedor:
        * Exibe o palpite dele + **Efeito Visual** indicando se *aquele usuário* acertou ou errou.
    * O usuário clica em uma Categoria ➔ Vai para **Tela de Categoria Única** (Apenas Leitura).

#### Aba 2: Ranking
* **Tela: Ranking Global** (Exibe a lista de posições e pontuações dos usuários)
    * Usuário clica em um perfil da lista ➔ Redireciona para a tela **Palpites do Usuário** (no Modo B: Palpites de Outro Usuário).

---

#### Tela: Categoria Única (Detalhes)
Acessada ao clicar em qualquer card de categoria. O comportamento muda conforme o status da votação no banco de dados:

* `SE` status é `votacao_aberta`:
    * Exibe lista de indicados ➔ Usuário seleciona um indicado ➔ Clica em "Confirmar Voto" ➔ Salva ou Atualiza o palpite ➔ Retorna para a tela **Lista de Palpites**.
* `SE` status é `votacao_fechada` e sem vencedor:
    * Bloqueia qualquer interação de votação.
    * Exibe na tela a mensagem informativa: **"Votação encerrada. Aguardando vencedor..."** ⏳
* `SE` status é `votacao_fechada` e com vencedor (`indicado_vencedor_id` preenchido):
    * Bloqueia qualquer interação de votação.
    * Exibe o palpite feito pelo usuário e destaca de forma nítida o indicado que foi o **Vencedor Oficial**.

---

### 3. Fluxo do Administrador
`SE` o usuário que realizou o login possuir a role de Admin, ele ganha acesso exclusivo a uma aba de gerenciamento:

* **Tela: Painel do Admin (Dashboard)**
    * **Ações em Lote:**
        * Fechar tudo: muda todas as categorias de `votacao_aberta` para `votacao_fechada`.
        * Reabrir tudo: muda todas as categorias de `votacao_fechada` para `votacao_aberta` e limpa todos os `indicado_vencedor_id`.
    * **Seção: Gerenciar Categorias** (Operações CRUD)
        * Criar, Editar e Excluir categorias do sistema.
        * Definir ou redefinir o indicado vencedor de uma categoria (somente se `votacao_fechada`).
    * **Seção: Gerenciar Indicados** (Operações CRUD)
        * Criar, Editar e Excluir indicados.
        * Vincular indicados às suas respectivas categorias (tabela pivô).
    * **Seção: Gerenciar Usuários**
        * Promover usuário comum para `admin`.
        * Rebaixar admin para `user`.


## Regras de Negócio (Business Rules)

### 1. Regras de Palpites e Votação
* **Apenas Um Voto:** Cada usuário só pode ter 1 palpite ativo por categoria. Um novo palpite na mesma categoria deve sobrescrever (atualizar) o anterior.
* **Trava de Tempo:** A criação ou edição de palpites só é permitida enquanto o status da categoria for `votacao_aberta`.
* **Bloqueio Pós-Fechamento:** Se o status da categoria for `votacao_fechada`, qualquer tentativa de enviar um palpite deve retornar 403 Forbidden.

### 2. Regras de Visibilidade e Privacidade (Anti-Espionagem)
* **Voto Secreto:** Enquanto uma categoria estiver com status `votacao_aberta`, nenhum usuário pode ver o palpite de outro. A API deve ocultar esse dado.
* **Revelação de Votos:** Os palpites de outros usuários só ficam visíveis quando o status da categoria mudar para `votacao_fechada`.

### 3. Regras de Pontuação e Ranking
* **Critério:** A pontuação de cada usuário é a contagem simples de acertos (palpites onde `indicado_id == indicado_vencedor_id` da categoria).
* **Cálculo sob demanda:** A pontuação **não é armazenada no banco**. A API calcula os resultados na query. Cache será implementado futuramente para mitigar problemas de performance.
* **Gatilho de Atualização:** O ranking deve ser recalculado assim que o Administrador definir ou redefinir o vencedor de uma categoria.

### 4. Regras de Permissão (Roles)
* **Primeiro usuário cadastrado** recebe automaticamente a role `admin`. Todos os demais recebem `user`.
* **Usuário Comum (`role: 'user'`):** Pode visualizar a lista de palpites (própria e de terceiros com restrições), visualizar o ranking global e votar em categorias abertas. Não tem acesso a rotas de modificação do sistema.
* **Administrador (`role: 'admin'`):** Tem controle total sobre o ciclo de vida das categorias e indicados (CRUD). Pode promover usuários a admin e rebaixar admins a usuário.

## Tempo Real e Notificações (SignalR)

### Eventos em Tempo Real (WebSocket)
O servidor emite eventos via SignalR para forçar atualização no cliente:
* **Vencedor definido:** quando o admin define o vencedor de uma categoria → clientes atualizam o feed de categorias.
* **Votações encerradas:** quando o admin fecha as votações → clientes visualizando o feed de outro usuário atualizam para revelar os palpites.

### Notificações
Notificações push/in-app disparadas pelos seguintes eventos:
* Votação encerrada em lote (`votacao_aberta` → `votacao_fechada`).
* Vencedor de uma categoria definido (`indicado_vencedor_id` atualizado).
* Evento encerrado (todas as categorias com `indicado_vencedor_id` preenchido).

---

## 5. Regras do Painel Administrativo (Ações do Admin)
* **Mudança de status sempre em lote:** O admin não pode alterar o status de categorias individualmente. As ações disponíveis são:
  * **Fechar tudo:** todas as categorias passam de `votacao_aberta` para `votacao_fechada`.
  * **Reabrir tudo:** todas as categorias passam de `votacao_fechada` para `votacao_aberta` e o `indicado_vencedor_id` de todas é removido (set null).
* **Definição de vencedor:** O admin pode definir ou redefinir o `indicado_vencedor_id` de uma categoria individualmente, mas somente se ela estiver com status `votacao_fechada`.

---

## Requisitos

### Funcionais

#### Autenticação
- RF01 — O sistema deve permitir o cadastro de usuários com username, email e senha. O primeiro usuário cadastrado recebe automaticamente a role `admin`; os demais recebem `user`.
- RF02 — O sistema deve autenticar usuários via email e senha, retornando um access token (JWT) e um refresh token.
- RF03 — O sistema deve disponibilizar um endpoint de refresh para emitir um novo access token a partir de um refresh token válido.
- RF04 — O sistema deve proteger rotas com base no access token JWT.
- RF05 — O sistema deve diferenciar permissões entre os roles `user` e `admin`.

#### Categorias
- RF06 — O sistema deve listar todas as categorias com seus respectivos status, indicado vencedor (quando definido) e o palpite do usuário autenticado.
- RF07 — O sistema deve exibir os detalhes de uma categoria, incluindo seus indicados e o vencedor (quando definido).
- RF08 — O admin deve poder criar, editar e excluir categorias.
- RF09 — O admin deve poder fechar a votação de todas as categorias simultaneamente (`votacao_aberta` → `votacao_fechada`).
- RF10 — O admin deve poder reabrir a votação de todas as categorias simultaneamente (`votacao_fechada` → `votacao_aberta`), removendo o `indicado_vencedor_id` de todas.
- RF11 — O admin deve poder definir ou redefinir o indicado vencedor de uma categoria individualmente, desde que ela esteja com status `votacao_fechada`.

#### Indicados
- RF12 — O admin deve poder criar, editar e excluir indicados.
- RF13 — O admin deve poder vincular e desvincular indicados de categorias.

#### Usuários
- RF14 — O admin deve poder promover um usuário comum para `admin` e rebaixar um admin para `user`.

#### Palpites
- RF15 — O usuário deve poder registrar ou atualizar seu palpite em uma categoria com status `votacao_aberta` (upsert).
- RF16 — O sistema deve bloquear criação ou edição de palpites em categorias com status `votacao_fechada`, retornando 403.
- RF17 — O sistema deve impedir que o usuário vote em um indicado que não pertença à categoria.
- RF18 — O usuário deve poder visualizar seus próprios palpites em todas as categorias.
- RF19 — O usuário deve poder visualizar os palpites de outro usuário, ocultando os de categorias com status `votacao_aberta`.

#### Ranking
- RF20 — O sistema deve calcular e retornar o ranking global dos usuários, ordenado pela contagem de acertos (palpites onde `indicado_id == indicado_vencedor_id` da categoria).
- RF21 — O cálculo do ranking deve ser feito sob demanda, sem persistência de pontuação no banco.

#### Tempo Real e Notificações
- RF22 — O sistema deve emitir um evento SignalR quando o admin definir o vencedor de uma categoria, para que os clientes atualizem o feed.
- RF23 — O sistema deve emitir um evento SignalR quando as votações forem encerradas em lote, para que clientes visualizando o feed de outro usuário revelem os palpites.
- RF24 — O sistema deve enviar notificação quando as votações forem encerradas em lote.
- RF25 — O sistema deve enviar notificação quando o vencedor de uma categoria for definido.
- RF26 — O sistema deve enviar notificação quando o evento encerrar (todas as categorias com `indicado_vencedor_id` preenchido).

### Não-Funcionais

#### Tecnologia
- RNF01 — O backend deve ser desenvolvido em .NET 10.
- RNF02 — O acesso a dados deve ser feito via Dapper.
- RNF03 — O banco de dados deve ser PostgreSQL, com colunas em snake_case.
- RNF04 — A autenticação deve utilizar JWT com access token de curta duração e refresh token armazenado no banco.
- RNF05 — As imagens dos indicados devem ser armazenadas no Firebase Storage, com a URL persistida no banco.
- RNF06 — A comunicação em tempo real deve ser implementada com SignalR.

#### Segurança
- RNF07 — Senhas devem ser armazenadas como hash.
- RNF08 — Rotas administrativas devem ser acessíveis apenas para usuários com role `admin`.

#### Performance
- RNF09 — O cálculo do ranking deve ser feito sob demanda. Cache deve ser implementado futuramente para mitigar impacto de performance.

#### Arquitetura
- RNF10 — O projeto deve seguir separação em camadas: `Core`, `Infrastructure` e `API`.
