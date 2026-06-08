
# 📓Descrição Pipeline de Coleta, Leitura, Tratamento, Filtragem, Publicação de Dados desenvolvido em linguagem Python (SFTP → Local → OneDrive)

LEANDRO DE SOUZA REIS SILVA
Graduado em Ciência de Dados

### 🪪 Identificação do Projeto

- Nome do Projeto: Automatização do Pipeline de Dados Sistema GED 
- Criado por: Leandro Souza Reis Silva 
- Data de criação: 12/03/2026 
- Plataforma: Liguagem Python, Jupyter Notebook, VSCode
- O projeto proporcionou um ganho de aproximadamente 80% na produtividade de coleta e tratamento de dados.
- Com foco em regras de negócio, o código em Python não foi inserido no repositório

## 🔎 Visão Geral

O projeto implementa um pipeline **profissional** para automatizar a Coleta, Transformação e Carregamento (ETL) dos Dados seguindo os passos abaixo:
1.  **Baixar** 65 arquivos de dados via **SFTP** (FileZilla Server) com reconexão e tolerância a falhas,
2.  **Tratar nomes dinâmicos** no servidor (sufixos com data/metadata),
3.  **Extrair** arquivos `.zip` e **ler** os `.csv` resultantes,
4.  **Filtrar** os dados conforme regras do negócio do setor,
5.  **Exportar** resultados para a pasta local `./RelatorioFinal`,
6.  **Publicar** automaticamente no **OneDrive** (replace atômico + retry).

> **Importante:** Os nomes dos arquivos **finais** salvos **não são alterados** pelo algoritmo (ex.: `DOCENG.csv`, `RELATORIO_GERENCIAL.csv`, `WORKFLOW_ACTIVE.csv`, `WORKFLOW_HISTORY.csv`).
***

## 🎯 Objetivos

*   **Confiabilidade**: lidar com **quedas de conexão** e **timeouts** no SFTP (keepalive, retry, reconexão).
*   **Generalização**: baixar e ler relatórios com **nomes dinâmicos**, p.ex.  
    `SIGLA_SITE_VW_RELATORIO_GERENCIAL_2_ALL_20260225.csv.zip` **ou** `SIGLA_SITE_VW_RELATORIO_GERENCIAL_2_ALL.csv.zip`.
*   **Performance prática**: evitar releitura repetida de arquivos grandes (cache).
*   **Prontidão para operação**: logs claros por etapa (sucesso/erro) e exportação robusta (arquivo em uso/OneDrive).
*   **Portabilidade**: configuração via `.env` (sem credenciais no código).
***

## 🧱 Escopo (O que este projeto faz)

*   **Lê** a planilha `Relatórios.xlsx` com as colunas mínimas: `Arquivo`, `Unidade`.
*   **Para cada linha**:<br>
    *   resolve o **prefixo** a partir de `Arquivo` (remove `.csv.zip`);<br>
    *   encontra no SFTP o **.zip mais recente** que **começa com o prefixo**;<br>
    *   baixa o `.zip` e **extrai** o `.csv`;<br>
    *   **lê** o `.csv` resultante **por prefixo** (funciona com e sem data no nome);<br>
    *   adiciona a coluna **`UO`** (Unidade) conforme a planilha.<br>
*   **Aplica filtros** específicos do negócio.<br>
*   **Exporta** CSVs finais em `./RelatorioFinal`.<br>
*   **Publica** no **OneDrive** (pasta definida no `.env`), com **retry** + **replace atômico**.<br>
***

## 🧩 Padrão de Nomes (dinâmicos) — Como o notebook resolve

*   **No SFTP (download)**: busca por `prefixo + qualquer_sufixo + ".zip"`, pega o **mais recente** (ordem lexical).<br>
    *   Exemplos de prefixos aceitos (vindos da planilha):<br>
        *   `SIGLA_SITE_VW_RELATORIO_DOCENG_ALL`<br>
        *   `SIGLA_SITE_VW_RELATORIO_GERENCIAL_2_ALL`<br>
        *   `SIGLA_SITE_VW_WORKFLOW_ACTIVE_ALL`<br>
        *   `SIGLA_SITE_VW_WORKFLOW_HISTORY_ALL`<br>
*   **No disco (leitura)**: busca por `prefixo + ("_*.csv" ou ".csv")`, priorizando o **mais recente**.<br>
> Assim, **não importa** se o nome possui `_20260225`, `_YYYYMMDD`, ou outra convenção definida pela TI — o pipeline encontra e processa corretamente.<br>
***
 Variáveis de Ambiente (arquivo `.env`)<br>
**Obrigatórias**<br>
*   `HOST` — servidor SFTP<br>
*   `PORT` — porta SFTP (ex.: `22`)<br>
*   `USERNAME` — usuário SFTP<br>
*   `PASSWORD` — senha SFTP<br>
*   `CAMINHO` — diretório remoto dos arquivos do SFTP`.zip`<br>
**Publicação OneDrive**<br>
*   `ONEDRIVE_DIR` — pasta destino para publicar os CSVs (OneDrive empresarial)<br>
    *(opcionalmente `LOCAL` como fallback; opcional `ONEDRIVE_DIR_ALT` como destino alternativo se a pasta principal for somente leitura)*<br>
**(Opcional se necessário) Chave SSH**<br>

*   `PKEY_PATH` — caminho da chave privada (quando o servidor requer public key)<br>
*   `PKEY_PASS` — passphrase da chave (se houver)<br>
*   `SFTP_OTP` — código OTP para keyboardinteractive/MFA (se aplicável)<br>
***

## 🧭 Fluxo (alto nível)

1.  **Inicialização**<br>
    *   Carrega `.env`, configura logging, cria pastas (`Arquivos`, `RelatorioFinal`).<br>
2.  **Leitura da planilha**<br>
    *   `Relatórios.xlsx` (milhares de linhas, por exemplo).<br>
3.  **Download & Extração**<br>
    *   Deduplica por `Arquivo` (baixa apenas 1 de cada).<br>
    *   Resolve prefixos, encontra `.zip` no SFTP, baixa e extrai.<br>
4.  **Leitura dos CSVs**<br>
    *   Encontra por prefixo no disco (com/sem data).<br>
    *   Aplica cache para não reler o mesmo arquivo muitas vezes.<br>
5.  **Filtros**<br>
    *   Aplica critérios de negócio (Fase, Projeto/SE, Datas, Criadores, etc.).<br>
6.  **Exportação**<br>
    *   Salva CSVs em `./RelatorioFinal` com writetotemp + `os.replace` + retry.<br>
7.  **Publicação no OneDrive**<br>
    *   Copia para `.tmp` dentro do OneDrive e faz replace atômico (retry se bloqueado).<br>
    *   **Se a pasta estiver somente leitura**, pode publicar no `ONEDRIVE_DIR_ALT` (fallback).<br>
***

## 📝 Entradas e Saídas

**Entrada**<br>
*   `Relatórios.xlsx` — planilha com:<br>
    *   `Arquivo` — nome base do relatório (ex.: `SIGLA_SITE_VW_RELATORIO_DOCENG_ALL.csv.zip`)<br>
    *   `Unidade Organizacional` — valor para preencher a coluna `UO` ao consolidar<br>
**Saída (em `./RelatorioFinal`)**<br>
*   `DOCENG.csv`<br>
*   `RELATORIO_GERENCIAL.csv`<br>
*   `WORKFLOW_ACTIVE.csv`<br>
*   `WORKFLOW_HISTORY.csv`<br>
*(Os nomes acima são fixos por exigência do processo no Power BI.)*<br>
***

## 🛡️ Resiliência, Logs e Tratamento de Erros

*   **SFTP**: reconexão com backoff para erros transitórios (`SSHException`, `timeout`), **sem retry** para `AuthenticationException` (credenciais/método).<br>
*   **Download**: baixa em **`.part`** e faz **replace** ao final.<br>
*   **Leitura CSV**: resolução por **prefixo** (com/sem data) + cache (em caso de alteração no nome do arquivo).<br>
*   **Exportação**: escreve em `.tmp` e **troca atômica**; retry para `PermissionError` (arquivo em uso).<br>
*   **OneDrive**: replace atômico; retry para bloqueio; suporte a **fallback** quando a pasta principal é somente leitura.<br>
*   **Logging**: mensagens por etapa (INFO/WARN/ERROR) no console e arquivo (`./logs/pipeline_YYYYMMDD.log`).<br>
***

## 🔐 Segurança

*   **Nunca** versionar `.env` com credenciais.<br>
*   Use chaves SSH quando o servidor exigir (e proteção por passphrase).<br>
*   Se houver MFA/keyboardinteractive, prover `SFTP_OTP` de forma segura (ou integrar com TOTP).<br>
***

## ▶️ Como Executar

1.  Crie e preencha o `.env` com as variáveis acima.<br>
2.  Garanta a planilha `Relatórios.xlsx` no diretório do notebook.<br>
3.  Dê um cuplo clique no arquivo `RodaPipelineGED.bat` para excutar o script `pipelineGED.py`<br>
4.  Se optar por utilizar o arquivo Execute as células **em ordem** ou selecionado **Run All Cells** no Jupyter.<br>
5.  Verifique os resultados em:<br>
    *   `./RelatorioFinal` (arquivos consolidados)<br>
    *   Logs em `./logs/`<br>
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

*   Se a publicação no OneDrive falhar por **permissão (somente leitura)**, peça **“Editar”** na pasta ou configure `ONEDRIVE_DIR_ALT` até a liberação.<br>
*   Caso o servidor não aceite **senha**, habilite **public key** no `.env` (ou `keyboardinteractive`/MFA, conforme política da TI).<br>
*   O pipeline foi desenhado para **operar diariamente** com mínima intervenção, respeitando nomes dinâmicos de arquivos no servidor.<br>
