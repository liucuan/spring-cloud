## Gateway 工作机制
Spring cloud gateway 的工作机制大体如下：

① Gateway 接收客户端请求。

② 客户端请求与路由信息进行匹配，匹配成功的才能够被发往相应的下游服务。

③ 请求经过 Filter 过滤器链，执行 pre 处理逻辑，如修改请求头信息等。

④ 请求被转发至下游服务并返回响应。

⑤ 响应经过 Filter 过滤器链，执行 post 处理逻辑。

⑥ 向客户端响应应答。

![avatar](spring_cloud_gateway_diagram.png)


## 基本组件介绍
### Route
Route 是 gateway 中最基本的组件之一，表示一个具体的路由信息载体。
```
public class Route implements Ordered {

	private final String id;

	private final URI uri;

	private final int order;

	private final AsyncPredicate<ServerWebExchange> predicate;

	private final List<GatewayFilter> gatewayFilters;
```
Route 主要定义了如下几个部分：

①  id，标识符，区别于其他 Route。

②  destination uri，路由指向的目的地 uri，即客户端请求最终被转发的目的地。

③  order，用于多个 Route 之间的排序，数值越小排序越靠前，匹配优先级越高。

④  predicate，谓语，表示匹配该 Route 的前置条件，即满足相应的条件才会被路由到目的地 uri。

⑤  gateway filters，过滤器用于处理切面逻辑，如路由转发前修改请求头等。

###AsyncPredicate
Predicate 即 Route 中所定义的部分，用于条件匹配，请参考 Java 8 提供的  Predicate 和 Function。
```
public interface AsyncPredicate<T> extends Function<T, Publisher<Boolean>> {

	default AsyncPredicate<T> and(AsyncPredicate<? super T> other) {
		Assert.notNull(other, "other must not be null");

		return t -> Flux.zip(apply(t), other.apply(t))
				.map(tuple -> tuple.getT1() && tuple.getT2());
	}

	default AsyncPredicate<T> negate() {
		return t -> Mono.from(apply(t)).map(b -> !b);
	}

	default AsyncPredicate<T> or(AsyncPredicate<? super T> other) {
		Assert.notNull(other, "other must not be null");

		return t -> Flux.zip(apply(t), other.apply(t))
				.map(tuple -> tuple.getT1() || tuple.getT2());
	}

}
```
AsyncPredicate 定义了 3 种逻辑操作方法：

①  and ，与操作，即两个 Predicate 组成一个，需要同时满足。

②  negate，取反操作，即对 Predicate 匹配结果取反。

③  or，或操作，即两个 Predicate 组成一个，只需满足其一。
###GatewayFilter
```
public interface GatewayFilter extends ShortcutConfigurable {

   	String NAME_KEY = "name";
 
   	String VALUE_KEY = "value";
 
   	Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```
Filter 最终是通过 filter chain 来形成链式调用的，每个 filter 处理完 pre filter 逻辑后委派给  filter chain，filter chain 再委派给下一下 filter。
```
public interface GatewayFilterChain {

	/**
	 * Delegate to the next {@code WebFilter} in the chain.
	 * @param exchange the current server exchange
	 * @return {@code Mono<Void>} to indicate when request handling is complete
	 */
	Mono<Void> filter(ServerWebExchange exchange);

}
```
###构建Route

####1.外部化配置

#####1.1Hello World
```
spring:
     cloud:
       gateway: # ①
         routes: # ②
         - id: cookie_route # ③
           uri: http://example.org # ④
           predicates: # ⑤
           - Cookie=chocolate, ch.p # ⑥
           filters: # ⑦ 
           - AddRequestHeader=X-Request-Foo, Bar # ⑧
```
① "spring.cloud.gateway" 为固定前缀。

② 定义路由信息列表，即可定义多个路由。

③ 声明了一个 id 为 "cookie_route" 的路由。

④ 定义了路由的目的地 uri，即请求转发的目的地。

⑤ 声明 predicates，即请求满足相应的条件才能匹配成功。

⑥ 定义了一个 Predicate，当名称为 chocolate 的 Cookie  的值匹配ch.p时 Predicate 才能够匹配，它由 CookieRoutePredicateFactory 来生产。

⑦ 声明 filters，即路由转发前后处理的过滤器。

⑧ 定义了一个 Filter，所有的请求转发至下游服务时会添加请求头 X-Request-Foo:Bar ，由AddRequestHeaderGatewayFilterFactory 来生产。

####2.编程方式
```
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) { // ①
    return builder.routes() // ②
            .route(r -> r.host("**.abc.org").and().path("/image/png") // ③
                .filters(f ->
                        f.addResponseHeader("X-TestHeader", "foobar")) // ④
                .uri("http://httpbin.org:80") // ⑤
            )
            .build();
}
```
① RouteLocatorBuilder bean 在 spring-cloud-starter-gateway 模块自动装配类中已经声明，可直接使用。RouteLocator 封装了对 Route 获取的定义，可简单理解成工厂模式。

② RouteLocatorBuilder 可以构建多个路由信息。

③ 指定了 Predicates，这里包含两个：

请求头Host需要匹配**.abc.org，通过 HostRoutePredicateFactory 产生。

请求路径需要匹配/image/png，通过 PathRoutePredicateFactory 产生。

④ 指定了一个 Filter，下游服务响应后添加响应头X-TestHeader:foobar，通过AddResponseHeaderGatewayFilterFactory 产生。

⑤ 指定路由转发的目的地 uri。

如果说外部化配置完全是个黑盒，那么通过编程的方式使开发者向白盒靠近了一步。因为开发者需要使用 gateway 的 api，需要开发者对其有工作机制有所了解。

###Route 构建的原理


