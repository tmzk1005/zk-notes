---
title: "Spring-Cloud-Gateway请求转换并审计"
date: 2021-11-11T14:00:00+08:00
draft: false
toc: false
---

作为一个网关组件，有时候需要做协议转化，或者需要将请求和响应的内容审计下来，虽然Spring-Cloud-Gateway内部有提供了`ModifyRequestBodyGatewayFilterFactory`和`ModifyResponseBodyGatewayFilterFactory`，但是不够通用， 也没有审计功能，因此需要模仿它们实现自己的协议转换器，把转换的过程抽象出来。

### 转换过程抽象接口定义

ProtocolConverter.java
```java
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;

public interface ProtocolConverter {

    String PROTOCOL_CONVERT_EXCEPTION = "protocolConvertException";

    HeadersAndBody convertRequest(Flux<DataBuffer> rawBody, ServerWebExchange exchange) throws ProtocolConvertException;

    HeadersAndBody convertResponse(Flux<DataBuffer> rawBody, ServerWebExchange exchange) throws ProtocolConvertException;

}
```

使用到的异常类：

ProtocolConvertException.java
```java
public class ProtocolConvertException extends Exception {

    public ProtocolConvertException(String message) {
        super(message);
    }

    public ProtocolConvertException(Throwable throwable) {
        super(throwable);
    }

}
```

以及打包Header和Body部分的数据结构：

HeadersAndBody.java
```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpHeaders;
import reactor.core.publisher.Flux;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class HeadersAndBody {

    private HttpHeaders httpHeaders;

    private Flux<DataBuffer> body;

}
```

### 转换Filter

ProtocolConvertFilter.java
```java

import java.util.List;
import java.util.Objects;
import java.util.function.Function;

import lombok.extern.slf4j.Slf4j;
import org.reactivestreams.Publisher;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.factory.rewrite.CachedBodyOutputMessage;
import org.springframework.cloud.gateway.support.BodyInserterContext;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ReactiveHttpOutputMessage;
import org.springframework.http.codec.HttpMessageReader;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.http.server.reactive.ServerHttpResponseDecorator;
import org.springframework.lang.NonNull;
import org.springframework.web.reactive.function.BodyInserter;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.client.ClientResponse;
import org.springframework.web.reactive.function.server.HandlerStrategies;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import static org.springframework.cloud.gateway.support.ServerWebExchangeUtils.ORIGINAL_RESPONSE_CONTENT_TYPE_ATTR;

@Slf4j
public class ProtocolConvertFilter implements GatewayFilter {

    private final List<HttpMessageReader<?>> messageReaders;

    private final ProtocolConverter protocolConverter;

    public ProtocolConvertFilter(List<HttpMessageReader<?>> messageReaders, @NonNull ProtocolConverter protocolConverter) {
        this.messageReaders = messageReaders;
        this.protocolConverter = protocolConverter;
        Objects.requireNonNull(protocolConverter, "protocolConverter can not be null!");
    }

    public ProtocolConvertFilter(ProtocolConverter protocolConverter) {
        this(HandlerStrategies.withDefaults().messageReaders(), protocolConverter);
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String key = ProtocolConvertFilter.class.getName() + "#" + protocolConverter.getClass().getName();
        if (Boolean.TRUE.equals(exchange.getAttributeOrDefault(key, false))) {
            return chain.filter(exchange);
        }
        exchange.getAttributes().put(key, Boolean.TRUE);

        ServerRequest serverRequest = ServerRequest.create(exchange, messageReaders);

        HeadersAndBody headersAndBody;
        try {
            headersAndBody = protocolConverter.convertRequest(serverRequest.bodyToFlux(DataBuffer.class), exchange);
        } catch (ProtocolConvertException exception) {
            log.error("Exception happened while convert request", exception);
            return Mono.error(exception);
        }

        BodyInserter<Flux<DataBuffer>, ReactiveHttpOutputMessage> bodyInserter = BodyInserters.fromPublisher(headersAndBody.getBody(), DataBuffer.class);

        HttpHeaders requestHeaders = new HttpHeaders();
        requestHeaders.putAll(headersAndBody.getHttpHeaders());
        CachedBodyOutputMessage outputMessage = new CachedBodyOutputMessage(exchange, requestHeaders);

        return bodyInserter.insert(outputMessage, new BodyInserterContext())
                .then(
                        Mono.defer(() -> {
                            Exception exception = exchange.getAttribute(ProtocolConverter.PROTOCOL_CONVERT_EXCEPTION);
                            if (Objects.nonNull(exception)) {
                                return Mono.error(exception);
                            }
                            ServerHttpRequest request = decorate(exchange, requestHeaders, outputMessage);
                            ServerHttpResponse response = new ConvertedHttpResponse(exchange);
                            ServerWebExchange newExchange = exchange.mutate().request(request).response(response).build();
                            return chain.filter(newExchange);
                        })
                ).onErrorResume((Function<Throwable, Mono<Void>>) throwable -> outputMessage.getBody().map(DataBufferUtils::release).then(Mono.error(throwable)));
    }

    private ServerHttpRequestDecorator decorate(
            ServerWebExchange exchange,
            HttpHeaders headers,
            CachedBodyOutputMessage outputMessage
    ) {
        return new ServerHttpRequestDecorator(exchange.getRequest()) {

            @Override
            @NonNull
            public HttpHeaders getHeaders() {
                long contentLength = headers.getContentLength();
                HttpHeaders httpHeaders = new HttpHeaders();
                httpHeaders.putAll(headers);
                if (contentLength > 0) {
                    httpHeaders.setContentLength(contentLength);
                } else {
                    httpHeaders.set(HttpHeaders.TRANSFER_ENCODING, "chunked");
                }
                return httpHeaders;
            }

            @Override
            @NonNull
            public Flux<DataBuffer> getBody() {
                return outputMessage.getBody();
            }
        };
    }

    private class ConvertedHttpResponse extends ServerHttpResponseDecorator {

        private final ServerWebExchange exchange;

        public ConvertedHttpResponse(ServerWebExchange exchange) {
            super(exchange.getResponse());
            this.exchange = exchange;
        }

        @Override
        @NonNull
        public Mono<Void> writeWith(@NonNull Publisher<? extends DataBuffer> body) {
            String originalResponseContentType = exchange.getAttribute(ORIGINAL_RESPONSE_CONTENT_TYPE_ATTR);
            HttpHeaders httpHeaders = new HttpHeaders();
            httpHeaders.add(HttpHeaders.CONTENT_TYPE, originalResponseContentType);
            ClientResponse clientResponse = prepareClientResponse(body, httpHeaders);

            HeadersAndBody headersAndBody;
            try {
                headersAndBody = protocolConverter.convertResponse(clientResponse.bodyToFlux(DataBuffer.class), exchange);
            } catch (ProtocolConvertException exception) {
                log.error("Exception happened while convert response", exception);
                return Mono.error(exception);
            }

            BodyInserter<Flux<DataBuffer>, ReactiveHttpOutputMessage> bodyInserter = BodyInserters.fromPublisher(headersAndBody.getBody(), DataBuffer.class);
            CachedBodyOutputMessage outputMessage = new CachedBodyOutputMessage(exchange, headersAndBody.getHttpHeaders());
            return bodyInserter.insert(outputMessage, new BodyInserterContext())
                    .then(Mono.defer(() -> {
                        Exception exception = exchange.getAttribute(ProtocolConverter.PROTOCOL_CONVERT_EXCEPTION);
                        if (Objects.nonNull(exception)) {
                            return Mono.error(exception);
                        }
                        Mono<DataBuffer> messageBody = DataBufferUtils.join(outputMessage.getBody());
                        HttpHeaders newResponseHeader = headersAndBody.getHttpHeaders();
                        if (!newResponseHeader.containsKey(HttpHeaders.TRANSFER_ENCODING) || newResponseHeader.containsKey(HttpHeaders.CONTENT_LENGTH)) {
                            messageBody = messageBody.doOnNext(data -> newResponseHeader.setContentLength(data.readableByteCount()));
                        }
                        return getDelegate().writeWith(messageBody);
                    }));
        }

        @Override
        @NonNull
        public Mono<Void> writeAndFlushWith(@NonNull Publisher<? extends Publisher<? extends DataBuffer>> body) {
            return writeWith(Flux.from(body).flatMapSequential(p -> p));
        }

        private ClientResponse prepareClientResponse(@NonNull Publisher<? extends DataBuffer> body, HttpHeaders httpHeaders) {
            return ClientResponse.create(Objects.requireNonNull(exchange.getResponse().getStatusCode()), messageReaders)
                    .headers(headers -> headers.putAll(httpHeaders))
                    .body(Flux.from(body))
                    .build();
        }

    }

}
```
