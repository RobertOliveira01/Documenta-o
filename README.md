# 📚 Documentação da Automação OmniChat ↔ HubSpot (v3.4)

Este repositório contém o pipeline de automação (ETL) para integração entre OmniChat, MongoDB e HubSpot CRM.

## 🚀 Destaque da Versão 3.4
O sistema prioriza a inteligência de vendas e mantém o cadastro de pessoas físicas atualizado:
1.  **Prioridade ao Negócio Manual:** Se um vendedor criar um Negócio manualmente no HubSpot (e vincular ao contato), a automação detecta que este é o negócio **mais recente** e passa a jogar os pedidos e conversas nele.
2.  **Auto-Correção (Migração):** Se um pedido já estava vinculado a um negócio automático, mas o sistema detecta que um Negócio Manual novo foi aberto depois, o **Script 5** move o pedido para esse novo negócio automaticamente.

---

## ⚙️ Arquitetura e Responsabilidades

| Container / Script | Função Principal | Quem ele Atualiza? | Quando? |
| :--- | :--- | :--- | :--- |
| **Script 1** (Ingestão) | Backup de Mensagens | MongoDB (Raw Data) | A cada 60s |
| **Script 2** (Chat & CRM) | **Gestão de Sessão** e **Abertura de Negócio** | Notas de Chat, Contatos e **Criação do Negócio Inicial (#0000)** | A cada 60s |
| **Script 3** (Pedidos) | **Conversão de Venda** | **Atualiza o Negócio Aberto** (Nome, Valor, Itens) | A cada 2 min |
| **Script 4** (Clientes) | **Saneamento Cadastral (PF)** | **Contatos** (CPF, Telefones, E-mail, Endereço) | A cada 60s |
| **Script 5** (Monitor) | Monitoramento e **Migração de Negócio** | Negócios (Valor/Frete) e Notas (Anotações) | A cada 60s |

---

## 🔄 Fluxo Detalhado: O que está rodando?

### 1. O Ciclo do Contato e Conversa
**Responsável:** `script_2_mensagens_cliente_empresa_hubspot.py`

* **QUANDO:** O cliente envia mensagens novas e não integradas.
* **AÇÃO 1 (Histórico):** Agrupa mensagens em sessões e gera a Nota de Conversa no HubSpot.
* **AÇÃO 2 (Regra de Negócio Aberto):**
    * O script pergunta ao HubSpot: *"Existe algum negócio para este contato que **NÃO** esteja fechado (`hs_is_closed=false`)?"*
    * **Se encontrar (seja automático ou criado manualmente):** Anexa a conversa nesse negócio existente.
    * **Se NÃO encontrar (só existem ganhos/perdidos):** (`hs_is_closed=true`) Cria um novo negócio "Rascunho" (`#0000`).

### 2. O Ciclo da Venda (Conversão)
**Responsável:** `script_3_pedidos_negocios_hubspot.py`

* **QUANDO:** Um pedido é gerado na OmniChat.
* **AÇÃO:**
    1.  Busca o negócio **EM ABERTO** mais recente no HubSpot.
    2.  **CENÁRIO A (Prioridade Manual):** Se o vendedor criou um negócio manual recentemente, o script identifica ele e usa esse ID.
    3.  **CENÁRIO B (Fluxo Automático):** Se não houver manual, ele usa o negócio `#0000` criado pelo chat.
    4.  **Execução:** Atualiza nome, valor, itens e cria a nota do pedido.

### 3. Saneamento e Enriquecimento Cadastral
**Responsável:** `script_4_rotina_atualizar_cliente_banco.py`

* **QUANDO:** A cada 60 segundos (monitora clientes ativos nos últimos 30 dias).
* **AÇÃO:** Compara MongoDB vs. API OmniChat.
* **ATUALIZAÇÕES ATIVAS:** * Nome e Sobrenome
    * E-mail
    * Telefones (Formatados BR)
    * CPF
    * Data de Nascimento
    * Gênero
* **OBS:** A atualização de Empresas (CNPJ/Razão Social) e Vendedores (Owners) está **desativada** no código atual.

### 4. O Ciclo de Monitoramento (Pós-Venda & Migração)
**Responsável:** `script_5_rotina_atualizar_pedidos_banco.py`

* **QUANDO:** Constantemente monitorando pedidos dos últimos 30 dias.
* **FUNCIONALIDADE 1 (Auto-Correção/Migração):**
    * Verifica: *"O negócio que este pedido está vinculado no Banco é realmente o mais recente aberto no HubSpot?"*
    * Se o script descobrir que **existe um negócio manual mais novo** aberto, ele:
        1.  **Migra** o vínculo do pedido para o novo negócio.
        2.  Recria os itens de linha no novo negócio.
* **FUNCIONALIDADE 2 (Sincronia):**
    * Se valor, frete ou anotações mudarem na OmniChat, reflete no HubSpot imediatamente.

---

## 📊 Mapeamento Cadastral (Script 4)

Abaixo, a relação de campos que o Script 4 efetivamente atualiza nos **Contatos**.

### 👤 Contatos (Contacts)

| Campo HubSpot (Internal Name) | Fonte OmniChat | Transformação |
| :--- | :--- | :--- |
| `email` | `email` | - |
| `firstname` | `name` + `lastName` | Concatena Nome + Sobrenome |
| `phone` | `phoneCountryCode` + `Area` + `Number` | Formata: `+5511999999999` |
| **`cpf`** | `taxDocumentNumber` | Cópia direta |
| **`gender`** | `gender` | Traduz: `male`→`Masculino`, `female`→`Feminino` |
| **`date_of_birth`** | `birthDate` | Formata: `YYYY-MM-DD` |

> *Nota: Os campos de Empresa (CNPJ, IE, Endereço da Empresa) existem no código mas estão comentados/inativos.*

---

## 📊 Estrutura de Vendas (Deals & Notes)

### Negócio (Deal)
| Campo | Quem Preenche Inicialmente? | Quem Atualiza depois? |
| :--- | :--- | :--- |
| **Nome** | **Script 2** (`#0000`) | **Script 3** (`#{ID}`) |
| **Valor** | **Script 2** (`0`) | **Script 3** (Valor Real) |
| **Itens** | Ninguém | **Script 3** (Cria) -> **Script 5** (Sincroniza/Migra) |
| **Frete** | Ninguém | **Script 3** (Define) -> **Script 5** (Corrige) |

### Notas (Notes)
1.  **Nota de Conversa (Script 2):** Histórico de Chat + Link Dinâmico.
2.  **Nota de Pedido (Script 3 e 5):** Valor do Frete + Anotações do Pedido.

---

## ⚠️ Regra de Status (Aberto vs Fechado)

O sistema utiliza a propriedade nativa `hs_is_closed` para decidir se cria um novo negócio ou atualiza o existente.

* **Negócio em Andamento (`hs_is_closed = false`):** O sistema **ATUALIZA** este negócio.
* **Negócio Ganho (`hs_is_closed = true`):** O sistema **CRIA UM NOVO**.
* **Negócio Perdido (`hs_is_closed = true`):** O sistema **CRIA UM NOVO**.

> **Importante:** Manter os funis organizados. Se deixar um negócio antigo "esquecido" em uma etapa aberta (`hs_is_closed = false`), o sistema continuará atualizando as informações de pedido dentro dele. Para iniciar um novo ciclo de venda, o negócio anterior deve ter o campo "O negócio está fechado?" (`hs_is_closed = true`).
