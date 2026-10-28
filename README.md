# 📐 Modelagem do Sistema — AL Cakes

Modelagem do sistema **como ele está implementado hoje** (extraída do código em `backend/src` e dos scripts de banco). Os diagramas usam Mermaid, que o GitHub renderiza automaticamente.

## 1. Diagrama de Classes (domínio)

Baseado nas interfaces de `backend/src/tipos/`.

```mermaid
classDiagram
    class Usuario {
        +string id
        +string nome
        +string email
        +string telefone
        +Role role
        +Date criado_em
    }
    class Cliente {
        +string id
        +string nome
        +string telefone
        +Date criado_em
    }
    class Bolo {
        +string id
        +string nome
        +string descricao
        +number preco
        +string imagem_url
        +number estoque
        +boolean sob_encomenda
        +boolean ativo
    }
    class CarrinhoItem {
        +string id
        +string usuario_id
        +string bolo_id
        +number quantidade
        +Date criado_em
        +Date atualizado_em
        +subtotal() number
    }
    class Pedido {
        +string id
        +number numero_pedido
        +string clienteId
        +number total
        +PedidoStatus status
        +Date data
    }
    class ItemPedido {
        +string id
        +string pedidoId
        +string boloId
        +number quantidade
        +number precoUnitario
    }
    class Role {
        <<enumeration>>
        cliente
        admin
    }
    class PedidoStatus {
        <<enumeration>>
        pendente
        confirmado
        em_preparo
        pronto_retirada
        saiu_entrega
        entregue
        cancelado
    }

    Usuario "1" --> "*" CarrinhoItem : possui
    CarrinhoItem "*" --> "1" Bolo : referencia
    Cliente "1" --> "*" Pedido : realiza
    Pedido "1" *-- "1..*" ItemPedido : composto por
    ItemPedido "*" --> "1" Bolo : refere-se a
    Usuario --> Role
    Pedido --> PedidoStatus
```

> **Observação de modelagem:** `Usuario` (conta com login) e `Cliente` (identificado por telefone, criado/recuperado no momento do pedido) são entidades separadas no código atual. Essa duplicidade é registrada como limitação no documento de melhorias.

## 2. Diagrama de Classes (camadas da aplicação)

```mermaid
classDiagram
    direction LR
    class Rotas {
        rotasAuth
        rotasBolos
        rotasPedidos
        rotasCarrinho
        rotasUsuarios
        rotasWhatsApp
    }
    class Middlewares {
        autenticacao()
        autorizacao(roles)
    }
    class Controladores {
        controladorAuth
        controladorBolo
        controladorPedido
        controladorCarrinho
        controladorUsuario
        controladorWhatsApp
    }
    class Servicos {
        servicoAuth
        servicoBolo
        servicoPedido
        servicoCarrinho
        servicoCliente
        servicoUsuario
        servicoWhatsApp
    }
    class Supabase {
        <<PostgreSQL>>
    }
    Rotas --> Middlewares : usa
    Rotas --> Controladores : delega
    Controladores --> Servicos : chama
    Servicos --> Supabase : consulta/grava
```

## 3. Modelo Entidade-Relacionamento (banco de dados)

Baseado em `backend/scripts/setup-database.js` e nas migrações em `backend/scripts/migrations/`.

```mermaid
erDiagram
    USUARIOS ||--o{ CARRINHO_ITENS : possui
    BOLOS ||--o{ CARRINHO_ITENS : "aparece em"
    CLIENTES ||--o{ PEDIDOS : realiza
    PEDIDOS ||--|{ PEDIDO_ITENS : contem
    BOLOS ||--o{ PEDIDO_ITENS : "aparece em"

    USUARIOS {
        uuid id PK
        text nome
        text email UK
        text senha_hash
        text telefone
        text role "cliente | admin"
        timestamptz criado_em
    }
    CLIENTES {
        uuid id PK
        text nome
        text telefone UK
        timestamptz criado_em
    }
    BOLOS {
        uuid id PK
        text nome
        text descricao
        numeric preco
        text imagem_url
        int estoque
        boolean sob_encomenda
        boolean ativo
        timestamptz criado_em
    }
    PEDIDOS {
        uuid id PK
        uuid cliente_id FK
        int numero_pedido
        numeric total
        text status
        timestamptz criado_em
    }
    PEDIDO_ITENS {
        uuid id PK
        uuid pedido_id FK
        uuid bolo_id FK
        int quantidade
        numeric preco_unitario
    }
    CARRINHO_ITENS {
        uuid id PK
        uuid usuario_id FK
        uuid bolo_id FK
        int quantidade
        timestamptz atualizado_em
    }
```

## 4. Casos de Uso

```mermaid
flowchart LR
    V([Visitante])
    C([Cliente logado])
    A([Administrador])

    subgraph Sistema AL Cakes
        UC1(Consultar cardápio)
        UC2(Ver detalhes do bolo)
        UC3(Registrar conta)
        UC4(Fazer login)
        UC5(Gerenciar carrinho)
        UC6(Finalizar pedido)
        UC7(Acompanhar pedidos)
        UC8(Enviar pedido via WhatsApp)
        UC9(Gerenciar bolos - CRUD)
        UC10(Atualizar status do pedido)
        UC11(Gerenciar usuários)
        UC12(Configurar mensagens WhatsApp)
    end

    V --> UC1 & UC2 & UC3 & UC4
    C --> UC1 & UC2 & UC5 & UC6 & UC7 & UC8
    A --> UC9 & UC10 & UC11 & UC12 & UC7
```

## 5. Diagrama de Sequência — Finalizar pedido

Fluxo implementado em `servicoPedido.criar`.

```mermaid
sequenceDiagram
    actor Cliente
    participant API as controladorPedido
    participant SP as servicoPedido
    participant SC as servicoCarrinho
    participant SCl as servicoCliente
    participant SB as servicoBolo
    participant DB as Supabase

    Cliente->>API: POST /pedidos/criar (JWT, nome, telefone)
    API->>SP: criar(usuarioId, dados)
    SP->>SC: listar(usuarioId)
    SC-->>SP: itens do carrinho
    SP->>SCl: buscarOuCriar(nome, telefone)
    SCl->>DB: select/insert clientes
    loop cada item
        SP->>SB: buscarPorId(boloId)
        SB-->>SP: bolo (valida estoque se não for sob encomenda)
    end
    SP->>DB: insert pedidos (status = pendente, numero_pedido)
    SP->>DB: insert pedido_itens
    SP->>SB: reduzirEstoque(boloId, qtd)
    SP->>SC: limpar(usuarioId)
    SP-->>API: pedido criado
    API-->>Cliente: 201 + pedido
```

## 6. Diagrama de Estados — Pedido

Estados definidos em `PedidoStatus` (`backend/src/tipos/Pedido.ts`).

```mermaid
stateDiagram-v2
    [*] --> pendente
    pendente --> confirmado
    confirmado --> em_preparo
    em_preparo --> pronto_retirada
    em_preparo --> saiu_entrega
    saiu_entrega --> entregue
    pronto_retirada --> entregue
    pendente --> cancelado
    confirmado --> cancelado
    entregue --> [*]
    cancelado --> [*]
```

> As transições acima representam o fluxo esperado do negócio. Hoje a API aceita qualquer troca de status (não há validação de transição); isso está listado como melhoria.

## 7. Arquitetura

```mermaid
flowchart TD
    F["Frontend<br/>HTML + CSS + JS (jQuery)"] -->|HTTP / REST + JWT| B["Backend<br/>Node.js + Express + TypeScript"]
    B -->|supabase-js| D[("Supabase<br/>PostgreSQL")]
    B -.->|links wa.me| W[WhatsApp]
    B --> S["Swagger UI<br/>/api-docs"]
```
