# Como Rodar — E-Commerces-WebScraper

Guia completo de preparação e execução do projeto neste ambiente (Linux, Python 3.10.0 via asdf).

---

## Sumário

1. [Visão geral do projeto](#1-visão-geral-do-projeto)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Problemas conhecidos neste ambiente (e correções)](#3-problemas-conhecidos-neste-ambiente)
4. [Passo a passo de preparação](#4-passo-a-passo-de-preparação)
5. [Configuração do `.env`](#5-configuração-do-env)
6. [Preparando a entrada (URL do produto)](#6-preparando-a-entrada-url-do-produto)
7. [Executando o projeto](#7-executando-o-projeto)
8. [Estrutura de saída](#8-estrutura-de-saída)
9. [Solução de problemas](#9-solução-de-problemas)

---

## 1. Visão geral do projeto

Scraper de produtos de e-commerce que extrai nome, preço, desconto, descrição e imagens de:

| Plataforma   | Método                        | Autenticação | Script principal |
| ------------ | ----------------------------- | ------------ | ---------------- |
| Amazon       | Automação de navegador (Playwright) | Opcional     | `Amazon.py`      |
| Mercado Livre| HTTP Requests (BeautifulSoup) | Não          | `MercadoLivre.py`|
| AliExpress   | Automação de navegador (Playwright) | Opcional     | `AliExpress.py`  |
| Shein        | Automação de navegador (Playwright) | Sim (perfil Chrome) | `Shein.py` |
| Shopee       | Automação de navegador (Playwright) | Sim (perfil Chrome) | `Shopee.py` |

Fluxo principal:

1. `url_input_normalizer.py` normaliza as URLs de `Inputs/urls.txt`
2. `main.py` detecta a plataforma pela URL, roda o scraper específico, baixa imagens,
   remove duplicatas e gera a pasta de saída em `Outputs/`
3. (Opcional) `Gemini.py` gera conteúdo de marketing se `GEMINI_API_KEY` estiver configurada

---

## 2. Pré-requisitos

- **Python >= 3.8** (neste ambiente: 3.10.0 via asdf — verificado ✅)
- **pip** (neste ambiente: pip 21.2.3 — será atualizado no passo a passo)
- **git** (para clonar/atualizar o repositório)
- **Conexão com a internet**
- **Sem Chrome necessário para Amazon**: o `Amazon.py` usa o Chromium do Playwright.
  Chrome real + perfil (`CHROME_PROFILE_PATH`) só é necessário para Shopee/Shein.
- **Google Gemini API Key** (opcional — só para gerar templates de marketing)
- `make` (opcional, para os comandos do Makefile)
- `play`, `aplay` ou similar (opcional — som de notificação ao finalizar; aufência é ignorável)

---

## 3. Problemas conhecidos neste ambiente

Os itens abaixo **impedem a execução** com Python 3.10.0 neste clone e precisam ser corrigidos
antes da instalação:

| # | Problema | Impacto | Correção |
| - | -------- | ------- | -------- |
| 1 | `requirements.txt` está em codificação **UTF-16** | `pip install -r requirements.txt` falha com erro de decode | Converter para UTF-8 (ver passo 4.3) |
| 2 | `numpy==2.4.3` nos requirements **exige Python >= 3.11** | Instalação falha no Python 3.10.0 | Fixar `numpy==2.2.6` (última versão que suporta 3.10; usada apenas por `Scripts/affiliate_pages_downloader.py`, não pelo fluxo principal) |
| 3 | Seu Python 3.10 (asdf) foi compilado **sem suporte a tkinter** (`_tkinter` ausente) e `main.py` importa `tkinter` no topo | `python main.py` quebra no import | Parche: import opcional de tkinter com fallback (o tkinter só serve para janelas de aviso GUI; com `--headerless True` não é usado) |
| 4 | Não existe arquivo `.env` | Scraper roda sem chaves (Gemini desabilitado; Chrome com perfil padrão) | Copiar `.env.example` → `.env` |
| 5 | Pasta `Inputs/` e `Inputs/urls.txt` não existem | O `main.py` cria o vazio automaticamente, mas sem URL não faz nada | Criar o arquivo e adicionar a URL do produto |

---

## 4. Passo a passo de preparação

### 4.1 (Se ainda não clonou) Clonar o repositório

```bash
git clone https://github.com/BrenoFariasdaSilva/E-Commerces-WebScraper.git
cd E-Commerces-WebScraper
```

> Neste ambiente o clone já existe em `/home/guilherme/Apps/E-Commerces-WebScraper`.

### 4.2 Criar e ativar o ambiente virtual

```bash
python3.10 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip   # pip 21.2.3 -> versão atual
```

O Makefile usa automaticamente o `venv/` (targets `Makefile: dependencies`), então
os comandos `make ...` também funcionam sem ativar o venv manualmente.

### 4.3 Corrigir a codificação e o pin do requirements.txt

**Corrigir codificação (UTF-16 → UTF-8):**

```bash
iconv -f UTF-16LE -t UTF-8 requirements.txt > /tmp/opencode/requirements_utf8.txt
mv /tmp/opencode/requirements_utf8.txt requirements.txt
```

**Corrigir o pin do numpy (Python 3.10):**

```bash
sed -i 's/^numpy==2.4.3$/numpy==2.2.6/' requirements.txt
```

> `numpy 2.4.x` exige Python >= 3.11. `2.2.6` é a última versão compatível com 3.10.

### 4.4 Instalar as dependências

```bash
python -m pip install -r requirements.txt
```

### 4.5 Instalar os navegadores do Playwright (obrigatório para Amazon/AliExpress/Shein/Shopee)

```bash
python -m playwright install chromium
```

> Se der erro de dependências do sistema no Linux:
> `python -m playwright install-deps chromium` (requer sudo).

### 4.6 Criar o arquivo `.env`

```bash
cp .env.example .env
```

E ajustar os valores conforme a [seção 5](#5-configuração-do-env).

---

## 5. Configuração do `.env`

```env
# Opcional — deixa vazio para rodar sem gerar templates de marketing via IA
# Formato: Main:SUA_CHAVE,Backup:OUTRA_CHAVE
GEMINI_API_KEY=

# Caminho do perfil do Chrome (só necessário para Shopee/Shein).
# Para Amazon, pode deixar vazio (usa o Chromium do Playwright).
# Linux: /home/SEU_USUARIO/.config/google-chrome
CHROME_PROFILE_PATH=
CHROME_EXECUTABLE_PATH=

# False = navegador visível (bom para debug/primeira execução)
# True  = navegador invisível (produção)
HEADLESS=False
```

Variáveis reconhecidas pelos scrapers (em `Amazon.py` e similares):

- `GEMINI_API_KEY` — chaves da API do Google Gemini (opcional)
- `CHROME_PROFILE_PATH` — diretório de dados do Chrome (opcional; se vazio, o Chromium puro do Playwright é usado)
- `CHROME_EXECUTABLE_PATH` — executável do Chrome se não estiver no local padrão (opcional)
- `HEADLESS` — `True`/`False` para rodar com ou sem janela do navegador

> ⚠️ Se usar `CHROME_PROFILE_PATH`, feche **todas** as janelas do Chrome antes de rodar
> (o perfil fica travado se o Chrome estiver aberto).

---

## 6. Preparando a entrada (URL do produto)

O scraper lê as URLs de `Inputs/urls.txt` — **uma URL por linha**.

```bash
mkdir -p Inputs
echo "https://www.amazon.com.br/d/XXXXXXXXXX" > Inputs/urls.txt
```

Formatos aceitos por linha:

| Formato | Exemplo | Uso |
| ------- | ------- | --- |
| Só a URL (scraping online) | `https://www.amazon.com.br/d/XXXXXXXXXX` | Caso padrão |
| URL + caminho de HTML local | `https://... ./Inputs/produto/index.html` | scraping offline de página salva |
| URL + caminho de um .zip | `https://... ./Inputs/produto.zip` | scraping offline de arquivo zipado |

> ⚠️ O `main.py` **remove automaticamente** a linha do `urls.txt` após processar o
> produto com sucesso (constante `CLEAR_INPUT_FILE = True` em `main.py`).

Recomendação de URL da Amazon: use o link canônico do produto
(`https://www.amazon.com.br/d/<ASIN>` ou a URL completa da página do produto).

---

## 7. Executando o projeto

### Opção A — Rodar apenas o scraper principal (recomendado para um produto)

```bash
source venv/bin/activate
python main.py --headerless True
```

- `--headerless True` desativa as janelas de aviso GUI do tkinter (não precisamos delas).

O script limpa o terminal, processa cada URL de `Inputs/urls.txt`, e mostra/registra logs.

### Opção B — Fluxo completo via Makefile

```bash
make run       # roda normalizador + affiliate downloader + main.py + reestruturação + weekly posts
make main      # roda apenas main.py (com sort) + reestruturação de saídas
make local     # variante para entradas locais/offline
```

> ⚠️ O `make run` também executa `Scripts/affiliate_pages_downloader.py`, que depende de
> PyAutoGUI/screenshot e precisa de um display gráfico (X11/Wayland) — em um servidor
> headless ele falha. Para ficar só no scraping de produto, use a **Opção A** ou `make main`.

### Opção C — Só preparar os insumos, sem rodar o scraper

```bash
make normalize_urls      # normaliza Inputs/urls.txt
make prepare_inputs      # normaliza + adiciona zips + renomeia arquivos
```

---

## 8. Estrutura de saída

Cada execução cria uma pasta de run com timestamp em `Outputs/`:

```
Outputs/
└── 1. 2026-10-26 - 15h30m00s/          # pasta do run (índice + data/hora)
    ├── Amazon - Nome do Produto/       # pasta do produto (prefixo da plataforma)
    │   ├── Nome do Produto.txt         # descrição extraída (template do marketing)
    │   ├── Nome do Produto_Template.txt# conteúdo gerado pela IA (se GEMINI_API_KEY setada)
    │   ├── image_1.webp                # imagens baixadas (dedupe + filtro de qualidade)
    │   ├── image_2.webp
    │   ├── video_1.mp4                 # vídeos (se houver)
    │   ├── index.html                  # snapshot da página + assets/ localizados
    │   └── assets/                     # imagens/css/js referenciados pelo snapshot
    └── Logs/                           # logs do run podem ficar aqui
Logs/main.log                            # log global na raiz do projeto
```

Nomes de pasta de produto são sanitizados e truncados em 80 caracteres
(`product_utils.normalize_product_name`).

---

## 9. Solução de problemas

| Problema | Causa provável | Solução |
| -------- | -------------- | ------- |
| `UnicodeDecodeError` ao instalar requirements | arquivo em UTF-16 | refazer conversão do passo 4.3 |
| `numpy` não instala / exige 3.11 | pin antigo | garantir `numpy==2.2.6` no `requirements.txt` |
| `ModuleNotFoundError: _tkinter` | Python compilado sem tkinter | usar `--headerless True` e import opcional de tkinter (patch) |
| `Executable doesn't exist` (Playwright) | navegadores não instalados | `python -m playwright install chromium` |
| `Unable to open user data directory` | Chrome aberto com o mesmo perfil | fechar todo o Chrome |
| `No product data extracted` | sem login / seletores mudaram / anti-bot | rodar com `HEADLESS=False`, conferir logs em `Logs/main.log` |
| Rate limiting / IP bloqueado | muitas requisições seguidas | aumentar `DELAY_BETWEEN_REQUESTS` em `main.py` |
| Sem display no Linux ao usar `make run` | PyAutoGUI precisa de X11 | usar `python main.py --headerless True` (Opção A) |

---

## Referências rápidas

- Entrada: `Inputs/urls.txt`
- Saída: `Outputs/<N. data - hora>/<Plataforma> - <Produto>/`
- Log principal: `Logs/main.log`
- Scraper Amazon: `Amazon.py`
- Orquestrador: `main.py`
