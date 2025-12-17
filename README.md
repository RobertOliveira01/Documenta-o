# 📚 Documentação da Automação OmniChat ↔ HubSpot (v2.2)

Este repositório contém o pipeline de automação (ETL) responsável pela ingestão de dados da API da OmniChat no MongoDB e pela integração e sincronização desses dados (Clientes, Mensagens e Pedidos) com o HubSpot CRM.

## ⚙️ Arquitetura do Sistema

O sistema é composto por cinco scripts Python independentes. A arquitetura utiliza um modelo de **consistência eventual** para garantir que dados preenchidos durante o atendimento (como o e-mail informado tardiamente) sejam capturados corretamente.

| Script | Função Principal | Frequência Sugerida |
| :--- | :--- | :--- |
| **1_consultar_api_omni.py** | Ingestão de Mensagens e **Tentativa de Enriquecimento** | Cada 60 segundos |
| **2_mensagens_cliente_empresa_hubspot.py** | Criação de Contatos, Empresas e Histórico de Chat **(Processamento)** | Cada 60 segundos |
| **3_pedidos_negocios_hubspot.py** | Criação de Negócios (Deals), **Itens de Linha** e Repescagem | Cada 2 minutos |
| **4_rotina_atualizar_cliente_banco.py** | **Saneamento Cadastral (Correção de Timing)** | Cada 1 hora |
| **5_rotina_atualizar_pedidos_banco.py** | Sincronização de Valores e Itens de Pedido (Diffing) | Cada 6 horas |

---

## 🔁 Fluxo Lógico (Pipeline)

### 1. Captura de mensagens e informações do Cliente (Fonte)
**Script:** `1_consultar_api_omni.py`

* **Objetivo:** Monitorar novas mensagens em tempo real e garantir o histórico.
* **Comportamento das informações do Cliente:**
    1.  Ao receber uma mensagem, o script verifica se o cliente já existe no banco.
    2.  Se não existir, ele consulta a API de Clientes e atualiza as informações.
    3.  Se o cliente iniciou a conversa agora e ainda não informou o e-mail ao atendente, este script salvará as mensagens e (apenas nome/telefone), no script de rotina os dados do cliente serão consultados novamente para atualização.

### 2. Atualizando informações do Cliente (A "Rede de Segurança")
**Script:** `4_rotina_atualizar_cliente_banco.py`

* **Objetivo:** Este script roda periodicamente buscando clientes ativos que possam ter sido atualizados na OmniChat *após* a primeira mensagem.
* **Lógica:**
    1.  Varre clientes ativos nos últimos 30 dias no MongoDB.
    2.  Consulta novamente a API `/customers` da OmniChat.
    3.  Se encontrar um e-mail/CNPJ que antes não existia, atualiza o MongoDB.
    4.  **Impacto:** Isso "destrava" os scripts 2 e 3 para processarem este cliente na próxima execução.

### 3. Processamento e Vendas (HubSpot)
Esta fase depende estritamente de um **E-mail**.

**Script:** `2_mensagens_cliente_empresa_hubspot.py`
* **Objetivo:** Criar a "Identidade" do cliente e da empresa no CRM.
* **Ação:** Cria Contato ➔ **Cria Empresa (com CNPJ)** ➔ Cria Nota (Histórico do Chat).
* **Saída:** Salva o `contact_id` e `hubspot_company_id` no documento do cliente no MongoDB.

**Script:** `3_pedidos_negocios_hubspot.py`
* **Objetivo:** Registrar a venda (Deal) e detalhar os produtos (Line Items).
* **Dependência:** Cliente precisa ter **E-mail** e já estar integrado (`contact_id`).
* **Lógica de Segurança:**
    1.  **Salvar Primeiro:** O pedido é salvo no MongoDB imediatamente.
    2.  **Validar:** Se o cliente não tiver e-mail, o processamento para por aqui (status pendente).
    3.  **Integrar:**
        * Cria o Negócio (`deals`).
        * Cria cada produto como **Item de Linha (`line_items`)** e associa ao Negócio.
        * Trata descontos como um item de linha negativo (preço < 0).
    4.  **Snapshot:** Salva no MongoDB uma lista com IDs do HubSpot e o estado atual dos itens (`synced_data`) para comparação futura.
    5.  **Repescagem:** Ao final de cada ciclo, tenta reprocessar pedidos pendentes dos últimos 30 dias (geralmente liberados pelo **Script 4**).

### 4. Manutenção de Pedidos (Sincronização Inteligente)
**Script:** `5_rotina_atualizar_pedidos_banco.py`
* **Objetivo:** Manter o conteúdo do pedido (produtos e valores) sempre fiel à OmniChat, caso haja alterações após a venda.
* **Alvo da Atualização:** **Negócios (Deals)** e seus **Itens de Linha (Line Items)** no HubSpot.
* **Lógica (Diffing):**
    1.  Busca pedidos **já integrados** no MongoDB.
    2.  Consulta dados atuais na API da OmniChat.
    3.  Compara item a item (Snapshot Banco vs API) e reflete no HubSpot:
        * **Criar:** Gera um novo **Item de Linha** no HubSpot e o associa ao Negócio existente.
        * **Atualizar:** Altera preço ou quantidade no **Item de Linha** específico.
        * **Deletar:** Exclui o **Item de Linha** do HubSpot caso ele não exista mais na OmniChat.
    4.  **Atualização do Negócio:** Após alinhar os itens, recalcula o valor total e atualiza a propriedade `amount` no objeto **Negócio (Deal)**.

---

## 📊 Mapeamento de Dados (De-Para)

### 🟢 Script 2: Criação de Entidades

#### 1. Contato (`contacts`)
Chave de unificação: **E-mail**.

| Campo HubSpot | Fonte OmniChat | Regra / Transformação |
| :--- | :--- | :--- |
| `email` | `email` | Obrigatório. Se ausente, aguarda atualização pelo **Script 4**. |
| `firstname` | `name` + `lastName` | Concatena nome e sobrenome. |
| `phone` | `phoneNumber` | Formata com DDI (ex: `+55...`). |
| `omnichat_id` | `objectId` (Cliente) | ID original para rastreabilidade. |

#### 2. Empresa (`companies`)
Criada e vinculada automaticamente ao contato.

| Campo HubSpot | Fonte OmniChat | Regra / Transformação |
| :--- | :--- | :--- |
| `name` | `businessName` | Se vazio, usa o Nome do Cliente. |
| `phone` | `phoneNumber` | Mesmo do contato. |
| `domain` | Domínio do `email` | Extrai `@empresa.com`. Ignora domínios públicos. |
| **`cnpj`** | **`businessTaxId`** | **Internal Name: `cnpj`. Envia apenas se existir valor.** |

---

### 🔵 Script 3 e 5: Vendas e Produtos

#### 1. Negócio (`deals`)
Representa a transação financeira no Pipeline.

| Campo HubSpot | Fonte OmniChat | Regra / Transformação |
| :--- | :--- | :--- |
| `dealname` | `objectId` | Formato: `"Pedido OmniChat #123456"` |
| `amount` | *Cálculo* | `(Soma Itens + Frete) - Desconto` |
| `closedate` | `createdAt` | Data original do pedido. |
| `dealstage` | *Fixo* | `"appointmentscheduled"` (Na criação). |
| `pipeline` | *Fixo* | `"default"`. |

#### 2. Itens de Linha (`line_items`) [NOVO]
Substitui a antiga "Nota de Texto". Cria registros nativos de produto no CRM.

| Campo HubSpot | Fonte OmniChat | Regra / Transformação |
| :--- | :--- | :--- |
| `name` | `items[].name` | Nome do produto ou "Produto sem nome". |
| `quantity` | `items[].quantity` | Quantidade vendida. |
| `price` | `items[].price` | Preço unitário. |
| **(Desconto)** | `discount` | Cria item com **valor negativo** se houver desconto. |

---

## 🛡️ Mecanismos de Segurança e Integridade

1.  **Padrão de Repescagem (Retry Pattern - Script 3):**
    * Pedidos que chegam antes do e-mail do cliente estar disponível não são perdidos. Eles ficam salvos no banco com status `PENDENTE`.
    * Assim que o **Script 4** identificar que o e-mail foi preenchido, a repescagem do Script 3 processa o pedido.

2.  **Snapshot de Sincronização (Script 3 e 5):**
    * O sistema salva no MongoDB (`hubspot_line_items`) o "estado conhecido" dos itens no HubSpot. Isso permite detectar exatamente o que mudou (Diffing) sem precisar consultar a API do HubSpot a cada ciclo, economizando consultas a API.

3.  **Idempotência e Prevenção de Duplicidade:**
    * Antes de criar qualquer Negócio, o script verifica se aquele Pedido já possui um `hubspot_deal_id` salvo no banco. Isso impede a criação de deals duplicados em caso de falhas de rede.

<!-- 4.  **Escopo da Automação:**
    * **Etapas do Funil:** A automação **NÃO** altera a etapa do negócio (`dealstage`) após a criação. O movimento dos cards no funil (ex: de "Agendado" para "Fechado") deve ser feito manualmente pela equipe de vendas ou por automações internas (Workflows) do próprio HubSpot. -->
