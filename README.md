# Building Microservices with Nest.js 

Nest.js is a progressive Node.js framework for building efficient, reliable, and scalable server-side applications. 

Nest.js can be seen as Angular on the backend (as one of my friends called it) as it provides a plethora of useful features, and - just as Angular can be a bit overwhelming at first glance. To avoid overloading with the info, I'll skip to the most crucial ones from my point of view.

1)Built with TypeScript

2)Many technologies supported out of the box (GraphQL, Redis, Elasticsearch, TypeORM, microservices, CQRS, …)

3)Built with Node.js and Supports both Express.js and Fastify

4)Dependency Injection

# It consists of:

1) PostgreSQL Typeorm

2)CQRS

3)Domain Driven Design

4)Event Sourcing

5)Healthchecks

6) Unit tests

7).env support

8) RabbitMQ Event Bus Connection

9)Dockerfile and docker-compose

# Core Concepts of Nest.js
 1) Modules
 
 2)Controllers
 
 3)Services.

# Modules
Modules encapsulate logic into reusable pieces of code (components).
// app.module.ts

@Module({

  imports: [],       // Other modules
  
  controllers: [],   // REST controllers
  
  
  providers: [],     // Services, Pipes, Guards, etc
  
})

export class AppModule {}

# Controllers

Used to handle REST operations (HTTP methods)

// app.controller.ts


@Controller()      // Decorator indicating that the following TypeScript class is a REST controller

export class AppController {
  
  constructor(private readonly appService: AppService) {}    // Service available through Dependency Injection  

 
 @Get()     // HTTP method handler
 
 getHello(): string {
   
   
   return this.appService.getHello();     // Calling a service method
  }

}

# Services
Services are used to handle logic and functionality. Service methods are called from within a controller.
// app.service.ts

@Injectable()       // Decorator that marks a TypeScript class a provider (service)

export class AppService {

constructor() {}       // Other services, repositories, CQRS handlers can be accessed through Dependency Injection

 
 getHello(): string {
 
 return 'Hello World!';      // Plain functionality
  }
}

# Microservices
There is a great articles series by Chris Richardson regarding Microservices available on https://microservices.io/. Make sure to read it first if you are not familiar with this concept.

Ok, let's jump to the code! You will need two repositories that I have prepared for this tutorial:

Make sure to clone them and install all required dependencies. We will also need a Docker installed on our system and a Database Management Tool of your choice (I am using Table Plus). Also, a Postman would be needed to test endpoints.

# Refactoring basic server to microservices
In this section what we will do is we will be converting two basic Nest.js servers to a main server (API Gateway) and microservice (responsible for handling item operations).

If you get lost at some point, inside the repositories there are commits and branches that will help you do the refactor step by step.

# Repositories
There are two repositories ready to serve as a simple example and they are very similar Nest.js servers with small differences:

nest-microservice:

.env.example file with environment variables that you would need to copy to .env file for the docker-compose.yml to work.
# Database
POSTGRES_VERSION=13-alpine
POSTGRES_USERNAME=postgres_user
POSTGRES_PASSWORD=postgres_password
POSTGRES_DATABASE=item
POSTGRES_PORT=5433
POSTGRES_HOST=localhost
docker-compose.yml file with configuration of PostgreSQL image to serve our database.
// docker-compose.yml

version: '3.7'
services:
  postgres:
    container_name: microservice_postgres
    image: postgres:${POSTGRES_VERSION}
    environment: 
      - POSTGRES_USER=${POSTGRES_USERNAME}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DATABASE}
    ports:
      - ${POSTGRES_PORT}:5432
    volumes:
      - /data/postgres/
    networks:
      - microservice-network

networks: 
    microservice-network:
        driver: bridge
required npm packages for the demo to work.
// package.json

...
  "dependencies": {
    "@nestjs/common": "^7.6.15",
    "@nestjs/config": "^0.6.3",
    "@nestjs/core": "^7.6.15",
    "@nestjs/microservices": "^7.6.15",
    "@nestjs/platform-express": "^7.6.15",
    "@nestjs/typeorm": "^7.1.5",
    "pg": "^8.6.0",
    "reflect-metadata": "^0.1.13",
    "rimraf": "^3.0.2",
    "rxjs": "^6.6.6",
    "typeorm": "^0.2.32"
  },
nest-demo:

required npm packages for the demo to work.
// package.json

...
  "dependencies": {
    "@nestjs/common": "^7.6.15",
    "@nestjs/config": "^0.6.3",
    "@nestjs/core": "^7.6.15",
    "@nestjs/microservices": "^7.6.15",
    "@nestjs/platform-express": "^7.6.15",
    "reflect-metadata": "^0.1.13",
    "rimraf": "^3.0.2",
    "rxjs": "^6.6.6"
  },
Both projects are basic Nest.js servers:
// main.ts

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();

// app.module.ts

import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

// app.controller.ts

import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}

// app.service.ts

import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}

If we run these servers using yarn dev or npm run dev commands we would see in the browser Hello World.

Now into the actual refactoring

What we will do in this sections is we will take the code from the basic server and refactor it to API Gateway and a microservice. 

nest-demo:

Inside app.module.ts we will add a ClientsModule with some option to allow our server to communicate to the microservice using TCP connection.
// app.module.ts

import { Module } from '@nestjs/common';

import { ClientsModule, Transport } from '@nestjs/microservices';

import { AppController } from './app.controller';


import { AppService } from './app.service';

@Module({


  imports: [
  
    ClientsModule.register([{ name: 'ITEM_MICROSERVICE', transport: Transport.TCP }])
    
  ],
  
  controllers: [AppController],
  
  providers: [AppService],
  
})
export class AppModule {}

Inside app.controller.ts we will add two new endpoints that would allow us to test both READ and WRITE functionality.
// app.controller.ts

import { Body, Controller, Get, Param, Post } from '@nestjs/common';

import { AppService } from './app.service';

@Controller()

export class AppController {


  constructor(private readonly appService: AppService) {}
  

  @Get()
  
  getHello(): string {
  
    return this.appService.getHello();
    
  }

  @Get('/item/:id')
  
  getById(@Param('id') id: number) {
  
    return this.appService.getItemById(id);
    
  }

  @Post('/create')
  
  
  create(@Body() createItemDto) {
  
    return this.appService.createItem(createItemDto);
    
    
  }
}
Inside app.service.ts we will add two additional methods to handle new endpoints by sending a message pattern and data to the corresponding microservice.
// app.service.ts

import { Inject, Injectable } from '@nestjs/common';

import { ClientProxy } from '@nestjs/microservices';


@Injectable()

export class AppService {


  constructor(
  
    @Inject('ITEM_MICROSERVICE') private readonly client: ClientProxy
  ) {}

  getHello(): string {
    return 'Hello World!';
  }

  createItem(createItemDto) {
  
    return this.client.send({ role: 'item', cmd: 'create' }, createItemDto);
  }
  

  getItemById(id: number) {
  
    return this.client.send({ role: 'item', cmd: 'get-by-id' }, id); 
  }
}
In here we are injecting the ITEM_MICROSERVICE client that we have declared in app.module.ts in order to later use it in certain methods to send the message.
send method accepts two arguments; messagePattern in a form of an object, and a data.

** If you need to pass more than one variable (i.e. firstName and lastName) create an object out of it and send them in that form as a second argument.
Make sure to remember or copy the value of messagePattern because we will need it in that exact form in the microservice to respond to this message.

And that will be it for the nest-demo project. Do not run the project yet as the microservice is not ready yet to handle requests.

# nest-microservice:
Create item.entity.ts file. It will be used to model our database tables.
// item.entity.ts

import { BaseEntity, Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()

export class ItemEntity extends BaseEntity {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;
}
In here we are declaring a table with two columns; id and name.

Create item.repository.ts file to be able to work with the entity on the database (create, find, delete, etc).
// item.repository.ts

import { EntityRepository, Repository } from "typeorm";

import { ItemEntity } from "./item.entity";

@EntityRepository(ItemEntity)

export class ItemRepository extends Repository<ItemEntity> {}

In here we could create our methods for for working with entity but for this tutorial we will only need the default ones provided by typeorm.

Modify app.module to connect to the PostgreSQL database Docker container and load ItemRepository and ItemEntity.
// app.module.ts

import { Module } from '@nestjs/common';

import { TypeOrmModule } from '@nestjs/typeorm';

import { AppController } from './app.controller';

import { AppService } from './app.service';

import { ItemEntity } from './item.entity';

import { ItemRepository } from './item.repository';

@Module({

  imports: [
  
    TypeOrmModule.forRoot({
    
      type: 'postgres',
      
      host: 'localhost',
      
      port: 5433,
      
      username: 'postgres_user',
      
      password: 'postgres_password',
      
      database: 'item',
      
      synchronize: true,
      
      autoLoadEntities: true,
    }),
    
    TypeOrmModule.forFeature([ItemRepository, ItemEntity])
    
  ],
  controllers: [AppController],
  
  providers: [AppService],
  
})
export class AppModule {}

** *For the real application remember to not use credentials in plain values but use environment variables or/and @nestjs/config package.

Refactor main.ts file from basic Nest.js server to Nest.js Microservice.
// main.ts

import { NestFactory } from '@nestjs/core';

import { Logger } from '@nestjs/common';

import { Transport } from '@nestjs/microservices';

import { AppModule } from './app.module';

const logger = new Logger('Microservice');

async function bootstrap() {

  const app = await NestFactory.createMicroservice(AppModule, {
  
  
    transport: Transport.TCP,
    
  });

  await app.listen(() => {
  
    logger.log('Microservice is listening');
    
  });
}
bootstrap();

Refactor app.controller.ts to listen to messages rather than HTTP methods (messagePattern from nest-demo will be needed here).
// app.controller.ts

import { Body, Controller, Get, Param, Post } from '@nestjs/common';

import { MessagePattern } from '@nestjs/microservices';

import { AppService } from './app.service';

@Controller()

export class AppController {

  constructor(private readonly appService: AppService) {}

  @Get()
  
  getHello(): string {
  
    return this.appService.getHello();
  }

  @MessagePattern({ role: 'item', cmd: 'create' })
  
  createItem(itemDto) {
  
    return this.appService.createItem(itemDto)
  }

  @MessagePattern({ role: 'item', cmd: 'get-by-id' })
  
  getItemById(id: number) {
  
    return this.appService.getItemById(id);
  }
}
In here we are using the messagePattern from nest-demo to react for the messages with certain pattern and we trigger methods inside appService.

Refactor app.service to handle the READ and WRITE methods.
// app.service.ts

import { Injectable } from '@nestjs/common';

import { ItemEntity } from './item.entity';

import { ItemRepository } from './item.repository';

@Injectable()
export class AppService {
  constructor(
    private readonly itemRepository: ItemRepository,
  ) {}

  getHello(): string {
    return 'Hello World!';
  }

  createItem(itemDto) {
    const item = new ItemEntity();
    item.name = itemDto.name;
    return this.itemRepository.save(item);
  }

  getItemById(id) {
    return this.itemRepository.findOne(id);
  }
}
In here we are using the injected itemRepository to save a new ItemEntity or find existing one by id.

# Running all API Gateway, Microservice and Database Container

To run all services I would recommend to open two terminal windows or three if you are not using Docker Desktop.

1) Run PostgreSQL container by using docker-compose up in nest-microservice project or by using Docker Desktop.

2)Run yarn dev or npm run dev in nest-microservice project to start microservice.

3)Run yarn dev or npm run dev in nest-demo project to start API Gateway
.
4)Testing if everything is working correctly

Connect to your PostgreSQL container with TablePlus using the same credentials that you used for your Nest.js application in TypeORM module.
Trigger a POST endpoint in Postman to http://localhost:3000/create with name of your item in the body

You should see the response in Postman and also a new record in the TablePlus.

To test even further you could also send a GET request to http://localhost:3000/item/:id where :id will be 1. And you should see the correct item object that we got from the PostgreSQL.

![Screenshot (54)](https://user-images.githubusercontent.com/93249038/213341819-cb4ec8d9-aedf-4a59-82d3-77c4f1a939f9.png)



