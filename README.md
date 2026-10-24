# Objetivo

1 - Construir diagrama C4 nível de sistemas ( c4.md ). Tome cuidado para os elementos não ficarem um em cima do outro.

2 - Construir um diagrama de sequência (UML) para detalhar um processo crítico ( sequencia.md )

Utilizando mermaid crie esses diagramas usando o escopo abaixo.

# Escopo 
Uma operadora de turismo vende reservas de hotél com vagas limitadas. O sistema em foco é uma aplicação web de reservas: o cliente consulta a região e datas, o sistema disponibiliza os hotéis e quartos disponíveis, o cliente seleciona o quarto desejado, realiza a marcação de reserva e o pagamento. 

Fora do sistema ficam apenas:

- Gateway de Viagens: Que disponíbiliza os hoteis disponiveis dentro de uma região e data; assim, como serviços explicitos para marcação de reserva. Cadastro de hotéis, pacotes, cancelamento, reembolso ocorrem dentro do Gateway de Viagens e estão fora do escopo dessa solução.

- Gateway de Pagametnos: Que permite o pagamento da reserva;

- Serviços de Email: Que envia a confirmação de email para o cliente quando tudo for finalizado

O processo crítico é a confirmação da reserva: o cliente escolhe data e quantidade de vagas, o sistema verifica a disponibilidade no gateway de viagens, pede a cobrança ao gateway de pagamento e, se o pagamento for aprovado, grava a reserva e dispara o e-mail de confirmação. Esse fluxo é o que o diagrama de sequência deve detalhar. O critico aqui é o pagamento é realizado fora do gateway de viagens, então existe duas decisões:

1 - Cobrar e depois marcar a reserva ( o preço pode mudar nesse tempo de 1 seg );
    -> Cliente paga X
    -> Ao marcar a reserva o preço flutuou e ficou mais caro;
        -> Caso isso aconteça, reserva passa por um processo humano de negociação com os fornecedores;
    -> Ao marcar a reserva o preço persistiu:
        -> Tudo certo com a reserva;

2 - Marcar, cobrar e se der falha cancelar a reserva
    -> Operação clara, porém, há hotéis que não permitem cancelamento gratuito;
    -> Operadora leva prejuizo;

Por uma decisão de negócio, optou-se pelo item 1.

# Contexto C4 — Aplicação Web de Reservas

A aplicação web é o único sistema da solução. Cliente e equipe de negociação são pessoas. Gateway de Viagens, Gateway de Pagamentos e Serviços de E-mail ficam fora da fronteira. A equipe entra no contexto porque a divergência de preço não é resolvida por sistema: ela é o caminho de exceção da decisão de cobrar primeiro.

```mermaid
C4Context
    title Contexto — Aplicação Web de Reservas

    %% Linha 1: atores
    Person(cliente, "Cliente", "Consulta região e datas, escolhe o quarto e paga a reserva.")
    Person(negociacao, "Equipe de negociação", "Trata a reserva quando o preço sobe entre a cobrança e a marcação.")

    %% Linha 2: sistema principal e gateway operacional
    System(reservas, "Aplicação Web de Reservas", "Orquestra consulta, cobrança, marcação e confirmação. Sistema em foco.")
    System_Ext(viagens, "Gateway de Viagens", "Informa hotéis e quartos por região e data e executa a marcação da reserva.")

    %% Linha 3: serviços de apoio
    System_Ext(pagamentos, "Gateway de Pagamentos", "Cobra o valor cotado da reserva.")
    System_Ext(email, "Serviços de E-mail", "Envia a mensagem ao cliente.")
    Rel_D(cliente, reservas, "Consulta, seleciona e paga")
    Rel_D(negociacao, viagens, "Negocia com os fornecedores")
    Rel_R(reservas, viagens, "Consulta disponibilidade e marca a reserva")
    Rel_D(reservas, pagamentos, "Solicita a cobrança do preço cotado")
    Rel_D(reservas, email, "Solicita o envio da confirmação ao cliente")
    Rel_L(negociacao, reservas, "Acompanha reserva com preço divergente")

    UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="1")
```

# Sequência — Confirmação da reserva

O fluxo segue a decisão de negócio de cobrar antes de marcar. O cliente paga o preço cotado e só depois a aplicação pede a marcação ao Gateway de Viagens. A comparação acontece na resposta desse gateway. Preço igual encerra o processo com gravação e e-mail de confirmação. Preço maior deixa a cobrança feita e abre negociação humana. Nesse segundo caso o e-mail não confirma a reserva; ele avisa que ela está em análise.

```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant App as Aplicação Web de Reservas
    participant GV as Gateway de Viagens
    participant GP as Gateway de Pagamentos
    participant Email as Serviços de E-mail
    actor Negociacao as Equipe de negociação

    Cliente->>App: Informa região, datas e quantidade de vagas
    App->>GV: Consulta disponibilidade e preço
    GV-->>App: Quartos disponíveis e preço cotado

    alt Sem disponibilidade
        App-->>Cliente: Informa indisponibilidade
    else Há vaga no preço cotado
        App-->>Cliente: Apresenta quarto e preço cotado
        Cliente->>App: Confirma a reserva e autoriza o pagamento
        App->>GP: Cobra o preço cotado
        GP-->>App: Resultado da cobrança

        alt Pagamento recusado
            App-->>Cliente: Informa pagamento não aprovado
        else Pagamento aprovado
            Note over App,GV: A marcação ocorre depois da cobrança. Nessa janela o preço pode mudar.
            App->>GV: Marca a reserva pelo preço cotado
            GV-->>App: Resultado da marcação e preço vigente

            alt Preço vigente igual ao cotado
                App->>App: Grava a reserva confirmada
                App->>Email: Solicita e-mail de confirmação
                Email-->>Cliente: Envia confirmação da reserva
                App-->>Cliente: Reserva confirmada
            else Preço vigente maior que o cotado
                App->>App: Grava a reserva em negociação, com o pagamento já capturado
                App->>Negociacao: Encaminha a diferença de preço
                Negociacao->>GV: Negocia a condição com os fornecedores
                App->>Email: Solicita aviso de reserva em análise
                Email-->>Cliente: Envia aviso de negociação em andamento
            end
        end
    end
```
