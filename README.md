
# 📓Descrição Pipeline de Download, Leitura, Filtragem e Publicação de Relatórios desenvolvido em linguagem Python (SFTP → Local → OneDrive)

LEANDRO DE SOUZA REIS SILVA
Graduado em Ciência de Dados
🪪 Identificação do Projeto 
- Nome do Projeto: Automatização do Pipeline de Dados do GED 
- Criado por: Leandro Souza Reis Silva 
- Data de criação: 12/03/2026 
- Plataforma: Liguagem Python, Jupyter Notebook, VSCode
- O projeto proporcionou um ganho de aproximadamente 80% na produtividade de coleta e tratamento de dados.
- Com foco em regras de negócio, o código em Python não foi inserido no repositório

## 🔎 Visão Geral
O projeto implementa um pipeline **profissional e simples** para:
1.  **Baixar** relatórios via **SFTP** (FileZilla Server) com reconexão e tolerância a falhas,
2.  **Tratar nomes dinâmicos** no servidor (sufixos com data/metadata),
3.  **Extrair** arquivos `.zip` e **ler** os `.csv` resultantes,
4.  **Filtrar** os dados conforme regras do negócio,
5.  **Exportar** resultados para a pasta local `./RelatorioFinal`,
6.  **Publicar** automaticamente no **OneDrive** (replace atômico + retry).

> **Importante:** Os nomes dos arquivos **finais** salvos **não são alterados** pelo notebook (ex.: `DOCENG.csv`, `RELATORIO_GERENCIAL.csv`, `WORKFLOW_ACTIVE.csv`, `WORKFLOW_HISTORY.csv`).
***

## 🎯 Objetivos
*   **Confiabilidade**: lidar com **quedas de conexão** e **timeouts** no SFTP (keepalive, retry, reconexão).
*   **Generalização**: baixar e ler relatórios com **nomes dinâmicos**, p.ex.  
    `AB_VW_RELATORIO_GERENCIAL_2_ALL_20260225.csv.zip` **ou** `AB_VW_RELATORIO_GERENCIAL_2_ALL.csv.zip`.
*   **Performance prática**: evitar releitura repetida de arquivos grandes (cache).
*   **Prontidão para operação**: logs claros por etapa (sucesso/erro) e exportação robusta (arquivo em uso/OneDrive).
*   **Portabilidade**: configuração via `.env` (sem credenciais no código).
***

## 🧱 Escopo (O que este projeto faz)
*   **Lê** a planilha `Relatórios.xlsx` com as colunas mínimas: `Arquivo`, `Unidade`.
*   **Para cada linha**:
    *   resolve o **prefixo** a partir de `Arquivo` (remove `.csv.zip`);
    *   encontra no SFTP o **.zip mais recente** que **começa com o prefixo**;
    *   baixa o `.zip` e **extrai** o `.csv`;
    *   **lê** o `.csv` resultante **por prefixo** (funciona com e sem data no nome);
    *   adiciona a coluna **`UO`** (Unidade) conforme a planilha.
*   **Aplica filtros** específicos do negócio.
*   **Exporta** CSVs finais em `./RelatorioFinal`.
*   **Publica** no **OneDrive** (pasta definida no `.env`), com **retry** + **replace atômico**.
***

## 🧩 Padrão de Nomes (dinâmicos) — Como o notebook resolve
*   **No SFTP (download)**: busca por `prefixo + qualquer_sufixo + ".zip"`, pega o **mais recente** (ordem lexical).
    *   Exemplos de prefixos aceitos (vindos da planilha):
        *   `SIGLA_SITE_VW_RELATORIO_DOCENG_ALL`
        *   `SIGLA_SITE_VW_RELATORIO_GERENCIAL_2_ALL`
        *   `SIGLA_SITE_VW_WORKFLOW_ACTIVE_ALL`
        *   `SIGLA_SITE_VW_WORKFLOW_HISTORY_ALL`
*   **No disco (leitura)**: busca por `prefixo + ("_*.csv" ou ".csv")`, priorizando o **mais recente**.
> Assim, **não importa** se o nome possui `_20260225`, `_YYYYMMDD`, ou outra convenção definida pela TI — o pipeline encontra e processa corretamente.
***
 Variáveis de Ambiente (arquivo `.env`)
**Obrigatórias**
*   `HOST` — servidor SFTP
*   `PORT` — porta SFTP (ex.: `22`)
*   `USERNAME` — usuário SFTP
*   `PASSWORD` — senha SFTP
*   `CAMINHO` — diretório remoto dos arquivos do SFTP`.zip`
**Publicação OneDrive**
*   `ONEDRIVE_DIR` — pasta destino para publicar os CSVs (OneDrive empresarial)  
    *(opcionalmente `LOCAL` como fallback; opcional `ONEDRIVE_DIR_ALT` como destino alternativo se a pasta principal for somente leitura)*
**(Opcional se necessário) Chave SSH**

*   `PKEY_PATH` — caminho da chave privada (quando o servidor requer public key)
*   `PKEY_PASS` — passphrase da chave (se houver)
*   `SFTP_OTP` — código OTP para keyboardinteractive/MFA (se aplicável)
***

## 🧭 Fluxo (alto nível)
1.  **Inicialização**
    *   Carrega `.env`, configura logging, cria pastas (`Arquivos`, `RelatorioFinal`).
2.  **Leitura da planilha**
    *   `Relatórios.xlsx` (milhares de linhas, por exemplo).
3.  **Download & Extração**
    *   Deduplica por `Arquivo` (baixa apenas 1 de cada).
    *   Resolve prefixos, encontra `.zip` no SFTP, baixa e extrai.
4.  **Leitura dos CSVs**
    *   Encontra por prefixo no disco (com/sem data).
    *   Aplica cache para não reler o mesmo arquivo muitas vezes.
5.  **Filtros**
    *   Aplica critérios de negócio (Fase, Projeto/SE, Datas, Criadores, etc.).
6.  **Exportação**
    *   Salva CSVs em `./RelatorioFinal` com writetotemp + `os.replace` + retry.
7.  **Publicação no OneDrive**
    *   Copia para `.tmp` dentro do OneDrive e faz replace atômico (retry se bloqueado).
    *   **Se a pasta estiver somente leitura**, pode publicar no `ONEDRIVE_DIR_ALT` (fallback).
***

## 📝 Entradas e Saídas
**Entrada**
*   `Relatórios.xlsx` — planilha com:
    *   `Arquivo` — nome base do relatório (ex.: `AB_VW_RELATORIO_DOCENG_ALL.csv.zip`)
    *   `Unidade Organizacional` — valor para preencher a coluna `UO` ao consolidar
**Saída (em `./RelatorioFinal`)**
*   `DOCENG.csv`
*   `RELATORIO_GERENCIAL.csv`
*   `WORKFLOW_ACTIVE.csv`
*   `WORKFLOW_HISTORY.csv`
*(Os nomes acima são fixos por exigência do processo no Power BI.)*
***

## 🛡️ Resiliência, Logs e Tratamento de Erros
*   **SFTP**: reconexão com backoff para erros transitórios (`SSHException`, `timeout`), **sem retry** para `AuthenticationException` (credenciais/método).
*   **Download**: baixa em **`.part`** e faz **replace** ao final.
*   **Leitura CSV**: resolução por **prefixo** (com/sem data) + cache (em caso de alteração no nome do arquivo).
*   **Exportação**: escreve em `.tmp` e **troca atômica**; retry para `PermissionError` (arquivo em uso).
*   **OneDrive**: replace atômico; retry para bloqueio; suporte a **fallback** quando a pasta principal é somente leitura.
*   **Logging**: mensagens por etapa (INFO/WARN/ERROR) no console e arquivo (`./logs/pipeline_YYYYMMDD.log`).
***

## 🔐 Segurança
*   **Nunca** versionar `.env` com credenciais.
*   Use chaves SSH quando o servidor exigir (e proteção por passphrase).
*   Se houver MFA/keyboardinteractive, prover `SFTP_OTP` de forma segura (ou integrar com TOTP).
***

## ▶️ Como Executar
1.  Crie e preencha o `.env` com as variáveis acima.
2.  Garanta a planilha `Relatórios.xlsx` no diretório do notebook.
3.  Dê um cuplo clique no arquivo `RodaPipelineGED.bat` para excutar o script `pipelineGED.py`
4.  Se optar por utilizar o arquivo Execute as células **em ordem** ou selecionado **Run All Cells** no Jupyter.
5.  Verifique os resultados em:
    *   `./RelatorioFinal` (arquivos consolidados)
    *   Logs em `./logs/`
***

## 📂 Estrutura esperada
```
    /Nova_Versao_GED
    ├─ /.ipynb_checkpoints              # Pasta com checkpoints do Jupyter
    ├─ /Arquivos                        # Pasta para salvamento dos ZIP/CSV baixados (limpado a cada execução)
    ├─ /dados                           # Pasta com os dados gerados pelo script Python (Desconsiderar)
    ├─ /logs                            # Arquivos ".txt" logs diários da execução
    ├─ /RelatorioFinal                  # saída final dos CSVs
    ├─ .env                             # credenciais e caminhos (NÃO versionar)
    ├─ DownloadRelatorios.ipynb         # Arquivo Jupyter ".ipynb"
    ├─ pipelineGED.py                   # Script Python para execução do pipeline
    ├─ README.md                        # Arquivo README com o informações sobre o projeto
    ├─ Relatórios.xlsx                  # Arquivo que determina quais relatórios serão baixados do SFTP
    └─ RodaPipelineGED.bat              # Arquivo de lotes do Windows que executa o script "pipelineGED.py"
```
***

## 🧭 Observações finais
*   Se a publicação no OneDrive falhar por **permissão (somente leitura)**, peça **“Editar”** na pasta ou configure `ONEDRIVE_DIR_ALT` até a liberação.
*   Caso o servidor não aceite **senha**, habilite **public key** no `.env` (ou `keyboardinteractive`/MFA, conforme política da TI).
*   O pipeline foi desenhado para **operar diariamente** com mínima intervenção, respeitando nomes dinâmicos de arquivos no servidor.
