# Painel de Incidentes (APP-CORE)

Painel web para gerenciar, em tempo real, os IPs monitorados e os logs de firewall que o Apache Spark analisa. Tanto os IPs quanto os logs ficam no Elasticsearch, e você também consegue vê-los e inseri-los pelo Kibana.

## Como o sistema é feito

É uma página única (`index.html`, HTML + CSS + JavaScript puro, sem framework e sem build) servida por um Nginx em Docker. Não existe backend próprio: o navegador conversa direto com o Elasticsearch, e o Nginx faz a ponte.

```
Navegador ──► Nginx (porta 80)
                ├─ /          → index.html
                └─ /es/       → proxy para o Elasticsearch (elasticsearch:9200)

Kibana ──────► Elasticsearch ◄────── Spark
                 ├─ ips-monitorados  (lido pelo Spark)
                 ├─ logs-firewall    (lido pelo Spark)
                 └─ incidentes       (gravado pelo Spark)
```

| Arquivo | Função |
|---|---|
| `index.html` | Toda a interface e a lógica (abas, CRUDs, atualização automática) |
| `nginx.conf` | Serve a página e encaminha `/es/` ao Elasticsearch |
| `docker-compose.yml` | Sobe o Nginx e o conecta à rede do Elasticsearch |

O proxy `/es/` existe para evitar CORS e porque o navegador não enxerga o hostname `elasticsearch`, que só existe dentro do Docker.

## Interface

- **Navegação:** menu lateral com as páginas Painel (`#painel`) e Datasource (`#datasource`). A página aberta é lembrada ao recarregar. No celular o menu vira uma barra no topo.
- **Tema:** claro ou escuro automático, conforme o sistema operacional.
- **Feedback:** erros de validação aparecem embaixo do campo, ações bem-sucedidas mostram um aviso no canto da tela, e remover pede confirmação em uma janela própria.
- **Estados:** carregamento (esqueleto), lista vazia, sem resultado na busca e erro de conexão com botão "Tentar novamente".
- **Tempo real:** o indicador "Ao vivo" no topo fica vermelho se o Elasticsearch parar de responder. As datas aparecem como "há 7 minutos", com a data exata ao passar o mouse.
- **Atalhos:** na edição de um IP, `Enter` salva e `Esc` cancela; no Datasource, `Esc` cancela a edição do log.
- **Acessibilidade:** campos com rótulo, foco visível, botões de ícone com `aria-label` e respeito a "reduzir animações".
- Sem bibliotecas: o CSS e os ícones (SVG) estão dentro do próprio `index.html`.

## Aba "Painel"

- **Cartões de resumo:** quantidade de IPs monitorados, de incidentes e de incidentes com ação `deny`.
- **IPs monitorados:** lista com busca, e CRUD completo no índice `ips-monitorados` (veja abaixo).
- **Incidentes recentes:** os 50 últimos documentos do índice `incidentes` do Elasticsearch, atualizados sozinhos a cada 2 segundos. É onde os alertas detectados pelo Spark devem aparecer.

## Aba "Datasource"

É a fonte de dados dos logs de firewall, ou seja, o índice `logs-firewall` do Elasticsearch. Em vez de inserir logs fictícios só pelo Kibana, você pode criá-los, editá-los e removê-los direto por aqui.

- Formulário com **IP**, **ação** (`deny` ou `allow`) e **mensagem**.
- O IP é gravado no campo `src_ip`, que é o campo que o script do Spark lê.
- A tabela lista os 100 logs mais recentes e se atualiza a cada 5 segundos, então o que você inserir pelo Kibana também aparece.
- Logs inseridos pelo Kibana costumam trazer tudo dentro de `message` (`src_ip=... action=...`). A tela extrai o IP e a ação desse texto quando os campos não existem.

## Como funciona o CRUD

**IPs monitorados** (aba Painel): cada IP é um documento do índice `ips-monitorados`, no formato `{"ip": "...", "@timestamp": "..."}`. O `_id` do documento é o próprio IP, então o Elasticsearch não deixa duplicar.

| Operação | Chamada ao Elasticsearch |
|---|---|
| Ler | `POST /es/ips-monitorados/_search` (ordem de cadastro) |
| Criar | `PUT /es/ips-monitorados/_doc/<ip>?refresh=true` |
| Editar | remove o `_id` antigo e cria o novo, porque o `_id` é o IP |
| Remover | `DELETE /es/ips-monitorados/_doc/<ip>?refresh=true` |

Antes de gravar, o IP é validado (formato IPv4 e sem duplicados). Como o índice é do Elasticsearch, também dá para ver e editar a lista pelo Kibana.

**Logs** (aba Datasource): cada log é um documento do índice `logs-firewall`.

| Operação | Chamada ao Elasticsearch |
|---|---|
| Ler | `POST /es/logs-firewall/_search` (ordenado por `@timestamp`, mais recentes primeiro) |
| Criar | `POST /es/logs-firewall/_doc?refresh=true`, com `@timestamp` automático |
| Editar | `POST /es/logs-firewall/_update/<id>?refresh=true` (mescla os campos e preserva o resto) |
| Remover | `DELETE /es/logs-firewall/_doc/<id>?refresh=true` |

O `refresh=true` faz a alteração aparecer na busca imediatamente.

## Formato dos documentos

| Índice | Quem grava | Campos |
|---|---|---|
| `ips-monitorados` | Painel (aba Painel) | `ip`, `@timestamp`. O `_id` é o IP. |
| `logs-firewall` | Painel (aba Datasource) e Kibana | `src_ip`, `action`, `message`, `@timestamp`. Os logs do Kibana podem ter só `message` e `@timestamp`, com `src_ip=... dest_ip=... action=...` dentro do texto. |
| `incidentes` | Spark | `@timestamp`, `src_ip`, `dest_ip`, `action`, `message` (cópia do log que deu match). |

## Como o Spark funciona (`SPARK/apps/deny_list.py`)

O script roda em loop (a cada 2 segundos) dentro do cluster Spark:

1. Lê os IPs de `ips-monitorados` (por isso um IP novo ou removido vale sem reiniciar).
2. Lê até 1000 logs de `logs-firewall`. Se `src_ip` e `action` não existirem como campos, extrai do texto de `message` (`pega_campo`).
3. Cria um DataFrame do Spark e filtra os logs cujo `src_ip` está na lista de IPs monitorados.
4. Para cada match ainda não visto (controlado pelo `_id` do log, em memória), grava um documento em `incidentes` e uma linha em `danylist.csv`, e imprime `[ALERTA]` no terminal. Se a gravação falhar, tenta de novo no ciclo seguinte.

Como a memória de "já visto" é do processo, ao reiniciar o script os logs que já deram match geram incidentes duplicados.

## Como foi feito

- Começou como um painel com a lista de IPs em um arquivo JSON (`logs-ips.json`) servido pelo Nginx com `PUT`.
- Foi criada a aba Datasource para cadastrar, editar e remover os logs de firewall que antes só entravam pelo Kibana.
- A lista de IPs foi migrada do arquivo JSON para o índice `ips-monitorados`, e o Spark passou a ler de lá. Com isso o arquivo, a pasta `dados/`, o `PUT` do Nginx e o volume do Spark deixaram de existir.
- Escolhas deliberadas: sem backend e sem framework (o Nginx basta como proxy), o IP como `_id` para evitar duplicados e `refresh=true` nas gravações para a tela refletir a mudança na hora.

## Como executar

Pré-requisito: Docker com Docker Compose. Rode na ordem abaixo, a partir da raiz do repositório. O Elasticsearch precisa subir primeiro porque cria a rede `elasticsearch_elastic-net`, que o painel usa.

```bash
cd ELASTICSEARCH && docker compose up -d && cd ..
cd SPARK && docker compose up -d && cd ..
cd APP-CORE && docker compose up -d && cd ..
```

| Serviço | Endereço |
|---|---|
| Painel | http://localhost |
| Kibana | http://localhost:5601 |
| Elasticsearch | http://localhost:9200 |
| Spark Master | http://localhost:8080 |

O índice `ips-monitorados` é criado quando o primeiro IP é cadastrado pelo painel.

Para ver o Spark em ação, rode o `deny_list.py` dentro do cluster. A cada 2 segundos ele relê `ips-monitorados` e `logs-firewall`, compara o `src_ip` dos logs com os IPs monitorados, grava os matches no índice `incidentes` (que aparece na aba Painel) e no `danylist.csv`, e imprime os alertas no terminal. IP novo ou removido vale sem reiniciar. Pare com `Ctrl+C`.

```bash
docker exec spark-master /opt/spark/bin/spark-submit /opt/spark-apps/deny_list.py
```

Para parar tudo, rode `docker compose down` em cada uma das três pastas.

### Testar o fluxo completo

1. Suba os containers e rode o `deny_list.py` (comando acima), deixando o terminal aberto.
2. No Painel, cadastre um IP, por exemplo `8.8.4.4`. No terminal do Spark aparece `[NOVO IP MONITORADO] 8.8.4.4`.
3. Na aba Datasource, crie um log com esse IP, ação `deny` e uma mensagem.
4. No terminal do Spark aparece `[ALERTA] IP da deny list encontrado: 8.8.4.4`, seguido de `Novos incidentes gravados: 1`.
5. Em até 2 segundos o incidente aparece em "Incidentes recentes" na aba Painel, e a linha é adicionada ao `SPARK/apps/danylist.csv`.
6. No Kibana, o log aparece no Discover (veja abaixo).

### Ver os logs no Kibana

O Kibana começa sem Data Views. Em Stack Management → Data Views, crie um apontando para `logs-firewall` (campo de tempo `@timestamp`). Depois, o que você criar no Datasource aparece no Discover.

## Limitações conhecidas

- Sem autenticação: o Elasticsearch roda com a segurança desativada (`xpack.security.enabled=false`) e o painel é aberto. Use só em laboratório.
- A lista de IPs mostra até 1000 itens.
- O `deny_list_v2.py` é uma versão de teste que só lê e imprime os logs; ele lê apenas o campo `src_ip`, então logs antigos do Kibana (com tudo dentro de `message`) vêm como `None` nele. O `deny_list.py` não tem esse problema.
