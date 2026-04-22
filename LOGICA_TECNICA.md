# 🔧 Lógica Técnica — Conciliação de Antecipações

> Documento técnico explicando como funciona o processamento dos dados,
> o casamento entre Realizado e Retorno, o saldo acumulado e a exibição mês a mês.

---

## 1. Visão Geral do Fluxo

```
CSVs na pasta
     ↓
Leitura + Detecção de Layout
     ↓
Validação de anomes
     ↓
Reindexação do Retorno (M+1 → M)
     ↓
Absorção de Retornos Órfãos
     ↓
Cálculo do Saldo Acumulado por CPF
     ↓
Compressão gzip+base64 → HTML
```

---

## 2. Leitura dos Arquivos

O script detecta **tipo** (realizado/retorno) e **empresa** automaticamente pelo nome do arquivo:

```python
stem = Path(f.name).stem.lower()     # "realizado_gdf"
p    = stem.split('_', 1)
tipo    = p[0]                        # "realizado"
empresa = p[1].upper()               # "GDF"
```

### Layouts suportados

Cada convênio pode ter colunas diferentes. O script normaliza tudo:

```python
# REALIZADO — Padrão (GDF, RJ)
cpf    = row['cpf']
anomes = row['anomes']                # já no formato YYYYMM
valor  = row['valor_antecipado']
qtd    = row['qtd_antecipacoes']

# REALIZADO — Maranhão (data ISO)
cpf    = row['cpf']
anomes = row['data_antecipacao'][:4] + row['data_antecipacao'][5:7]
#        "2025-09-10T19:30:40.760Z" → "202509"
valor  = row['request_value']
qtd    = 1  # uma linha = uma antecipação

# RETORNO — Padrão (GDF)
cpf      = row['cpf']
matricula = row['matricula']
desconto = row['desconto']
anomes   = row['anomes']

# RETORNO — Layout novo (RJ, Maranhão)
cpf      = row['document_number']
matricula = row['register_number']
desconto = row['discount']
anomes   = row['year_month']
```

---

## 3. Validação de anomes

Antes de processar qualquer linha, valida se o campo `anomes` é uma data real:

```python
def anomes_valido(v):
    if not v or len(v) != 6 or not v.isdigit():
        return False
    ano, mes = int(v[:4]), int(v[4:])
    return 2020 <= ano <= 2030 and 1 <= mes <= 12
```

**Por que isso importa?**
O arquivo RJ chegou com dados corrompidos onde o valor do desconto caía na coluna `anomes`
(ex: `desconto=766.64` → `anomes=76664`). A validação descarta essas linhas automaticamente.

---

## 4. ⭐ O Casamento: Reindexação do Retorno (M+1 → M)

Esta é a parte mais importante. O **Realizado** e o **Retorno** vivem em meses diferentes:

```
Realizado de JANEIRO (202501) → Retorno chega em FEVEREIRO (202502)
```

Se não corrigirmos isso, o realizado de jan/2025 e o retorno de jan/2025
ficam em linhas separadas e o saldo nunca fecha.

**A solução:** ao ler cada linha de retorno, calculamos o mês ao qual ela pertence
subtraindo 1 mês do `anomes_folha`:

```python
def mes_anterior(anomes):
    ano, mes = int(anomes[:4]), int(anomes[4:])
    mes -= 1
    if mes < 1:      # janeiro → dezembro do ano anterior
        mes = 12
        ano -= 1
    return f'{ano}{mes:02d}'

# Exemplo:
# anomes_folha = "202502" (fevereiro — mês que veio no arquivo de retorno)
# anomes_ref   = "202501" (janeiro — mês do realizado correspondente)
anomes_ref = mes_anterior(anomes_folha)
```

**Resultado:** o retorno de fevereiro é indexado como janeiro, ficando na mesma linha
do realizado de janeiro. A coluna **Mês Desconto** guarda o mês original (fevereiro)
para rastreabilidade.

---

## 5. ⭐ Absorção de Retornos Órfãos

**Problema:** a base histórica começa em out/2024. Mas alguns retornos chegaram em out/2024
referentes a set/2024 (que não existe na base). Ao subtrair 1 mês, esses retornos apontariam
para set/2024 — um mês inexistente para aquele CPF.

**Detecção:**

```python
# Para cada CPF, saber qual é o primeiro mês de realizado
primeiro_real = {}
for (emp, cpf, anomes) in realizados:
    k = (emp, cpf)
    if k not in primeiro_real or anomes < primeiro_real[k]:
        primeiro_real[k] = anomes

# Ao indexar o retorno:
anomes_ref = mes_anterior(anomes_folha)
primeiro   = primeiro_real.get((empresa, cpf))

if primeiro is None:
    # CPF só tem retorno, sem nenhum realizado
    # → não desloca, usa o mês do próprio arquivo
    dest = anomes_folha

elif anomes_ref < primeiro:
    # ÓRFÃO: o retorno referencia um mês antes da base
    # → absorve no primeiro mês de realizado do CPF
    dest = primeiro

else:
    # CASO NORMAL
    dest = anomes_ref
```

**Exemplo prático:**
```
CPF 62068423120
  Primeiro realizado: 202410
  Retorno no arquivo: anomes=202410 → anomes_ref=202409

  202409 < 202410 → ÓRFÃO → absorve em 202410

  Resultado na linha 202410:
    Realizado: R$ 2.061,12
    Retorno:   R$ 2.061,12  ← inclui o retorno órfão
    Saldo:     R$ 0,00      ← quitado
```

Sem essa absorção, o retorno órfão ficaria "perdido" e o saldo nunca fecharia.

---

## 6. ⭐ Saldo Acumulado Mês a Mês por CPF

Depois de indexar todos os retornos corretamente, calculamos o saldo.
**O saldo carrega entre os meses** — não é calculado por linha isolada.

```python
# Para cada CPF+empresa, ordenar os meses e acumular
saldo = 0.0

for anomes in sorted(todos_os_meses_do_cpf):
    realizado = realizados.get((empresa, cpf, anomes), 0)
    retorno   = sum(r['desconto'] for r in retornos_do_mes)

    saldo = round(saldo + realizado - retorno, 2)
    # saldo > 0 → Dívida (falta retornar)
    # saldo < 0 → Crédito (retornou a mais)
    # saldo = 0 → Quitado
```

**Exemplo:**
```
CPF 00123456789

  Mês       Realizado   Retorno   Saldo Mês   Saldo Acum
  202501    1.000,00    800,00    +200,00      +200,00  → Dívida
  202502      500,00    500,00       0,00      +200,00  → Dívida (carrega)
  202503    1.000,00    750,00    +250,00      +450,00  → Dívida
  202504      800,00  1.250,00    -450,00         0,00  → Quitado ✅
```

O saldo do mês 202502 não é zero — ele carrega os R$ 200 da dívida anterior.
Só zera quando o total acumulado fecha.

---

## 7. CPFs Só-Retorno

CPFs que têm retorno mas nenhum realizado na base.
Esses registros **não deslocam o mês** (ficam no `anomes_folha` original):

```python
if primeiro_real.get((empresa, cpf)) is None:
    dest = anomes_folha  # não subtrai 1 mês
```

**Por que?** Sem um realizado para ancorar, subtrair 1 mês criaria uma linha fantasma
em um mês fora da base (ex: apareceria set/2024 quando a base começa em out/2024).

---

## 8. Visão Sumária (sem filtro de CPF)

Quando nenhum CPF está filtrado, o HTML consolida tudo por mês:

```javascript
// Agrupar todos os registros filtrados por mês
// (sem distinção de convênio)
const groups = {};
for (const r of records) {
    const key = r.anomes;
    if (!groups[key]) groups[key] = { anomes: key, real: 0, ret: 0, qtd: 0 };
    groups[key].real += r.realizado;
    groups[key].ret  += r.retorno;
    groups[key].qtd  += r.qtd;
}

// Recalcular saldo acumulado consolidado, mês a mês
let saldo = 0;
const rows = Object.values(groups)
    .sort((a, b) => a.anomes < b.anomes ? -1 : 1)
    .map(row => {
        saldo = Math.round((saldo + row.real - row.ret) * 100) / 100;
        return { ...row, saldo };
    });
```

**Por que recalcular o saldo aqui?**
O saldo nos dados originais é calculado por CPF individualmente.
No sumário, precisamos do saldo do **total consolidado** — que é diferente
da soma dos saldos individuais.

---

## 9. Múltiplas Antecipações por CPF+Mês (Maranhão)

O Maranhão tem uma linha por antecipação, então o mesmo CPF pode aparecer
várias vezes no mesmo mês. O script soma automaticamente:

```python
key = (empresa, cpf, anomes)
if key not in realizados:
    realizados[key] = {'val': 0.0, 'qtd': 0}
realizados[key]['val'] += valor   # soma os valores
realizados[key]['qtd'] += 1       # conta as antecipações
```

---

## 10. Compressão dos Dados no HTML

Os dados são comprimidos antes de embutir no HTML para reduzir o tamanho:

```python
import json, gzip, base64

# Serializar
json_str   = json.dumps(dados, ensure_ascii=False)

# Comprimir (redução de ~87%)
compressed = gzip.compress(json_str.encode('utf-8'), compresslevel=9)

# Codificar para texto (seguro para embutir em JS)
b64_data   = base64.b64encode(compressed).decode('ascii')

# No HTML: const DATA_B64 = "H4sIAB4j4W..."
```

No browser, o JavaScript descomprime na hora:

```javascript
async function decompress(b64) {
    const bytes = Uint8Array.from(atob(b64), c => c.charCodeAt(0));
    const ds    = new DecompressionStream('gzip');
    // ... lê os chunks e retorna o JSON original
}
```

---

## 11. Resumo dos Pontos Críticos

| # | Ponto | Erro comum sem o tratamento |
|---|---|---|
| 1 | **Reindexação M+1→M** | Realizado e retorno ficam em linhas separadas, saldo nunca fecha |
| 2 | **Absorção de órfãos** | Retornos "perdidos" fora da base, R$ 932k fora do saldo |
| 3 | **Saldo acumulado** | Calcular por linha isolada em vez de acumular entre meses |
| 4 | **CPF só-retorno sem deslocar** | Linha fantasma em mês inexistente (set/2024) |
| 5 | **Validação de anomes** | Dados corrompidos causam crash ou valores errados |
| 6 | **Soma por CPF+mês** | Maranhão tem N linhas por CPF — somar, não sobrescrever |

---

## 12. Script Completo de Reprocessamento

Para regenerar o `conciliacao.html` com novos CSVs:

```python
import csv, json, gzip, base64, re
from collections import defaultdict
from pathlib import Path

PASTA = '/Users/ramon.fsilva/Desktop/Conciliacao'

def mes_anterior(anomes):
    ano, mes = int(anomes[:4]), int(anomes[4:])
    mes -= 1
    if mes < 1: mes, ano = 12, ano - 1
    return f'{ano}{mes:02d}'

def anomes_valido(v):
    if not v or len(v) != 6 or not v.isdigit(): return False
    ano, mes = int(v[:4]), int(v[4:])
    return 2020 <= ano <= 2030 and 1 <= mes <= 12

def iso_to_anomes(dt):
    try: return dt[:4] + dt[5:7]
    except: return ''

realizados = {}
retornos_raw = []
empresas = set()

for f in Path(PASTA).glob('*.csv'):
    stem = Path(f.name).stem.lower()
    p = stem.split('_', 1)
    tipo, empresa = p[0], p[1].upper() if len(p) > 1 else stem.upper()
    empresas.add(empresa)

    with open(f, newline='', encoding='utf-8-sig') as fh:
        rows = list(csv.DictReader(fh))

    if tipo == 'realizado':
        for row in rows:
            cpf = row.get('cpf','').strip().zfill(11)
            if 'data_antecipacao' in row:
                anomes = iso_to_anomes(row.get('data_antecipacao',''))
                valor  = float(row.get('request_value', 0) or 0)
                qtd    = 1
            else:
                anomes = row.get('anomes','').strip()
                valor  = float(row.get('valor_antecipado', 0) or 0)
                qtd    = int(row.get('qtd_antecipacoes', 0) or 0)
            if not anomes_valido(anomes): continue
            key = (empresa, cpf, anomes)
            if key not in realizados:
                realizados[key] = {'val': 0.0, 'qtd': 0}
            realizados[key]['val'] += valor
            realizados[key]['qtd'] += qtd

    elif tipo == 'retorno':
        for row in rows:
            if 'document_number' in row:
                cpf  = row.get('document_number','').strip().zfill(11)
                mat  = row.get('register_number','').strip()
                desc = float(row.get('discount', 0) or 0)
                anomes = row.get('year_month','').strip()
            else:
                cpf  = row.get('cpf','').strip().zfill(11)
                mat  = row.get('matricula','').strip()
                desc = float(row.get('desconto', 0) or 0)
                anomes = row.get('anomes','').strip()
            if not anomes_valido(anomes): continue
            retornos_raw.append({
                'empresa': empresa, 'cpf': cpf,
                'matricula': mat, 'desconto': desc,
                'anomes_folha': anomes,
            })

# Primeiro mês de realizado por (empresa, cpf)
primeiro_real = {}
for (emp, cpf, anomes) in realizados:
    k = (emp, cpf)
    if k not in primeiro_real or anomes < primeiro_real[k]:
        primeiro_real[k] = anomes

# Indexar retornos com absorção de órfãos
ret_idx = defaultdict(list)
for r in retornos_raw:
    anomes_ref = mes_anterior(r['anomes_folha'])
    k = (r['empresa'], r['cpf'])
    primeiro = primeiro_real.get(k)
    if primeiro is None:
        dest = r['anomes_folha']
    elif anomes_ref < primeiro:
        dest = primeiro
    else:
        dest = anomes_ref
    ret_idx[(r['empresa'], r['cpf'], dest)].append(r)

# Montar extrato com saldo acumulado
todos_keys = set()
for (emp, cpf, anomes) in realizados:    todos_keys.add((emp, cpf, anomes))
for (emp, cpf, anomes) in ret_idx:      todos_keys.add((emp, cpf, anomes))

por_cpf = defaultdict(list)
for (emp, cpf, anomes) in todos_keys:
    por_cpf[(emp, cpf)].append(anomes)

registros = []
for (emp, cpf), meses in por_cpf.items():
    saldo = 0.0
    for anomes in sorted(set(meses)):
        real_row    = realizados.get((emp, cpf, anomes))
        ret_rows    = ret_idx.get((emp, cpf, anomes), [])
        realizado   = real_row['val'] if real_row else 0.0
        qtd         = real_row['qtd'] if real_row else 0
        retorno     = sum(r['desconto'] for r in ret_rows)
        matriculas  = list({r['matricula'] for r in ret_rows
                            if r['matricula'] not in ('', '0000000000', '00000000000')})
        meses_folha = sorted({r['anomes_folha'] for r in ret_rows})
        saldo = round(saldo + realizado - retorno, 2)
        registros.append({
            'empresa': emp, 'cpf': cpf, 'anomes': anomes,
            'matriculas': matriculas, 'qtd': qtd,
            'realizado': round(realizado, 2),
            'retorno':   round(retorno, 2),
            'meses_folha': meses_folha,
            'saldo': saldo,
        })

# Comprimir e injetar no HTML
out = {'empresas': sorted(empresas), 'registros': registros, 'total': len(registros)}
compressed = gzip.compress(json.dumps(out, ensure_ascii=False).encode('utf-8'), compresslevel=9)
b64_data   = base64.b64encode(compressed).decode('ascii')

html_path = Path(PASTA) / 'conciliacao.html'
with open(html_path) as f:
    html = f.read()
html = re.sub(r'const DATA_B64 = "[^"]+";', f'const DATA_B64 = "{b64_data}";', html)
with open(html_path, 'w') as f:
    f.write(html)

print(f"✅ {len(registros)} registros processados")
print(f"✅ HTML atualizado: {html_path}")
```

Para rodar: `python3 reprocessar.py` (salvar o script acima como `reprocessar.py` na pasta)

---

*Documentação gerada com HubAI Nitro — Ramon Fsilva*
