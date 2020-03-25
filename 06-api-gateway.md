# API Gateway

No momento em que quebramos o Monólito, extraindo os serviços de Pagamentos e Distância, tivemos que alterar a UI do Caelum Eats.

UI passou a chamar diretamente os serviços de Pagamentos, de Distância e o Monólito e, para isso, precisou conhecer as URLs dos novos serviços.

O arquivo `environment.ts` do código da UI reflete esse fato:

####### fj33-eats-ui/src/environments/environment.ts

```typescript
export const environment = {
  production: false,
  baseUrl: '//localhost:8080',
  pagamentoUrl: '//localhost:8081',
  distanciaUrl: '//localhost:8082'
};
```

![Todos os serviços expostos {w=73}](imagens/06-api-gateway/todos-os-servicos-como-edge-services.png)

Esse cenário, em que a UI invoca diretamente as APIs de cada serviço, traz alguns problemas. Os principais deles são:

- _Falta de encapsulamento:_ expondo todos os serviços, a UI conhece detalhes sobre a implementação do sistema como um todo. E isso é problemático porque a implementação muda constantemente: novos serviços serão criados, outros serviços serão extraídos do Monólito, alguns serão divididos em dois e outros mesclados em apenas um serviço. A cada alteração, a UI deverá ser modificada em conjunto com os serviços modificados. O mesmo ocorrerá com o deploy.
- _Segurança:_ sob uma perspectiva de Segurança da Informação, há uma grande _superfície exposta_ na Arquitetura atual do Caelum Eats. Cada serviço exposto pode ser alvo de ataques.

Poderíamos criar um _edge service_, que fica exposto na fronteira da rede e funciona como um _proxy reverso_ para os demais serviços, também chamados de _downstream services_.

> Um _proxy_, ou _forward proxy_, age como intermediário entre seus clientes e a Internet. É comumente usado em redes internas de organizações para limitar acesso a redes sociais e monitorar o tráfego de rede.
> Um _reverse proxy_ age como intermediário entre a Internet e servidores de uma rede interna, afim de proteger o acesso e limitar o conhecimento dos clientes externos sobre a estrutura interna da rede.

Assim, a UI chamaria apenas esse _edge service_, sem conhecer as URLs dos _downstream services_. Aumentaríamos o encapsulamento e diminuiríamos a superfície exposta no perímetro da rede.

A API desse _edge service_ serviria como a _API pública_ da organização.

Chamadas entre serviços, feitas na rede interna, não precisariam passar por esse _edge service_.

Em uma Arquitetura de Microservices, esse tipo de _edge service_ é chamado de **API Gateway**.

> **Pattern: API Gateway**
>
> Um _edge service_ que é o ponto de entrada de uma API para seus clientes externos.

![API Gateway na fronteira da rede {w=73}](imagens/06-api-gateway/api-gateway-como-edge-services.png)

### Implementações de API Gateway

Existem diferentes implementações de API Gateway disponíveis:

- [AWS API Gateway](https://aws.amazon.com/pt/api-gateway/), disponibilizado pela Amazon na plataforma AWS
- [Kong](https://github.com/Kong/kong), open-source, um conjunto de extensões do NGINX [escritos em Lua](https://docs.konghq.com/1.4.x/getting-started/introduction/#what-is-kong-technically) que provê diversas capacidades e permite a criação de plugins
- [Traefik](https://github.com/containous/traefik), open-source, implementado em Go
- [Spring Cloud Gateway](https://cloud.spring.io/spring-cloud-gateway/), parte da plataforma Spring, implementado com o framework Web reativo Spring WebFlux
- [Zuul](https://github.com/Netflix/zuul), open-source, um projeto da Netflix feito em Java

No orquestrador de containers [Kubernetes](https://kubernetes.io/), há o conceito de [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), uma abstração que provê rotas HTTP e HTTPS de fora do cluster para [Services](https://kubernetes.io/docs/concepts/services-networking/service/) internos. Um Ingress deve ter uma implementação, chamada de [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). Um Ingress Controller mantido pelo time do Kubernetes é o NGINX. Podem ser usados como Ingress Controller o Traefik e Kong, mencionados anteriormente. Há também diversas outras implementações como [Ambassador](https://www.getambassador.io/), [Contour](https://projectcontour.io/) da VMWare, [Gloo](https://solo.io/gloo/).

#### Zuul

No artigo de lançamento do Zuul no blog de Tecnologia da Netflix, [Announcing Zuul: Edge Service in the Cloud](https://medium.com/netflix-techblog/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee) (COHEN; HAWTHORNE, 2013), é afirmado que o Zuul é usado na Netflix API de diversas formas. Entre elas:

- Autenticação e Segurança em geral
- Insights
- Teste de Stress
- Testes (ou releases) Canário, um release parcial para um subconjunto das máquinas de produção, para minimizar o impacto e o risco de uma nova versão
- Roteamento Dinâmico, que veremos em capítulos posteriores

Mikey Cohen, na palestra [Zuul @ Netflix](https://www.slideshare.net/MikeyCohen1/zuul-netflix-springone-platform) (COHEN, 2016), diz que, na arquitetura da Netflix, há mais de 20 clusters Zuul em produção, que lidam com dezenas de bilhões de requests por dia em 3 regiões AWS diferentes.

Há duas versões do Zuul, cujas diferenças são explicadas com detalhes na palestra [Zuul's Journey to Non-Blocking](https://www.youtube.com/watch?v=2oXqbLhMS_A) (GONIGBERG, 2017):

- Zuul 1, que usa a API de Servlet sobre um Tomcat
- Zuul 2, que usa I/O Assíncrono com Netty

> Curiosidade: na palestra mencionada anteriormente, Arthur Gonigberg lembra que Zuul é o nome do mostro do filme Ghostbusters conhecido como The Gatekeeper (em português, algo como porteiro).

Usaremos o projeto [Spring Cloud Netflix Zuul](https://cloud.spring.io/spring-cloud-netflix/multi/multi__router_and_filter_zuul.html), que integra o Zuul 1 ao Spring Boot.

## Implementando um API Gateway com Zuul

Pelo navegador, abra `https://start.spring.io/`.

Em _Project_, mantenha _Maven Project_.
Em _Language_, mantenha _Java_.
Em _Spring Boot_, mantenha a versão padrão.
No trecho de _Project Metadata_, defina:

- `br.com.caelum` em _Group_
- `api-gateway` em _Artifact_

Mantenha os valores em _More options_.

Mantenha o _Packaging_ como `Jar`.
Mantenha a _Java Version_ em `8`.

Em _Dependencies_, adicione:

- Zuul
- DevTools

Clique em _Generate Project_.

Extraia o `api-gateway.zip` e copie a pasta para seu Desktop.

Adicione a anotação `@EnableZuulProxy` à classe `ApiGatewayApplication`:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/ApiGatewayApplication.java

```java
@EnableZuulProxy
@SpringBootApplication
public class ApiGatewayApplication {

  public static void main(String[] args) {
    SpringApplication.run(ApiGatewayApplication.class, args);
  }

}
```

Não deixe de adicionar o import:

```java
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
```

No arquivo `src/main/resources/application.properties`:

- modifique a porta para 9999
- desabilite o Eureka, por enquanto (o abordaremos mais adiante)
- para as URLs do serviço de pagamento, parecidas com `http://localhost:9999/pagamentos/algum-recurso`, redirecione para `http://localhost:8081`. Para manter o prefixo `/pagamentos`, desabilite a propriedade `stripPrefix`.
- para as URLs do serviço de distância, algo como `http://localhost:9999/distancia/algum-recurso`, redirecione para `http://localhost:8082`. O prefixo `/distancia` será removido, já que esse é o comportamento padrão.
- para as demais URLs, redirecione para `http://localhost:8080`, o monólito.

O arquivo ficará semelhante a:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
server.port = 9999

ribbon.eureka.enabled=false

zuul.routes.pagamentos.url=http://localhost:8081
zuul.routes.pagamentos.stripPrefix=false

zuul.routes.distancia.url=http://localhost:8082

zuul.routes.monolito.path=/**
zuul.routes.monolito.url=http://localhost:8080
```

Com as configurações anteriores, a URL `http://localhost:9999/pagamentos/1`  será direcionada para `http://localhost:8081/pagamentos/1`, mantendo o prefixo `pagamentos`.

Já a URL `http://localhost:9999/distancia/restaurantes/mais-proximos/71503510` será direcionada para `http://localhost:8082/restaurantes/mais-proximos/71503510`, removendo o prefixo `distancia`.

Outras URLs, que não iniciam com `/pagamentos` ou `/distancia`, serão direcionadas para o Monólito. Por exemplo, a URL `http://localhost:9999/restaurantes/1`, será direcionada para `http://localhost:8080/restaurantes/1`.


## Fazendo a UI usar o API Gateway

Remova as URLs específicas dos serviços de distância e pagamento, mantendo apenas a `baseUrl`, que deve apontar para o API Gateway:

####### fj33-eats-ui/src/environments/environment.ts

```typescript
export const environment = {
  production: false,

  b̶a̶s̶e̶U̶r̶l̶:̶ ̶'̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶'̶
  baseUrl: '//localhost:9999' // modificado

  ,̶ ̶p̶a̶g̶a̶m̶e̶n̶t̶o̶U̶r̶l̶:̶ ̶'̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶1̶'̶
  ,̶ ̶d̶i̶s̶t̶a̶n̶c̶i̶a̶U̶r̶l̶:̶ ̶'̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶'̶
};
```

Em `PagamentoService`, troque `pagamentoUrl` por `baseUrl`:

####### fj33-eats-ui/src/app/services/pagamento.service.ts

```typescript
export class PagamentoService {

  p̶r̶i̶v̶a̶t̶e̶ ̶A̶P̶I̶ ̶=̶ ̶e̶n̶v̶i̶r̶o̶n̶m̶e̶n̶t̶.̶p̶a̶g̶a̶m̶e̶n̶t̶o̶U̶r̶l̶ ̶+̶ ̶'̶/̶p̶a̶g̶a̶m̶e̶n̶t̶o̶s̶'̶;̶
  private API = environment.baseUrl + '/pagamentos'; // modificado

  // restante do código ...

}
```

Use apenas `baseUrl` em `RestauranteService`, alterando o atributo `DISTANCIA_API`:

####### fj33-eats-ui/src/app/services/restaurante.service.ts

```typescript
export class RestauranteService {

  private API = environment.baseUrl;

  p̶r̶i̶v̶a̶t̶e̶ ̶D̶I̶S̶T̶A̶N̶C̶I̶A̶_̶A̶P̶I̶ ̶=̶ ̶e̶n̶v̶i̶r̶o̶n̶m̶e̶n̶t̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶U̶r̶l̶;̶
  private DISTANCIA_API = environment.baseUrl + '/distancia'; // modificado

  // código omitido ...

}
```

## Exercício: API Gateway com Zuul

1. Em um Terminal, clone o repositório `fj33-api-gateway` para o seu Desktop:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-api-gateway.git
  ```

  No workspace de microservices do Eclipse, importe o projeto `fj33-api-gateway`, usando o menu _File > Import > Existing Maven Projects_ e apontando para o diretório `fj33-api-gateway` do Desktop.

2. Execute a classe `ApiGatewayApplication`, certificando-se que os serviços de pagamento e distância estão no ar, assim como o monólito.

  Alguns exemplos de URLs:

  - `http://localhost:9999/pagamentos/1`
  - `http://localhost:9999/distancia/restaurantes/mais-proximos/71503510`
  - `http://localhost:9999/restaurantes/1`

  Note que as URLs anteriores, apesar de serem invocados no API Gateway, invocam o serviço de pagamento, o de distância e o monólito, respectivamente.

3. Vá até a branch `cap6-ui-chama-api-gateway` do projeto `fj33-eats-ui`:

  ```sh
  cd ~/Desktop/fj33-eats-ui
  git checkout -f cap6-ui-chama-api-gateway
  ```

4. Com o monólito, os serviços de pagamentos e distância e o API Gateway no ar, suba o front-end por meio do comando `ng serve`.

  Faça um novo pedido e efetue o pagamento. Deve funcionar!

  Tente fazer o login como administrador (`admin`/`123456`) e acessar a página de restaurantes em aprovação. Deve ocorrer um erro _401 Unauthorized_, que não acontecia antes da UI passar pelo API Gateway. Por que será que acontece esse erro?

## Desabilitando a remoção de cabeçalhos sensíveis no Zuul

Na Netflix, o Zuul é usado para Autenticação. Os tokens provenientes da UI são validados e, se o usuário for válido, é repassado para os _downstream services_.

Por padrão, o Zuul remove os cabeçalhos HTTP `Cookie`, `Set-Cookie`, `Authorization`.

Por enquanto, no Caelum Eats, não será feita nenhuma Autenticação no API Gateway.

Por isso, vamos desabilitar essa remoção no `application.properties`:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
zuul.sensitiveHeaders=
```

## Exercício: cabeçalhos sensíveis no Zuul

1. Pare o API Gateway.

  Obtenha o código da branch `cap6-cabecalhos-sensiveis-no-zuul` do projeto `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap6-cabecalhos-sensiveis-no-zuul
  ```

  Execute a classe `ApiGatewayApplication`. Zuul no ar!

2. Faça novamente login como administrador (`admin`/`123456`) e acesse a página de restaurantes em aprovação. Deve funcionar!

## API Composition no API Gateway

Após escolher um restaurante, um cliente do Caelum Eats pode ver detalhes do restaurantes como o nome, descrição, tipo de cozinha, tempos de entrega, distância ao CEP informado, cardápio e avaliações.

![Cliente vê detalhes do restaurante escolhido {w=60}](imagens/01-conhecendo-o-projeto/cliente-3-ve-cardapio.png)

Para obter todos os dados necessários para a tela de detalhes do restaurante, o front-end Angular do Caelum Eats precisa disparar as seguintes chamadas `GET` via AJAX:

- `http://localhost:9999/restaurantes/1`, para buscar os dados básicos do restaurante
- `http://localhost:9999/restaurantes/1/cardapio`, para buscar os dados do cardápio
- `http://localhost:9999/restaurantes/1/avaliacoes`, para buscar as avaliações
- `http://localhost:9999/distancia/restaurantes/71503510/restaurante/1`, para buscar a distância do CEP informado

![Chamadas AJAX para obter detalhes de um restaurante](imagens/06-api-gateway/chamadas-ajax-detalhes-de-um-restaurante.png)

As três primeiras chamadas mostradas anteriormente são para o Monólito. Poderia ser feita uma otimização no módulo de Restaurante para, dado um id de um restaurante, obter os dados básicos juntamente ao cardápio. A chamada que busca avaliações faz parte do módulo Pedido do Monólito. Se desejássemos manter a separação entre os módulos, facilitando uma posterior extração como serviço, é interessante deixar a busca de avaliações separada da busca dos dados do restaurante.

A quarta chamada é feita ao serviço de Distância, afim de obter a distância do restaurante ao CEP determinado pelo cliente. São buscadas informações, portanto, de outro serviço.

São 4 chamadas ao backend, passando pelo API Gateway, para obter os dados de uma tela.

Uma Arquitetura de Microservices vai levar a uma granularidade mais fina dos serviços. E, se não tomarmos cuidado, essa granularidade terá o efeito de aumentar o número de requisições feitas pelo front-end.

Muitas chamadas ao backend acabam impactando a performance. Cada chamada terá, além da demora do processamento nos serviços, uma latência de rede adicional. Como lembra Sam Newman, no livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015), seria um uso muito ineficiente do plano de dados de um Mobile.

No post [Optimizing the Netflix API](https://medium.com/netflix-techblog/optimizing-the-netflix-api-5c9ac715cf19) (CHRISTENSEN, 2013), Ben Christensen mostra que essa "tagarelice" (em inglês, _chattiness_) era um problema da Netflix API. Devido à natureza genérica e granular das APIs, cada chamada só retornava uma pequena porção dos dados necessários. As aplicações do Netflix para cada dispositivo (smartphones, TVs, videogames) tinham que realizar diversas chamadas para unir os dados necessários em uma determinada experiência de usuário.

A solução encontrada na Netflix, semelhante a citada por Sam Newman, é unir os vários requests necessários para cada dispositivo em um único request otimizado. O preço da latência da rede WAN (Internet) seria pago apenas uma vez e trocado por chamadas entre os servidores, numa rede LAN de menor latência. Um benefício adicional, além da eficiência, é que as chamadas na rede local são mais estáveis e confiáveis.

<!--

TODO:

Colocar figuras do post "Optimizing the Netflix API" mostrando as diversas chamadas e a latência acumulada.

https://miro.medium.com/max/1127/0*hH8iqS2HLJI9DUbz.png

https://miro.medium.com/max/1127/0*pjGnJ3RF8wdXOLrC.png

-->

Essa ideia de compor APIs de serviços, com granularidade mais fina, em uma API de granularidade maior é chamada de **API Composition**.

### API Composition e consultas

Chris Richardson, no livro [Microservice Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a), tem uma outra abordagem ao descrever API Composition: o foco é em como fazer consultas em uma Arquitetura de Microservices.

Se, com um BD Monolítico, uma consulta poderia ser feita com um SQL e alguns joins, com Microservices, os dados tem que ser recuperados de diversos BDs de serviços diferentes. Os clientes dos serviços recuperariam os dados pelas APIs do serviços e combinariam os resultados em memória. No fim das contas, trata-se de um _in-memory join_.


> **Pattern: API Composition**
>
> Implemente uma consulta que recupera dados de diversos serviços por sua API e combina os resultados.


Para Richardson, uma API Composition envolve dois participantes:

- um _API Composer_, que invoca os serviços e combina os resultados
- dois ou mais _Provider services_, que são a fonte dos dados consultados

Richardson discute onde implementar a API Composition:

- no front-end. É o caso que discutimos no Caelum Eats e na Netflix API. Há uma ineficiência no uso da rede, devido à alta latência da Internet.
- no API Gateway. Faz sentido se as consultas fazem parte da API pública da organização. A performance de rede é aumentada, pois as consultas aos _downstream services_ são feitas em uma LAN, de menor latência. Deve-se evitar lógica de negócio no API Gateway. 
- em um serviço específico para a composição. Útil para agregações invocadas internamente pelos serviços, que não fazem parte da API pública e para casos em que há uma lógica de negócio complexa envolvida na composição. Problemático quando não há um time responsável pelo serviço de composição.

Podemos implementar a API Composition no API Gateway. Temos que mantê-la estritamente técnica, tomando cuidado para que lógica de negócio não acabe vazando para o API Gateway.

### Avaliando pontos negativos de API Composition

Chris Richardson ainda discute sobre algumas desvantagens de API Composition como um todo, em qualquer uma das abordagens descritas anteriormente:

- há uma certa ineficiência em invocar _n_ APIs e realizar um _in-memory join_. Mesmo em uma rede interna, a latência vai afetar negativamente a performance da consulta. Pode haver um grande impacto especialmente em datasets maiores.
- há um impacto na disponibilidade, já que as _n_ APIs consultadas precisam estar no ar ao mesmo tempo para que a consulta seja realizada com sucesso.
- há a possibilidade de inconsistência nos dados, já que evitamos transações distribuídas.

Para mitigar algumas dessas desvantagens, Richardson sugere:

- caches, que aumentam a disponibilidade e a performance, mas impactam ainda mais a consistência, com o risco de termos dados desatualizados.
- uma solução que envolve Mensageria que discutiremos mais adiante

## API Composition no API Gateway do Caelum Eats

Conforme mencionamos anteriormente, na tela de detalhes do restaurante, a UI faz 4 chamadas AJAX:

- 3 chamadas distintas ao Monólito para buscar os dados básicos do restaurante, o cardápio e as avaliações.
- 1 chamada ao serviço de distância, para buscar a distância do restaurante ao CEP informado.

Implementaremos um API Composition em que o API Gateway buscará em 1 chamada os dados básicos do restaurante junta à distância do restaurante ao CEP.

> Poderíamos fazer um API Composition em que 1 chamada obteria todos os dados necessários para a tela de detalhe de restaurantes. Mas isso fica como um desafio!

O API Gateway com o Spring Cloud Netflix Zuul é código feito com Spring Boot.

Então, para implementar essa API Composition criaremos clientes REST que invocam o serviço de Distância e o Monólito. Utilizaremos o RestTemplate do Spring para invocar o serviço de Distância e o Feign para invocar o Monólito. Seria possível utilizar só o Feign, mas estudaremos a diferença entre o RestTemplate e Feign em integrações com algumas ferramentas do Spring Cloud.

Para expor a API Composition para a UI, utilizaremos um bom e velho `@RestController` do Spring MVC. Para não precisamos criar um _domain model_ do serviço de Distância e do Monólito no código do API Gateway, mesclaremos os dados de cada _downstream service_ com um simples `Map`.

## Invocando o serviço de distância a partir do API Gateway com RestTemplate

Adicione o Lombok como dependência no `pom.xml` do projeto `api-gateway`:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
```

Crie uma classe `RestClientConfig` no pacote `br.com.caelum.apigateway`, que fornece um `RestTemplate` do Spring:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/RestClientConfig.java

```java
@Configuration
class RestClientConfig {

  @Bean
  RestTemplate restTemplate() {
    return new RestTemplate();
  }

}
```

Faça os imports adequados:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;
```

_Observação: estamos usando o `RestTemplate` ao invés do Feign porque estudaremos a diferença entre os dois mais adiante._

Ainda no pacote `br.com.caelum.apigateway`, crie um `@Service` chamado `DistanciaRestClient` que recebe um `RestTemplate` e o valor de `zuul.routes.distancia.url`, que contém a URL do serviço de distância.

No método `comDistanciaPorCepEId`, dispare um `GET` à URL do serviço de distância que retorna a quilometragem de um restaurante a um dado CEP.

Como queremos apenas mesclar as respostas na API Composition, não precisamos de um _domain model_. Por isso, podemos usar um `Map` como tipo de retorno.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/DistanciaRestClient.java

```java
@Service
class DistanciaRestClient {

  private RestTemplate restTemplate;
  private String distanciaServiceUrl;

  DistanciaRestClient(RestTemplate restTemplate,
      @Value("${zuul.routes.distancia.url}") String distanciaServiceUrl) {
    this.restTemplate = restTemplate;
    this.distanciaServiceUrl = distanciaServiceUrl;
  }

  Map<String,Object> porCepEId(String cep, Long restauranteId) {
    String url = distanciaServiceUrl + "/restaurantes/" + cep + "/restaurante/" + restauranteId;
    return restTemplate.getForObject(url, Map.class);
  }

}
```

Ajuste os imports:

```java
import java.util.Map;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
```

Observação: é possível resolver o warning de _unchecked conversion_ usando um `ParameterizedTypeReference` com o método `exchange` do `RestTemplate`.

## Invocando o monólito a partir do API Gateway com Feign

Adicione o Feign como dependência no `pom.xml` do projeto `api-gateway`:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Na classe `ApiGatewayApplication`, adicione a anotação `@EnableFeignClients`:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/ApiGatewayApplication.java

```java
@EnableFeignClients // adicionado
@EnableZuulProxy
@SpringBootApplication
public class ApiGatewayApplication {

  public static void main(String[] args) {
    SpringApplication.run(ApiGatewayApplication.class, args);
  }

}
```

O import a ser adicionado está a seguir:

```java
import org.springframework.cloud.openfeign.EnableFeignClients;
```

Crie uma interface `RestauranteRestClient`, que define um método `porId` que recebe um `id` e retorna um `Map`. Anote esse método com as anotações do Spring Web, para que dispare um GET à URL do monólito que detalha um restaurante.

A interface deve ser anotada com `@FeignClient`, apontando para a configuração do monólito no Zuul. 

```java
@FeignClient("monolito")
interface RestauranteRestClient {

  @GetMapping("/restaurantes/{id}")
  Map<String,Object> porId(@PathVariable("id") Long id);

}
```

Ajuste os imports:

```java
import java.util.Map;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
```

A configuração do monólito no Zuul precisa ser ligeiramente alterada para que o Feign funcione:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶m̶o̶n̶o̶l̶i̶t̶o̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶
monolito.ribbon.listOfServers=http://localhost:8080
```

Mais adiante estudaremos cuidadosamente o Ribbon.

## Compondo chamadas no API Gateway

No `api-gateway`, crie um `RestauranteComDistanciaController`, que invoca dado um CEP e um id de restaurante obtém:

- os detalhes do restaurante usando `RestauranteRestClient`
- a quilometragem entre o restaurante e o CEP usando `DistanciaRestClient`

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteComDistanciaController.java

```java
@RestController
@AllArgsConstructor
class RestauranteComDistanciaController {

  private RestauranteRestClient restauranteRestClient;
    private DistanciaRestClient distanciaRestClient;

    @GetMapping("/restaurantes-com-distancia/{cep}/restaurante/{restauranteId}")
    public Map<String, Object> porCepEIdComDistancia(@PathVariable("cep") String cep,
                                                     @PathVariable("restauranteId") Long restauranteId) {
      Map<String, Object> dadosRestaurante = restauranteRestClient.porId(restauranteId);
      Map<String, Object> dadosDistancia = distanciaRestClient.porCepEId(cep, restauranteId);
      dadosRestaurante.putAll(dadosDistancia);
      return dadosRestaurante;
    }
}
```

Não esqueça dos imports:

```java
import java.util.Map;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import lombok.AllArgsConstructor;
```

Se tentarmos acessar, pelo navegador ou pelo cURL, a URL `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1` termos como status da resposta um `401 Unauthorized`.

Isso ocorre porque, como o prefixo não é `pagamentos` nem `distancia`, a requisição é repassada ao monólito pelo Zuul.

Devemos configurar uma rota no Zuul, usando o `forward` para o endereço local:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
zuul.routes.local.path=/restaurantes-com-distancia/**
zuul.routes.local.url=forward:/restaurantes-com-distancia
```

A rota acima deve ficar logo antes da rota do monólito, porque esta última é `/**`,  um "coringa" que corresponde a qualquer URL solicitada.

Um novo acesso a URL `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1` terá como resposta um JSON com os dados do restaurante e de distância mesclados.

## Chamando a composição do API Gateway a partir da UI

No projeto `eats-ui`, adicione um método que chama a nova URL do API Gateway em `RestauranteService`:

####### fj33-eats-ui/src/app/services/restaurante.service.ts

```typescript
export class RestauranteService {

  // código omitido ...

  porCepEIdComDistancia(cep: string, restauranteId: string): Observable<any> {
    return this.http.get(`${this.API}/restaurantes-com-distancia/${cep}/restaurante/${restauranteId}`);
  }

}
```

Altere ao `RestauranteComponent` para que chame o novo método `porCepEIdComDistancia`.

Não será mais necessário invocar o método `distanciaPorCepEId`, porque o restaurante já terá a distância.

####### fj33-eats-ui/src/app/pedido/restaurante/restaurante.component.ts

```java
export class RestauranteComponent implements OnInit {

  // código omitido ...

  ngOnInit() {

    t̶h̶i̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶S̶e̶r̶v̶i̶c̶e̶.̶p̶o̶r̶I̶d̶(̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶)̶
    this.restaurantesService.porCepEIdComDistancia(this.cep, restauranteId) // modificado
      .subscribe(restaurante => {

        this.restaurante = restaurante;
        this.pedido.restaurante = restaurante;

        t̶h̶i̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶S̶e̶r̶v̶i̶c̶e̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶P̶o̶r̶C̶e̶p̶E̶I̶d̶(̶t̶h̶i̶s̶.̶c̶e̶p̶,̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶)̶
          .̶s̶u̶b̶s̶c̶r̶i̶b̶e̶(̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶C̶o̶m̶D̶i̶s̶t̶a̶n̶c̶i̶a̶ ̶=̶>̶ ̶{̶
            t̶h̶i̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶C̶o̶m̶D̶i̶s̶t̶a̶n̶c̶i̶a̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶;̶
        }̶)̶;̶

        // código omitido ...

      });

  }

  // restante do código ...

}
```

Ao buscar os restaurantes a partir de um CEP e escolhermos um deles ou também ao acessar diretamente uma URL como `http://localhost:4200/pedidos/71503510/restaurante/1`, deve ocorrer um _Erro no servidor_.

No Console do navegador, podemos perceber que o erro é relacionado a CORS:

_Access to XMLHttpRequest at 'http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1' from origin 'http://localhost:4200' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource._

Para resolver o erro de CORS, devemos adicionar ao API Gateway uma classe `CorsConfig` semelhante a que temos nos serviços de pagamentos e distância e também no monólito:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/CorsConfig.java

```java
@Configuration
class CorsConfig implements WebMvcConfigurer {

  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**").allowedMethods("*").allowCredentials(true);
  }

}
```

Depois de reiniciar o API Gateway,  os detalhes do restaurante devem ser exibidos, assim como sua distância a um CEP informado.

CORS é uma tecnologia do front-end, já que é uma maneira de relaxar a _same origin policy_ de chamadas AJAX de um navegador.

Como apenas o API Gateway será chamado diretamente pelo navegador e não há restrições de chamadas entre servidores Web, podemos apagar as classes `CorsConfig` dos serviços de pagamento e distância, assim como a do módulo `eats-application` do monólito.

## Exercício: API Composition no API Gateway

1. Pare o API Gateway.

  Faça o checkout da branch `cap6-api-composition-no-api-gateway` do projeto `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap6-api-composition-no-api-gateway
  ```

  Certifique-se que o monólito e o serviço de distância estejam no ar.
  
  Rode novamente a classe `ApiGatewayApplication`.

  Tente acessar a URL `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1`

  Deve ser retornado algo parecido com:

  ```json
  {
    "restauranteId":1,
    "distancia":11.393642891403121808480136678554117679595947265625,
    "cep":"70238500",
    "descricao":"O melhor da China aqui do seu lado.",
    "endereco":"ShC/SUL COMERCIO LOCAL QD 404-BL D LJ 17-ASA SUL",
    "nome":"Long Fu",
    "taxaDeEntregaEmReais":6.00,
    "tempoDeEntregaMaximoEmMinutos":25,
    "tempoDeEntregaMinimoEmMinutos":40,
    "tipoDeCozinha":{
      "id":1,
      "nome":"Chinesa"
    },
    "id":1
  }
  ```

2. Como a UI chama apenas o API Gateway e CORS é uma tecnologia de front-end, devemos remover a classe `CorsConfig` do monólito modular e dos serviços de pagamento e distância. Essa classe já está incluída no código do API Gateway.

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap6-api-composition-no-api-gateway

  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap6-api-composition-no-api-gateway

  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap6-api-composition-no-api-gateway
  ```

  Reinicie o monólito e os serviços de pagamento e distância.

3. Obtenha o código da branch `cap6-api-composition-no-api-gateway` do projeto `fj33-eats-ui`:

  ```sh
  cd ~/Desktop/fj33-eats-ui
  git checkout -f cap6-api-composition-no-api-gateway
  ```

  Com os serviços de distância e o monólito rodando, inicie o front-end com o comando `ng serve`.

  Digite um CEP, busque os restaurantes próximos e escolha algum. Na página de detalhes de um restaurantes, chamamos a API Composition. Veja se os dados do restaurante e a distância são exibidos corretamente.

## Para saber mais: BFF

Microservices devem ser independentes. O time de desenvolvimento de um microservice deve dominar todo o seu ciclo de vida, da concepção a operação. Como disse o CTO da Amazon, Werner Vogels, em [entrevista a Association for Computing Machinery (ACM)](https://dl.acm.org/doi/pdf/10.1145/1142055.1142065?download=true) (VOEGELS, 2006): "You build it, you run it."

À medida que mais responsabilidades são colocadas no API Gateway e mais API Compositions são implementadas, múltiplos times passam a contribuir com o mesmo código. A independência dos microservices e seus times é afetada.

Para piorar o problema, raramente temos apenas uma UI. Além da Web, podemos ter UIs Mobile e/ou Desktop. Cada UI tem necessidades diferentes.  Por exemplo, uma UI Mobile vai ter menos espaço na tela e, portanto, vai exibir menos dados. Também no Mobile, há limitações de bateria e no plano de dados do usuário. E uma UI Mobile tem capacidades extra, como uma câmeras e GPS. Diferentes experiências de usuário requerem diferentes dados dos serviços e, em consequência, da API.

Como resolver isso?

<!--@note
  Alexandre Aquiles: Coloquei as figuras de alguns posts da Netflix e o post do Sam Newman sobre BFF em uns slides:
  https://slides.com/alexandreaquiles/apresentacao-api-gateway-composition-e-bff/fullscreen#/
-->

Poderíamos ter um time para a API. Só que esse time teria uma coordenação com cada _downstream service_ e também com cada UI.

Uma solução melhor, usada pela SoundCloud e descrita por Phil Calçado no post [The Back-end for Front-end Pattern (BFF)](https://philcalcado.com/2015/09/18/the_back_end_for_front_end_pattern_bff.html) (CALÇADO, 2015) é ter um API Gateway com API Compositions específicas para cada front-end. Calçado relata que Nick Fisher, tech lead do time Web da SoundCloud, cunhou o termo _Back-end for Front-end_, ou simplesmente **BFF**.

<!--@note
Calçado fala que tinha cunhado o termo BEFFE mas descobriu que era um palavrão em holandês.
https://en.wiktionary.org/wiki/beffen#Verb
-->

![BFF {w=30}](imagens/06-api-gateway/bff.png)

BFF não é uma tecnologia nem algo que você compra, baixa ou configura. BFF é uma maneira de utilizar um API Gateway mais alinhada com os times de front-end.

Os times de front-end passam a ter a necessidade de um (ou mais) desenvolvedor(es) com habilidades de back-end, que mantém o código que compõe os _downstream services_ da maneira adequada para o front-end. Phil Calçado deixa claro no post mencionado anteriormente que o BFF é parte da aplicação e pode ser usado para implementar um _presentation model_, que amplia o _domain model_ para incluir dados relevantes apenas para UI como flags e títulos de janelas.

Sam Newman, no post [Backends For Frontends](https://samnewman.io/patterns/architectural/bff/) (NEWMAN, 2015b), discute que um detalhe negativo do uso de BFFs é que pode levar a duplicação de código. Mas Newman diz que prefere aceitar um pouco de duplicação a criar abstrações que levem a uma necessidade de coordenação entre serviços. Porém, se a duplicação puder ser modelada em termos de domínio, pode ser extraída para um novo serviço.

Newman ainda sugere que, se o tipo de usuário final for diferente, é interessante termos BFFs separados. No Caelum Eats, por exemplo, se tivermos uma aplicação Web para os clientes e outra diferente para os donos de restaurante, teríamos dois BFFs distintos para cada tipo de usuário.

No mesmo post, Newman diz que um BFF pode ser usado como cache para resultados de API Composition e para server-side rendering.

### Um BFF por plataforma?

Aplicações de diferentes plataformas Mobile, como Android e iOS, devem utilizar o mesmo BFF ou não?

Isso é discutido por Sam Newman no post [Backends For Frontends](https://samnewman.io/patterns/architectural/bff/) (NEWMAN, 2015b). Para Newman, há algumas motivações para termos BFFs distintos para cada plataforma:

- se as experiências de usuário do Android e iOS são suficientemente distintas.
- se há um time distinto para as aplicações Android e iOS.

![BFF por plataforma {w=45}](imagens/06-api-gateway/bff-por-mobile.png)

Na palestra [Evoluindo uma Arquitetura inteiramente sobre APIs](https://www.infoq.com/br/presentations/evoluindo-uma-arquitetura-soundcloud/) (CALÇADO, 2013), menciona uma outra motivação para BFFs separados por plataforma: uma app iOS é publicada na Apple Store em questão de semanas ou meses, enquanto uma app Android é publicada na Play Store em questão de horas. Se tivermos apenas um BFF, teremos que manter APIs compatíveis com ambas as versões das apps. Já com um BFF, há maior controle, ainda que haja alguma duplicação.

## Zuul Filters

A descrição da arquitetura do Zuul, feita no artigo [Announcing Zuul: Edge Service in the Cloud](https://medium.com/netflix-techblog/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee) (COHEN; HAWTHORNE, 2013), revela que o Zuul é um simples proxy reverso que pode ser usado com filtros feitos em alguma linguagem da JVM. Os Zuul Filters podem realizar diferentes ações durante o roteamento de requests e responses HTTP como: roteamento dinâmico, _load balacing_, _rate limiting_, _encoding_ para devices específicos, _failover_, autenticação, etc.

Os Zuul Filters possuem as seguintes características:

- **Type**: em geral, a fase do roteamento em que o filtro será aplicado. Pode conter tipos customizados.
- **Execution Order**: define a prioridade do filtro. Quanto menor, mais prioritário.
- **Criteria**: define os critérios necessários para a execução do filtro.
- **Action**: as ações a serem efetuadas pelo filtro.

Os Zuul já contem alguns Type padronizados para os Zuul Filters:

- `pre`: executados antes do roteamento. Pode ser usado para autenticação e load balancing, por exemplo.
- `routing`: executados durante o roteamento.
- `post`: executados depois que o roteamento foi feito. Pode ser usado para manipular cabeçalhos HTTP e registrar estatísticas, por exemplo.
- `error`: executados no caso de um erro em outras fases.

Um exemplo de um filtro implementado em Java, que atrasa em 20 segundos o roteamento de requests vindos de um Master System:

```java
class DeviceDelayFilter extends ZuulFilter {

    @Override
    public String filterType() {
       return "pre";
    }

    @Override
    public int filterOrder() {
       return 5;
    }

    @Override
    public boolean shouldFilter() {
       String deviceType = RequestContext.getRequest().getParameter("deviceType");
       return deviceType != null && deviceType.equals("MasterSystem");
    }

    @Override
    public Object run() {
       Thread.sleep(20000);
    }
}
```

Filtros dinâmicos podem ser implementados em Groovy e são lidos do disco, compilados e executados dinamicamente no próximo request roteado. Um diretório do servidor é consultado periodicamente para obter mudanças ou filtros adicionais.

De acordo com a [documentação](https://cloud.spring.io/spring-cloud-netflix/multi/multi__router_and_filter_zuul.html) do Spring Cloud Netflix Zuul, já há uma série de Zuul Filters implementados em Java.

Há alguns `pre` filters, como:

- `ServletDetectionFilter`: detecta se a request passou pela Servlet Dispatcher do Spring.
- `FormBodyWrapperFilter`: faz o parse de dados de forms.
- `DebugFilter`: se `debug` estiver setado, seta outras propriedades de debug do routing e do request.
- `PreDecorationFilter`: usa o `RouteLocator` para determinar onde e como fazer o roteamento e seta cabeçalhos relacionados a proxy como `X-Forwarded-Host` e `X-Forwarded-Port`.

Há alguns `route` filters:

- `SendForwardFilter`: repassa o resultado do response roteado para o response original.
- `SimpleHostRoutingFilter`: envia os requests para a URL configurada na propriedade `routeHost` do `RequestContext`.
- `RibbonRoutingFilter`: usa ferramentas que veremos posteriormente para fazer roteamento dinâmico e balanceamento de carga.

Há um `error` filter:

- `SendForwardFilter`: faz o forward de qualquer exceção para `/error`.

## LocationRewriteFilter no Zuul para além de redirecionamentos

Ao usar um API Gateway como Proxy, precisamos ficar atentos a URLs retornadas nos payloads e cabeçalhos HTTP.

O cabeçalho `Location` é comumente utilizado por redirects (status `301 Moved Permanently`, `302 Found`, entre outros). Esse cabeçalho contém um novo endereço que o cliente HTTP, em geral um navegador, tem que acessar logo em seguida.

Esse cabeçalho `Location` também é utilizado, por exemplo, quando um novo recurso é criado no servidor (status `201 Created`).

O Spring Cloud Netflix Zuul tem um Filter padrão, o `LocationRewriteFilter`,que reescreve as URLs, colocando no `Location` o endereço do próprio Zuul, ao invés de manter o endereço do serviço.

Porém, esse Filter só funciona para redirecionamentos (`3XX`) e não para outros status como `2XX`.

Por exemplo, vamos criar um novo pagamento usando o Zuul como Proxy:

```sh
curl -X POST -i -H 'Content-Type: application/json' 
 -d '{"va51.8, "nome": "JOÃO DA SILVA", "numero": "1111 2222 3333 4444", "expiracao": "2022-07", "codigo": "123", "formaDePagamentoId": 2, "pedidoId": 1}'
  http://localhost:9999/pagamentos
```

O response, incluindo cabeçalhos, será semelhante a:

```txt
HTTP/1.1 201 
Location: http://localhost:8081/pagamentos/2
Date: Mon, 06 Jan 2020 17:47:56 GMT
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
```

<!-- separador -->

```json
{"id":2, "valor":51.8, "nome":"JOÃO DA SILVA", "numero":"1111 2222 3333 4444",
"expiracao":"2022-07", "codigo":"123", "status":"CRIADO",
"formaDePagamentoId":2, "pedidoId":1}
```

Perceba que, apesar de invocarmos o serviço de pagamentos pelo Zuul, o cabeçalho `Location` contém a porta `8081`, do serviço original, na URL:

```txt
Location: http://localhost:8081/pagamentos/2
```

Vamos customizá-lo, para que funcione com respostas bem sucedidas, de status `2XX`.

> O código do `LocationRewriteFilter` do Spring Cloud Netflix Zuul pode ser encontrado em:
> http://bit.ly/spring-cloud-location-rewrite-filter

Para isso, crie uma classe `LocationRewriteConfig` no pacote `br.com.caelum.apigateway`, definindo uma subclasse anônima  de `LocationRewriteFilter`, modificando alguns detalhes.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/LocationRewriteConfig.java

```java
@Configuration
class LocationRewriteConfig {

  @Bean
  LocationRewriteFilter locationRewriteFilter() {
    return new LocationRewriteFilter() {
      @Override
      public boolean shouldFilter() {
        int statusCode = RequestContext.getCurrentContext().getResponseStatusCode();
        return HttpStatus.valueOf(statusCode).is3xxRedirection() || HttpStatus.valueOf(statusCode).is2xxSuccessful();
      }
    };
  }

}
```

Tome bastante cuidado com os imports:

```java
import org.springframework.cloud.netflix.zuul.filters.post.LocationRewriteFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;

import com.netflix.zuul.context.RequestContext;
```

Agora sim! Ao receber um status `201 Created`, depois de criar algum recurso em um serviço, o API Gateway terá o `Location` dele próprio, e não do serviço original.

## Exercício: Customizando o LocationRewriteFilter do Zuul

1. Através de um cliente REST, tente adicionar um pagamento passando pelo API Gateway. Para isso, utilize a porta `9999`.

  Com o cURL é algo como:

  ```sh
  curl -X POST
    -i
    -H 'Content-Type: application/json'
    -d '{ "valor": 51.8, "nome": "JOÃO DA SILVA", "numero": "1111 2222 3333 4444", "expiracao": "2022-07", "codigo": "123", "formaDePagamentoId": 2, "pedidoId": 1 }'
    http://localhost:9999/pagamentos
  ```

  Lembrando que um comando semelhante ao anterior, mas com a porta `8081`, está disponível em: https://gitlab.com/snippets/1859389

  Note no cabeçalho `Location` do response que, mesmo utilizando a porta `9999` na requisição, a porta da resposta é a `8081`.

  ```txt
  Location: http://localhost:8081/pagamentos/40
  ```

2. Pare o API Gateway.

  No projeto `fj33-api-gateway`, faça o checkout da branch `cap6-customizando-location-filter-do-zuul`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap6-customizando-location-filter-do-zuul
  ```

  Execute a classe `ApiGatewayApplication`.

3. Teste novamente a criação de um pagamento com um cliente REST. Perceba que o cabeçalho `Location` agora tem a porta `9999`, do API Gateway.

4. (desafio - opcional) Se você fez os exercícios opcionais de Spring HATEOAS, note que as URLs dos links ainda contém a porta `8081`. Implemente um Filter do Zuul que modifique as URLs do corpo de um _response_ para que apontem para a porta `9999`, do API Gateway.

## Exercício opcional: um ZuulFilter de Rate Limiting

1. Adicione, no `pom.xml` de `api-gateway`, uma dependência a biblioteca Google Guava:

  ####### fj33-api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>15.0</version>
  </dependency>
  ```

  A biblioteca Google Guava possui uma implementação de _rate limiter_, que restringe o acesso a recurso em uma determinada taxa configurável.

2. Crie um `ZuulFilter` que retorna uma falha com status `429 TOO MANY REQUESTS` se a taxa de acesso ultrapassar 1 requisição a cada 30 segundos:

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/RateLimitingZuulFilter.java

  ```java
  @Component
  public class RateLimitingZuulFilter extends ZuulFilter {

    private final RateLimiter rateLimiter = RateLimiter.create(1.0 / 30.0); // permits per second

    @Override
    public String filterType() {
      return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
      return Ordered.HIGHEST_PRECEDENCE + 100;
    }

    @Override
    public boolean shouldFilter() {
      return true;
    }

    @Override
    public Object run() {
      try {
        RequestContext currentContext = RequestContext.getCurrentContext();
        HttpServletResponse response = currentContext.getResponse();

        if (!this.rateLimiter.tryAcquire()) {
          response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
          response.getWriter().append(HttpStatus.TOO_MANY_REQUESTS.getReasonPhrase());
          currentContext.setSendZuulResponse(false);
        }
      } catch (IOException e) {
        throw new RuntimeException(e);
      }
      return null;
    }
  }
  ```

  Os imports são os seguintes:

  ```java
  import java.io.IOException;

  import javax.servlet.http.HttpServletResponse;

  import org.springframework.cloud.netflix.zuul.filters.support.FilterConstants;
  import org.springframework.core.Ordered;
  import org.springframework.http.HttpStatus;
  import org.springframework.stereotype.Component;

  import com.google.common.util.concurrent.RateLimiter;
  import com.netflix.zuul.ZuulFilter;
  import com.netflix.zuul.context.RequestContext;
  ```

3. Garanta que `ApiGatewayApplication` foi reiniciado e acesse alguma várias vezes seguidas pelo navegador, uma URL como `http://localhost:9999/restaurantes/1`.

  Deve ocorrer um erro `429 Too Many Requests`.

4. Apague (ou desabilite comentando a anotação `@Component`) a classe `RateLimitingZuulFilter` para que não cause erros na aplicação.

## Para saber mais: Micro front-ends

Quebramos o código do back-end em alguns serviços independentes e alinhados com verticais de negócio. Mas e o front-end? Continua monolítico!

Toda mudança relevante no back-end, em qualquer serviço (ou no monólito), traria a necessidade de uma mudança no front-end. Todo deploy relevante do back-end, iria requerer um deploy no front-end.

Idealmente, uma Arquitetura de Microservices, teria times independentes, multidisciplinares e orientados pelo domínio, cuidando da concepção à operação do software. E isso deveria ser realmente ponta a ponta (em inglês, _end-to-end_), incluindo a UI.

Michael Geers criou, em 2017, o site [micro-frontends.org](https://micro-frontends.org/) (GEERS, 2017), em que descreve uma abordagem que aplica os princípios de Microservices em websites e webapps.

A ideia de Micro Front-ends é que os times cuidem de um aspecto de negócio específico de ponta a ponta, incluindo concepção, front-end, back-end e operações.

<!-- @note
  Michael Geers cita um artigo [Documents‐to‐Applications Continuum](https://ar.al/notes/the-documents-to-applications-continuum/), em que Aral Balkan fala da ideia de um continuum (tipo uma reta numérica) com documentos totalmente estáticos de um lado (websites) e aplicações em que não há conteúdo prévio e o usuário preenche o conteúdo (webapps), como um Editor de Fotos online.
-->

![Times "ponta a ponta" com Micro Front-ends {w=63}](imagens/06-api-gateway/micro-front-ends-times.png)

Com Micro Front-ends, uma página é composta por componentes de diferentes times.

No caso de sites estáticos, a composição da página pode ser feita no lado do servidor.

Já para single-page apps, a composição deve ser feita no próprio navegador, com código JavaScript.

No site [microservices.io](https://microservices.io/), mantido por Chris Richardson, autor do livro [Microservice Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a), a composição no servidor é chamada de _Server-side page fragment composition_ e no navegador é chamada de _Client-side UI composition_.

Navegadores modernos são compatíveis com [Web Components](https://www.webcomponents.org/introduction), um conjunto de especificações que permitem a criação de componentes customizados como, por exemplo, `<lista-de-recomendacoes>`. As principais especificações são: [Custom Elements](https://w3c.github.io/webcomponents/spec/custom/), [Shadow DOM](https://w3c.github.io/webcomponents/spec/shadow/), [ES Modules](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-module-system) e [HTML Template](https://html.spec.whatwg.org/multipage/scripting.html#the-template-element/). É uma capacidade oferecida por frameworks como Angular, React e Vue, mas com APIs do próprio navegador. Dessa maneira, esses diferentes frameworks podem ter Web Componentes como destino do build.

![Times diferentes cuidam de partes de cada tela {w=72}](imagens/06-api-gateway/micro-front-ends-telas.png)

<!-- @note
  Uma dúvida é: dá pra fazer isso com apps mobile? Será que só com híbridas ou nativas também?
  Creio que dá pra criar componentes customizados, publicados por meio de jars e compô-los em .
-->

Michael Geers declara alguns princípios de Micro Front-ends em seu site:

- **Independência de Tecnologia**: cada time deve escolher suas tecnologias com o mínimo de coordenação com outros times. Uma tecnologia como os Web Components, mencionados anteriormente, ajudam a implementar esse princípio.
- **Código Isolado**: evite variáveis globais e estado compartilhado, mesmo se os times utilizarem o mesmo framework.
- **Uso de Prefixos**: onde o isolamento não é possível, como CSS, Local Storage, Cookies e eventos, estabeleça prefixos e convenções de nomes para evitar colisões e deixar claro qual time cuida de qual componente.
- **Uso de Recursos Nativos**: prefira recursos nativos do navegador a APIs customizadas. Por exemplo, para comunicação entre componentes, use [DOM Events](https://developer.mozilla.org/en-US/docs/Web/Events) ao invés de um mecanismo específico de um framework.
- **Resiliência**: construa a aplicação de maneira que ainda seja útil no caso de uma demora ou uma falha no JavaScript. Conceitos como Universal Rendering e Progressive Enhancement ajudam nesse cenário.

## Discussão: ESB x API Gateway

No livro [SOA in Practice](https://www.amazon.com.br/Soa-Practice-Distributed-System-Design/dp/0596529554) (JOSUTTIS, 2007), Nicholai Josuttis afirma que, entre as responsabilidades de um Enterprise Service Bus (ESB), estão: interoperabilidade entre diferentes plataformas e linguagens de programação, incluindo a tradução de protocolos; transformação de dados; roteamento inteligente; segurança; confiabilidade; gerenciamento de serviços; monitoramento e logging; e orquestração.

Como mencionado anteriormente, o time de Tecnologia da Netflix, no post [Announcing Zuul: Edge Service in the Cloud](https://medium.com/netflix-techblog/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee) (COHEN; HAWTHORNE, 2013), afirma que o API Gateway Zuul é usado para: roteamento dinâmico; segurança; insights (ou seja, monitoramento); entre outras responsabilidades. Além disso, vimos durante o capítulo que podemos usar um API Gateway como um API Composer.

Há certa semelhança entre as responsabilidades de um ESB e de um API Gateway. Seria um API Gateway uma reencarnação do ESB para uma Arquitetura de Microservices?

Uma diferença importante é a **topologia de rede**.

Em geral, um ESB é implantado como um ponto central de comunicação, em uma topologia conhecida como _hub and spoke_, de maneira a evitar uma comunicação ponto a ponto. Assim, os serviços não precisam conhecer os protocolos e endereços uns dos outros, mas apenas os do ESB. Numa implantação _hub and spoke_, a ideia é que qualquer comunicação entre serviços seja feita pelo ESB, levando a um baixo acoplamento.

Já um API Gateway é um proxy reverso, implantado no perímetro da rede. Chamadas de aplicações externas passam pelo API Gateway, porém as chamadas entre os serviços são feitas diretamente, em uma comunicação ponto a ponto. Isso poderia levar a um alto acoplamento entre os serviços, mas nos próximos capítulos veremos maneiras de mitigar esse risco.

Uma outra diferença é relacionada a **presença ou não de regras de negócio**.

No SOA clássico, há a ideia de que serviços podem ser compostos em outros serviços formando novos processos de negócio. O ESB passa a coordenar a execução de diversos serviços de acordo com uma lógica de negócio. O ESB funciona como um maestro que coordena os diferentes músicos (os serviços) de uma orquestra, num processo conhecido como Orquestração. Tecnologias como BPEL ajudam nessa tarefa. Isso pode levar a necessidade de coordenação no desenvolvimento e deploy de diversos serviços.

Autonomia é um dos conceitos mais importantes de uma Arquitetura de Microservices. A centralização de uma Orquestração dá lugar à descentralização de uma Coreografia, em que os diferentes serviços reagem em conjunto a eventos de negócio. Assim, não há um coordenador ou maestro e, em consequência, não há a necessidade de colocar regras de negócio fora dos serviços. Canais de comunicação devem ser "ignorantes". Martin Fowler e James Lewis, em seu seminal artigo [Microservices](https://martinfowler.com/articles/microservices.html) (FOWLER; LEWIS, 2014), cunharam um lema nesse sentido: _smart endpoints, dumb pipes_.

Para seguir o lema _smart endpoints, dumb pipes_ e favorecer a autonomia dos serviços, um API Gateway não deveria conter regras de negócio. Um API Gateway deveria ajudar na implementação de requisitos transversais (também chamados de não-funcionais): segurança, resiliência, escalabilidade, entre diversos outros.

Em uma [thread no Twitter de Maio de 2019](https://twitter.com/samnewman/status/1131117455907205120) (NEWMAN, 2019b), Sam Newman critica o uso do API Gateway como orquestrador de regras de negócio:

_As pessoas [no SOA clássico] fariam a orquestração de processos de negócios nas camadas do message broker. Agora é comum ver pessoas fazendo o mesmo nos API Gateways, que estão rapidamente se tornando o Enterprise Service Bus dos Microservices._

_A inserção da lógica de negócios no middleware é problemática do ponto de vista da implantação, pois é necessário coordenar o rollout, mas também em termos de controle, pois as pessoas que construíram o aplicativo geralmente eram diferentes das pessoas que gerenciavam alterações no middleware._

E as nossas API Compositions não seriam, no fim das contas, colocar regras de negócio no API Gateway? Não, já que evitamos processar os dados, usando a API Composition apenas como agregação de chamadas remotas, visando a otimização das chamadas da UI externa.
