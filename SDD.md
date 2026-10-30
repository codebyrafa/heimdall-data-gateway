# Documento de Arquitetura de Sistema (SDD) - Heimdall Data Gateway
## Visão Geral do Produto
O Heimdall Data Gateway atua como um "porteiro" (gatekeeper) para pipelines de engenharia de dados. A ferramenta valida arquivos brutos (CSV, JSON, Parquet) contra regras estritas definidas em contratos de dados (YAML) antes que sejam ingeridos em um Data Lake ou banco de dados. 

A solução operará de forma híbrida, oferecendo uma Interface de Linha de Comando (CLI) para automação em CI/CD e uma Interface Web interativa para usuários de negócios.

## Stack Tecnológica Base
A escolha das ferramentas prioriza alta performance, tipagem rigorosa e facilidade de integração em pipelines modernos:
- **Linguagem**: Python 3.10+ com tipagem estrita (Type Hints) e Docstrings.
- **Processamento de Dados**: polars (escolhido no lugar do Pandas pela velocidade superior de processamento e uso eficiente de memória).
- **Leitura de Contratos**: pyyaml para interpretar as regras declarativas.
- **Interface de Linha de Comando (CLI)**: typer (para gestão de argumentos com base em type hints) e rich (para formatação visual, tabelas e cores no terminal).
- **Interface Web**: streamlit para a criação do dashboard interativo de validação (SaaS-like) sem necessidade de código frontend externo.
- **Testes e Qualidade**: pytest para testes unitários e de integração.
- **Gerenciamento de Dependências**: poetry configurado via pyproject.toml.

## Arquitetura de Módulos Desacoplados
O sistema será dividido em três camadas principais para garantir isolamento de responsabilidades e reaproveitamento de código:

1. **Core (src/core/)**: Motor central da aplicação. Independe da interface gráfica ou do terminal.
    - **parser.py**: Lê e valida sintaticamente o arquivo de contrato YAML.
    - **reader.py**: Utiliza o Polars para carregar os arquivos CSV, JSON ou Parquet em dataframes.
    - **validator.py**: Compara o dataframe contra as regras do contrato, aplicando verificações de tipos, nulos, limites numéricos, unicidade e regex. Gera a lista padronizada de falhas.

2. **CLI (src/cli/)**: Consome o Core para execução em terminal.
    - **main.py**: Configura os comandos (ex: heimdall validate), gerencia os argumentos recebidos pelo usuário e formata a saída usando o Rich.

3. **Web (src/web/)**: Interface gráfica construída em Streamlit.
    - **app.py**: Disponibiliza uploaders de arquivos temporários, consome o Core e exibe os resultados em cartões de métricas, gráficos de barra para distribuição de erros e tabelas filtráveis para as linhas reprovadas.

## Estrutura de Diretórios do Repositório
Para refletir um padrão de engenharia de software profissional, o repositório deve seguir a topologia abaixo:
´´´
heimdall/
├── pyproject.toml              # Dependências unificadas via Poetry
├── README.md                   # Documentação detalhando uso da CLI e Web
├── src/
│   ├── __init__.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── parser.py           # Parsing do YAML
│   │   ├── reader.py           # Ingestão de dados nativa (Polars)
│   │   └── validator.py        # Motor de validação estrita
│   ├── cli/
│   │   ├── __init__.py
│   │   └── main.py             # App Typer e console Rich
│   └── web/
│       ├── __init__.py
│       └── app.py              # Interface Streamlit
├── examples/
│   ├── contratos/              # Mock de schemas YAML para testes rápidos
│   └── dados/                  # Mock de arquivos brutos (válidos e corrompidos)
└── tests/                      
    ├── test_cli.py             # Testes de integração (CliRunner)
    ├── test_reader.py          # Testes de ingestão de arquivos múltiplos
    └── test_validator.py       # Testes unitários de quebra de regras
´´´

## Fluxo de Execução (Data Flow)
1. **Entrada de Arquivos**: O usuário fornece o caminho local (via CLI) ou faz o upload (via Web) do arquivo de dados bruto e do arquivo YAML.
2. **Parsing do Contrato**: O parser.py valida se o YAML possui a estrutura base obrigatória (dataset, colunas, tipos esperados) antes de ler os dados massivos.
3. **Lazy Loading de Dados**: O reader.py tenta inferir o esquema inicial e carrega os dados via Polars.
4. **Inspeção de Qualidade**: O validator.py itera sobre as regras (nulos, boundaries numéricos, regex) utilizando operações vetorizadas do Polars para máxima performance.
5. **Veredito**: Retorna um booleano de is_valid e uma lista estruturada de erros identificando linha, coluna e violação para a interface correspondente (Terminal ou Dashboard Web). Se reprovado na CLI, emite exit code 1 para interromper pipelines de CI/CD.