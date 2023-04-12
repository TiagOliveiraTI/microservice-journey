FROM confluentinc/cp-kafka:6.1.1 AS kafka

FROM confluentinc/cp-zookeeper:3.7.0 AS zookeeper

FROM alpine:3.14.0

COPY --from=kafka /usr/bin/kafka-* /usr/bin/
COPY --from=zookeeper /usr/bin/zookeeper-server-start /usr/bin/zookeeper-server-stop /usr/bin/
COPY --from=zookeeper /usr/share/java/* /usr/share/java/

RUN apk add --no-cache bash

CMD ["bash"]
