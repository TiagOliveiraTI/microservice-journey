# Passo a passo para rodar o Kafka com PHP 

Nesse passo a passo, vamos adicionar as libs necessárias para conectar o PHP ao Kafka

1. o PHP possui um lib nativa para conectar ao Kafkam chamada RDKafka, vamos instalar ela na nossa imagem docker

    Adicione essas libs: autoconf librdkafka-dev

    e entao instale via PECL: RUN pecl install rdkafka

    RUN ln -s /usr/local/etc/php/php.ini-development /usr/local/etc/php/php.ini && \
    echo "extension=rdkafka.so" >> /usr/local/etc/php/php.ini

2. Como estamos usando o Symfony, vamos adicionar as seguintes dependencias:

    ```shell
    composer require enqueue/async-event-dispatcher enqueue/enqueue-bundle  enqueue/rdkafka symfony/messenger
    ```

3. Adicione as seguintes linhas ao Bundle do Symfony:

    ```php
    Enqueue\Bundle\EnqueueBundle::class => ['all' => true],
    Enqueue\MessengerAdapter\Bundle\EnqueueAdapterBundle::class => ['all' => true],
    ```

4. Adicione tambem as variaveis de ambiente no arquivo .env

    ```env
    ###> messenger ###
    MESSENGER_TRANSPORT_DSN=enqueue://default?topic[name]=customers&queue[name]=customers
    KAFKA_BROKER_LIST=kafka
    # KAFKA_BROKER_LIST=localhost:9092,node-2.kafka.host:9092,node-3.kafka.host:9092
    ###< messenger ###

    ```

5. crie o seguinte arquivo
    ```yaml
    # config/packages/enqueue.yaml
    enqueue:
        default:
            transport:
            dsn: "rdkafka://"
            global:
                group.id: 'myapp'
                metadata.broker.list: "%env(KAFKA_BROKER_LIST)%"
            topic:
                auto.offset.reset: beginning
            commit_async: true
            client: ~

    ```

6. Crie os seguintes arquivos
    ```php
    # src/Message/KafkaMessage.php
    <?php

    declare(strict_types=1);

    namespace App\Message;

    class KafkaMessage
    {
        public function __construct(
            private string $content,
        ) {
        }

        public function getContent(): string
        {
            return $this->content;
        }
    }
    ```

    ```php
    # src/MessageHandler/KafkaHandler.php
    <?php

    declare(strict_types=1);

    namespace App\MessageHandler;

    use App\Message\KafkaMessage;
    use Symfony\Component\Messenger\Attribute\AsMessageHandler;

    #[AsMessageHandler]
    class KafkaHandler
    {
        public function __invoke(KafkaMessage $message)
        {
            # faz alguma coisa
        }
    }
    ```

7. Altere o seguinte arquivo:

    ```yaml
    # config/packages/messenger.yaml
    framework:
        messenger:
            # failure_transport: failed

            transports:
                # https://symfony.com/doc/current/messenger.html#transport-configuration
                async:
                    dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                    options:
                        topic:
                            name: customers
                        # queue:
                        #     customers: ~
                    retry_strategy:
                        max_retries: 3
                        multiplier: 2
                failed: 'doctrine://default?queue_name=failed' # configurar a fila de falha
                # sync: 'sync://'

            routing:
                Symfony\Component\Mailer\Messenger\SendEmailMessage: async
                Symfony\Component\Notifier\Message\ChatMessage: async
                Symfony\Component\Notifier\Message\SmsMessage: async
                App\Message\KafkaMessage: async
                # Route your messages to the transports
                # 'App\Message\YourMessage': async

    ```