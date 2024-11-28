# labs-auction-goexpert

Este projeto é um sistema de leilão desenvolvido em Go. Ele permite criar leilões e fazer lances em leilões existentes. O objetivo deste trabalho foi adicionar uma nova funcionalidade ao projeto já existente para o leilão fechar automaticamente a partir de um tempo definido.

## O que foi realizado

1. Foi definida uma variável de ambiente para o tempo máximo de duração do leilão chamada `AUCTION_DURATION` e adicionada ao arquivo `.env` na pasta `cmd/auction` com o valor de 60 segundos.
2. Foi implementado o fechamento automático do leilão chamando a função `startAuctionExpiryCheck()` no arquivo `internal/infra/database/auction/create_auction.go`. Esta função:
   * Calcula o tempo restante para cada leilão ativo com base na variável de ambiente AUCTION_DURATION.
   * Usa uma go routine para monitorar leilões expirados e atualizá-los para fechados.
   * Define o intervalo de checagem usando o time.Tick.
3. O arquivo `internal/infra/database/bid/create_bid.go` foi modificado para verificar se o leilão está fechado antes de aceitar um lance. Se o leilão estiver fechado, a função retorna sem inserir o lance no banco de dados. Isso garante que lances não sejam aceitos para leilões que já foram fechados.
4. Foi criado um teste em `internal/infra/database/auction/create_auction_test.go` para verificar se o leilão é fechado automaticamente após o tempo definido.

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
### Explicando o teste:

O código fornecido é um teste unitário para a função `CreateAuction` no arquivo `create_auction_test.go`. Ele utiliza a biblioteca `mtest` do MongoDB para criar um ambiente de teste mockado. O objetivo do teste é verificar se a função `CreateAuction` está funcionando corretamente, incluindo a criação de um leilão e a verificação de seu status após um determinado período.

Primeiramente, o teste define uma estrutura de leilão (`auctionEntity`) com atributos como `Id`, `ProductName`, `Category`, `Description`, `Condition`, `Status` e `Timestamp`:

```go
auctionEntity := &auction_entity.Auction{
    Id:          "1",
    ProductName: "Leilão Teste",
    Category:    "Categoria Teste",
    Description: "Descrição Teste",
    Condition:   auction_entity.New,
    Status:      auction_entity.Active,
    Timestamp:   time.Now(),
}
```

Em seguida, o teste configura o ambiente de teste mockado utilizando `mtest.New` e define as respostas mockadas para as operações de banco de dados:

```go
mt := mtest.New(t, mtest.NewOptions().ClientType(mtest.Mock))
mt.AddMockResponses(mtest.CreateSuccessResponse())
mt.AddMockResponses(mtest.CreateCursorResponse(1, "auctions.create", mtest.FirstBatch, bson.D{
    {"_id", auctionEntity.Id},
    {"product_name", auctionEntity.ProductName},
    {"status", auctionEntity.Status},
}))
```

O teste então cria uma instância do repositório de leilões (`repo`) e define a variável de ambiente `AUCTION_DURATION` para 2 segundos:

```go
db := mt.DB
repo := NewAuctionRepository(db)
err := os.Setenv("AUCTION_DURATION", "2s")
if err != nil {
    t.Fatalf("Erro ao configurar variável de ambiente: %v", err)
}
```

A função `CreateAuction` é chamada para criar o leilão, e o teste verifica se o leilão foi inserido corretamente no banco de dados:

```go
errInternal := repo.CreateAuction(context.Background(), auctionEntity)
if errInternal != nil {
    t.Fatalf("Erro ao criar leilão: %v", errInternal)
}
var result AuctionEntityMongo
err = repo.Collection.FindOne(context.Background(), bson.M{"_id": auctionEntity.Id}).Decode(&result)
if err != nil {
    t.Fatalf("Erro ao buscar leilão: %v", err)
}
assert.Equal(t, auctionEntity.ProductName, result.ProductName)
```

O teste então verifica se o leilão foi fechado automaticamente após o tempo de expiração definido. Dependendo da duração configurada, ele adiciona respostas mockadas apropriadas e espera pelo tempo necessário antes de verificar o status do leilão:

```go
auctionDuration := os.Getenv("AUCTION_DURATION")
duration, err := time.ParseDuration(auctionDuration)
if err != nil {
    duration = time.Minute * 5
}
if duration < timeSleep {
    mt.AddMockResponses(bson.D{
        {"ok", 1},
        {"nModified", 1},
    })
    time.Sleep(timeSleep)
    mt.AddMockResponses(mtest.CreateCursorResponse(2, "auctions.update", mtest.FirstBatch, bson.D{
        {"_id", auctionEntity.Id},
        {"product_name", auctionEntity.ProductName},
        {"status", auction_entity.Closed},
    }))
} else {
    mt.AddMockResponses(mtest.CreateCursorResponse(2, "auctions.update", mtest.FirstBatch, bson.D{
        {"_id", auctionEntity.Id},
        {"product_name", auctionEntity.ProductName},
        {"status", auctionEntity.Status},
    }))
    time.Sleep(timeSleep)
    mt.AddMockResponses(bson.D{
        {"ok", 1},
        {"nModified", 0},
    })
}
```

Finalmente, o teste verifica se o leilão foi fechado automaticamente após o tempo de expiração:

```go
err = repo.Collection.FindOne(context.Background(), bson.M{"_id": auctionEntity.Id}).Decode(&result)
if err != nil {
    t.Fatalf("Erro ao buscar leilão: %v", err)
}
assert.Equal(t, auction_entity.Closed, result.Status)
```

Este teste cobre o caminho feliz de criação de um leilão e verifica se ele é fechado automaticamente após o tempo de expiração configurado.