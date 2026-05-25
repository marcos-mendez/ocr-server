# OCR Server 2.1.1 — © Agência Nacional de Telecomunicações (Anatel)

Servidor de OCR automático para arquivos PDF. Monitora diretórios de entrada e aplica reconhecimento óptico de caracteres via [Tesseract](https://github.com/tesseract-ocr/tesseract), produzindo um novo PDF/A com camada de texto pesquisável oculta — sem alterar a aparência visual do documento original.

Distribuído sob os termos da licença [LGPL v2.1](lgpl-2.1.txt).

**Autor:** Guilherme Chehab

---

## Documentação adicional

- [Diagrama de fluxo (PDF)](workflow.pdf) — fluxo completo de processamento do servidor
- [Diagrama de fluxo (Visio)](workflow.vsd) — arquivo-fonte do diagrama

---

## Como funciona

O servidor opera como daemon Unix e monitora continuamente os diretórios configurados. A cada ciclo:

1. Detecta novos arquivos PDF no diretório `Entrada/`
2. Verifica se o arquivo já possui texto extraível — se sim, ignora
3. Processa cada página em paralelo via Tesseract (um processo por página)
4. Gera um novo PDF/A com camada de texto pesquisável oculta
5. Move o resultado para `Saida/` e o original para `Originais_Processados/`
6. Em caso de falha, move o arquivo para `Erro/`

A sincronização entre instâncias paralelas é feita por renomeação de arquivo e `flock`, permitindo que múltiplos servidores monitorem o mesmo diretório simultaneamente (útil em ambientes com NFS/SAMBA).

---

## Execução via Docker (recomendado)

A forma recomendada de implantação é via container Docker, que inclui todas as dependências (Tesseract, Ghostscript, ImageMagick, cpdf, pdftk, poppler-utils) sem necessidade de instalação manual.

### Pré-requisito

Docker instalado no host.

### Build da imagem

```sh
docker build -t ocr-server .
```

### Iniciando o serviço

```sh
docker run --name <NOME_CONTAINER> -d -v <DIRETORIO_BASE>:/var/ocr-server ocr-server
```

| Parâmetro | Descrição |
|-----------|-----------|
| `--name` | Nome da instância do container (ex: `ocr-server`) |
| `-d` | Executa em segundo plano |
| `-v <DIRETORIO_BASE>:/var/ocr-server` | Mapeia o diretório do host onde os PDFs serão monitorados |

O container criará automaticamente a seguinte estrutura de subdiretórios em `<DIRETORIO_BASE>`:

```
<DIRETORIO_BASE>/
├── Entrada/                # Coloque aqui os PDFs para processar
├── Saida/                  # PDFs com OCR aplicado
├── Originais_Processados/  # Cópias dos arquivos originais
└── Erro/                   # Arquivos que não puderam ser processados
```

### Visualizando os logs

```sh
# Últimas entradas de log
docker logs <NOME_CONTAINER>

# Acompanhamento em tempo real
docker logs -f <NOME_CONTAINER>
```

---

## Configuração

As variáveis de configuração estão no arquivo `/usr/local/bin/ocr`:

| Variável | Descrição | Padrão |
|----------|-----------|--------|
| `@BASE_DIRS` | Lista de diretórios base monitorados — cada diretório gera sua própria instância | — |
| `@SUB_DIRS` | Subdiretórios de entrada, saída, backup de originais, temporário e de erros | — |
| `$MAX_FILES` | Número máximo de arquivos processados simultaneamente por diretório | `2` |
| `$MAX_PGS` | Número máximo de páginas processadas simultaneamente por arquivo | nº de CPUs |

O número máximo de processos simultâneos resulta de: **Nº de diretórios × MAX\_FILES × MAX\_PGS**.

### Dimensionamento de recursos

Recomenda-se aproximadamente **1,5 GB de RAM por core** para evitar uso de swap. Caso não seja possível, reduza `$MAX_FILES` ou `$MAX_PGS`.

Como referência de desempenho: em uma CPU Xeon E5 4670 @ 2,6 GHz, cada página leva ~18 segundos. Com 16 cores, o desempenho agregado é de ~1,2 segundos por página.

### Operação multi-instância

Para escalar horizontalmente, instale múltiplos servidores apontando para o mesmo diretório de entrada via compartilhamento SAMBA/CIFS ou NFS. O mecanismo de sincronização garante que cada PDF seja processado por apenas uma instância.

---

## Instalação manual (sem Docker)

> A instalação manual é destinada a ambientes sem suporte a containers. O uso do Docker (seção acima) é fortemente recomendado.

### Pré-requisitos — Ubuntu 22.04 / 24.04

```sh
# Habilitar repositório universe (necessário no Ubuntu 24.04)
add-apt-repository universe

apt-get update && apt-get upgrade && apt-get install -y \
    tesseract-ocr tesseract-ocr-por tesseract-ocr-eng tesseract-ocr-spa \
    leptonica-progs poppler-utils pdftk-java unpaper ghostscript imagemagick \
    rsyslog perl libfile-find-rule-perl libfile-touch-perl libunix-syslog-perl

# cpdf — binário pré-compilado (não disponível nos repositórios oficiais)
wget https://github.com/coherentgraphics/cpdf-binaries/raw/v2.9/Linux-Intel-64bit/cpdf \
     -O /usr/local/bin/cpdf
chmod 755 /usr/local/bin/cpdf

# Habilitar leitura/escrita de PDFs no ImageMagick
sed -i 's/rights="none" pattern="PDF"/rights="read|write" pattern="PDF"/' \
    /etc/ImageMagick-6/policy.xml
```

### Instalação do serviço

```sh
# Script principal
cp usr/local/bin/ocr /usr/local/bin/
chmod 755 /usr/local/bin/ocr

# Script de inicialização
cp etc/init.d/ocr /etc/init.d/
chmod 755 /etc/init.d/ocr
update-rc.d ocr defaults

# Criar diretórios de trabalho
mkdir -p /var/ocr-server/{Entrada,Saida,Originais_Processados,Erro}
chmod -R 777 /var/ocr-server

# Iniciar o serviço
service ocr start
```

---

## Histórico de versões

- **1.0** — Primeira versão de produção. Suporte a múltiplos tipos de imagem; script init.d para Ubuntu e CentOS.
- **2.0** — Saída em formato PDF/A com melhor compressão via Ghostscript. Reescrita do processo de extração e transformação de imagens. Verificação de assinatura digital (arquivos assinados não são modificados). Uso preferencial de pacotes do sistema operacional.
- **2.1 / 2.1.1** — Filtros de processamento e redução de cor desabilitados por padrão. Container Docker baseado em Ubuntu.

---

## Limitações e pendências conhecidas

### Pendências (TODO)

- Suporte a páginas PDF com mais de uma imagem por página — a versão atual assume que o número de páginas é igual ao número de imagens
- Mover todas as configurações para arquivo externo (`.conf`)
- Interface web para controle de filas e submissão de arquivos pelo usuário final

### Problemas conhecidos

- Em PDFs com imagens do tipo *stencil* ou *encoding*, as informações de recorte são perdidas no processamento, fazendo com que a página gerada difira visualmente do original. Isso decorre de uma limitação do `pdftoppm` na extração dessas imagens.

---

## Erros ou Sugestões

1. [Abrir Issue](https://github.com/anatelgovbr/ocr-server/issues) no repositório do GitHub em caso de erro na instalação ou na operação do serviço.
2. Ao abrir a Issue, utilize o modelo **"1 - Reportar Erro"**.
