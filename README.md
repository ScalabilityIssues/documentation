# Simurgh flight management system

## Description

This project is an implementation of an online IT system for an airline company operating in Europe, focusing on managing ticket sales and pricing. The system handles direct flights for a single airline, allowing administrators to define flights and schedules. It supports dynamic ticket pricing based on seasonality and economic factors, offering customers variable pricing options. Customers receive email notifications for ticket updates, ensuring they are informed about any changes. The system also includes a robust ticket validation mechanism for reliable verification before boarding, ensuring a seamless and efficient travel experience. This project aims to enhance the airline's operational efficiency and customer satisfaction through effective ticket management and adaptive pricing strategies.

## How to run

1. Install Docker from https://docs.docker.com/get-docker/
1. Run `docker compose up -d`
1. Enjoy at http://localhost:80

## System architecture description

<img src="img/arch.jpg" alt="architecture_sketch" class="center" />



- microservices
  - dedicated databases
  - choreography
  - no api gateway
  - decomposition by subdomain
- monolithic frontend
- event based


## Service descriptions

### Sale service

- Handle the ticket's discover and purchase flow
  - Generate ticket offers (JWT with flight details and price)
  - Payments are simulated
- Afterwards creates ticket

## Price estimation service
Code <a href="https://github.com/ScalabilityIssues/price_estimator">here</a>

The "Price Estimation" microservice incorporates several key features to provide accurate and timely price predictions for airline tickets.

- Firstly, it employs a container named "ml-data-scraper" capable of periodic execution and configurable to extract flight and pricing information from various airline companies via <a href="https://www.kayak.com">Kayak</a> website.
Once the data retrieval process concludes, it uploads the gathered information to a distributed and efficient MinIO database stored within a designated bucket, while concurrently dispatching a notification through RabbitMQ signaling the completion of the task.
- Additionally, the microservice comprises a container labeled "ml-training" which remains on standby for incoming events indicating the arrival of new flight data. Upon receipt, it initiates the training of a new Machine Learning model tailored for price prediction. Once the training phase is complete, the newly created model is uploaded to a designated MinIO bucket.
- Finally, the microservice encompasses a container exposing a gRPC interface specifically designed for price prediction. This interface accepts input parameters such as airport information and dates and produces reliable price predictions. In the event of a newly trained model, it seamlessly incorporates the updated model for predictions; otherwise, it utilizes the latest available model stored within the MinIO repository, ensuring up-to-date and accurate price estimations for users.

### Ticket service

- Manage tickets creation, update and delete
- Before a ticket creation check that a seat is available for that flight
- After a ticket creation or update, publish on the broker to notify the client


### Validation service
- Handle the cryptographic signing of tickets through a rpc method
- When a ticket is signed, a qr code is generated containing the ticket and the signature; this is necessary to check the validity of the ticket
- Handle the keys to verify tickets


### Flight management service
- Manage airports creation, update and delete
- Manage planes creation, update and delete
- Manage flights creation, update and delete
- After a flight modification, publish on the broker to notify clients that have a ticket on that flight


### Update service
- Listen from the broker for changes on flights and send emails to users that have a ticket for that flight
- Listen from the broker for changes on tickets and send an email to the user owner of the ticket


### Frontend
- Customer side
  - flight search
  - ticket purchase
  - ticket visualization
- Staff side
  - tickets validation
- Admin side
  - flight management
    - create flights
    - create planes


## Production Considerations
