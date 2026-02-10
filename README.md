# 📚 Documentação da Automação OmniChat ↔ HubSpot (v4.0 - Fluxo Manual)

Este repositório contém o pipeline de automação (ETL) para integração entre **OmniChat**, **MongoDB** e **HubSpot CRM**.

## 🚀 Destaque da Versão 4.0: "Negócio 100% Manual"
Nesta versão, a lógica de criação de negócios foi alterada para dar controle total ao time comercial e evitar poluição no CRM.

1.  **Criação Bloqueada:** A automação **NUNCA** cria um Negócio (Deal) automaticamente.
2.  **Responsabilidade Humana:** O vendedor deve criar o Negócio manualmente no HubSpot quando julgar necessário.
3.  **Preenchimento Automático:** Assim que o negócio manual for criado, a automação detecta, vincula o pedido e insere os produtos (Itens de Linha) automaticamente.

---

## ⚙️ Arquitetura e Responsabilidades

O sistema é dividido em 5 containers (workers) que operam de forma independente e assíncrona:

| Script | Nome Docker | Função Técnica | Papel no Fluxo |
| :--- | :--- | :--- | :--- |
| **Script 1** | `worker_1_ingestao` | **Input (Coleta)** | Baixa mensagens da OmniChat a cada 60s e salva no MongoDB. Garante que nenhuma conversa seja perdida. |
| **Script 2** | `worker_2_hubspot_crm` | **CRM Core** | Lê o Banco. Se o cliente tiver e-mail, cria/atualiza **Contato** e **Empresa** no HubSpot e salva o histórico da conversa na **Nota**. *Não cria Negócios.* |
| **Script 3** | `worker_3_hubspot_deals` | **Pedidos** | Lê Pedidos novos. Verifica se existe um **Negócio Manual** aberto no HubSpot. <br>🟢 **Se sim:** insere produtos e valores. <br>🔴 **Se não:** deixa o pedido como `PENDENTE` e tenta depois. |
| **Script 4** | `worker_4_rotina_clientes` | **Reparo** | Monitora a base da OmniChat. Se um cliente ganhar um e-mail novo, "destrava" as mensagens antigas para o Script 2 processar. |
| **Script 5** | `worker_5_rotina_pedidos` | **Sincronia** | Monitora pedidos já concluídos. Se houver alteração de valor, produtos ou troca de negócio, atualiza os dados no HubSpot. |

---

## 🔄 Novo Fluxo de Trabalho (Workflow Detalhado)

### 1. Ingestão e Identificação
* O **Script 1** baixa os dados brutos.
* O **Script 2** processa esses dados. Se o cliente **não tiver e-mail**, o script ignora (logs de aviso). Se tiver e-mail, ele garante que o Contato exista no HubSpot.

### 2. O Fluxo do Pedido (Script 3)
Quando um pedido entra, o script executa a seguinte lógica de decisão:

1.  **Busca no HubSpot:** Procura o Negócio (Deal) mais recente associado àquele contato que esteja **ABERTO**.
2.  **Cenário A: Nenhum Negócio Encontrado** ❌
    * **Ação:** Nenhuma.
    * **Status:** O pedido permanece no MongoDB como `PENDENTE`.
    * **Log:** `[AGUARDANDO] Nenhum negócio aberto encontrado`.
    * **Repescagem:** O script tentará novamente no próximo ciclo.
3.  **Cenário B: Negócio Encontrado** ✅
    * **Ação:** O script assume esse negócio.
    * **Atualização:**
        * Renomeia o Deal para: `Nome Cliente #ID_Pedido`.
        * Limpa itens antigos (se houver).
        * Cria os novos **Line Items** (produtos) com quantidade e preço.
        * Atualiza o valor total do Deal.
    * **Status:** Marca o pedido como `CONCLUIDO` no MongoDB.

---

## 📊 Mapeamento de Dados

### Contato (Contact)
* **Chave Única:** E-mail.
* **Dados:** Nome, Telefone, Link do WhatsApp.

### Negócio (Deal)
A automação agora apenas **LÊ** e **ATUALIZA** negócios existentes.

| Campo | Quem Preenche? | Observação |
| :--- | :--- | :--- |
| **Criação do Deal** | **USUÁRIO (Manual)** | O robô não cria mais deals. |
| **Nome do Deal** | **Script 3** | Atualiza para o padrão `Nome #ID`. |
| **Valor (Amount)** | **Script 3** | Atualiza com o somatório dos itens. |
| **Itens (Produtos)** | **Script 3** | Cria os itens nativos do HubSpot. |
| **Pipeline/Fase** | **USUÁRIO** | O robô não move o card de etapa. |

### Notas (Notes)
1.  **Nota de Conversa (Script 2):** Criada e vinculada apenas ao **Contato**. Contém o link para o chat.
2.  **Nota de Pedido (Script 3 e 5):** Detalhes técnicos do pedido (Frete, Descontos), vinculada ao **Negócio**.

---

## 🛠️ Manutenção e Comandos Úteis

### Subir o ambiente
```bash
docker compose up -d
