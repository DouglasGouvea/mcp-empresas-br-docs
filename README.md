# Empresas BR — CNPJ due diligence MCP server

Remote MCP server for checking Brazilian companies by **CNPJ** using public data.
One call returns registry data, federal sanctions, federal contracts and CVM
(securities regulator) registration — each fact with its **source, status and
query date**.

Built for agents doing **KYC, compliance and due diligence** on Brazilian companies.

> Responses are in Portuguese (field names and values), as the source data is.

## Why

Scraping four government sources gives an agent a pile of data and no way to
tell *"clean record"* from *"source was down"*. This server makes that explicit:

- **Per-source status** (`ok`, `nao_encontrado`, `erro`, `timeout`, `limite_excedido`).
  An empty sanctions list with status `ok` means *nothing on record*. A failed
  source is never turned into an empty list.
- **Partial results instead of failures**: `relatorio_completo` runs the four
  checks in parallel; a section that fails comes back `null` and is listed in
  `secoes_com_falha`, with `completo: false`.
- **Facts only**: no risk score, no rating. Your agent interprets.
- **LGPD-safe**: never returns partners (QSA), emails or phone numbers. For sole
  proprietors / MEI, name and address are omitted (`dados_omitidos_lgpd: true`).
  Sanction records against individuals (CPF) are discarded.

## Tools

| Tool | What it returns | Source |
|---|---|---|
| `cadastro_empresa` | Legal name, status, opening date, legal nature, size, share capital, CNAE (main + secondary), city/state, address, Simples/MEI | Receita Federal (via BrasilAPI, with OpenCNPJ fallback) |
| `sancoes_empresa` | CEIS, CNEP, CEPIM and leniency agreements — sanctioning body, type, dates, legal basis, case number | Portal da Transparência (CGU) |
| `contratos_publicos` | Most recent federal contracts as supplier (`limite` 1–100): object, agency, dates, values | Portal da Transparência (CGU) — **federal Executive branch only** |
| `registro_cvm` | Whether it is a CVM-registered public company, registration status, category, sector, auditor | CVM open data (daily file) |
| `relatorio_completo` | All four above in one call, in parallel, with partial results | All of the above |

Input for every tool: `cnpj` (with or without punctuation; check digits are
validated before any source is called).

## Example

Real response from `relatorio_completo` for Petrobras (`33000167000101`),
trimmed — full version in [`examples/relatorio_completo-petrobras.json`](examples/relatorio_completo-petrobras.json):

```json
{
  "cnpj": "33000167000101",
  "consultado_em": "2026-10-04T09:39:08-03:00",
  "fontes": [
    {"nome": "BrasilAPI (Receita Federal)", "status": "ok", "de_cache": true, "idade_cache_s": 45573, "...": "..."},
    {"nome": "Portal da Transparência – CEIS", "status": "ok", "...": "..."},
    "... 5 more sources"
  ],
  "dados": {
    "cadastro": {
      "razao_social": "PETROLEO BRASILEIRO S A PETROBRAS",
      "situacao_cadastral": "ATIVA",
      "data_inicio_atividade": "1966-09-28",
      "natureza_juridica": "Sociedade de Economia Mista",
      "cnae_principal": {"codigo": "0600001", "descricao": "Extração de petróleo e gás natural"},
      "uf": "RJ",
      "dados_omitidos_lgpd": false,
      "...": "..."
    },
    "sancoes": {"ceis": [], "cnep": [], "cepim": [], "acordos_leniencia": [], "total_registros": 0},
    "contratos": {
      "escopo": "Poder Executivo Federal",
      "total_encontrado": 55,
      "contratos": [{"orgao": {"sigla": "UFRJ"}, "data_assinatura": "2018-11-27", "valor_final": 857706249.0, "...": "..."}]
    },
    "cvm": {
      "registrada": true,
      "registros": [{"codigo_cvm": "9512", "situacao": "ATIVO", "categoria_registro": "Categoria A", "auditor": {"nome": "KPMG AUDITORES INDEPENDENTES LTDA."}}]
    },
    "completo": true,
    "secoes_com_falha": []
  },
  "aviso": "Dados públicos ... Não é aconselhamento jurídico ou financeiro. ..."
}
```

## Connect

Endpoint (streamable HTTP): `https://mcp-empresas-br.mcpize.run/mcp`

1. Get an API key at **[mcpize.com/mcp/mcp-empresas-br](https://mcpize.com/mcp/mcp-empresas-br)**.
   The **free plan includes 100 calls/month**; Pro is US$ 9/month for 1,000 calls.
2. Send it as `Authorization: Bearer <API key>`.

Claude Code:

```bash
claude mcp add --transport http empresas-br https://mcp-empresas-br.mcpize.run/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

Generic MCP client config:

```json
{
  "mcpServers": {
    "empresas-br": {
      "type": "http",
      "url": "https://mcp-empresas-br.mcpize.run/mcp",
      "headers": {"Authorization": "Bearer YOUR_API_KEY"}
    }
  }
}
```

Also listed in the official MCP Registry as `io.github.DouglasGouvea/mcp-empresas-br`.

## Coverage and limits

- Sanctions and contracts cover the **federal level only**. No record here does
  not prove there is no record at state or municipal level.
- Contracts: federal Executive branch only (the PNCP API cannot filter by supplier CNPJ).
- Sanctions are matched on the same CNPJ root (head office and branches);
  records from another establishment are flagged `outro_estabelecimento: true`.
- Caching: registry 7 days, sanctions and contracts 24 h, CVM file reloaded daily.
  Every source shows `de_cache` and `idade_cache_s`.

## Data sources and attribution

- Receita Federal do Brasil — CNPJ open data, via [BrasilAPI](https://brasilapi.com.br) / [OpenCNPJ](https://opencnpj.org)
- [Portal da Transparência](https://portaldatransparencia.gov.br) (CGU) — open data under Decree 8.777/2016
- [CVM open data](https://dados.cvm.gov.br) — ODbL

Public data, no guarantee of completeness or timeliness. Not legal or financial advice.

## Contact

Issues and feature requests: [open an issue](../../issues).
X: [@DougManoel89](https://x.com/DougManoel89)

---

**Português:** servidor MCP de checagem de empresa por CNPJ — cadastro na Receita,
sanções federais (CEIS, CNEP, CEPIM, leniência), contratos com o Executivo Federal
e registro na CVM, com fonte, status e data de cada fato. Plano gratuito de 100
consultas/mês em [mcpize.com/mcp/mcp-empresas-br](https://mcpize.com/mcp/mcp-empresas-br).
