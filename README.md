# Tutorial NestJS and KafkaJS

## Preparation

### Kafdrop

```bash
$ open /Applications/Docker.app
$ git clone https://github.com/obsidiandynamics/kafdrop.git
$ cd kafdrop
$ cd docker-compose/kafka-kafdrop
$ docker-compose up
```

### NestJS

Create project folder:

```bash
mkdir kafka-nestjs-microservices
cd kafka-nestjs-microservices
```

Create api gateway app:

```bash
$ nest new api-getway
```

Create billing app:

```bash
$ nest new billing
```

Create auth app:

```bash
$ nest new auth
```

Install dependencies:

```bash
$ cd api-gateway
$ npm install @nestjs/microservices kafkajs
```

Repeat for the other 2 microservices.

## Start the microservices

From each microservices folder:

```bash
$ npm run start:dev
```

## Code

### API-GATEWAY

In the api-gateway service `src` folder create a Data Transfer Object file:

```ts
// create-order-request.dto.ts

export class CreateOrderRequest {
	userId: string;
	price: number;
}
```

Back in the controller add a post route:

```ts
// app.controller.ts

import { Body, Controller, Get, Post } from '@nestjs/common';
import { CreateOrderRequest } from './create-order-request.dto';
...
@Post()
  createOrder(@Body() createOrderRequest: CreateOrderRequest) {
    this.appService.createOrder(createOrderRequest);
  }
```

In the service add the function `createOrder()` and the `constructor`:

```ts
// app.service.ts

import { CreateOrderRequest } from './create-order-request.dto';
...

constructor(
    @Inject('BILLING_SERVICE') private readonly billingClient: ClientKafka,
  ) {}

createOrder({ userId, price }: CreateOrderRequest) {
    this.billingClient.emit(
      'order_created',
      new OrderCreatedEvent('123', userId, price),
    );
  }
```

Setup our kafka communication to billing in the app.module:

```ts
// app.modules.ts

import { ClientsModule, Transport } from '@nestjs/microservices';
...
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'BILLING_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'billing',
            brokers: ['localhost:9092'],
          },
          consumer: {
            groupId: 'billing-consumer',
          },
        },
      },
    ]),
  ],
  controllers: [AppController],
  providers: [AppService],
})
```

Create an order event file in the `src` folder:

```ts
// order-created.event.ts

export class OrderCreatedEvent {
	constructor(
		public readonly orderId: string,
		public readonly userId: string,
		public readonly price: number
	) {}

	toString() {
		return JSON.stringify({
			orderId: this.orderId,
			userId: this.userId,
			price: this.price,
		});
	}
}
```

### BILLING

In the file `main.ts` we remove the code in the bootstrap function and replace by this

```ts
// main.ts

import { MicroserviceOptions, Transport } from '@nestjs/microservices';
...
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.KAFKA,
      options: {
        client: {
          brokers: ['localhost:9092'],
        },
        consumer: {
          groupId: 'billing-consumer',
        },
      },
    },
  );
  app.listen();
}
bootstrap();
```

In the app controller:

```ts
// app.controller.ts

import { Controller, Get, Inject, OnModuleInit } from '@nestjs/common';
import { ClientKafka, EventPattern } from '@nestjs/microservices';
...

@Controller()
export class AppController implements OnModuleInit {
  constructor(
    private readonly appService: AppService,
    @Inject('AUTH_SERVICE') private readonly authClient: ClientKafka,
  ) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @EventPattern('order_created')
  handleOrderCreated(data: any) {
    this.appService.handleOrderCreated(data);
  }

  onModuleInit() {
    this.authClient.subscribeToResponseOf('get_user');
  }
}
```

Create an event file:

```ts
// order-created.event.ts
export class OrderCreatedEvent {
	constructor(
		public readonly orderId: string,
		public readonly userId: string,
		public readonly price: number
	) {}
}
```

In the app service:

```ts
// app.service.ts

import { Inject, Injectable } from "@nestjs/common";
import { ClientKafka } from "@nestjs/microservices";
import { GetUserRequest } from "./get-user-request.dto";
import { OrderCreatedEvent } from "./order-created.event";

@Injectable()
export class AppService {
	constructor(
		@Inject("AUTH_SERVICE") private readonly authClient: ClientKafka
	) {}

	getHello(): string {
		return "Hello World!";
	}

	handleOrderCreated(orderCreatedEvent: OrderCreatedEvent) {
		console.log(orderCreatedEvent);
		this.authClient
			.send("get_user", new GetUserRequest(orderCreatedEvent.userId))
			.subscribe((user) => {
				console.log(
					`Billing user with stripe ID ${user.stripeUserId} a price of $${orderCreatedEvent.price}...`
				);
			});
	}
}
```

Test with postman by sending a `post` request to localhost:3000 witht eh following body:

```json
{
	"userId": "345",
	"price": 34.33
}
```

It should console log in the billing microservice:

```bash
{ orderId: '123', userId: '345', price: 34.33 }
Billing user with stripe ID 27279 a price of $34.33...
```

Now need to set up a handle to an external service in app module:

```ts
// app.module.ts

import { Module } from "@nestjs/common";
import { ClientsModule, Transport } from "@nestjs/microservices";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";

@Module({
	imports: [
		ClientsModule.register([
			{
				name: "AUTH_SERVICE",
				transport: Transport.KAFKA,
				options: {
					client: {
						clientId: "auth",
						brokers: ["localhost:9092"],
					},
					consumer: {
						groupId: "auth-consumer",
					},
				},
			},
		]),
	],
	controllers: [AppController],
	providers: [AppService],
})
export class AppModule {}
```

Create a new DTO:

```ts
// get-user-request.dto.ts

export class GetUserRequest {
	constructor(public readonly userId: string) {}

	toString() {
		return JSON.stringify({
			userId: this.userId,
		});
	}
}
```

### AUTH

Update the `main.ts` in auth:

```ts
import { NestFactory } from "@nestjs/core";
import { MicroserviceOptions, Transport } from "@nestjs/microservices";
import { AppModule } from "./app.module";

async function bootstrap() {
	const app = await NestFactory.createMicroservice<MicroserviceOptions>(
		AppModule,
		{
			transport: Transport.KAFKA,
			options: {
				client: {
					brokers: ["localhost:9092"],
				},
				consumer: {
					groupId: "auth-consumer",
				},
			},
		}
	);
	app.listen();
}
bootstrap();
```

Update the auth controller:

```ts
import { Controller, Get } from "@nestjs/common";
import { MessagePattern } from "@nestjs/microservices";
import { AppService } from "./app.service";

@Controller()
export class AppController {
	constructor(private readonly appService: AppService) {}

	@Get()
	getHello(): string {
		return this.appService.getHello();
	}

	@MessagePattern("get_user")
	getUser(data: any) {
		// console.log('data auth is: ' + data);
		return this.appService.getUser(data);
	}
}
```

Create get user request DTO:

```ts
// get-user-request.dto.ts
export class GetUserRequest {
	constructor(public readonly userId: string) {}

	toString() {
		return JSON.stringify({
			userId: this.userId,
		});
	}
}
```

In the app services file:

```ts
// app.services.ts

import { Injectable } from "@nestjs/common";
import { GetUserRequest } from "./get-user-request.dto";

@Injectable()
export class AppService {
	private readonly users: any[] = [
		{
			userId: "123",
			stripeUserId: "43234",
		},
		{
			userId: "345",
			stripeUserId: "27279",
		},
	];

	getHello(): string {
		return "Hello World!";
	}

	getUser(getUserRequest: GetUserRequest) {
		return this.users.find((user) => user.userId === getUserRequest.userId);
	}
}
```

## Resources

The code github repository is available [here](https://github.com/mguay22/nestjs-kafka-microservices)

[youtube video](https://www.youtube.com/watch?v=JJEKPqSlXvk&t=146s)

[Kafdrop](https://github.com/obsidiandynamics/kafdrop) is a web UI for viewing Kafka topics and browsing consumer groups. The tool displays information such as brokers, topics, partitions, consumers, and lets you view messages.
