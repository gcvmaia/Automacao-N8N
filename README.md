# NFS-e automática por pagamento com n8n e Asaas

Automação em n8n que emite uma nota fiscal de serviço (NFS-e) assim que um pagamento é confirmado e envia o PDF ao cliente por e-mail. Projeto do bootcamp Santander de automação com n8n.

## Sobre o Asaas

O [Asaas](https://www.asaas.com) é uma plataforma brasileira de cobranças e pagamentos. Com ela, uma empresa gera cobranças por Pix, boleto ou cartão, cadastra clientes, recebe os pagamentos e emite notas fiscais de serviço. Ela também avisa outros sistemas, por webhook, quando algo acontece, como um pagamento recebido ou uma nota autorizada. É esse aviso que dispara a automação.

O projeto roda no **sandbox do Asaas**, um ambiente de testes que imita o Asaas real. Nele os clientes, as cobranças e as notas são fictícios, nada é cobrado de verdade e nenhuma nota chega à prefeitura. Serve para construir e validar a automação sem risco.

## O que faz

1. Quando recebe um pagamento, o Asaas ativa o webhook do n8n.
2. O n8n agenda a NFS-e no Asaas e pede a emissão imediata, que leva cerca de 20 s.
3. Quando o Asaas avisa que a nota foi autorizada, o n8n baixa o PDF e envia ao cliente por e-mail pelo Gmail SMTP.
4. Cada passo fica registrado numa tabela de controle, uma Data Table do n8n.
5. Se algo falha, você recebe um e-mail de alerta.

## Arquitetura

| Parte | Função |
|---|---|
| Workflow `NFS-e automática por pagamento` | Recebe os avisos, agenda e emite a nota, envia o e-mail e registra tudo |
| Workflow `Manter endereço do webhook atualizado (túnel)` | A cada 2 minutos confere o endereço do túnel Cloudflare e atualiza o webhook no Asaas se mudou |
| Data Table `notas_fiscais_controle` | Colunas `payment_id`, `customer_id`, `invoice_id`, `invoice_number`, `status`, `detalhe` e `valor` |
| Túnel Cloudflare em Docker | Expõe o n8n local para o Asaas |

Status na tabela: `AGENDADA`, `EMITIDA`, `EMITIDA_SEM_EMAIL`, `ERRO_AO_AGENDAR` e `ERRO_EMISSAO`.

## Fluxo principal

- **Entrada:** webhook protegido por token no cabeçalho `asaas-access-token`. Aviso sem token recebe 403. Os eventos são normalizados e roteados entre `PAYMENT_RECEIVED`, `INVOICE_AUTHORIZED` e `INVOICE_ERROR`. Os demais são ignorados.
- **Pagamento recebido:** checa na tabela se o pagamento já foi agendado ou emitido, o que evita nota duplicada, agenda a NFS-e e pede a emissão imediata.
- **Nota autorizada:** confere se a nota ainda não foi enviada, busca o cliente, baixa o PDF, envia o e-mail e marca `EMITIDA`. Se o envio falhar, marca `EMITIDA_SEM_EMAIL` e avisa você com o link do PDF.
- **Erro:** registra o motivo na tabela e avisa por e-mail.
- **Reprocessar:** `POST /webhook/reprocessar-nfse` com o token e o corpo `{"paymentId": "pay_..."}` refaz o agendamento de um pagamento que tinha dado `ERRO_AO_AGENDAR`, por exemplo depois de completar o endereço do cliente.

## Como usar

Importe os arquivos da pasta `workflows/` no seu n8n e ajuste o que for preciso para o seu ambiente. Isso inclui as credenciais do Asaas, do token do webhook e do Gmail SMTP, a Data Table de controle, o ID do serviço municipal e os e-mails. As notas adesivas dentro de cada workflow explicam cada grupo de nós.

O segundo workflow, `Manter endereço do webhook atualizado (túnel)`, só é necessário se você expõe o n8n por um túnel rápido da Cloudflare. Esse túnel muda de endereço a cada reinício, e o workflow atualiza o webhook no Asaas sozinho para os avisos continuarem chegando. Se o seu n8n já tem um endereço público fixo, cadastre esse endereço no Asaas e não importe o segundo workflow.
