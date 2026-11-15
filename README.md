# Análise de Boletins de Urna (BUWeb) — Agente Auditor MCP (Eleições RN)

Agente inteligente de auditoria eleitoral em linguagem natural construído sobre os dados oficiais de **Boletins de Urna (BUWeb)** do Tribunal Superior Eleitoral (TSE) para o estado do Rio Grande do Norte (RN).

O projeto adota o protocolo aberto **MCP (Model Context Protocol)** através da biblioteca **FastMCP**, separando a inteligência da consulta da execução matemática com **pandas**. Um modelo local via **Ollama** atua como agente ReAct (Reasoning + Acting), orquestrando chamadas de ferramentas e respondendo em português sem inventar dados (*alucinações*).

---

## 🏛️ Capacidades do Agente

O agente consulta e audita o CSV oficial do TSE respondendo a perguntas sobre:

- **Votos para Governador e Presidente** (2º Turno);
- **Totais por tipo de voto**: nominais, brancos e nulos;
- **Auditoria de zonas e seções eleitorais** (identificação de líderes de votos ou anomalias);
- **Eleitorado**: total de aptos, comparecimento e abstenções;
- **Biometria**: eleitores sem biometria válida por seção, zona ou município;
- **Comparação direta entre seções eleitorais**.

O modelo de linguagem **não calcula valores por conta própria** nem acessa a internet. Ele descobre as ferramentas publicadas pelo servidor MCP e submete as consultas ao banco de dados em memória.

---

## 📁 Estrutura do Projeto

```text
pj_eleicao_MCP/
├── data/
│   └── bweb_2t_RN_261020260742.csv     # CSV oficial do BUWeb (TSE - 2º Turno RN)
├── notebooks/
│   ├── Pj_eleicao_MCP.ipynb            # Notebook experimental (Colab / Jupyter)
│   └── pj_eleicao_MCP.pdf              # Exportação em PDF do notebook executado
├── src/
│   └── bu_agent/                       # Pacote do agente e servidor MCP
│       ├── __init__.py                 # Ponto de entrada e exports do pacote
│       ├── agent.py                    # Host MCP e loop de decisão ReAct
│       ├── cli.py                      # Interface interativa de linha de comando
│       ├── config.py                   # Configurações de caminhos e modelo Ollama
│       ├── data.py                     # Carregamento e normalização do CSV via pandas
│       ├── mcp_server.py               # Servidor FastMCP (comunicação via stdio)
│       ├── prompt.py                   # System prompt do auditor eleitoral
│       ├── prompts.py                  # Alias de compatibilidade
│       ├── summary.py                  # Formatador de respostas amigáveis (pt-BR)
│       ├── tools.py                    # Implementação auditada das 5 ferramentas
│       └── tools_schema.py             # Schemas JSON das ferramentas para a LLM
├── tests/
│   ├── conftest.py                     # Inclusão do diretório src no sys.path
│   └── test_tools.py                   # Suíte com 13 testes unitários (pytest)
├── .vscode/
│   └── mcp.json                        # Configuração para iniciar o MCP no VS Code
├── codigo_gerado.sh                    # Script auxiliar para baixar o modelo no Ollama
├── LICENSE                             # Licença de uso
├── README.md                           # Documentação completa
└── requirement.txt                     # Dependências do projeto
```

### Arquivos principais

- <a href="https://github.com/arleswasb/Projeto_AI_Agents_II_MCP_eleicao/blob/main/notebooks/Pj_eleicao_MCP.ipynb"><img src="https://img.shields.io/badge/-Notebook-191A1B?style=flat-square&logo=jupyter" height="20" alt="Notebook" /></a> <a href="https://colab.research.google.com/github/arleswasb/Projeto_AI_Agents_II_MCP_eleicao/blob/main/notebooks/Pj_eleicao_MCP.ipynb"><img src="https://colab.research.google.com/assets/colab-badge.svg" height="20" alt="Open in Colab" /></a> [**`notebooks/Pj_eleicao_MCP.ipynb`**](./notebooks/Pj_eleicao_MCP.ipynb): Notebook interativo para desenvolvimento, experimentos no Google Colab e validação passo a passo do fluxo MCP e ReAct.
- [**`notebooks/pj_eleicao_MCP.pdf`**](./notebooks/pj_eleicao_MCP.pdf): Relatório completo gerado a partir do notebook executado.
- [**`data/bweb_2t_RN_261020260742.csv`**](./data/bweb_2t_RN_261020260742.csv): Arquivo de dados oficial preservado na íntegra.

---

## ⚙️ Arquitetura MCP (Model Context Protocol)

O projeto opera sob o padrão cliente/servidor MCP:

```text
┌─────────────────┐        stdio        ┌─────────────────────────┐
│     Agente      │ ◄─────────────────► │      Servidor MCP       │
│  (bu_agent.cli  │   JSON-RPC (MCP)    │ (bu_agent.mcp_server)   │
│   / Ollama LLM) │                     └────────────┬────────────┘
└─────────────────┘                                  │
                                                     ▼
                                        ┌─────────────────────────┐
                                        │  Gabarito com Pandas    │
                                        │ (data/bweb_...csv)      │
                                        └─────────────────────────┘
```

### 1. Servidor FastMCP (`src/bu_agent/mcp_server.py`)
Publica via transporte `stdio`:
- **5 Ferramentas de análise**;
- **1 Resource**: `eleicao-RN://schema` (documenta as colunas oficiais do BU);
- **1 Prompt**: `recomendar_analise` (orienta o usuário sobre perguntas relevantes).

### 2. Ferramentas Disponíveis

| Ferramenta | Parâmetros | Descrição |
|---|---|---|
| `search_secoes` | `tipo_voto`, `cargo`, `zona`, `min_votos`, `top_n` | Busca e ranqueia as seções com maior número de votos. |
| `secao_details` | `zona`, `secao` | Dados detalhados de uma seção: aptos, comparecimento, abstenções, biometria e votos por cargo. |
| `compare_secoes` | `zona_a`, `secao_a`, `zona_b`, `secao_b`, `tipo_voto` | Compara duas seções eleitorais e identifica qual teve maior votação. |
| `total_por_tipo` | `tipo_voto`, `cargo` | Total consolidado de votos no arquivo (Nominal, Branco ou Nulo) por cargo. |
| `total_sem_biometria` | `zona`, `municipio` | Total de eleitores aptos sem biometria válida (geral, por zona ou por cidade). |

---

## 🚀 Instalação e Configuração

### 1. Preparar o Ambiente Virtual

Requisitos:
- Python 3.10 ou superior;
- [Ollama](https://ollama.com/) instalado e em execução (para consultas com LLM local).

Clone o repositório e crie o ambiente virtual:

```bash
git clone https://github.com/arleswasb/Projeto_AI_Agents_II_MCP_eleicao.git
cd Projeto_AI_Agents_II_MCP_eleicao

python3 -m venv .venv
source .venv/bin/activate   # No Windows: .venv\Scripts\activate
pip install -r requirement.txt
```

Confirme que o arquivo de dados está presente em `data/`:

```bash
test -f data/bweb_2t_RN_261020260742.csv && echo "Dados encontrados com sucesso."
```

---

## 🧪 Validação e Testes

### 1. Executar a Suíte de Testes Automatizados

O projeto inclui 13 testes unitários que validam todas as 5 ferramentas e as regras de tratamento de erros com dados reais:

```bash
.venv/bin/pytest -v
```

Resultado esperado:
```text
13 passed in 1.xx s
```

### 2. Testar o Servidor MCP Standalone (sem LLM)

Você pode verificar as ferramentas, resource e prompt do servidor FastMCP diretamente via Python:

```bash
PYTHONPATH=src .venv/bin/python - <<'PY'
import asyncio
from pathlib import Path
from fastmcp import Client

async def main():
    async with Client(Path("src/bu_agent/mcp_server.py")) as client:
        tools = await client.list_tools()
        print("✅ Ferramentas MCP:", [tool.name for tool in tools])

        total = await client.call_tool(
            "total_por_tipo",
            {"tipo_voto": "Nulo", "cargo": "Presidente"},
        )
        print("✅ Retorno total_por_tipo:", total.data)

        schema = await client.read_resource("eleicao-RN://schema")
        print("✅ Resource eleicao-RN://schema:", bool(schema))

        prompt = await client.get_prompt(
            "recomendar_analise",
            {"cargo": "Presidente", "tipo_voto": "Nulo"},
        )
        print("✅ Prompt recomendar_analise:", bool(prompt.messages))

asyncio.run(main())
PY
```

Ou usando o utilitário CLI do FastMCP:

```bash
fastmcp list src/bu_agent/mcp_server.py --resources --prompts --input-schema
fastmcp call src/bu_agent/mcp_server.py total_por_tipo tipo_voto=Nulo cargo=Presidente
```

---

## 💬 Executando o Agente Interativo (CLI)

### 1. Iniciar o Ollama e baixar o modelo

O modelo padrão é `qwen2.5:7b` (ou a versão leve `qwen2.5:3b`):

```bash
# Inicie o serviço em segundo plano caso não esteja rodando:
ollama serve

# Baixe o modelo desejado:
ollama pull qwen2.5:7b
# ou para máquinas mais modestas:
ollama pull qwen2.5:3b
```

Para selecionar o modelo de 3 bilhões de parâmetros, defina a variável de ambiente:

```bash
export BU_MODEL=qwen2.5:3b
```

### 2. Iniciar a CLI do Agente

Execute:

```bash
PYTHONPATH=src .venv/bin/python -m bu_agent.cli
```

Exemplos de perguntas para testar no terminal:

```text
❓ Quantos votos nulos para presidente existem?
❓ Quais seções têm mais votos nulos para governador?
❓ Me dê os detalhes da seção 505 da zona 3.
❓ Compare os votos nominais da seção 505 com a 504 da zona 3.
❓ Quantos eleitores estão sem biometria válida no total?
❓ Quantos eleitores sem biometria existem no município de Natal?
```

O agente exibirá o ciclo ReAct detalhado:
```text
── volta 1 ──
🤖 o modelo pediu → total_por_tipo(cargo='Presidente', tipo_voto='Nulo')
🔧 nós executamos → 61.879 votos (Nulo para Presidente) em 1.543 linhas
── volta 2 ──
🤖 o modelo respondeu com texto → No 2º turno do RN foram registrados 61.879 votos nulos para Presidente.
```

Para encerrar o agente, digite `sair`, `exit` ou `q`.

---

## 💻 Integração com VS Code

O projeto conta com o arquivo [`.vscode/mcp.json`](./.vscode/mcp.json) pré-configurado.

Para iniciar o servidor MCP pelo VS Code:
1. Abra a pasta do projeto no VS Code;
2. Pressione `Ctrl+Shift+P` (ou `Cmd+Shift+P` no macOS);
3. Digite e selecione **MCP: List Servers**;
4. Localize `eleicao-rn` e selecione **Start**.

---

## 📓 Execução pelo Notebook

- <a href="https://github.com/arleswasb/Projeto_AI_Agents_II_MCP_eleicao/blob/main/notebooks/Pj_eleicao_MCP.ipynb"><img src="https://img.shields.io/badge/-Notebook-191A1B?style=flat-square&logo=jupyter" height="20" alt="Notebook" /></a> <a href="https://colab.research.google.com/github/arleswasb/Projeto_AI_Agents_II_MCP_eleicao/blob/main/notebooks/Pj_eleicao_MCP.ipynb"><img src="https://colab.research.google.com/assets/colab-badge.svg" height="20" alt="Open in Colab" /></a> [**`Pj_eleicao_MCP.ipynb`**](./notebooks/Pj_eleicao_MCP.ipynb) — Auditoria Eleitoral BUWeb RN (MCP + Agente ReAct)
  - **No Google Colab:** Clique na tag **Open in Colab** acima para abrir e executar diretamente na nuvem.
  - **Localmente:** Abra [`notebooks/Pj_eleicao_MCP.ipynb`](./notebooks/Pj_eleicao_MCP.ipynb) no VS Code ou Jupyter Lab com o ambiente virtual `.venv` ativo.
  - **Relatório:** Visualize o relatório gerado em [`notebooks/pj_eleicao_MCP.pdf`](./notebooks/pj_eleicao_MCP.pdf).

As células são organizadas linearmente:
- **Download e extração** do BUWeb direto do CDN do TSE;
- **Inspeção de dados** estruturais com pandas;
- **Verificação do Ollama** e diagnóstico de GPU (`nvidia-smi`);
- **Criação do servidor MCP** (`mcp_server.py`) e inspeção de schemas JSON-RPC;
- **Loop ReAct do agente** com bateria de perguntas e validações.

---

## 🛡️ Fonte dos Dados Oficiais

Os dados devem ser obtidos exclusivamente a partir do CDN oficial do TSE:
- [Repositório de Dados Eleitorais do TSE](https://dadosabertos.tse.jus.br/)
- Padrão de URL de download oficial:
  ```text
  https://cdn.tse.jus.br/estatistica/sead/eleicoes/eleicoes{ANO}/buweb/bweb_{TURNO}t_{UF}_{GERACAO_BU}.zip
  ```

---

## 📄 Licença

Este projeto está sob a licença [MIT](./LICENSE).
