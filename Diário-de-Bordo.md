# Diário de Bordo — Escalonador de Processos e System Calls

## Data
01/10/2026

## Tema
Escalonador de Processos e Chamadas de Sistema (System Calls)

---

## Objetivo

Compreender o funcionamento das Chamadas de Sistema (System Calls) e do escalonamento de processos, analisando um estudo de caso envolvendo processos interativos e processos Batch em um servidor de núcleo único.

O objetivo principal é identificar por que o algoritmo FCFS pode prejudicar a responsividade de aplicações interativas e analisar uma alternativa de escalonamento que permita distribuir melhor o uso da CPU.

---

# Estudo de Caso — CloudData

A startup CloudData possui um servidor de núcleo único responsável por executar dois tipos de tarefas:

- **Processos Interativos:** requisições rápidas de usuários que precisam de respostas imediatas.
- **Processos Batch:** geração de relatórios financeiros que realizam processamento pesado e operações de entrada e saída.

O servidor está configurado utilizando o algoritmo **FCFS (First-Come, First-Served)** e os usuários estão relatando congelamentos e lentidão na interface web.

---

# Parte A — Chamadas de Sistema

## 1. Como o processo solicita a leitura do disco?

Um processo não acessa diretamente o hardware. Quando o processo de geração de relatório precisa ler dados financeiros armazenados no disco, ele solicita esse serviço ao Sistema Operacional por meio de uma **System Call (chamada de sistema)**.

As System Calls funcionam como uma interface entre os programas e o núcleo do Sistema Operacional. Por meio delas, um processo pode solicitar serviços que exigem acesso controlado aos recursos do computador.

Uma operação de leitura pode ser representada de forma simplificada como:

```text
Processo
   ↓
System Call
   ↓
Kernel
   ↓
Disco
   ↓
Kernel
   ↓
Processo
```

Segundo Bernal (2005), as chamadas de sistema fornecem aos processos uma interface para utilização dos serviços disponibilizados pelo núcleo do Sistema Operacional.

---

## 2. O que acontece com o processo durante a solicitação?

Normalmente, uma aplicação é executada em **Modo Usuário**, que possui acesso limitado aos recursos do computador.

Ao realizar uma System Call, ocorre uma transição para o **Modo Kernel (ou Modo Supervisor)**. Nesse modo, o Sistema Operacional possui os privilégios necessários para realizar operações protegidas, como o acesso aos dispositivos de entrada e saída.

No caso da leitura do relatório, o processo solicita a operação ao kernel e pode ficar **bloqueado** enquanto aguarda a conclusão do acesso ao disco.

Enquanto esse processo aguarda o I/O, o Sistema Operacional pode utilizar a CPU para executar outro processo.

O fluxo pode ser representado assim:

```text
Modo Usuário
     ↓
System Call
     ↓
Modo Kernel
     ↓
Solicitação de leitura
     ↓
Processo bloqueado aguardando I/O
     ↓
Leitura concluída
     ↓
Processo volta para o estado Pronto
```

Dessa forma, uma operação de I/O não significa necessariamente que a CPU ficará parada. O processo que está esperando pelo disco pode ficar bloqueado enquanto outro processo utiliza o processador.

---

# Parte B — Diagnosticando o FCFS

## 1. Por que o FCFS causa o congelamento da interface?

O **FCFS (First-Come, First-Served)** executa os processos de acordo com a ordem em que eles chegam à fila de processos prontos.

Um problema desse algoritmo aparece quando um processo longo chega antes de processos menores e interativos.

No caso da CloudData, um relatório Batch pode receber a CPU antes de uma requisição web. Como o FCFS é não preemptivo, a requisição interativa precisa aguardar o processo que está executando liberar a CPU.

Exemplo:

```text
Relatório Batch chega
        ↓
Recebe a CPU
        ↓
Requisição web chega
        ↓
Requisição fica esperando
        ↓
Relatório continua executando
        ↓
Interface demora para responder
```

A UFRGS apresenta o FCFS como uma política de escalonamento não preemptiva e destaca como uma de suas características o fato de processos que chegam posteriormente terem que aguardar aqueles que estão à sua frente na fila.

Nesse cenário, isso prejudica principalmente os processos interativos, que precisam de um baixo tempo de resposta.

---

## 2. O que significa FCFS ser não preemptivo?

Um algoritmo **não preemptivo** não retira a CPU à força de um processo que está executando.

No FCFS, quando um processo recebe a CPU, ele continua executando até:

- terminar sua execução;
- liberar voluntariamente a CPU;
- ou realizar uma operação que faça o processo ser bloqueado, como uma operação de I/O.

Portanto, se um relatório Batch estiver utilizando a CPU e uma requisição web chegar posteriormente, a requisição não poderá simplesmente interromper o relatório para executar imediatamente.

```text
Relatório recebe CPU
        ↓
Requisição web chega
        ↓
FCFS não realiza preempção
        ↓
Requisição espera
```

Em um servidor de núcleo único, esse problema fica ainda mais evidente, pois existe apenas uma CPU disponível para executar os processos.

---

# Parte C — Propondo uma Solução

## 1. Round-Robin

Entre os algoritmos apresentados no estudo de caso — **SJF, SRTN e Round-Robin** — o **Round-Robin (RR)** é uma alternativa adequada para melhorar a responsividade da interface web.

O Round-Robin utiliza um intervalo de tempo chamado **quantum**. Cada processo recebe a CPU durante esse intervalo. Quando o quantum termina e o processo ainda não terminou, ele pode ser retirado da CPU por meio de **preempção** e colocado novamente na fila de processos prontos.

Exemplo:

```text
Processo Batch
      ↓
Recebe CPU
      ↓
Quantum termina
      ↓
Preempção
      ↓
Processo Interativo recebe CPU
      ↓
Requisição é processada
      ↓
Outro processo recebe CPU
```

A UFRGS apresenta o Round-Robin como um algoritmo preemptivo baseado na utilização de um quantum para distribuir o tempo de processador entre os processos.

Essa característica é importante para a CloudData porque impede que um processo Batch mantenha a CPU continuamente enquanto processos interativos aguardam.

### SJF

O **SJF (Shortest Job First)** seleciona o processo com menor tempo de execução estimado.

Ele pode favorecer tarefas curtas, mas existe uma dificuldade: o Sistema Operacional precisa estimar quanto tempo cada processo levará para executar.

### SRTN

O **SRTN (Shortest Remaining Time Next)** utiliza uma abordagem preemptiva baseada no menor tempo restante de execução.

Apesar de poder favorecer processos curtos, ele também depende de estimativas sobre o tempo restante dos processos.

### Round-Robin

O **Round-Robin** utiliza preempção e quantum para distribuir o uso da CPU entre os processos.

Por isso, considerando especificamente a necessidade de melhorar a **responsividade da interface web**, ele se encaixa bem no cenário apresentado.

O podcast *Café Debug*, ao abordar threads, paralelismo e Sistemas Operacionais, também trata de conceitos relacionados à execução de tarefas e operações I/O, ajudando a compreender a relação entre aplicações e gerenciamento de recursos pelo Sistema Operacional.

---

# Starvation e Aging

## 2. O que é Starvation?

**Starvation**, também chamada de inanição, ocorre quando um processo permanece esperando por recursos ou pela oportunidade de execução durante um período excessivamente longo.

No cenário da CloudData, imagine que a empresa utilize um algoritmo baseado em prioridades:

```text
Prioridade alta
      ↓
Processos Interativos

Prioridade baixa
      ↓
Processos Batch
```

Se novas requisições interativas de alta prioridade continuarem chegando, os processos Batch podem permanecer esperando por muito tempo.

Exemplo:

```text
Processo Web → alta prioridade
Processo Web → alta prioridade
Processo Web → alta prioridade
Processo Web → alta prioridade
Processo Batch → baixa prioridade
```

Nesse cenário existe o risco de ocorrer **Starvation**.

---

## Aging

Uma forma de reduzir o problema é utilizar o mecanismo de **Aging (envelhecimento)**.

O Aging aumenta gradualmente a prioridade de um processo conforme ele permanece esperando.

```text
Processo Batch
Prioridade baixa
      ↓
Permanece esperando
      ↓
Prioridade aumenta
      ↓
Continua esperando
      ↓
Prioridade aumenta novamente
      ↓
Processo recebe CPU
```

Dessa forma, mesmo que um processo tenha inicialmente uma prioridade baixa, sua prioridade pode aumentar enquanto ele aguarda, reduzindo o risco de ficar indefinidamente sem utilizar a CPU.

---

# Relação entre os conceitos

Durante a pesquisa, foi possível relacionar os principais conceitos estudados:

```text
                  SISTEMA OPERACIONAL
                         │
             ┌───────────┴───────────┐
             │                       │
        System Calls            Escalonamento
             │                       │
        Modo Usuário            Processos
             ↓                       │
        Modo Kernel            ┌─────┴─────┐
             │                 │           │
             ↓               FCFS      Round-Robin
            I/O                 │           │
             │              Não Preemptivo  │
             ↓                             ↓
      Processo Bloqueado               Preempção
                                           │
                                           ↓
                                  Maior responsividade
                                           │
                                           ↓
                                    Processos Web
             
             Prioridades
                  │
                  ↓
              Starvation
                  │
                  ↓
                Aging
```

---

# Conclusão

O estudo de caso demonstrou como o funcionamento do Sistema Operacional influencia diretamente o desempenho de uma aplicação.

As **System Calls** permitem que os processos solicitem serviços ao kernel, como operações de leitura e escrita. Durante essas solicitações, o Sistema Operacional controla o acesso aos recursos protegidos e o processo pode ficar bloqueado enquanto aguarda uma operação de I/O.

No caso da CloudData, o **FCFS** pode prejudicar a responsividade porque é um algoritmo não preemptivo. Um processo Batch que esteja executando pode fazer com que uma requisição interativa tenha que esperar.

O **Round-Robin** apresenta uma alternativa utilizando quantum e preempção, permitindo que os processos interativos tenham oportunidades frequentes de utilizar a CPU.

Por fim, uma política de prioridades pode gerar **Starvation** para processos de baixa prioridade. O **Aging** pode ser utilizado para aumentar gradualmente a prioridade dos processos que permanecem esperando.

---

# Referências

## Vídeo

UNIVERSIDADE DE SÃO PAULO. **PCS 3446 - Sistemas Operacionais**. São Paulo: e-Aulas USP, [s.d.]. Disponível em: <https://eaulas.usp.br/portal/skins/default/video?idItem=18843>. Acesso em: 1 out. 2026.

## Podcasts

SOFTWARE SESSIONS. **Elizabeth Figura on Wine and Proton**. [S. l.]: Software Sessions, 2025. Podcast. Disponível em: <https://podcasts.apple.com/us/podcast/elizabeth-figura-on-wine-and-proton/id1479514262?i=1000728248651>. Acesso em: 1 out. 2026.

CAFÉ DEBUG. **#167 Threads, Paralelismo e SO na Prática para Devs**. [S. l.]: Café Debug, 2025. Podcast. Disponível em: <https://podcasts.apple.com/sn/podcast/167-threads-paralelismo-e-so-na-pr%C3%A1tica-para-devs/id1367730836?i=1000717118967>. Acesso em: 1 out. 2026.

## Fontes acadêmicas

BERNAL, Volnys Borges. **Introdução aos Sistemas Operacionais**. São Paulo: Universidade de São Paulo, 2005. Disponível em: <https://www.lsi.usp.br/~volnys/courses/psi2653/2005/03-SistOperacional/01-IntroSO-v3.pdf>. Acesso em: 1 out. 2026.

TOSCANI, Simão Sirineo; CARISSIMI, Alexandre da Silva; OLIVEIRA, Rômulo Silva de. **Sistemas Operacionais: escalonamento de processos**. Porto Alegre: Instituto de Informática, Universidade Federal do Rio Grande do Sul, [s.d.]. Disponível em: <https://www.inf.ufrgs.br/~asc/livro/transparencias/cap4.pdf>. Acesso em: 1 out. 2026.
