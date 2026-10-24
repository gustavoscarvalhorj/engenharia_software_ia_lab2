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
