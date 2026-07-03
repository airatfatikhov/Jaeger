# Для развертывания полного стека Jaeger (версии 2.x, которая является современной и рекомендуемой) через Docker Compose, обычно используются следующие компоненты:
* Jaeger All-in-One (или отдельные микросервисы): В новых версиях рекомендуется использовать образ jaegertracing/all-in-one для простоты или отдельные компоненты для продакшена.
* Elasticsearch (или Cassandra): Для хранения трейсов (persistent storage). Jaeger по умолчанию использует in-memory хранилище в режиме all-in-one, но для реального использования нужна внешняя БД.
* Jaeger Query: UI для просмотра трейсов.
* Jaeger Collector: Принимает спаны от приложений.
* Jaeger Ingester (опционально): Если используется Kafka как буфер.
* Kafka (опционально): Для буферизации данных между Collector и Ingester.

# 🧪 Быстрый тест (curl)
Проверим, что collector принимает данные:
````
curl -X POST http://localhost:14268/api/traces \
  -H "Content-Type: application/json" \
  -d '{
    "data": [{
      "traceID": "test123",
      "spans": [{
        "traceID": "test123",
        "spanID": "span1",
        "operationName": "test-op",
        "startTime": 1700000000000000,
        "duration": 100000,
        "processID": "p1"
      }],
      "processes": {"p1": {"serviceName": "test-service"}}
    }]
  }'
````

#⚙️ Как настроить sampling
Вариант 1: Через переменные окружения (в приложении)
````
# Сохранять 10% трейсов
JAEGER_SAMPLER_TYPE=probabilistic
JAEGER_SAMPLER_PARAM=0.1

# Или сохранять все (для отладки)
JAEGER_SAMPLER_TYPE=const
JAEGER_SAMPLER_PARAM=1
````

Вариант 2: Через конфигурационный файл Jaeger Agent|Collector
````
{
  "service_strategies": [
    {
      "service": "frontend",
      "type": "probabilistic",
      "param": 0.1,
      "operation_strategies": [
        {
          "operation": "/health",
          "type": "const",
          "param": 0
        },
        {
          "operation": "/checkout",
          "type": "const",
          "param": 1
        }
      ]
    },
    {
      "service": "payment-service",
      "type": "ratelimiting",
      "param": 5
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.01
  }
}
````

# 🧪 Как проверить, что sampling работает
В Jaeger UI откройте трейс и посмотрите на поле Sampling Probability — там будет указано, с какой вероятностью этот трейс был отобран. Если видите 1.0 — трейс сохранён принудительно, если 0.01 — попал по вероятности 1%.