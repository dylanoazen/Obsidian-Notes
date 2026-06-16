

# Protocols

Um protocolo é um conjunto de regras que define como solicitar dados e como eles são transmitidos e recebidos em uma rede. Sem protocolos, o receptor não saberia onde a mensagem termina ou como interpretá-la.

## TCP vs UDP

| | [[TCP]] | [[UDP]] |
|---|---|---|
| Conexão | Fixada entre dois lados | Sem conexão fixa |
| Entrega | Garantida | Não garantida |
| Ordem | Garantida | Não garantida |
| Velocidade | Mais lento | Mais rápido |
| Caso de uso | Chat, API, transferência de arquivos | Games, streaming, chamadas de vídeo |

**TCP** é como uma ligação telefônica — a linha permanece aberta entre os dois lados até alguém desligar. Tudo que é enviado vai diretamente para o outro lado, em ordem.

**UDP** é como enviar uma carta — cada pacote é independente, pode tomar rotas diferentes e não há garantia de que chegará ou chegará em ordem.

### Quando usar cada um:
- **TCP** → quando ordem e entrega são importantes
- **UDP** → quando velocidade importa mais do que perfeição

## TCP

TCP é um byte stream — ele não preserva limites de mensagem:
- Uma mensagem pode chegar fragmentada
- Duas mensagens podem chegar ao mesmo tempo (coalesced)

É por isso que você precisa de framing.

## Framing

Framing é uma técnica para marcar onde uma mensagem começa e onde ela termina. Veja [[Framing]] para o detalhamento completo.

### Tipos de Framing

