Develop NestJs microservices using [Dapr](https://dapr.io/) pubsub and bindings.

# Description

Dapr Module for [Nest](https://github.com/nestjs/nest) built on top of the [Dapr JS SDK](https://github.com/dapr/js-sdk).

# Installation

```bash
npm i --save @borgius/nest-dapr
```

# Requirements

Install [Dapr](https://dapr.io/) as per getting started [guide](https://docs.dapr.io/getting-started/). Ensure Dapr is running with

```bash
dapr --version
```

Output:

```
CLI version: 1.7.1
Runtime version: 1.7.4
```

# Quick start

The following scaffolds a [Nest](https://github.com/nestjs/nest) project with the [nest-dapr](https://www.npmjs.com/package/@borgius/nest-dapr) package and demonstrates using Nest with Dapr using [RabbitMQ](https://www.rabbitmq.com/) pubsub & queue bindings.

Install Nest [CLI](https://docs.nestjs.com/cli/overview)

```bash
npm install -g @nestjs/cli
```

Scaffold Nest project

```
nest new nest-dapr
cd nest-dapr/
```

Install [nest-dapr](https://www.npmjs.com/package/@borgius/nest-dapr) package

```bash
npm i --save @borgius/nest-dapr
```

Import `DaprModule` in `AppModule` class

```typescript
@Module({
  imports: [DaprModule.register()],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Import `DaprClient` from `@dapr/dapr` package and add dependency to `AppController` class

```typescript
import { DaprClient } from '@dapr/dapr';
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(
    private readonly appService: AppService,
    private readonly daprClient: DaprClient,
  ) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

Create pubsub & topic names used for pubsub operations and message interface

```typescript
const pubSubName = 'my-pubsub';
const topicName = 'my-topic';

interface Message {
  hello: string;
}

@Controller()
```

Create endpoint to publish topic message

```typescript
@Post('pubsub')
async pubsub(): Promise<boolean> {
  const message: Message = { hello: 'world' };

  return this.daprClient.pubsub.publish(pubSubName, topicName, message);
}
```

Create pubsub handler which will subscribe to the topic and log the received message

```typescript
@DaprPubSub(pubSubName, topicName)
pubSubHandler(message: Message): void {
  console.log(`Received topic:${topicName} message:`, message);
}
```

Create Dapr [pubsub](https://docs.dapr.io/developing-applications/building-blocks/pubsub/) component in `components` folder

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-pubsub
  namespace: default
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: amqp://guest:guest@localhost:5674
```
Save file as `components/rabbitmq-pubsub.yaml`

Create `docker-compose.yml` in the project root used to run [RabbitMQ](https://www.rabbitmq.com/)

```yaml
version: '3.9'
services:
  pubsub:
    image: rabbitmq:3-management-alpine
    ports:
      - 5674:5672
      - 15674:15672
```

Start RabbitMQ

```bash
docker-compose up
```

Create script to bootstrap your Nest project using Dapr sidecar. Update `package.json` and add script

```json
"scripts": {
  ..
  "start:dapr": "dapr run --app-id nest-dapr --app-protocol http --app-port 50001 --dapr-http-port 50000 --components-path ./components npm run start"
},
```

Start Nest app with Dapr

```bash
npm run start:dapr
```

Invoke endpoint to publish the message

```bash
curl -X POST localhost:3000/pubsub
```

This should publish a message to RabbitMQ which should be consumed by the handler and written to the console:

```
== APP == Received topic:my-topic message: { hello: 'world' }
```

Full example

```typescript
import { DaprClient } from '@dapr/dapr';
import { DaprPubSub } from '@borgius/nest-dapr';
import { Controller, Get, Post } from '@nestjs/common';
import { AppService } from './app.service';

const pubSubName = 'my-pubsub';
const topicName = 'my-topic';

interface Message {
  hello: string;
}

@Controller()
export class AppController {
  constructor(
    private readonly appService: AppService,
    private readonly daprClient: DaprClient,
  ) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Post('pubsub')
  async pubsub(): Promise<boolean> {
    const message: Message = { hello: 'world' };

    return this.daprClient.pubsub.publish(pubSubName, topicName, message);
  }

  @DaprPubSub(pubSubName, topicName)
  pubSubHandler(message: Message): void {
    console.log(`Received topic:${topicName} message:`, message);
  }
}
```

# DaprModule

`DaprModule` is a global Nest [Module](https://docs.nestjs.com/modules) used to register `DaprServer` & `DaprClient` as [providers](https://docs.nestjs.com/providers) within your project. It also registers all your handlers which listen to Dapr pubsub and input bindings so that when messages are received by Dapr, they are forwarded to the handler. Handler registration occurs during the `onApplicationBootstrap` lifecycle hook.

To use `nest-dapr`, import the `DaprModule` into the root `AppModule` and run the `register()` static method.

```typescript
@Module({
  imports: [DaprModule.register()],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`register()` takes an optional `DaprModuleOptions` object which allows passing arguments to `DaprServer` instance.

```typescript
export interface DaprModuleOptions {
  serverHost?: string;
  serverPort?: string;
  daprHost?: string;
  daprPort?: string;
  communicationProtocol?: CommunicationProtocolEnum;
  clientOptions?: DaprClientOptions;
}
```

See Dapr JS [docs](https://docs.dapr.io/developing-applications/sdks/js/js-server/) for more information about these arguments.

## Async configuration

You can pass your options asynchronously instead of statically. In this case, use the `registerAsync()` method, which provides several ways to deal with async configuration. One of which is to use a factory function:

```typescript
DaprModule.registerAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    serverHost: configService.get('DAPR_SERVER_HOST'),
    serverPort: configService.get('DAPR_SERVER_PORT'),
    daprHost: configService.get('DAPR_HOST'),
    daprPort: configService.get('DAPR_PORT'),
    communicationProtocol: CommunicationProtocolEnum.GRPC,
    clientOptions: {
      logger: {
        level: LogLevel.Verbose,
      },
    },
  }),
  inject: [ConfigService],
})
```

# DaprServer & DaprClient providers

`DaprModule` registers [DaprServer](https://docs.dapr.io/developing-applications/sdks/js/js-server/) and [DaprClient](https://docs.dapr.io/developing-applications/sdks/js/js-client/) as Nest [providers](https://docs.nestjs.com/providers). These can be injected into your controllers and services like any other provider.

```typescript
import { DaprClient } from '@dapr/dapr';
import { Controller, Post } from '@nestjs/common';

@Controller()
export class AppController {
  constructor(readonly daprClient: DaprClient) {}

  @Post()
  async pubsub(): Promise<boolean> {
    return this.daprClient.pubsub.publish('my-pub-sub', 'my-topic', {
      hello: 'world',
    });
  }
}
```

# Dapr decorators

`nest-dapr` provides two TypeScript [decorators](https://www.typescriptlang.org/docs/handbook/decorators.html#decorators) which are used to declaratively configure subscriptions and bindings. These are used by `DaprModule` in conjunction with the handler method to define the handler implementations.

## DaprPubSub decorator

`DaprPubSub` decorator is used to set-up a handler for receiving pubsub topic messages. The handler has 3 arguments (`name`, `topicName` & `route`). `name` specifies the pubsub component `name` as defined in the Dapr component `metadata` section. `topicName` is the name of the pubsub topic. Route is an optional argument and defines possible [routing](https://docs.dapr.io/developing-applications/building-blocks/pubsub/howto-route-messages/) values.

Example:

```typescript
@DaprPubSub('my-pubsub', 'my-topic')
pubSubHandler(message: any): void {
  console.log('Received message:', message);
}
```

RabbitMQ pubsub Component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-pubsub
  namespace: default
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: amqp://guest:guest@localhost:5674
```

Publish message:

```typescript
await this.daprClient.pubsub.publish('my-pubsub', 'my-topic', { hello: 'world' });
```

In this example the handler `pubSubHandler` method will receive messages from the `my-topic` topic through the `my-pubsub` component which in this case is RabbitMQ.

## DaprBinding decorator

`DaprBinding` decorator is used to set-up a handler for receiving input binding data. The handler has one argument `name` which specifies the binding component `name` as defined in the Dapr component `metadata` section.

Example:

```typescript
@DaprBinding('my-queue-binding')
bindingHandler(message: any): void {
  coneole.log('Received message:', message);
}
```

RabbitMQ binding component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-queue-binding
  namespace: default
spec:
  type: bindings.rabbitmq
  version: v1
  metadata:
  - name: queueName
    value: queue1
  - name: host
    value: amqp://guest:guest@localhost:5674
  - name: durable
    value: true
  - name: deleteWhenUnused
    value: false
  - name: ttlInSeconds
    value: 60
  - name: prefetchCount
    value: 0
  - name: exclusive
    value: false
  - name: maxPriority
    value: 5
  - name: contentType
    value: "text/plain"
```

Send message:

```typescript
await this.daprClient.binding.send('my-queue-binding', 'create', { hello: 'world' });
```

In this example the handler `bindingHandler` method will receive messages from the `queue1` queue defined in the `my-queue-binding` component which in this case is RabbitMQ.

## Writing handlers

`DaprModule` uses reflection to register all handlers found either in [Controller](https://docs.nestjs.com/controllers) or [Provider](https://docs.nestjs.com/providers) classes. These classes must be registered in a Nest [module](https://docs.nestjs.com/modules). Providers must be decorated with the `@Injectable()` decorator at the class level. Once this is done and your provider is added to your module's [providers] array then `nest-dapr` will use Nest dependency injection container to resolve the provider instance and call your handler when the message is received.

Here's an example of a [Provider](https://docs.nestjs.com/providers) containing a Dapr handler.

```typescript
import { DaprPubSub } from '@borgius/nest-dapr';
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  @DaprPubSub('my-pubsub', 'my-topic')
  pubSubHandler(message: any): void {
    this.logger.log(`Received topic message:`, message);
  }
}
```

# Examples

| Example | Description |
|---|---|
| [Basics](examples/basics/README.md) | Demonstrates pubsub & input binding using RabbitMQ |

# Troubleshooting

[Dapr](https://dapr.io/) is a complex set of tools and services and must be set-up and deployed carefully to ensure your system operates correctly. `nest-dapr` is merely [syntactic sugar](https://en.wikipedia.org/wiki/Syntactic_sugar) over the existing Dapr [js-sdk](https://github.com/dapr/js-sdk). If things are not working out for you please review Dapr & SDK documentation and issues. Also please use one of the examples provided in this repo. They are updated and tested regularly and should work out of the box. If you find that both Dapr and the Javascript SDK is both working fine but `nest-dapr` is not working in some way, please file an issue and state clearly the problem and provide a reproducable code example. Filing an issue with something like: "It doesn't work" is likely to be ignored. Thank you.

# Credits

Inspiration for this project is taken from [dapr-nestjs-pubsub](https://github.com/avifatal/dapr-nestjs-pubsub) which I believe is the first attempt at integrating [Dapr](https://dapr.io/) with [Nest](https://github.com/nestjs/nest). Unfortunately this repo supports only pubsub messaging and I wanted to support input bindings as well. In the end I adopted the patterns established in Nest's own [event-emitter](https://github.com/nestjs/event-emitter) repo and I pretty much *borrowed* all the metadata, reflection & decorator utils from there.

So full credit goes to the [Nest](https://github.com/nestjs/nest) development team :heart:

