# Automação de ofertas no WhatsApp

Automação de vendas pelo WhatsApp usando n8n, Evolution API, PostgreSQL, Redis e Docker.
O sistema agenda o envio diário de ofertas com imagem, descrição e preço para grupos configurados do WhatsApp, nos horários de 08:00, 14:00 e 20:00. Também permite pausar automaticamente os envios ao marcar o produto como indisponível.
Inclui configuração local com Docker, conexão do WhatsApp por QR Code, controle de produto/preço, agendamento de mensagens e proteção de credenciais com .env.

O projeto usa quatro serviços em containers Docker:

| Serviço | Porta | Função |
| --- | ---: | --- |
| n8n | `5678` | Monta e agenda o fluxo de automação. |
| Evolution API | `8080` | Mantém a conexão com o WhatsApp e envia as mensagens. |
| PostgreSQL | interna | Banco de dados da Evolution API. |
| Redis | interna | Cache usado pela Evolution API. |

## Como o sistema funciona

1. O **Schedule Trigger** do n8n inicia o fluxo nos horários programados.
2. O nó **Edit Fields 1** define o nome do produto, o preço e se ele continua disponível.
3. O nó **Produto disponível?** encerra o fluxo caso o campo `produto_disponivel` esteja como `false`.
4. O n8n lê a imagem em `produtos/produto.png`.
5. O texto e a imagem são unidos pelo nó **Merge**.
6. O nó **Code in JavaScript** cria uma cópia da mensagem para cada grupo configurado.
7. O nó **HTTP Request** pede à Evolution API que envie a imagem e a legenda para cada grupo.

```text
Schedule Trigger ─┐
                  ├─> Edit Fields 1 ─> Produto disponível? ─> mensagem + imagem
Manual Trigger ───┘                                        └─> grupos ─> Evolution API ─> WhatsApp
```

## Requisitos

- Docker Desktop (Windows/macOS) ou Docker Engine + Docker Compose (Linux).
- Uma conta e um número de WhatsApp para conectar à Evolution API.
- A pasta `produtos/` com a imagem que será enviada.

## Iniciar localmente

No terminal, dentro da pasta do projeto:

```bash
docker compose up -d
```

Confira se todos os serviços foram iniciados:

```bash
docker compose ps
```

Painéis locais:

- n8n: <http://localhost:5678>
- Evolution Manager: <http://localhost:8080/manager>

Para parar os containers:

```bash
docker compose down
```

> `down` não apaga os dados, porque PostgreSQL, Redis, n8n e as instâncias do WhatsApp usam volumes Docker. Não use `docker compose down -v` a menos que queira apagar esses dados.

## Conectar o WhatsApp

1. Abra <http://localhost:8080/manager>.
2. Informe a URL do servidor e a chave global da Evolution API configurada no `docker-compose.yml`.
3. Crie ou abra a instância `vendas_condominio`.
4. Gere o QR Code.
5. No WhatsApp do celular, abra **Configurações > Aparelhos conectados > Conectar aparelho** e leia o QR Code.
6. Confirme no Manager que a instância está conectada.

## Configurar o fluxo no n8n

Abra <http://localhost:5678> e abra o workflow da automação.

### Produto e preço

No nó **Edit Fields 1**, altere:

- `produto`: nome que aparecerá na mensagem;
- `preco`: preço que aparecerá na mensagem;
- `produto_disponivel`: controle de envio (`true` para enviar, `false` para parar).

Depois clique em **Save**.

### Imagem

Substitua o arquivo `produtos/produto.png` pela imagem do produto. O nó **Read/Write Files from Disk** usa o caminho:

```text
/files/produtos/produto.png
```

Esse caminho dentro do container corresponde à pasta `produtos/` deste projeto.

### Texto da mensagem

O texto é montado no nó **Edit Fields 2**, no campo `mensagem`. Edite-o para mudar descrição, tamanhos, emojis ou chamada para contato.

### Grupos de destino

Os identificadores dos grupos ficam no nó **Code in JavaScript**, no array `groups`. Para incluir ou remover grupos, ajuste essa lista.

Use apenas grupos nos quais você tem autorização para divulgar produtos.

## Horários automáticos

O nó **Schedule Trigger** está programado para todos os dias às:

- 08:00
- 14:00
- 20:00

O fuso configurado no projeto é `America/Sao_Paulo`.

Para a agenda funcionar, o workflow precisa estar **ativo** no n8n e o computador/servidor precisa estar ligado. Não use o botão **Execute workflow** para testar sem cuidado: ele pode enviar a mensagem imediatamente para todos os grupos configurados.

## Pausar após vender

Há duas formas de interromper os envios:

1. Recomendada: no **Edit Fields 1**, mude `produto_disponivel` para `false` e clique em **Save**. O workflow continua ativo, mas encerra antes de enviar qualquer mensagem.
2. Emergência: desligue o toggle **Active** do workflow no canto superior direito do n8n. Isso interrompe toda a automação.

Para voltar a anunciar, atualize produto/preço, deixe `produto_disponivel` como `true`, salve e mantenha o workflow ativo.

## Segurança antes de publicar no GitHub

As credenciais locais ficam no arquivo `.env`, que já está ignorado pelo Git. Antes de publicar o projeto em um repositório público:

1. Use `.env.example` como modelo para criar o seu `.env`; ele deve conter valores longos, únicos e privados.
2. Nunca publique o `.env`, QR Codes, sessões do WhatsApp, arquivos de banco, volumes Docker ou chaves de API.
3. O arquivo `docker-compose.yml.save` é uma cópia local antiga e também está ignorado pelo Git.
4. Os painéis locais são expostos apenas em `127.0.0.1`, isto é, apenas neste computador consegue acessá-los.

## Hospedagem

Na configuração atual o sistema roda em `localhost`, portanto o computador deve permanecer ligado e sem suspensão para os horários funcionarem. Para funcionar 24 horas sem o PC, é necessário migrar os containers e os volumes para um servidor.

## Uso responsável

Envie mensagens somente a contatos e grupos que esperam receber suas divulgações. Evite repetição excessiva, respeite as regras do WhatsApp e mantenha a opção de pausa ativada ao vender o produto.
