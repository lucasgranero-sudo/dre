# DRE - FECHAMENTO

Aplicação web de página única para calcular o DRE (fechamento de resultado) de vendedores do
Mercado Livre. Usada por mentorados de uma mentoria de Mercado Livre.

## Regra número 1: NÃO QUEBRAR ESTAS PREMISSAS

1. **Arquivo único, sem build.** Todo o app vive em `index.html` (HTML + CSS + JS inline).
   Única dependência externa: SheetJS via CDN. Não introduzir React, bundler, npm ou build step
   sem pedido explícito.
2. **100% offline e determinístico.** NÃO existem chamadas de IA/API. Toda análise (reclamações,
   CMV, tarifas) é feita em JavaScript puro no navegador. Isso é intencional: o app é distribuído
   para mentorados, precisa funcionar em qualquer lugar, sem custo e sem risco de "inventar" dado.
   **Nunca adicionar chamada para api.anthropic.com ou qualquer LLM.**
3. **Privacidade.** As planilhas enviadas pelo usuário nunca saem do navegador. Não enviar dados
   para servidor algum. Isso é um argumento comercial do produto, não um detalhe.
4. **Nunca inventar números.** Se um dado não existe na planilha, o app avisa o usuário
   (ex.: produto sem custo cadastrado) e deixa de fora do cálculo. Jamais estimar ou completar.
5. **Persistência = localStorage.** Sem backend, sem login. Dados ficam no navegador do usuário.

## Estrutura do DRE (ordem das seções — definida pelo cliente, não reordenar)

```
RECEITA
  Faturamento bruto Mercado Livre                        [auto]

DEDUÇÕES E CUSTOS OPERACIONAIS   (% = participação sobre o total de deduções, somam 100%)
  Comissão das plataformas   (tarifas de vendas totais)  [auto]
  Gastos com frete           (tarifas de envio)          [auto]
  Ads (Publicidade)          (investimento por campanha) [manual]
  Custo Fulls                (custos do Mercado Envios Full) [manual]
  Frete reverso              (tarifas de devolução)      [manual]
  Outras taxas               (outras tarifas)            [manual]
  = Total deduções operacionais  (SOMA TUDO, inclusive comissão)

REPASSE LÍQUIDO TOTAL = Faturamento - Total deduções

CUSTOS DOS PRODUTOS
  CMV                                                    [auto, via cruzamento]
  Custo com devoluções e perda de produtos               [manual]
  = Total custos de produtos      (%CMV = CMV / faturamento)

CUSTOS FIXOS E VARIÁVEIS  (aluguel, funcionários, contabilidade, energia, ferramentas,
  embalagem, manutenção, pró-labore, despesas diversas)  [todos manuais]
  = Total custos fixos e variáveis   (%Operação = total / faturamento)

EBITDA = Repasse líquido - Total custos de produtos - Total custos fixos
  Margem EBITDA = EBITDA / faturamento

IMPOSTOS
  Alíquota de impostos (%)  ->  Imposto = faturamento × alíquota

LUCRO LÍQUIDO FINAL = EBITDA - Imposto - (impostos adicionais customizados)
  Margem líquida = lucro / faturamento
```

### Decisões de cálculo já tomadas (não reverter sem pedir)

- **Comissão ENTRA na soma** do total de deduções. (Foi removida em uma iteração e reincluída a
  pedido do cliente — está correta assim.)
- **Os percentuais das 6 linhas de dedução são participação DENTRO do total de deduções**
  (somam 100%), não sobre o faturamento. Já validado contra os números do cliente:
  comissão 35,8% / frete 40,1% / ads 11,9% / full 8,6% / frete reverso 0,7% / outras 2,9%.
- Demais percentuais (CMV, operação, margens) são sobre o **faturamento**.
- Termômetro de margem líquida: <0% prejuízo · 0–5% apertada · 5–15% saudável · >15% forte.
  (Valores iniciais, ainda não calibrados pelo cliente.)

## Formatos numéricos — CUIDADO, JÁ CAUSOU BUG

- **Entrada do usuário: padrão brasileiro.** `15.094` = quinze mil e noventa e quatro
  (ponto = milhar). `15.094,50` = com centavos. A função `parseNum()` trata isso:
  ponto seguido de exatamente 3 dígitos = separador de milhar; caso contrário, decimal.
- **O relatório da ML vem em formato americano**: `1389.99` (ponto decimal, sem separador
  de milhar). Verificado: 0 vírgulas em 19.870 células. `parseNum()` cobre os dois casos
  porque valores monetários têm 1–2 casas decimais.
- Exibição sempre em pt-BR via `toLocaleString('pt-BR')` / `fmtBRL()`.

## Relatório de vendas do Mercado Livre (arquivo de entrada principal)

Export "Vendas BR" (Mercado Livre + Mercado Shops), `.xlsx`.

- **O cabeçalho NÃO está na linha 1** — há linhas de título antes (na amostra, cabeçalho na
  linha 6, 1-based). Por isso as colunas são localizadas **por nome**, com detecção automática
  da linha de cabeçalho (`findHeaderRow`). Nunca fixar índices de coluna.
- Colunas usadas (nomes normalizados sem acento/caixa):
  - `N.º de venda` → ID da venda
  - `Unidades` → quantidade vendida (atenção: o nome "Unidades" aparece em mais de uma coluna;
    `pickCol` pega a primeira ocorrência a partir do cabeçalho detectado)
  - `SKU` → chave do cruzamento de CMV
  - `Título do anúncio` → nome do produto / fallback do cruzamento
  - `Receita por produtos (BRL)` → faturamento bruto
  - `Tarifa de venda e impostos (BRL)` → comissão (vem NEGATIVO, usar Math.abs)
  - `Tarifas de envio (BRL)` → gastos com frete (vem NEGATIVO, usar Math.abs)
  - `Total (BRL)` → repasse informado pela ML (usado só como referência/conferência)
  - `Reclamação aberta` (Sim/Não), `Reclamação encerrada` (número), `Em mediação` (Sim/Não)

### O que NÃO existe no relatório de vendas (por isso é manual)

- **Ads**: a coluna `Venda por publicidade` é só uma marca Sim/Não, sem valor em R$.
  O investimento real está no relatório do Mercado Ads.
- **Custo Fulls**: não está nesse relatório (vem no faturamento de tarifas).
- **Frete reverso**: não vem separado.
- **Outras taxas**: não existe.

### Divergência conhecida (pendente de análise do cliente)

O repasse calculado pelo app difere do `Total (BRL)` declarado pela ML porque três colunas do
relatório não têm lugar no DRE atual. Na amostra:

- Cancelamentos e reembolsos: −53.164,27
- Descontos e bônus: +21.284,24
- Receita por envio: +12.584,05
- (Acréscimo no preço e taxa de parcelamento se anulam: +10.416,10 / −10.416,10)

Soma dessas três = exatamente a diferença (~19,3 mil). O cliente vai decidir se e como
incorporá-las. **Não implementar por conta própria.**

Também: o relatório é por **data da venda**, e existe a coluna `Mês de faturamento das suas
tarifas` — uma tarifa de venda de 30/06 pode ser cobrada em julho. Relevante se o fechamento
for por regime de caixa.

## Funcionalidades principais

### Upload único (topo, "Comece por aqui") — 2 passos

- **Passo 1 — Relatório de vendas.** Enviado UMA vez, distribuído por `distribuirRelatorio()`
  para todas as análises: preenche faturamento/comissão/frete, roda a análise de reclamações e
  extrai a quantidade vendida por produto. Etiquetas ("feeds") mostram para onde o dado foi.
- **Passo 2 — Produtos e custos.** Fica **travado** até o passo 1 existir. Dois caminhos:
  (a) enviar planilha própria, ou (b) **`gerarModeloCustos()`** — gera um `.xlsx` com todos os
  produtos vendidos (SKU, Produto, Qtd vendida, Custo unitário em branco) para o mentorado
  preencher e reenviar. Esse botão existe porque a maioria não tem planilha de custos pronta.
- Cada documento tem botão **✕ Remover** (`removerRelatorio()` / `removerCustos()`), que limpa
  tudo derivado dele — mas **preserva o que foi digitado à mão** (ver `limparAuto()`).

### Campos automáticos

Campos preenchidos por arquivo recebem a classe `.auto` (fundo destacado + etiqueta "auto") e
continuam editáveis. `limparAuto(field)` só limpa se o campo ainda estiver marcado como auto.

### Análise de reclamações

Uma venda entra na lista se: `Reclamação aberta = Sim` OU `Reclamação encerrada >= 1` OU
`Em mediação = Sim`. Status é composto (ex.: "Aberta / Encerrada"). Saída: tabela com resumo,
busca, filtros por tipo, "Copiar lista" (texto no formato `Venda [ID] — [Título] — Status da
reclamação: [status]` + total) e export Excel.

> Pendência: hoje as **encerradas entram** na lista. O cliente pode querer só abertas/mediação.

### CMV por cruzamento

`CMV = Σ (quantidade vendida × custo unitário)`, produto a produto.
Cruza pelo **SKU** (maiúsculo); se o produto não tiver SKU, cai para o **título normalizado**.
Ignora linhas "TOTAL/SOMA" do rodapé das planilhas. Produtos vendidos **sem custo cadastrado**
são listados num aviso e ficam de fora do total.

### Linhas flexíveis

- Toda linha de custo (padrão ou adicionada) pode ser **excluída** pelo `×`.
  `removeField()` esconde a linha e a exclui dos totais, **mas preserva o valor**;
  "↩ Restaurar linhas removidas" desfaz. Estado salvo junto com o fechamento.
- **"+ Adicionar custo"** em Deduções, Custos fixos e Impostos cria linhas personalizadas
  (nome + valor + excluir), que entram nos totais, no salvamento e no export.
- A linha "Alíquota de impostos (%)" não tem `×` de propósito (é taxa, não custo).

### Salvamento e export

- Salvar por **Cliente + Competência** em localStorage (sobrescreve se a dupla já existir).
- Export **Excel** do DRE completo (inclui custos personalizados nas seções certas).
- Export Excel só das reclamações.
- Botões de PDF/Backup/Importar foram **removidos a pedido do cliente** — não readicionar.

## Armadilhas conhecidas (aprendidas na marra)

- **`[hidden]` vs `display:flex`** — vários elementos são mostrados/escondidos pelo atributo
  `hidden`, mas o CSS de classe (`display:flex`) sobrepõe a regra padrão do navegador e o
  elemento continuava visível. Existe a regra global `[hidden]{display:none!important}` no topo
  do CSS. **Não remover.** Testes em jsdom NÃO reproduzem esse bug (jsdom reporta `display:none`
  incorretamente) — validar visualmente no navegador.
- Colunas do relatório da ML **por nome, nunca por índice** (a ML muda a ordem).
- IDs de venda são números longos — tratar como **string** (não usar parseInt, perde precisão).

## Identidade visual

Tema escuro com verde-neon, extraído do logo do cliente (um "F" wireframe neon).
Cor de destaque `--accent: #3EE0A4`, fundo `--bg: #0a0e0d`. O logo está **embutido em base64**
no badge do cabeçalho (mantém o arquivo self-contained — não externalizar).
Modo de impressão força preto no branco.

## Deploy

Site estático. `index.html` na raiz é tudo que precisa ir para o ar.
Netlify/Vercel/GitHub Pages servem sem configuração.

## Estado atual / próximos passos possíveis

- [ ] Cliente vai conferir os valores automáticos (faturamento/comissão/frete) contra a conta real.
- [ ] Decidir o que fazer com cancelamentos, descontos e receita por envio (ver divergência acima).
- [ ] Calibrar as faixas do termômetro de margem.
- [ ] Possível: margem de contribuição por produto (o relatório já tem tarifas por venda,
      dá para cruzar com custo e chegar em margem por SKU).
- [ ] Possível: excluir vendas canceladas do CMV.
