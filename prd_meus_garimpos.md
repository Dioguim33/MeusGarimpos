# PRD — Meus Garimpos

**Autor:** Diogo
**Status:** Rascunho v6 (reconstrução do zero)
**Última atualização:** 09/07/2026

*App de controle de compra e venda de itens usados (flips/garimpos) em marketplaces.*

---

## 1. Visão do produto

O Meus Garimpos é um app pessoal para registrar e acompanhar a operação de revenda de itens usados. Cada item comprado é um estoque que vira uma venda (ou troca) futura; o app registra o custo real, acompanha o item até a saída e mostra, de forma clara, quanto foi gasto, quanto entrou e quanto de lucro a operação gerou.

O objetivo de fundo é ter **controle desde cedo** para responder uma pergunta concreta: *essa operação de compra e venda está dando certo ou não?*

## 2. Problema

Hoje o controle é feito em planilha e/ou caderno — disperso, manual e difícil de consultar no momento da compra ou da venda. Falta visão rápida de estoque, lucro acumulado e histórico. O app substitui esse controle por algo estruturado, editável e acessível tanto no notebook/web quanto no celular.

## 3. Público-alvo

- **Agora:** uso 100% pessoal (Diogo, operando a revenda).
- **Depois:** não descartado abrir para outros usuários — por isso a arquitetura já nasce preparada para múltiplos usuários (ver seção 9).

## 4. Objetivos do MVP

Entregar um app enxuto que faça muito bem o ciclo básico: **registrar a compra → acompanhar o estoque → registrar a saída (venda ou troca) → ver o resultado.**

Métricas de sucesso do MVP (para o próprio uso):
- Registrar uma compra em menos de 30 segundos, com foto.
- Ter, na tela inicial, a resposta imediata de "quanto tenho em estoque, quanto já lucrei".
- Substituir a planilha/caderno por completo — zero registro paralelo.

## 5. Escopo

### 5.1. Dentro do MVP

- **Cadastro de compra (item):** custo de compra, descrição, foto.
- **Custos adicionais:** lista de custos extras por item (frete, taxa de plataforma, etc.), cada um com descrição + valor, para refletir o custo real gasto no produto.
- **Preço pedido ("a pedida"):** valor de anúncio, para comparar com o preço efetivamente vendido.
- **Edição** do item cadastrado.
- **Registro de saída:** marcar item como **vendido** (com preço e data) ou **trocado** (com tipo de troca e volta).
- **Resumo / Dashboard:** total gasto, total já vendido, lucro, quantidade em estoque, quantidade de vendas.
- **Login:** e-mail/senha e Google; biometria (Android) para quem já tem cadastro.
- **Plataformas:** Android e Web.
- **Funcionamento offline-first** com sincronização na nuvem.

### 5.2. Fora do MVP (v2+)

- **Controle de parcelas** (compra parcelada para revenda — ex: o caso do notebook). Tratado como exceção pontual/legado no MVP; à vista é o modo padrão daqui pra frente.
- Múltiplos usuários de verdade (embora a base já suporte).
- Relatórios avançados, gráficos históricos, categorização rica.

## 6. Modelo de dados (núcleo)

**Item** (entidade central — do momento da compra até a saída):

| Campo | Descrição |
|---|---|
| `id` | Identificador único |
| `descricao` | O que é o item |
| `custo_compra` | Quanto foi gasto na compra |
| `custos_adicionais` | Lista de `{ descricao, valor }` (frete, taxa, etc.) |
| `foto` | Imagem do item |
| `preco_pedido` | Valor de anúncio ("a pedida") — opcional |
| `status` | `em_estoque` \| `vendido` \| `trocado` |
| `data_compra` | Quando foi comprado |
| `preco_venda` | Preenchido ao marcar como vendido |
| `data_venda` | Preenchido ao marcar como vendido |
| `tipo_troca` | `troca_total` \| `volta_minha` \| `volta_cliente` (só quando `trocado`) |
| `valor_volta` | Valor em dinheiro da volta (só quando há volta) |
| `item_recebido` | Referência ao novo item gerado pela troca — opcional |
| `created_at` / `updated_at` | Controle de sincronização |

**Ciclo de vida do item (máquina de estados):**

```
                        ┌──→ vendido   (preco_venda, data_venda)
[compra] → em_estoque ──┤
                        └──→ trocado   (tipo_troca, valor_volta, item_recebido)
```

- **`troca_total`** — troca 100%, sem dinheiro dos dois lados.
- **`volta_minha`** — você deu um item + dinheiro por cima (volta sai do seu bolso).
- **`volta_cliente`** — você recebeu um item + dinheiro do cliente (volta entra pra você).

## 7. Especificação funcional

### 7.1. Cadastrar compra
Usuário registra um novo item com descrição, custo de compra e foto; opcionalmente adiciona custos extras e o preço pedido.
**Critérios de aceite:**
- Descrição e custo de compra são obrigatórios; foto, custos adicionais e preço pedido são opcionais.
- Ao salvar, o item entra com status `em_estoque` e aparece na lista de estoque.
- O total gasto no dashboard é atualizado somando custo de compra + custos adicionais.

### 7.2. Custos adicionais
**Critérios de aceite:**
- Usuário pode adicionar N linhas de custo, cada uma com descrição (ex: "frete", "taxa OLX") e valor.
- O custo total do item = `custo_compra` + Σ `custos_adicionais`.

### 7.3. Editar item
**Critérios de aceite:**
- Todos os campos podem ser editados, inclusive de itens já vendidos/trocados.
- Editar custos ou preço recalcula os totais do dashboard.

### 7.4. Registrar venda
**Critérios de aceite:**
- Usuário informa preço de venda e data.
- Status muda para `vendido`; item sai do estoque e entra na contagem de vendas.
- Lucro e total vendido são recalculados.

### 7.5. Registrar troca
**Critérios de aceite:**
- Usuário escolhe o `tipo_troca` (total, volta minha, volta cliente).
- Se houver volta, informa `valor_volta`.
- Opcionalmente registra o `item_recebido`.
- Status muda para `trocado`; item sai do estoque.
- **Tratamento no lucro (regra definida):** um item trocado **não conta como venda no lucro de caixa**. Em vez disso, o `item_recebido` entra como **novo item no estoque**, com custo-base = custo total do item que saiu ∓ `valor_volta` (`volta_minha` soma ao custo-base; `volta_cliente` subtrai). O lucro em reais só se realiza quando esse novo item for vendido. **Exceção:** se a volta do cliente superar o custo do item que saiu, o custo-base do item recebido é fixado em **zero** e o excedente entra como **lucro de caixa no momento da troca**. Assim o "lucro" do dashboard sempre representa dinheiro real.

### 7.6. Dashboard / Resumo
**Fórmulas:**
- **Custo total do item** = `custo_compra` + Σ `custos_adicionais`
- **Total gasto** = Σ custo total de todos os itens comprados
- **Total vendido** = Σ `preco_venda` dos itens `vendido`
- **Lucro (de vendas)** = Σ (`preco_venda` − custo total) dos itens `vendido`
- **ROI** = Lucro ÷ custo total dos itens vendidos × 100
- **Qtd em estoque** = contagem de itens `em_estoque`
- **Qtd de vendas** = contagem de itens `vendido`

*Nota (v2, parcelas):* quando houver item parcelado, o capital disponível para reinvestir após a venda = `preco_venda` − saldo devedor das parcelas daquele item (não o lucro simples). Regra já validada; entra com o módulo de parcelas.

## 8. Fluxo de telas

O MVP tem **6 telas**. O formulário de cadastro/edição é único (a mesma tela serve para criar e editar).

```
                        ┌───────────┐
                        │   Login   │
                        └─────┬─────┘
                              ▼
          ┌───────────┐   ┌───────────┐
          │ Dashboard │◄─►│  Estoque  │
          └───────────┘   └─────┬─────┘
                    novo item │  │ tocar item
                    ┌─────────┘  └─────────┐
                    ▼                      ▼
          ┌───────────────┐        ┌───────────────┐
          │  Cadastro /   │◄───────│   Detalhe do  │
          │    edição     │ editar │      item     │
          └───────────────┘        └───┬───────┬───┘
                                       ▼       ▼
                              ┌────────────┐ ┌────────────┐
                              │  Registrar │ │  Registrar │
                              │    venda   │ │    troca   │
                              └────────────┘ └────────────┘
```

**8.1. Login** — e-mail/senha, Google e (Android) biometria para retorno. Leva ao Dashboard.

**8.2. Dashboard / Resumo** — tela inicial. Mostra total gasto, total vendido, lucro, ROI, qtd em estoque, qtd de vendas. Alterna com o Estoque via navegação principal.

**8.3. Estoque** — lista dos itens. Ponto de partida para cadastrar (botão "novo item") e para abrir um item (toque → Detalhe). Idealmente permite filtrar por status (em estoque / vendidos / trocados).

**8.4. Cadastro / edição** — formulário único. Campos: descrição, custo de compra, foto, custos adicionais (lista), preço pedido. Vem vazio (novo) ou preenchido (edição).

**8.5. Detalhe do item** — visão de um item e suas ações: **editar**, **registrar venda**, **registrar troca**.

**8.6. Registrar venda / Registrar troca** — telas de saída. Venda pede preço + data. Troca pede tipo + volta (quando houver) + item recebido.

**Navegações implícitas importantes:**
- **Cadastro é reaproveitado:** "novo item" e "editar" abrem a mesma tela.
- **Troca gera item:** ao concluir uma troca com item recebido, o app abre o Cadastro já preparando esse novo item com o custo-base herdado.

## 9. Critérios de aceite detalhados (com casos de borda)

### 9.1. Cadastrar / editar item
- **Validação:** salvar sem descrição ou sem custo de compra é bloqueado, com mensagem clara.
- **Custo zero:** permitido (item recebido de graça / brinde) — não trava o salvamento.
- **Foto:** opcional; item sem foto exibe um placeholder.
- **Editar item vendido/trocado:** permitido; ao alterar custo/preço, os totais e o lucro são recalculados imediatamente.

### 9.2. Registrar venda
- **Prejuízo:** `preco_venda` menor que o custo total é permitido e resulta em lucro negativo daquele item (cenário real, sem fluxo especial).
- **Data:** data de venda não pode ser anterior à data de compra; venda no futuro é bloqueada ou alertada.
- **Item já vendido:** não aparece mais na lista de estoque, então não pode ser vendido de novo.
- **Reverter venda:** permitido. O usuário pode desfazer uma venda; o item volta ao status `em_estoque`, sai da contagem de vendas e os totais/lucro são recalculados. A mesma lógica vale para reverter uma troca — se a troca gerou um `item_recebido`, esse item também é removido do estoque ao reverter.

### 9.3. Registrar troca
- **Troca total:** sem `valor_volta`.
- **Com volta:** `valor_volta` obrigatório e maior que zero.
- **Custo-base do item recebido:** `volta_minha` soma ao custo total do item que saiu; `volta_cliente` subtrai.
- **Volta do cliente maior que o custo:** o custo-base do item recebido é fixado em **zero** e o excedente (`valor_volta` − custo total) entra como **lucro de caixa no momento da troca**.
- **Sem item recebido:** troca registrada normalmente, mas nenhum novo item é criado no estoque.

### 9.4. Dashboard
- **Estado vazio:** sem itens, todos os totais e contagens exibem zero (sem erro).
- **ROI com custo zero:** se o custo total dos vendidos for zero, o ROI não deve quebrar (exibir "—" em vez de divisão por zero).

### 9.5. Offline / sincronização
- **Criar/editar offline:** registros feitos sem internet ficam salvos localmente e sincronizam ao reconectar.
- **Conflito entre dispositivos:** resolução por *last-write-wins* usando `updated_at`.
- **Abrir app offline:** sessão fica em cache; o usuário consegue abrir e consultar sem internet.

## 10. Requisitos não-funcionais

- **Offline-first:** o app funciona 100% sem internet; registros feitos offline sincronizam automaticamente ao reconectar.
- **Sincronização multi-dispositivo:** o mesmo dado é acessível na web e no celular.
- **Plataformas:** Android e Web no primeiro momento.
- **Autenticação:** e-mail/senha e Google; biometria (Android) para retorno de usuário já cadastrado.
- **Preparado para escala:** modelo de dados e auth já comportam múltiplos usuários sem reescrita.

## 11. Decisão de arquitetura (ADR)

**Decisão:** Nuvem com login + arquitetura offline-first, para Android e Web.

**Racional:** o requisito de consultar no celular longe do notebook exige o mesmo dado em dois dispositivos → nuvem → login. O offline-first foi escolhido para usabilidade no momento da transação sem depender de sinal e para evitar reescrita caso o app escale.

**Stack de referência:** Flutter (Android + Web) + Firebase (Auth + Firestore). O Firestore já traz *offline persistence* embutido (cache local + sync automático) nas duas plataformas — na web via IndexedDB. Para um offline-first mais robusto, avaliar um banco local como fonte da verdade (decisão de implementação).

**Notas de plataforma:**
- **Biometria:** no Android é nativa (impressão digital/face). Na web não existe biometria direta. **Decisão:** biometria fica **só no Android** no MVP; a web usa e-mail/senha + Google.

**Alternativas descartadas:**
- *SQLite local puro:* não sincroniza entre dispositivos.
- *Nuvem sem offline:* dependeria de conexão no momento da transação.

## 12. Decisões

> ✅ **Contabilidade de troca** — item trocado não gera lucro de caixa; o item recebido entra no estoque com custo-base herdado. Ver seção 7.5.
> ✅ **Biometria** — só no Android no MVP; web usa e-mail/senha + Google. Ver seção 11.
> ✅ **Preço pedido** — é o valor do anúncio ("a pedida"), usado como referência. O número que entra no lucro é sempre o `preco_venda` real. Métrica de negociação (pedida × vendido) fica para v2.

> ✅ **Reverter venda** — permitido; o item volta ao estoque e os totais são recalculados. Mesma lógica para reverter troca. Ver seção 9.2.
> ✅ **Volta do cliente maior que o custo** — custo-base do item recebido vira zero; o excedente entra como lucro de caixa na hora. Ver seções 7.5 e 9.3.

**Opcional a avaliar (não bloqueia o MVP):**
- **Marcador de retirada vs. entrega na venda:** campo opcional `modo_entrega` (`retirada` \| `entrega`) para futuramente analisar se entregar compensa ou come margem.

## 13. Roadmap

- **v1 (MVP):** cadastro de compra com custos adicionais e preço pedido, edição, registro de venda e troca, dashboard, login (e-mail/senha + Google + biometria Android), offline-first, Android + Web.
- **v2:** módulo de parcelas (compra parcelada para revenda), com cálculo de capital para reinvestir.
- **v3+:** relatórios/gráficos históricos, métricas de negociação, abertura para múltiplos usuários.
