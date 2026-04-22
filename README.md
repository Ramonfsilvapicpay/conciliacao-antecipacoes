# ⚡ Conciliação de Antecipações — PicPay

> Sistema de conciliação financeira para confrontar antecipações de salário com retornos em folha de pagamento de múltiplos convênios.

---

## 🔗 Acesso Online

**Link público:** [https://lucky-lebkuchen-86bca2.netlify.app/](https://lucky-lebkuchen-86bca2.netlify.app/)

> Para link permanente: criar conta gratuita em [netlify.com](https://app.netlify.com/signup) e republicar via drag & drop.

---

## 📁 Estrutura da Pasta

```
Conciliacao/
├── Realizado_GDF.csv        → antecipações GDF (132.242 registros)
├── Retorno_GDF.csv          → descontos em folha GDF (127.636 registros)
├── Realizado_RJ.csv         → antecipações RJ (111.986 registros)
├── Retorno_RJ.csv           → descontos em folha RJ (111.007 registros)
├── Realizado_Maranhao.csv   → antecipações Maranhão (81.003 registros)
├── Retorno_Maranhao.csv     → descontos em folha Maranhão (18.064 registros)
├── conciliacao.html         → entregável final — abrir no browser
└── README.md                → este arquivo
```

---

## 📂 Padrão de Arquivos

```
tipo_nomeempresa.csv
```

| Exemplo | Tipo | Empresa |
|---|---|---|
| `realizado_gdf.csv` | Realizado | GDF |
| `retorno_gdf.csv` | Retorno | GDF |
| `realizado_rj.csv` | Realizado | RJ |
| `retorno_rj.csv` | Retorno | RJ |
| `realizado_maranhao.csv` | Realizado | MARANHAO |
| `retorno_maranhao.csv` | Retorno | MARANHAO |

> O sistema detecta tipo e empresa **automaticamente** pelo nome do arquivo.
> Para adicionar um novo convênio, basta jogar os CSVs na pasta e reprocessar.

---

## 📋 Layouts de CSV Suportados

### Realizado — Layout padrão (GDF, RJ)

| Coluna | Descrição |
|---|---|
| `cpf` | CPF do beneficiário |
| `anomes` | Mês de referência — formato `YYYYMM` |
| `valor_antecipado` | Valor antecipado no mês |
| `qtd_antecipacoes` | Quantidade de antecipações |

### Realizado — Layout Maranhão

| Coluna | Descrição |
|---|---|
| `cpf` | CPF do beneficiário |
| `request_value` | Valor antecipado |
| `data_antecipacao` | Data ISO (`2025-09-10T19:30:40.760Z`) → convertida para `YYYYMM` |

### Retorno — Layout padrão (GDF)

| Coluna | Descrição |
|---|---|
| `cpf` | CPF do beneficiário |
| `matricula` | Matrícula funcional |
| `desconto` | Valor descontado em folha |
| `anomes` | Mês de referência — formato `YYYYMM` |

### Retorno — Layout novo (RJ, Maranhão)

| Coluna | Descrição |
|---|---|
| `document_number` | CPF do beneficiário |
| `register_number` | Matrícula funcional |
| `discount` | Valor descontado em folha |
| `year_month` | Mês de referência — formato `YYYYMM` |

> ⚠️ Linhas com `anomes`/`year_month` fora do formato `YYYYMM` (ano 2020–2030, mês 01–12) são descartadas automaticamente.

---

## 🧮 Lógica de Conciliação

### Descasamento de meses

```
Realizado em Janeiro (202501) → Retorno chega em Fevereiro (202502)
```

O retorno de `202502` é automaticamente vinculado à linha `202501`.
A coluna **Mês Desconto** exibe o mês real do arquivo para rastreabilidade.

### Saldo Acumulado

```
Saldo(M) = Saldo(M-1) + Realizado(M) - Retorno(M)
```

| Saldo | Cor | Significado |
|---|---|---|
| 🔴 Positivo | Vermelho | **Dívida** — falta retornar |
| 🟢 Negativo | Verde | **Crédito** — retornou a mais |
| ⚪ Zero | Neutro | **Quitado** |

---

## ⚠️ Casos Especiais

### 1. Retornos Órfãos
Retorno cujo mês de referência (M-1) é anterior ao primeiro realizado do CPF.

**Causa:** limite da base histórica — retorno chegou no 1º mês do cliente, mas o realizado correspondente é anterior à base.

**Tratamento:** absorvido no **1º mês de realizado** do cliente. Mês Desconto preserva o mês real.

| Mês da Folha | Casos |
|---|---|
| 202410 | 807 |
| 202411 | 469 |
| 202501 | 162 |
| **Total** | **1.438 — R$ 932.284,67** |

### 2. CPFs Só-Retorno (26 casos GDF)

| Categoria | Qtd | Descrição |
|---|---|---|
| ✅ Zerados | 19 | Descontados e estornados em lote — jul/2025 |
| 🟢 Crédito | 3 | Só estorno negativo |
| 🔴 Dívida | 4 | Desconto sem realizado — verificar na origem |

> Todos os 29 registros negativos têm `matricula=0000000000` e `anomes=202507` — estorno em lote do GDF.

### 3. Múltiplas Matrículas
Mesmo CPF com mais de um vínculo funcional no mesmo mês.
Tratamento: soma os descontos e exibe todas as matrículas na célula.

### 4. Maranhão — Retorno Pendente
Realizado vai até `202604`, retorno disponível apenas até `202603`.
O mês `202604` aparece com dívida — será regularizado quando o arquivo de retorno chegar.

---

## 🖥️ Como usar o HTML

Abra `conciliacao.html` em qualquer browser moderno.
**Não precisa de internet, servidor ou instalação.**

### Filtros

| Filtro | Comportamento |
|---|---|
| CPF | Com ou sem formatação. Enter ou botão Filtrar |
| Convênio | Filtra por empresa específica |
| Mês Antecipação | Filtra por período |
| ✕ Limpar | Remove todos os filtros — volta ao consolidado total |

### Modos de visualização

| Modo | Quando ativa | O que mostra |
|---|---|---|
| 📊 **Sumário** | Sem filtro de CPF | Uma linha por mês — todos os convênios somados. Todos os meses visíveis sem paginação. |
| 🔍 **Detalhe** | Com filtro de CPF | Extrato linha a linha por CPF, ordenado por mês ASC |

### Gráfico de colunas mensais

- Duas barras por mês: 🟢 **Realizado** e 🔵 **Retorno**
- Valor abreviado acima de cada barra (ex: `5.3M`)
- **Tooltip** ao passar o mouse → valor completo com centavos (`R$ 5.325.830,36`)
- **Variação %** abaixo do mês: `retorno ÷ realizado` — 🟢 positivo / 🔴 negativo
- Reage automaticamente a todos os filtros

### Colunas da tabela

| Coluna | Descrição |
|---|---|
| Convênio | Empresa detectada pelo nome do arquivo |
| CPF | CPF formatado |
| Matrícula(s) | Matrícula(s) funcional(is) |
| Mês Antecipação | Período de referência do realizado |
| Qtd Ant. | Quantidade de antecipações no mês |
| Realizado | Valor antecipado no mês |
| Retorno (Folha M+1) | Valor descontado em folha |
| Mês Desconto | Mês real do arquivo de retorno |
| Saldo Acumulado | Posição acumulada até aquele mês |
| Status | Quitado / Dívida / Crédito |

---

## 🔄 Como atualizar os dados

1. Substituir ou adicionar CSVs na pasta `Conciliacao/` (padrão `tipo_empresa.csv`)
2. Pedir ao **HubAI Nitro (Paciente 0 / Zero)** para reprocessar
3. Republicar o HTML via **Netlify** — mesmo link, dados atualizados

---

## 📊 Base atual

| Convênio | Realizado | Retornado | Registros | Período |
|---|---|---|---|---|
| GDF | R$ 191.087.714,11 | R$ 186.781.982,99 | ~135k | Out/24 → Mar/26 |
| RJ | R$ 177.898.432,63 | R$ 175.657.062,42 | ~117k | Mai/25 → Jun/26 |
| MARANHÃO | R$ 16.943.885,74 | R$ 12.289.238,00 ⚠️ | ~25k | Jul/25 → Abr/26 |
| **Total** | **R$ 385.930.032,48** | **R$ 374.728.283,41** | **276.803** | Out/24 → Abr/26 |

> ⚠️ Maranhão: retorno de `202604` ainda pendente — aguardando arquivo.

---

## 🎨 Identidade Visual

Layout padrão PicPay:
- Verde principal `#21C25E` + teal `#00D4AA`
- Fundo escuro `#0D0D0D` / `#161616`
- Logo PicPay no header
- Cards com faixa gradiente no topo
- Tipografia Inter

---

*Gerado com HubAI Nitro — Ramon Fsilva*
