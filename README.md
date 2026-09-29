# Finam API SDK

Версия API: [2.19.0 (13.08.2026)](https://api.finam.ru/changelog/2-19-0/)

Документация: [https://api.finam.ru/getting-started/](https://api.finam.ru/getting-started/)

## Поддержка API 2.18-2.19

Rust-типы сгенерированы из официальных protobuf-схем [тега `2.19.0`](https://github.com/FinamWeb/finam-trade-api/tree/ac0abddcd07d4e1bc68a25181434ba628ea9d64e/proto).

| Релиз | Изменения в SDK |
| --- | --- |
| [2.18.0](https://api.finam.ru/changelog/2-18-0/) | `assets::Constituents.weight: Option<Decimal>` - вес инструмента в индексе, возвращаемый `get_constituents`. |
| [2.18.1](https://api.finam.ru/changelog/2-18-1/) | `assets::GetAssetParamsResponse.trade_lot_size: i64` - размер лота для торговых операций; `0` означает отсутствие значения. |
| [2.19.0](https://api.finam.ru/changelog/2-19-0/) | `marketdata::{Bar, Quote, Trade, StreamOrderBook}.is_data_snapshot: bool` - признак начального снепшота, позволяющий отличить его от последующих обновлений. |

Типы доступны через `finam::proto::grpc::tradeapi::v1`, а `Decimal` - через `finam::proto::google::r#type`.
Вес сохраняется как десятичная строка без преобразования в число с плавающей точкой; `None` отличается от явно переданного нулевого веса.
Признак `is_data_snapshot` находится в элементах ответов подписок, а не в самом ответе: `bars`, `quote`, `trades` и `order_book` соответственно.
При создании этих структур литералом необходимо указать новые поля либо использовать `..Default::default()`.

Остальные изменения этих релизов реализованы на стороне сервера и не требуют новых методов SDK:

- **2.18.0:** `get_asset` поддерживает неторгуемые и архивные инструменты; часть полей для них может быть не заполнена.
- **2.18.1:** после переподключения снепшот заявок сохраняет исходный `client_order_id`; исправлена передача токена в REST-методе `GET /v1/bonds/future` (SDK использует gRPC).
- **2.19.0:** улучшены сообщения об ошибках токенов; исправлен пустой `symbol` в `Trades` для инструментов, отсутствующих в справочнике.

## Пример

```rust
async fn main() {
    let secret = env::var("TOKEN").unwrap();
    let finam = finam::FinamSdk::new(&secret).await.unwrap();

    let quote_response = finam
        .market_data()
        .last_quote(QuoteRequest {
            symbol: "SBER@MISX".to_string(),
        })
        .await
        .unwrap()
        .into_inner();

    if let Some(quote) = quote_response.quote {
        println!("{:?} {:?}", quote.timestamp, quote.last);
    }

    let mut streaming = finam
        .market_data()
        .subscribe_latest_trades(SubscribeLatestTradesRequest {
            symbol: "SBER@MISX".to_string(),
        })
        .await
        .unwrap()
        .into_inner();

    loop {
        if let Some(message) = streaming.message().await.unwrap() {
            println!("{:?}", message.trades);
        }
    }
}
```

## Управление ресурсами

SDK автоматически управляет жизненным циклом JWT токенов. При создании экземпляра `FinamSdk` запускается фоновая задача, которая периодически обновляет токен каждые 10 минут.

### Автоматическая остановка фоновых задач

Когда экземпляр `FinamSdk` уничтожается (выходит из области видимости), все связанные с ним фоновые задачи обновления токенов автоматически останавливаются:

```rust
async fn example() {
    {
        let sdk = FinamSdk::new("secret").await.unwrap();
        // Использование SDK...
    } // SDK уничтожается здесь, фоновая задача остановлена

    // Фоновая задача больше не выполняется
}
```
