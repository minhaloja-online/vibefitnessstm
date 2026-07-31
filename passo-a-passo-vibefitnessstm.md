# Vibe Fitness STM — configurar Firebase, GitHub e Render

Este passo a passo assume que você já fez a parte de criar as "caixas vazias":

- Repositório do servidor no GitHub: **`servidor-pagamento-vibefitnessstm`**
- Repositório do site no GitHub: **`vibefitnessstm`**
- Web Service no Render: **`servidor-pagamento-vibefitnessstm`**
- Um projeto no Firebase (você não me disse o nome ainda)

Falta preencher essas caixas: subir os arquivos certos, ligar o Firestore, e configurar as variáveis no Render. É isso que os passos abaixo fazem, na ordem que menos trava.

> **Sobre o nome do projeto Firebase:** como você ainda não me passou o nome, sigo os passos como se fosse configurar do zero. O nome em si não entra em nenhum arquivo de código — o servidor lê o projeto pela chave de conta de serviço, não pelo nome. Pode manter o que você já criou, ou nomear como `vibefitnessstm` só por organização. Se quiser, me diga o nome depois e eu atualizo os documentos.

Tenha em mãos antes de começar:
- A **API Key de Sandbox** da Asaas (`sandbox.asaas.com` → Integrações → Chaves de API)
- Acesso ao projeto Firebase (o mesmo que a Vibe Fitness STM vai usar)

---

## 1. Firebase — banco de dados e chave de acesso

### 1.1 Ativar o Firestore

1. Abra o projeto no [console.firebase.google.com](https://console.firebase.google.com)
2. Menu lateral → **Firestore Database** → **Criar banco de dados**
3. Escolha a região mais próxima do seu público (ex.: `southamerica-east1`)
4. Comece em **modo de produção** — não em modo de teste. Modo de teste libera leitura e escrita para qualquer pessoa na internet, e isso inclui trocar o preço das suas roupas.

### 1.2 Publicar as regras de segurança

1. Ainda em **Firestore Database**, abra a aba **Regras**
2. Apague o conteúdo atual e cole o conteúdo do arquivo **`firestore.rules`** (está no pacote que gerei)
3. **Publicar**

Essas regras deixam qualquer visitante **ler** o catálogo (`produtos`) e as configurações da loja (`config`), mas só você, logado, pode **alterar** os dois. Pedidos (`pedidos`) só podem ser criados pelo próprio cliente, e só você lê ou atualiza depois.

### 1.3 Gerar a chave de conta de serviço

O servidor não faz login como pessoa — ele usa uma chave de máquina.

1. **⚙️ Configurações do projeto → Contas de serviço**
2. **Gerar nova chave privada** → baixa um arquivo `.json`
3. Guarde esse arquivo com cuidado: ele dá acesso total ao banco. Não sobe no GitHub, não vai por WhatsApp — só entra no Render, no próximo bloco.

---

## 2. GitHub — repositório `servidor-pagamento-vibefitnessstm`

Este repositório é **privado** e recebe só o backend — nunca o site.

Suba estes arquivos na raiz (arraste tudo de uma vez na tela de upload do GitHub):

```
server.js
package.json
.gitignore
```

E dentro de uma subpasta **`public/`**:

```
public/sucesso.html
public/pendente.html
public/falha.html
```

> Para criar a subpasta pelo upload do GitHub: arraste a pasta `public` inteira de uma vez, ou digite `public/sucesso.html` no campo de nome ao criar cada arquivo manualmente.

**Não suba** `.env`, nem o `.json` da conta de serviço do Firebase. As chaves vão direto no Render — nunca em arquivo dentro do repositório.

Clique em **Commit changes**.

> ⚠️ O Render sempre constrói a partir do que está no GitHub. Se você editar o `server.js` só na sua máquina sem commitar, o Render continua rodando a versão antiga.

---

## 3. GitHub — repositório `vibefitnessstm` (o site)

Aqui entram o `index.html` da loja e as três páginas de retorno, na mesma pasta:

```
index.html
sucesso.html
pendente.html
falha.html
```

As três páginas de retorno já estão prontas no pacote (pasta `vibefitnessstm-site/`), com o mesmo conteúdo das que foram para o repositório do servidor — elas precisam existir nos dois lugares.

Na prática, quem mais aparece é a `sucesso.html`: quando o pagamento confirma na hora (cartão ou Pix), a Asaas devolve o cliente para `URL_SITE/?pagamento=sucesso`, e é a sua página principal (`index.html`) que trata esse retorno. `pendente.html` e `falha.html` ficam no pacote por completude, mas dificilmente são abertas diretamente — a Asaas só redireciona em caso de sucesso, e boleto não redireciona nunca.

Depois de subir, ative o **GitHub Pages** deste repositório (**Settings → Pages → Branch: main**) se ainda não estiver ativo, e anote o endereço gerado — algo como `https://SEU-USUARIO.github.io/vibefitnessstm`. Você vai usar esse endereço em duas variáveis no próximo bloco.

---

## 4. Render — conectar o repositório e configurar o serviço

No serviço `servidor-pagamento-vibefitnessstm` que você já criou:

### 4.1 Conectar ao repositório

Em **Settings**, confira/defina:

| Campo | Valor |
|---|---|
| Repository | `servidor-pagamento-vibefitnessstm` |
| Branch | `main` |
| Root Directory | (vazio) |
| Build Command | `npm install` |
| Start Command | `npm start` |
| Instance Type | Free (ou o plano que preferir) |

### 4.2 Secret File — a chave do Firebase

Role até **Secret Files** (ou **Environment → Secret Files**):

- **Filename:** `firebase.json`
- **Contents:** abra o `.json` que você baixou do Firebase no Bloco de Notas, selecione tudo, copie e cole aqui **exatamente como está**. Não tire quebras de linha.

> O nome do arquivo aqui precisa bater **exatamente** com o que vai na variável `FIREBASE_SERVICE_ACCOUNT_PATH` abaixo. Se o Secret File se chama `firebase.json`, a variável tem que ser `/etc/secrets/firebase.json`. Errar isso derruba o servidor no start com `ENOENT: no such file or directory`.

### 4.3 Variáveis de ambiente

Em **Environment Variables**, adicione:

| Key | Value |
|---|---|
| `FIREBASE_SERVICE_ACCOUNT_PATH` | `/etc/secrets/firebase.json` |
| `ASAAS_API_KEY` | sua chave de Sandbox |
| `ASAAS_API_URL` | `https://api-sandbox.asaas.com/v3` |
| `URL_SITE` | `https://SEU-USUARIO.github.io/vibefitnessstm` |
| `ORIGENS_PERMITIDAS` | `https://SEU-USUARIO.github.io` |

Troque `SEU-USUARIO` pelo seu usuário real do GitHub (o do endereço que o Pages te deu no passo 3).

Repare na diferença entre as duas últimas:
- **`URL_SITE`** é o caminho completo até a pasta do site (o cliente volta para lá depois de pagar).
- **`ORIGENS_PERMITIDAS`** é só o domínio, sem caminho — é assim que o navegador se identifica.

Nenhuma das duas leva barra no final.

Faltam `URL_API` e `ASAAS_WEBHOOK_TOKEN`, que só existem depois que o serviço nascer. Salve o que já preencheu — o Render reinicia sozinho.

---

## 5. Fechar o ciclo: `URL_API`

Depois do deploy (uns 2 minutos), o Render mostra o endereço do serviço no topo da página — algo como:

```
https://servidor-pagamento-vibefitnessstm.onrender.com
```

1. Copie esse endereço
2. **Environment** → adicione:

| Key | Value |
|---|---|
| `URL_API` | o endereço que o Render mostrou |

3. Salve. Abra no navegador: `SEU-ENDERECO/api/status`

Tem que responder:

```json
{"ok":true,"ambiente":"sandbox"}
```

Se apareceu isso, o servidor está de pé e conectado ao Firebase. Confira nos **Logs** (aba **Logs** do Render):

```
Firebase conectado ao projeto: <nome do seu projeto>
Servidor rodando na porta 10000
```

---

## 6. Webhook na Asaas

É aqui que o pagamento é confirmado de verdade — a página de sucesso, por si só, não prova nada.

Painel da Asaas (Sandbox) → **Integrações → Webhooks → Adicionar Webhook**

| Campo | Valor |
|---|---|
| Este Webhook ficará ativo? | ligado |
| Nome do Webhook | `vibe-fitness-stm` |
| URL do Webhook | `https://servidor-pagamento-vibefitnessstm.onrender.com/api/webhook` |
| E-mail | o seu |
| Versão da API | `v3` |
| Token de autenticação | clique em **Gerar Token** e copie |
| Tipo de envio | Sequencial |
| Fila de sincronização ativada? | ligado |

Em **Adicionar Eventos → Cobranças**, marque pelo menos:

- `PAYMENT_CONFIRMED`
- `PAYMENT_RECEIVED`

**Salvar.**

Volte ao Render → **Environment** → adicione a última variável:

| Key | Value |
|---|---|
| `ASAAS_WEBHOOK_TOKEN` | o token que você acabou de gerar |

Tem que ser **exatamente igual** ao do painel da Asaas. Se não bater, o servidor devolve 401 e todo pagamento fica preso sem seu conhecimento.

> ⚠️ Confirme que a URL termina em `/api/webhook`. Só o domínio não funciona.

### Cadastre o domínio do site na Asaas

Em **Configurações da conta → Informações / Dados comerciais**, o site cadastrado precisa ser o mesmo do `URL_SITE`. Sem isso, o cliente não volta para a loja depois de pagar (o pagamento acontece normalmente — só a volta que não).

---

## 7. Ligar a loja ao servidor

1. Confirme que `index.html`, `sucesso.html`, `pendente.html` e `falha.html` estão publicados no repositório `vibefitnessstm`
2. Abra sua loja, entre como vendedor
3. **Configurações da Loja → Endereço do servidor de pagamento** → cole `https://servidor-pagamento-vibefitnessstm.onrender.com`
4. Salvar

O botão **💳 Pagar agora** aparece no carrinho na hora. Se apagar esse campo, o botão some e a loja volta a vender só por WhatsApp.

---

## 8. Primeira compra de teste

Monte um carrinho, preencha o CPF e clique em **Pagar agora**. A fatura da Asaas abre com Pix, boleto e cartão na mesma tela.

### Cartão de teste

| Resultado | Número | CCV | Validade |
|---|---|---|---|
| Aprovado | 4444 4444 4444 4444 | 123 | qualquer mês futuro |
| Recusado (Mastercard) | 5184 0197 4037 3151 | 123 | qualquer mês futuro |
| Recusado (Visa) | 4916 5613 5824 0741 | 123 | qualquer mês futuro |

### Pix ou boleto

No Sandbox eles não são pagos de verdade. Gere a cobrança, abra ela no painel do Sandbox (**Cobranças**) e clique em **CONFIRMAR PAGAMENTO** — é esse botão que dispara o webhook.

### O que conferir

- **Pagou** → o pedido aparece no painel do vendedor como **📦 Pendente de envio**, com o estoque debitado. ✅
- **Recusou ou desistiu** → o pedido continua invisível. Está certo, não é bug. ✅

Se pagou e nada apareceu, o webhook não chegou:
1. **Asaas → Integrações → Logs de Webhooks** — mostra se a Asaas chamou e o que recebeu de volta.
2. **Render → Logs** — procure `Webhook Asaas: pedido ... — PAYMENT_CONFIRMED`.

Se a Asaas registrou **401**, o `ASAAS_WEBHOOK_TOKEN` do Render está diferente do token cadastrado no painel.

---

## O plano gratuito do Render

O servidor **dorme** depois de uns 15 minutos sem uso. A primeira compra depois de um tempo parado demora uns 30–50 segundos para abrir a fatura. Isso também afeta o webhook: se a Asaas chamar com o servidor dormindo, a primeira tentativa pode dar timeout (ela tenta de novo, mas falhas repetidas pausam a fila de sincronização). Para testar, não atrapalha. Para vender de verdade, o plano pago mais barato resolve os dois pontos.

---

## Virar para produção

1. Na conta de **produção** (`asaas.com`), gere uma API Key nova.
2. Cadastre o webhook **de novo** lá (o de Sandbox não vale em produção). Gere um token novo.
3. No Render, troque:

| Key | Novo valor |
|---|---|
| `ASAAS_API_KEY` | a chave de produção |
| `ASAAS_API_URL` | `https://api.asaas.com/v3` |
| `ASAAS_WEBHOOK_TOKEN` | o token novo |

4. Confira em `/api/status` que aparece `"ambiente":"producao"`.
5. Faça uma compra real de R$ 1,00 e estorne depois.

---

## Se travar

| Sintoma | Causa provável |
|---|---|
| `ENOENT: /etc/secrets/firebase.json` | O Secret File não existe ou tem outro nome. Tem que ser `/etc/secrets/` + o nome exato que você deu. |
| `SyntaxError: "undefined" is not valid JSON` | Nenhuma das duas variáveis do Firebase chegou preenchida. |
| `/api/status` não abre | Veja **Logs** no Render — quase sempre é a chave do Firebase. |
| Erro 401 vindo da Asaas | Chave de um ambiente com a URL do outro (Sandbox × produção). |
| Webhook com 401 nos logs da Asaas | `ASAAS_WEBHOOK_TOKEN` diferente do token cadastrado no painel. |
| Erro de CORS no console do navegador | `ORIGENS_PERMITIDAS` está com caminho ou barra no fim. Só o domínio. |
| Cliente paga e não volta ao site | Domínio não cadastrado nos dados comerciais da Asaas — ou foi boleto, que não redireciona mesmo. |
| Botão "Pagar agora" não aparece | O campo em Configurações da Loja está vazio, ou não foi salvo. |
| Pedido pago continua invisível | Webhook não chegou. Veja os Logs de Webhooks na Asaas e os Logs no Render. |
| Fatura demora ~40s para abrir | Normal no plano free — o servidor estava dormindo. |
