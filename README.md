# APS 3 - Replanejamento de Rota por Bloqueio

**Aluna:** Heloyse Pereira
**Disciplina:** Modelagem de Software - Prof. Sidney L. da Silva
**Projeto:** Sistema de Logística de Entregas Autônomas (AGVs)

## A funcionalidade

A minha parte é o **replanejamento de rota**. Às vezes um corredor do parque fica bloqueado (uma empilhadeira parada, uma área interditada) enquanto um AGV está no meio da entrega. A API de Tráfego avisa o sistema e ele precisa:

1. descobrir quais rotas passam pelo trecho bloqueado;
2. mandar o AGV parar em um lugar seguro e, ao mesmo tempo, pedir uma rota nova para o Planejador de Rotas;
3. se a rota nova atrasar no máximo 15 minutos, enviar para o AGV e esperar ele confirmar;
4. se não tiver rota boa, ou o AGV não confirmar depois de 3 tentativas, a rota é invalidada e o operador é avisado.

Se o bloqueio for liberado no meio do processo, o AGV volta para a rota original.

## Operações de cada participante

- API de Orquestração: `informarBloqueio(BloqueioDTO)`, `onRotaConfirmada(ConfirmacaoDTO)`
- Banco de Dados: `buscarRotasNoSegmento(segmentoId)`, `atualizarStatusRota(rotaId, status)`, `salvarNovaRota(rotaId, NovaRotaDTO)`
- Planejador de Rotas: `recalcularRota(RecalculoDTO)`
- Broker MQTT: `enviarParada(agvId)`, `enviarNovaRota(agvId, NovaRotaDTO)`, `confirmarRota(agvId)`
- AGV: `parar()`, `receberRota(NovaRotaDTO)`
- Aplicação Web: `avisarReplanejamento(NovaRotaDTO)`, `avisarOperador(AlertaDTO)`

## Diagrama de Sequência

`->>` = síncrona (a chamada gRPC ao Planejador é síncrona), `-)` = assíncrona (webhook, MQTT, WebSocket), `-->>` = retorno.

```mermaid
sequenceDiagram
    participant Trafego as API de Tráfego
    participant API as API de Orquestração
    participant BD as Banco de Dados
    participant Planejador as Planejador de Rotas
    participant Broker as Broker MQTT
    participant AGV
    participant Web as Aplicação Web

    Trafego-)API: informarBloqueio(BloqueioDTO)
    API->>BD: buscarRotasNoSegmento(segmentoId)
    BD-->>API: lista de RotaDTO

    loop para cada rota afetada
        API->>BD: atualizarStatusRota(rotaId, RECALCULANDO)
        API-)Broker: enviarParada(agvId)
        Broker-)AGV: parar()
        API->>Planejador: recalcularRota(RecalculoDTO)
        Planejador-->>API: NovaRotaDTO (segmentos, atrasoMin)

        alt existe rota nova e atrasoMin <= 15
            API->>BD: salvarNovaRota(rotaId, NovaRotaDTO)
            API-)Broker: enviarNovaRota(agvId, NovaRotaDTO)
            Broker-)AGV: receberRota(NovaRotaDTO)
            AGV-)Broker: confirmarRota(agvId)
            Broker-)API: onRotaConfirmada(ConfirmacaoDTO)
            API->>BD: atualizarStatusRota(rotaId, ATIVA)
            API-)Web: avisarReplanejamento(NovaRotaDTO)
        else não existe rota ou atrasoMin > 15
            API->>BD: atualizarStatusRota(rotaId, INVALIDADA)
            API-)Web: avisarOperador(AlertaDTO)
        end
    end
```

## Diagrama de Atividades

O fork faz o AGV parar e o Planejador calcular ao mesmo tempo. O join garante que a rota nova só é enviada depois que o AGV já parou.

```mermaid
flowchart TB
    subgraph R1[API de Tráfego]
        I((início))
        T1[Detectar bloqueio e avisar o sistema]
    end

    subgraph R2[API de Orquestração]
        B1[Buscar rotas que passam pelo bloqueio]
        D0{Alguma rota afetada?}
        B2[Marcar rota como RECALCULANDO]
        FORK[ ]:::barra
        JOIN[ ]:::barra
        D1{Rota nova serve?}
        B3[Salvar e enviar rota nova]
        B4[Ativar rota]
        B5[Invalidar rota]
    end

    subgraph R3[Planejador de Rotas]
        C1[Calcular caminho desviando do bloqueio]
    end

    subgraph R4[AGV]
        G1[Parar em local seguro]
        G2[Receber rota e confirmar]
    end

    subgraph R5[Operador]
        E1[Ver aviso no painel]
        F((fim))
    end

    I --> T1 --> B1 --> D0
    D0 -->|"[nenhuma rota]"| F
    D0 -->|"[1 ou mais rotas]"| B2 --> FORK
    FORK --> G1 --> JOIN
    FORK --> C1 --> JOIN
    JOIN --> D1
    D1 -->|"[existe rota e atraso <= 15 min]"| B3 --> G2 --> B4 --> E1
    D1 -->|"[não existe rota ou atraso > 15 min]"| B5 --> E1
    E1 --> F

    classDef barra fill:#000,stroke:#000,color:#000
```

## Diagrama de Estados (Rota)

```mermaid
stateDiagram-v2
    [*] --> Calculada
    Calculada --> Ativa : missaoDespachada / enviarRotaParaAGV()

    state "Ativa" as Ativa
    Ativa : entry / registrarInicio()
    Ativa : do / acompanharTrajeto()

    Ativa --> Concluida : chegouNoDestino
    Ativa --> Replanejando : bloqueioInformado [segmento está na rota] / pararAGV()

    state Replanejando {
        [*] --> Recalculando
        Recalculando --> AguardandoConfirmacao : rotaEncontrada [atraso <= 15 min] / enviarNovaRota()
        AguardandoConfirmacao --> Recalculando : semResposta [tentativas < 3]
    }

    state "AguardandoConfirmacao" as AguardandoConfirmacao
    AguardandoConfirmacao : entry / iniciarTimer(5s)
    AguardandoConfirmacao : exit / pararTimer()

    AguardandoConfirmacao --> Ativa : agvConfirmou
    AguardandoConfirmacao --> Invalidada : semResposta [tentativas >= 3] / avisarOperador()
    Recalculando --> Invalidada : semRota [atraso > 15 min ou rota inexistente] / avisarOperador()
    Replanejando --> Ativa : bloqueioLiberado / voltarRotaOriginal()

    Concluida --> [*]
    Invalidada --> [*]
```

## C4 - Nível 3 (Planejador de Rotas)

```mermaid
flowchart TB
    api["API de Orquestração<br/>[Container: Node.js + NestJS]"]

    subgraph planejador[Planejador de Rotas - Container: Python]
        grpc["RotaService<br/>[Componente: servidor gRPC]"]
        replan["Replanejador<br/>[Componente: regra dos 15 min]"]
        grafo["MapaDoParque<br/>[Componente: NetworkX]"]
        astar["CalculadorDeCaminho<br/>[Componente: algoritmo A*]"]
        trafego["ClienteTrafego<br/>[Componente: requests]"]
    end

    bd[("Banco de Dados<br/>[Container: PostgreSQL]")]
    apiTrafego["API de Tráfego<br/>[Sistema externo]"]
    geo["API de Geolocalização<br/>[Sistema externo]"]

    apiTrafego -->|"HTTPS (webhook)"| api
    api -->|"gRPC (prazo de 2s)"| grpc
    grpc --> replan
    replan --> trafego
    replan --> astar
    astar --> grafo
    grafo -->|SQL| bd
    trafego -->|"HTTPS / REST"| apiTrafego
    replan -->|"HTTPS / REST"| geo

    classDef container fill:#1168bd,color:#fff
    classDef comp fill:#85bbf0,color:#000
    classDef ext fill:#999,color:#fff
    class api,bd container
    class grpc,replan,grafo,astar,trafego comp
    class apiTrafego,geo ext
```

## ADR

[ADR 0001 - Usar gRPC entre a API de Orquestração e o Planejador de Rotas](docs/architecture/adr-0001.md)

## Relação entre os diagramas

| O que acontece | Sequência | Estado da Rota |
|---|---|---|
| Bloqueio informado | `informarBloqueio()` | Ativa → Replanejando |
| Parar AGV + recalcular (fork) | `enviarParada()` e `recalcularRota()` | Recalculando |
| Rota nova enviada | `enviarNovaRota()` | Recalculando → AguardandoConfirmacao |
| AGV confirmou | `onRotaConfirmada()` | AguardandoConfirmacao → Ativa |
| Sem rota boa | `atualizarStatusRota(INVALIDADA)` | Recalculando → Invalidada |
