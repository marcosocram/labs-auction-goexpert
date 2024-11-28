# labs-auction-goexpert

Este projeto é um sistema de leilão desenvolvido em Go. Ele permite criar leilões e fazer lances em leilões existentes. O objetivo deste trabalho foi adicionar uma nova funcionalidade ao projeto já existente para o leilão fechar automaticamente a partir de um tempo definido.

## O que foi realizado

1. Foi definida uma variável de ambiente para o tempo máximo de duração do leilão chamada `AUCTION_DURATION` e adicionada ao arquivo `.env` na pasta `cmd/auction` com o valor de 60 segundos.
2. Foi implementado o fechamento automático do leilão chamando a função `startAuctionExpiryCheck()` no arquivo `internal/infra/database/auction/create_auction.go`. Esta função:
   * Calcula o tempo restante para cada leilão ativo com base na variável de ambiente AUCTION_DURATION.
   * Usa uma go routine para monitorar leilões expirados e atualizá-los para fechados.
   * Define o intervalo de checagem usando o time.Tick.
3. Foi criado um teste em `internal/infra/database/auction/create_auction_test.go` para verificar se o leilão é fechado automaticamente após o tempo definido.

## Pré-requisitos

- Go 1.20 ou superior
- Docker e Docker Compose instalados.

## Configuração

1. Clone o repositório:

   ```sh
   git clone https://github.com/marcosocram/labs-auction-goexpert.git
   cd labs-auction-goexpert
    ```

2. Configure as variáveis de ambiente:

   Crie um arquivo `.env` na pasta `cmd/auction` e adicione as seguintes variáveis:  
    ```sh
    BATCH_INSERT_INTERVAL=20s
    MAX_BATCH_SIZE=4
    AUCTION_INTERVAL=20s
    AUCTION_DURATION=60s
   
    MONGODB_URL=mongodb://admin:admin@mongodb:27017/auctions?authSource=admin
    MONGODB_DB=auctions
    ```
   
3. Inicie o serviço com Docker Compose:
    ```bash
    docker-compose up --build
    ```
   
4. Rodar o teste:
    ```bash
    go test ./...
    ```
   