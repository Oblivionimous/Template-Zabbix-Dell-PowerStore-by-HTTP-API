# Dell PowerStore by HTTP — Template Zabbix

Template para monitoramento de storages **Dell PowerStore** (modelos T e Q) pela **API REST** do PowerStoreOS, usando itens **HTTP agent** e **itens dependentes**. Não precisa de agente, scripts externos nem SNMP.

| | |
|---|---|
| **Template** | `Dell PowerStore by HTTP` |
| **Autor** | Mauro Paiva |
| **Versão** | 7.0 |
| **Zabbix** | 7.0 LTS ou superior |
| **Grupo de templates** | `PowerStore` |
| **Arquivo** | [`template_dell_powerstore_by_http.yaml`](template_dell_powerstore_by_http.yaml) |

## Sumário

- [Inspiração e créditos](#inspiração-e-créditos)
- [Visão geral](#visão-geral)
- [Requisitos](#requisitos)
- [Instalação](#instalação)
- [Como a coleta funciona](#como-a-coleta-funciona)
- [Macros](#macros)
- [Itens do template](#itens-do-template)
- [Descobertas (LLD)](#descobertas-lld)
- [Triggers](#triggers)
- [Gráficos](#gráficos)
- [Inventário](#inventário)
- [Mapeamentos de valores](#mapeamentos-de-valores)
- [Tags](#tags)
- [Dicas e solução de problemas](#dicas-e-solução-de-problemas)
- [Referências](#referências)
- [Changelog](#changelog)

## Inspiração e créditos

Este template foi inspirado no projeto **[zabbix_powerstore](https://github.com/HuiMi24/zabbix_powerstore)**, de **[HuiMi24](https://github.com/HuiMi24)**, que publicou um template Zabbix para monitorar o Dell EMC PowerStore pela API REST (o *Template PowerStore Cluster*, para Zabbix 6.2+) e um dashboard Grafana que o acompanha.

O projeto original mostrou que dava para monitorar o PowerStore só com HTTP agent consultando a API REST, e serviu de ponto de partida para este trabalho. A partir dessa ideia, a versão 7.0 deste template foi reescrita e ampliada para o Zabbix 7.0 LTS. As principais mudanças em relação ao original:

- macros com prefixo `{$POWERSTORE.*}` e senha guardada como **macro secreta**;
- chaves, nomes, tags e descrições padronizados, em português;
- itens dependentes a partir de uma única coleta por objeto, para reduzir as chamadas à API;
- descobertas de hardware, drives (desgaste de SSD), portas Ethernet/FC/SAS, NAS servers, host groups, volume groups e sessões de replicação;
- alertas ativos do PowerStore, versão do PowerStoreOS, status da API, inventário do host e protótipos de gráfico.

Agradecimentos ao **HuiMi24** por compartilhar o trabalho original com a comunidade. Se este template for útil para você, visite também o [repositório original](https://github.com/HuiMi24/zabbix_powerstore) e deixe uma estrela por lá.

## Visão geral

O template monitora:

- **Cluster e appliances**: estado, performance de bloco e de arquivo, capacidade física e lógica, redução de dados e eficiência;
- **Nodes**: performance de bloco, arquivo, NFS e SMB, utilização de CPU do workload de I/O e sessões de host conectadas;
- **Hardware**: estado de todos os componentes (nodes, PSUs, ventiladores, DIMMs, drives, SFPs e outros), part number e número de série;
- **Drives**: vida útil restante (endurance) dos SSDs;
- **Portas**: link, uso, velocidade, MTU, tráfego e erros das portas Ethernet; link, WWN e performance das portas FC; link e velocidade das portas SAS;
- **Objetos de armazenamento**: volumes, volume groups, file systems, NAS servers, hosts e host groups;
- **Replicação**: estado e última sincronização das sessões;
- **Alertas**: contagem de alertas ativos do PowerStore por severidade e lista dos alertas Critical e Major pendentes;
- **API**: código de status HTTP e indisponibilidade.

Em números:

| Objeto | Quantidade |
|---|---|
| Itens | 46 (17 HTTP agent + 29 dependentes) |
| Regras de descoberta | 15 |
| Protótipos de item | 271 |
| Triggers | 6 |
| Protótipos de trigger | 30 |
| Protótipos de gráfico | 20 |
| Macros | 36 |
| Mapeamentos de valores | 5 |

## Requisitos

- Zabbix Server ou Proxy **7.0 LTS** ou superior;
- acesso HTTPS (porta 443) do Zabbix Server/Proxy ao endereço de gerência do cluster PowerStore;
- usuário no PowerStore com perfil **Operator** (somente leitura). Pode ser usuário local ou LDAP.

## Instalação

1. **Crie o usuário no PowerStore.** No PowerStore Manager, acesse *Settings → Users* e crie um usuário com o perfil **Operator**.

2. **Importe o template.** No Zabbix, acesse *Data collection → Templates → Import* e envie o arquivo [`template_dell_powerstore_by_http.yaml`](template_dell_powerstore_by_http.yaml).

3. **Crie o host.** Em *Data collection → Hosts → Create host*, crie um host que represente o cluster PowerStore e vincule o template `Dell PowerStore by HTTP`. O host **não precisa de interface**: todas as requisições usam a URL da macro.

4. **Preencha as macros do host** (aba *Macros*):

   | Macro | Exemplo |
   |---|---|
   | `{$POWERSTORE.API.URL}` | `https://10.0.0.10/api/rest` (sem barra no final) |
   | `{$POWERSTORE.API.USER}` | `zabbix_monitor` |
   | `{$POWERSTORE.API.PASSWORD}` | senha do usuário (macro secreta) |

5. **Ajuste limites e filtros** pelas demais macros `{$POWERSTORE.*}`, se necessário (veja [Macros](#macros)).

6. **Inventário.** Para preencher o inventário do host automaticamente, deixe o modo de inventário em **Automático** (aba *Inventory*).

Após alguns minutos, os itens brutos são coletados e as descobertas criam os itens, triggers e gráficos de cada objeto.

## Como a coleta funciona

O template segue o padrão de **coleta mestre + itens dependentes**: cada objeto é consultado uma única vez por intervalo, e os valores são extraídos com JSONPath em itens dependentes. Os itens mestres têm sufixo `.get` e nome terminado em "dados brutos" ou "(bruto)", e não guardam histórico.

### Listas de objetos

Os objetos são lidos com `GET` em endpoints como `/cluster`, `/appliance`, `/node`, `/hardware`, `/volume`, `/file_system`, `/eth_port`, `/fc_port`, `/sas_port`, `/replication_session`, `/alert` e `/software_installed`, com `select=` e `limit={$POWERSTORE.API.LIMIT}`. A API retorna 100 itens por página por padrão; o template pede até 2000 para trazer tudo em uma única chamada. Os códigos HTTP aceitos são `200` e `206` (resposta parcial).

### Métricas

As métricas de performance, capacidade e desgaste são obtidas com `POST /metrics/generate`, informando a entidade, o ID do objeto e o intervalo `Five_Mins`. O pré-processamento guarda só a **amostra mais recente** (`$[-1]`). Entidades usadas:

| Grupo | Entidades |
|---|---|
| Performance de bloco | `performance_metrics_by_cluster`, `_by_appliance`, `_by_node`, `_by_volume`, `_by_vg`, `_by_host`, `_by_hg`, `_by_fe_fc_port` |
| Performance de arquivo | `performance_metrics_file_by_cluster`, `_file_by_appliance`, `_file_by_node`, `performance_metrics_by_file_system`, `performance_metrics_by_nas_server` |
| Protocolos | `performance_metrics_nfs_by_node`, `performance_metrics_smb_by_node` |
| Portas Ethernet | `performance_metrics_by_fe_eth_port` |
| Capacidade | `space_metrics_by_cluster`, `space_metrics_by_appliance`, `space_metrics_by_volume`, `space_metrics_by_vg` |
| Desgaste | `wear_metrics_by_drive` |

> O intervalo `Twenty_Sec` foi testado e levou mais de 2 minutos para responder; por isso o template usa amostras de 5 minutos.

### Conversões

- **Latências** chegam em microssegundos e são convertidas para **segundos** (multiplicador `0.000001`), para que o Zabbix exiba em ms/µs automaticamente.
- **Utilização de CPU** chega como fração e é convertida para **percentual**.
- **Throughput** é guardado em bytes por segundo (`Bps`); tráfego de portas Ethernet em bits por segundo (`bps`).

### Autenticação

Todas as requisições usam **HTTP Basic** com `{$POWERSTORE.API.USER}` e `{$POWERSTORE.API.PASSWORD}` e o cabeçalho `Accept: application/json`. A verificação de certificado TLS fica desabilitada (padrão do HTTP agent), o que permite usar o certificado autoassinado do PowerStore.

## Macros

### Conexão

| Macro | Padrão | Descrição |
|---|---|---|
| `{$POWERSTORE.API.URL}` | `https://CHANGE_ME/api/rest` | URL base da API REST, sem barra no final. |
| `{$POWERSTORE.API.USER}` | | Usuário local ou LDAP do PowerStore. O perfil Operator basta. |
| `{$POWERSTORE.API.PASSWORD}` | | Senha do usuário da API (macro secreta). |
| `{$POWERSTORE.API.TIMEOUT}` | `30s` | Timeout das requisições HTTP. |
| `{$POWERSTORE.API.LIMIT}` | `2000` | Parâmetro `limit` das listas (padrão da API: 100; máximo: 2000). |
| `{$POWERSTORE.NODATA.TIMEOUT}` | `10m` | Tempo sem resposta da API para disparar o trigger de indisponibilidade. |

### Intervalos de coleta

| Macro | Padrão | Descrição |
|---|---|---|
| `{$POWERSTORE.INTERVAL.CONFIG}` | `5m` | Listas de objetos (volumes, file systems, hosts e outros). |
| `{$POWERSTORE.INTERVAL.HEALTH}` | `2m` | Hardware, portas, alertas, replicação e status da API. |
| `{$POWERSTORE.INTERVAL.PERF}` | `5m` | Performance de cluster, appliances e nodes. |
| `{$POWERSTORE.INTERVAL.PERF.OBJ}` | `5m` | Performance de volumes, file systems, hosts, portas e NAS servers. |
| `{$POWERSTORE.INTERVAL.SPACE}` | `5m` | Métricas de capacidade. |

A versão do PowerStoreOS (`/software_installed`) é coletada a cada 1 hora.

### Limites de triggers

| Macro | Padrão | Descrição |
|---|---|---|
| `{$POWERSTORE.LATENCY.MAX.WARN}` | `0.005` | Latência média de atenção (s) para cluster, appliance e node (5 ms). Aceita contexto. |
| `{$POWERSTORE.LATENCY.MAX.HIGH}` | `0.02` | Latência média crítica (s) para cluster, appliance e node (20 ms). Aceita contexto. |
| `{$POWERSTORE.VOLUME.LATENCY.MAX.WARN}` | `0.02` | Latência média de atenção por volume (s). Aceita contexto com o nome do volume. |
| `{$POWERSTORE.CPU.UTIL.MAX}` | `80` | Utilização máxima de CPU de I/O (%). Aceita contexto. |
| `{$POWERSTORE.PUSED.MAX.WARN}` | `80` | Uso físico de atenção (%) para cluster e appliance. |
| `{$POWERSTORE.PUSED.MAX.HIGH}` | `90` | Uso físico crítico (%) para cluster e appliance. |
| `{$POWERSTORE.VOLUME.PUSED.MAX.WARN}` | `95` | Uso lógico de atenção (%) por volume. Aceita contexto com o nome do volume. |
| `{$POWERSTORE.FS.PUSED.MAX.WARN}` | `85` | Uso de atenção (%) por file system. Aceita contexto com o nome do file system. |
| `{$POWERSTORE.FS.PUSED.MAX.HIGH}` | `95` | Uso crítico (%) por file system. Aceita contexto com o nome do file system. |
| `{$POWERSTORE.DRIVE.ENDURANCE.MIN.WARN}` | `20` | Endurance restante mínima do SSD (%) para alerta de atenção. |
| `{$POWERSTORE.DRIVE.ENDURANCE.MIN.HIGH}` | `5` | Endurance restante mínima do SSD (%) para alerta crítico. |
| `{$POWERSTORE.PORT.ERRORS.MAX}` | `1` | Erros CRC por segundo tolerados em porta Ethernet. Aceita contexto com o nome da porta. |
| `{$POWERSTORE.REPL.STATE.CRIT}` | `^(Error\|System_Paused)$` | Regex dos estados de replicação tratados como erro. |
| `{$POWERSTORE.REPL.STATE.WARN}` | `^(Paused)$` | Regex dos estados de replicação tratados como atenção. |

### Filtros de descoberta

| Macro | Padrão | Descrição |
|---|---|---|
| `{$POWERSTORE.VOLUME.NAME.MATCHES}` | `.*` | Volumes a descobrir. |
| `{$POWERSTORE.VOLUME.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Volumes a ignorar. |
| `{$POWERSTORE.FS.NAME.MATCHES}` | `.*` | File systems a descobrir. |
| `{$POWERSTORE.FS.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | File systems a ignorar. |
| `{$POWERSTORE.HOST.NAME.MATCHES}` | `.*` | Hosts do storage a descobrir. |
| `{$POWERSTORE.HOST.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Hosts do storage a ignorar. |
| `{$POWERSTORE.PORT.NAME.MATCHES}` | `.*` | Portas Ethernet e FC a descobrir. |
| `{$POWERSTORE.PORT.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Portas Ethernet e FC a ignorar. |
| `{$POWERSTORE.HW.TYPE.MATCHES}` | `.*` | Tipos de hardware a descobrir. |
| `{$POWERSTORE.HW.TYPE.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Tipos de hardware a ignorar. Exemplo: `^(SFP\|Internal_M2)$`. |
| `{$POWERSTORE.HW.STATE.NOT_MATCHES}` | `^Empty$` | `lifecycle_state` ignorado na descoberta (slots vazios). |

#### Macros com contexto

Várias macros aceitam contexto, o que permite limites diferentes por objeto sem alterar o template. Exemplos:

```
{$POWERSTORE.FS.PUSED.MAX.HIGH:"fs_backup"} = 98
{$POWERSTORE.VOLUME.LATENCY.MAX.WARN:"vol_archive"} = 0.05
{$POWERSTORE.PORT.ERRORS.MAX:"BaseEnclosure-NodeA-IoModule0-FEPort1"} = 10
```

## Itens do template

| Grupo | Itens |
|---|---|
| **API** | código de status HTTP (`powerstore.api.status`) |
| **Cluster** | nome, estado, Global ID, endereço de gerência |
| **Appliance** | modelo e service tag do appliance principal |
| **Sistema** | versão e build do PowerStoreOS |
| **Alertas** | total ativos; Critical, Major, Minor e Info ativos; Critical e Major não reconhecidos; lista de Critical e Major pendentes |
| **Hardware** | componentes em falha ou desconectados; componentes fora do estado Healthy |
| **Inventário** | quantidade de clusters, appliances, nodes, volumes, volume groups, file systems, NAS servers, hosts, host groups e sessões de replicação |

## Descobertas (LLD)

Todas as descobertas são **itens dependentes** das coletas brutas e não fazem chamadas extras para descobrir objetos.

| Descoberta | Chave | Origem | Filtro | O que cria |
|---|---|---|---|---|
| Cluster | `powerstore.cluster.discovery` | `/cluster` | — | Performance de bloco e de arquivo, capacidade física/lógica, redução de dados, eficiência, economia por snapshots e thin provisioning |
| Appliances | `powerstore.appliance.discovery` | `/appliance` | — | Modelo, service tag, nodes, tolerância a falha de drives, performance de bloco e de arquivo, CPU de I/O, capacidade (inclusive uso lógico por volume, file system e vVol) |
| Nodes | `powerstore.node.discovery` | `/node` | — | Performance de bloco, arquivo, NFS e SMB, CPU de I/O e sessões de host conectadas |
| Hardware | `powerstore.hardware.discovery` | `/hardware` | tipo e `lifecycle_state` | Estado, part number e número de série de cada componente |
| Drives (desgaste de SSD) | `powerstore.drive.discovery` | `/hardware` | tipo `Drive` e `lifecycle_state` | Vida útil restante (%) |
| Portas Ethernet | `powerstore.eth_port.discovery` | `/eth_port` | nome | Link, em uso, velocidade, MTU, tráfego de entrada/saída, pacotes, erros CRC, erros de transmissão e descartes por falta de buffer |
| Portas FC | `powerstore.fc_port.discovery` | `/fc_port` | nome | Link, em uso, velocidade, WWN, IOPS, throughput e latência |
| Portas SAS | `powerstore.sas_port.discovery` | `/sas_port` | — | Link e velocidade |
| Volumes | `powerstore.volume.discovery` | `/volume` | nome | Estado, tamanho provisionado, capacidade lógica (usada, livre, %), thin savings, IOPS, throughput, latência e tamanho de I/O |
| Volume groups | `powerstore.volume_group.discovery` | `/volume_group` | — | Capacidade lógica, thin savings e performance |
| File systems | `powerstore.file_system.discovery` | `/file_system` | nome | Tamanho, espaço usado/livre (%) e performance |
| NAS servers | `powerstore.nas_server.discovery` | `/nas_server` | — | Estado operacional e performance |
| Hosts | `powerstore.host.discovery` | `/host` | nome | Performance por host |
| Host groups | `powerstore.host_group.discovery` | `/host_group` | — | Performance por host group |
| Replicação | `powerstore.replication.discovery` | `/replication_session` | — | Estado e última sincronização de cada sessão |

As métricas de performance de cada objeto incluem IOPS (leitura, escrita e total), throughput (leitura, escrita e total), latência média (leitura, escrita e geral) e tamanho médio de I/O (leitura, escrita e geral).

## Triggers

### Template

| Severidade | Nome | Condição |
|---|---|---|
| High | PowerStore: API sem resposta | Sem dados de `powerstore.api.status` por `{$POWERSTORE.NODATA.TIMEOUT}` |
| Average | PowerStore: API respondeu com código HTTP inesperado | Código diferente de `200` |
| Average | PowerStore: cluster fora do estado Configured | Estado do cluster diferente de `Configured` |
| Info | PowerStore: versão do PowerStoreOS alterada | Versão diferente da coleta anterior |
| High | PowerStore: alertas Critical ativos | Alertas Critical não reconhecidos > 0 |
| Average | PowerStore: alertas Major ativos | Alertas Major não reconhecidos > 0 |

### Protótipos

| Severidade | Objeto | Nome | Condição |
|---|---|---|---|
| High / Warning | Cluster, Appliance, Node | latência média alta / acima do esperado | `min(15m)` acima de `{$POWERSTORE.LATENCY.MAX.HIGH}` / `.WARN` |
| Average | Appliance, Node | utilização de CPU alta | `min(15m)` acima de `{$POWERSTORE.CPU.UTIL.MAX}` |
| High / Warning | Cluster, Appliance | capacidade física acima de N% | `min(30m)` acima de `{$POWERSTORE.PUSED.MAX.HIGH}` / `.WARN` |
| High | Hardware | componente em falha | Estado `Failed` ou `Prepare_Failed` |
| Average | Hardware | componente desconectado | Estado `Disconnected` |
| Warning | Hardware | componente fora do estado Healthy | Fora de `Healthy` por 30m |
| High / Warning | Drive | vida útil restante crítica / baixa | Abaixo de `{$POWERSTORE.DRIVE.ENDURANCE.MIN.HIGH}` / `.WARN` |
| High | Porta Ethernet, Porta FC | link down | Porta em uso e link `Down` |
| Warning | Porta Ethernet | erros CRC na recepção | `min(15m)` acima de `{$POWERSTORE.PORT.ERRORS.MAX}` |
| High | Porta SAS | link caiu | Link passou de `Up` para `Down` |
| High | Volume | offline | Estado `Offline` |
| Warning | Volume | latência média alta | `min(15m)` acima de `{$POWERSTORE.VOLUME.LATENCY.MAX.WARN}` |
| Warning | Volume | uso lógico acima de N% | `min(30m)` acima de `{$POWERSTORE.VOLUME.PUSED.MAX.WARN}` |
| High / Warning | File system | espaço usado acima de N% | `min(15m)` acima de `{$POWERSTORE.FS.PUSED.MAX.HIGH}` / `.WARN` |
| High | NAS server | parado | Estado `Stopped` |
| Average | NAS server | degradado ou em failover | Estado `Degraded` ou `Failover` |
| Average | Replicação | sessão com erro | Estado casa com `{$POWERSTORE.REPL.STATE.CRIT}` |
| Warning | Replicação | sessão pausada | Estado casa com `{$POWERSTORE.REPL.STATE.WARN}` |

Os triggers de atenção de latência, capacidade, desgaste de drive, file system, hardware e replicação dependem do trigger mais grave correspondente, para evitar alertas duplicados.

## Gráficos

Protótipos de gráfico criados pelas descobertas:

| Objeto | Gráficos |
|---|---|
| Cluster | capacidade física, IOPS, latência, throughput |
| Appliance | capacidade física, IOPS, latência, throughput, utilização de CPU |
| Node | IOPS, latência, utilização de CPU |
| Volume | capacidade lógica, IOPS, latência |
| File system | espaço, IOPS |
| Porta Ethernet | tráfego |
| Porta FC | IOPS, throughput |

## Inventário

Com o inventário do host em modo **Automático**, os campos abaixo são preenchidos:

| Campo do inventário | Item |
|---|---|
| Name | Cluster: nome |
| Serial number A | Cluster: Global ID |
| Serial number B | Appliance: service tag |
| Model | Appliance: modelo |
| OS | Sistema: versão do PowerStoreOS |
| OS (Full details) | Sistema: build do PowerStoreOS |
| OOB IP address | Cluster: endereço de gerência |

## Mapeamentos de valores

| Nome | Valores |
|---|---|
| PowerStore lifecycle state | 0 Healthy, 1 Empty, 2 Initializing, 3 Trigger_Update, 4 Disconnected, 5 Uninitialized, 6 Failed, 7 Prepare_Failed, 10 Unknown |
| PowerStore link status | 0 Down, 1 Up |
| PowerStore NAS server status | 0 Started, 1 Starting, 2 Stopping, 3 Stopped, 4 Failover, 5 Degraded, 10 Unknown |
| PowerStore volume state | 0 Ready, 1 Initializing, 2 Offline, 3 Destroying, 10 Unknown |
| PowerStore yes/no | 0 Não, 1 Sim |

## Tags

- Template: `class:storage`, `target:dell-powerstore`.
- Itens: `component` (`alerts`, `api`, `appliance`, `capacity`, `cluster`, `hardware`, `health`, `inventory`, `network`, `performance`, `raw`, `replication`, `system`) e tags com o nome do objeto (`cluster`, `appliance`, `node`, `volume`, `volume_group`, `filesystem`, `nas_server`, `storage.host`, `storage.host_group`, `port`, `port.type`, `hardware`, `hardware.name`, `replication`).
- Triggers: `scope` (`availability`, `capacity`, `notice`, `performance`).

Use as tags para filtrar problemas e montar dashboards e ações.

## Dicas e solução de problemas

- **Teste a API manualmente** a partir do Zabbix Server/Proxy:

  ```bash
  curl -k -u 'usuario:senha' -H 'Accept: application/json' \
    'https://<ip-do-powerstore>/api/rest/cluster?select=id,name,state'
  ```

- **`powerstore.api.status` = 401**: usuário ou senha incorretos, ou usuário bloqueado no PowerStore.
- **Timeout**: aumente `{$POWERSTORE.API.TIMEOUT}` ou os intervalos de coleta. Em clusters com muitos objetos, prefira um **Zabbix Proxy** próximo ao storage.
- **Muitos itens**: cada volume, file system, host e porta gera seus próprios itens de performance. Use os filtros `*.NAME.MATCHES` / `*.NAME.NOT_MATCHES` para limitar a descoberta aos objetos que interessam.
- **Componentes de hardware sem interesse** (SFPs, M.2 internos): use `{$POWERSTORE.HW.TYPE.NOT_MATCHES}`, por exemplo `^(SFP|Internal_M2)$`.
- **Itens "dados brutos"/"(bruto)"** não guardam histórico. Use *Test* na configuração do item ou *Latest data* para ver a resposta JSON da API.

## Referências

Documentação consultada para a criação do template.

### Dell PowerStore (fabricante)

- [Dell PowerStore REST API Developers Guide](https://www.dell.com/support/manuals/en-us/powerstore-1000/pwrstr-apidevg/) ([PDF](https://dl.dell.com/content/manual55475248-dell-powerstore-rest-api-developers-guide.pdf)): autenticação, filtros (`select`, `limit`, `eq.`), paginação (HTTP 206) e coleta de métricas por `POST /metrics/generate`.
- PowerStore REST API Reference Guide, disponível em [dell.com/powerstoredocs](https://www.dell.com/powerstoredocs): referência de recursos (`cluster`, `appliance`, `node`, `hardware`, `volume`, `file_system`, `nas_server`, `eth_port`, `fc_port`, `sas_port`, `replication_session`, `alert`, `software_installed`) e das entidades de métricas.
- [PowerStore: How to use PowerStore REST API (KB 000490125)](https://www.dell.com/support/kbdoc/en-us/000490125/powerstore-api-rest-powerstore): artigo da base de conhecimento com exemplos de uso da API.
- Swagger UI do próprio equipamento, em `https://<ip-de-gerencia>/swaggerui`: referência interativa da API na versão do PowerStoreOS instalada, usada para validar endpoints, campos e respostas.

### Zabbix 7.0

- [HTTP agent](https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/http)
- [Dependent items](https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/dependent_items)
- [Item value preprocessing](https://www.zabbix.com/documentation/7.0/en/manual/config/items/preprocessing) e [JSONPath functionality](https://www.zabbix.com/documentation/7.0/en/manual/config/items/preprocessing/jsonpath_functionality)
- [Low-level discovery](https://www.zabbix.com/documentation/7.0/en/manual/discovery/low_level_discovery)
- [User macros](https://www.zabbix.com/documentation/7.0/en/manual/config/macros/user_macros), [User macros with context](https://www.zabbix.com/documentation/7.0/en/manual/config/macros/user_macros_context) e [Secret user macros](https://www.zabbix.com/documentation/7.0/en/manual/config/macros/secret_macros)
- [Trigger expression](https://www.zabbix.com/documentation/7.0/en/manual/config/triggers/expression) e [Trigger dependencies](https://www.zabbix.com/documentation/7.0/en/manual/config/triggers/dependencies)
- [Value mapping](https://www.zabbix.com/documentation/7.0/en/manual/config/items/mapping)
- [Inventory](https://www.zabbix.com/documentation/7.0/en/manual/config/hosts/inventory)
- [Templates: export e import](https://www.zabbix.com/documentation/7.0/en/manual/xml_export_import/templates)
- [Zabbix template guidelines](https://www.zabbix.com/documentation/guidelines/en/template_guidelines): padrões de nomes, chaves, macros, tags e uso de item mestre com itens dependentes.

## Changelog

### 7.0

- Chaves, nomes, tags e descrições padronizados. Macros com prefixo `POWERSTORE` e senha como macro secreta.
- Correção do filtro JSONPath do item "em uso" das portas FC, da tag errada no trigger de link Ethernet e da URL duplicada de appliance.
- Remoção da macro inexistente `{$APPLIANCE_ID}` e unificação das descobertas de DIMM e drive em uma descoberta de hardware.
- Novos itens de alertas ativos, versão do PowerStoreOS, inventário, estado de volumes e NAS servers, capacidade percentual, desgaste de SSD, tráfego e erros de portas Ethernet, performance de portas FC, portas SAS, sessões de replicação, status da API e protótipos de gráfico.

---

**Autor:** Mauro Paiva · **Versão:** 7.0 · Inspirado em [HuiMi24/zabbix_powerstore](https://github.com/HuiMi24/zabbix_powerstore).
