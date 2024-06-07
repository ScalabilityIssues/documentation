# Simurgh flight management system


## Description

This project wants to be an online IT system for an airline company operating in Europe, focusing on managing ticket sales and pricing.


The system handles direct flights for a single airline, allowing administrators to define flights and schedules.
It supports dynamic ticket pricing based on seasonality and economic factors, offering customers variable pricing options. Customers receive email notifications for ticket updates, ensuring they are informed about any changes that affect them.
The system also includes a robust ticket validation mechanism for reliable verification before boarding, ensuring a seamless and efficient travel experience. Ticket validation is possible even if the whole infrastructure is down, improving the fault tolerance of the system.



## How to run


1. Install Docker from https://docs.docker.com/get-docker/
1. Run `docker compose up -d` on the project folder
1. Enjoy at http://localhost:80. Note that we inserted some data about airports, planes and flights; you can find flights from 2024-06-01 to 2024-12-31.
1. To interact with emails go to http://localhost:8025 


## API definition
The complete set of functionalities that the system provides is available in the [`proto` repository](https://github.com/ScalabilityIssues/proto).
It is a special repository that contains the definition of the GRPC services of all microservices.
This is included as a submodule in each repository and is necessary for the server to know what it should implement and for the client to know what remote procedures that it can call.




## System architecture description

![Architectural diagram](img/arch.jpg)


Based on the [architectural characteristics](architectural-characteristics.md) defined for our project, we opted for a microservice architecture with choreography approach.
This approach has numerous advantages, the most important is that microservices are loosely coupled and control is decentralized; this implies that microservices are easier to maintain.

Embracing this approach, these are the most important considerations we had:
1. We based on the requirements of the [architectural kata](architectural-kata.md) to identify what are the main duties of the whole system. 
   As a general principle, each duty is assigned to a microservice. 
   The services we identified are described in [Service descriptions](#service-descriptions).
2. Each microservice has its own database (if needed): 
   we identified the most appropriate way to store the data, considering how the data is accessed and stored and the scalability of each approach. More details on that for each microservice.
2. The main communication protocol we chose to use is GRPC.
   It allows for schema-first typed definitions of services and types, that then are used to generate the code that each service needs to send or receive requests. This cuts down considerably the amount of bugs related to the types of messages shared by the services and simplifies the integration between services.
3. GRPC is a good way to interact but not always the most appropriate. 
   Sometimes there are events that trigger multiple actions that are possibly independent; the best approach is to use the AMQP protocol, in which the event is published to the broker and all services that have some duties with respect to that kind of event can subscribe to a queue. 
   Our choice fell into [RabbitMQ](https://www.rabbitmq.com/) since it is easy to configure and use but at the same time is a powerful tool used in real-world deployments
4. Once each service is developed the challenge consists in running them together to make interactions possible. 
   To achieve this, we designed this strategy:
  - each microservice has its own GitHub repository
  - each repository has a GitHub action properly configured that at each push creates a Docker image of the service
  - each repository has also a `compose.yml` that allows to run the containers needed to develop and test the code of that repository
  - the `documentation` repo contains the `compose.yml` to run the whole application
5. We decided to use Rust to develop most of our services since it is a compiled, reliable, high-performance language. 
   Because the compiler is really strict, even if the language is not so beginner-friendly, it saves a lot of time by highlighting potential errors in advance. 
   The price estimation service, instead, is written in Python because it is the standard the facto programming language for Machine Learning and also allows a convenience library for data scraping. 
   The GUI in the frontend repository is developed with NextJS framework that provides fast and effective tools for building single-page web applications.
6. All the incoming requests need to be sent to the corresponding microservice; we relied on [Traefik](https://doc.traefik.io/traefik/getting-started/install-traefik/) reverse proxy that acts as an ingress controller and dispatches requests to the appropriate microservices. 


More considerations on the ~~perfect~~ least worst architecture for the application are available in section [Production consideration](#production-considerations)




## Service descriptions


### Sale service [[repo](https://github.com/ScalabilityIssues/sale-service)]

The **Sale Service** microservice is responsible for providing the functionalities for purchasing tickets. 

- It handles the discovery of tickets, allowing to list the available flights for a given route on a specific day. 
  For each possible flight, an offer is requested to the [Price Estimation](#price-estimation-service) service, which returns a price for that flight.
- Offers are placed into a JWT that is signed. 
  This allows to send offers to a client without the need to keep track of them; 
  when the client decides to buy one of them, the offer can be trusted if the signature is valid
- The service also handles the purchase flow, where payments are simulated
- After the successful purchase of a ticket, it sends a request to [Ticket Service](#ticket-service) to create the ticket
 
### Price estimation service [[repo](https://github.com/ScalabilityIssues/price_estimator)]

The **Price Estimation** microservice incorporates several key features to provide accurate and timely price predictions for airline tickets.

It is made up of three sub-components, as well as a MinIO block storage cluster.
- The first container is a web scraper capable of periodic execution and configurable to extract flight and pricing information from various airline companies using the [Kayak](https://www.kayak.com) website.
  Once the data retrieval process concludes, it uploads the gathered information to a distributed and efficient MinIO database stored within a designated bucket, while concurrently sending a notification through RabbitMQ signalling the completion of the task.
- The second container handles model training. 
  It remains on standby, listening for incoming AMQP events indicating the arrival of new flight data. 
  Upon receipt, it initiates the training of a new Machine Learning model tailored for price prediction. 
  Once the training phase is complete, the newly created model is uploaded to a designated MinIO bucket.
- The third container runs the GRPC server that handles price prediction requests. 
  This interface accepts input parameters such as airport information and dates and produces reliable price predictions. 
  In the event of a newly trained model, it seamlessly incorporates the updated model for predictions; 
  otherwise, it utilizes the latest available model stored within the MinIO repository, ensuring up-to-date and accurate price estimations for users.


### Ticket service [[repo](https://github.com/ScalabilityIssues/ticket-service)]

The **Ticket service** microservice is responsible for ticket CRUD operations.

- It manages ticket creation, update and delete. The ticket deletion is implemented as a soft delete, in which the ticket is moved to a collection of deleted tickets.
- Before a ticket creation it checks that a seat is available for that flight by making a request to the [Flight management service](#flight-management-service) to check how many seats the plane has and compare that with the number of seats already booked.
- After ticket creation or update, an AMQP event is emitted. 
  The notification process is handled by the [Update service](#update-service)
- It uses a MongoDB database to store the tickets. 
  We chose a non-relational database because ticket management is expected to be the limiting factor when it comes to scalability: 
  the number of flights, airports and planes to manage is negligible compared to the number of tickets. 
  MongoDB allows easier sharding, and since we do not need particular relations other than the flight the ticket refers to, we do not miss the relational schema.




### Validation service [[repo](https://github.com/ScalabilityIssues/validation-service)]

The **Validation Service** microservice is responsible for everything concerning ticket authenticity. 

- Since it deals with private keys, this service is isolated from [ticket service](#ticket-service) to ensure that the validation is completely detached from the ticket management
- The validation flow is designed to work offline. 
  By encoding signed ticket details inside a qr code, all that is necessary to validate them is a public key, which can be stored inside the device of a staff member.
- It handles the cryptographic signing of tickets through a grpc method, which allows other services to request to sign a ticket.
- When a ticket is signed, a qr code is generated; it will contain the ticket and the signature; this is necessary to check the validity of the ticket
- The service also handles the public keys to verify tickets validity and expose them through an grpc method




### Flight management service [[repo](https://github.com/ScalabilityIssues/flight-manager)]

The **Flight management service** microservice is responsible for the CRUD operations of airports, planes and flights. 

- Manages airports creation, update and delete. 
  Deletes are soft, which is achieved by setting the `deleted` attribute
- Manages plane creation, update and delete. 
  Deletes are soft, which is achieved by setting the `deleted` attribute
- Manages flight creation, update and delete. 
  Updates and delete are handled with other event tables, in particular
  - two for gate updates
  - one for the cancellation updates
  - one for delay updates
- We designed the resources handled by this service to be immutable, which ensures external references to them remain valid throughout the lifecycle of the application
- After a flight is updated an AMQP event is generated; 
  [Update service](#update-service) is then responsible for notifying clients who have a ticket on that flight
- It uses a PostgreSQL database to store the data about airports, planes and flights. 
  Here the requirements for the choice of the database are different with respect to the [Ticket service](#ticket-service).  
  We expect this service to get considerably more read than write interactions, which means that database replication should be enough to ensure scalability. 
  Furthermore, since the data handled by this service is crucial to most other services, the advantages offered by a relational database are significant. 


### Update service [[repo](https://github.com/ScalabilityIssues/update-service)]

The update service microservice is responsible for sending email updates to involved users when it detects modifications on tickets or flights. 

- It listens from the broker for changes on flights and sends emails to users that have a ticket for that flight
- It listens from the broker for changes on tickets and sends an email to the owner of the ticket
- It communicates with an SMTP server to forward emails
- To develop and demo this functionality we used [MailHog](https://github.com/mailhog/MailHog), a containerized SMTP server that captures sent emails and displays them in the browser (at http://localhost:8025)




### Frontend [[repo](https://github.com/ScalabilityIssues/frontend)]

The **Frontend** implements the monolithic GUI of the microservices-based system. 
The single-page web application provides the necessary functions for system administrators, the airline's staff, and customers. 
For convenience, the interface does not include user authentication and authorization; however, these ++features can be easily implemented in the future. 
Additionally, it is important to note that these security features are not considered requirements in the [architectural characteristics](architectural-characteristics.md) documentation.


Summarizing, the graphical interface offers the following features:
- **Customer side**:
  - Flight search
  - Ticket purchase
  - Ticket visualization
  - Flight status updates


- **Staff side**:
  - Ticket validation for passenger boarding


- **Admin side**:
  - Add flights to the system and visualize them
  - Add planes to the system and visualize them
  - Edit the flight status
 


## Production Considerations
In the project, we focus particularly on designing and implementing an application that is compliant with the [architectural characteristics](architectural-characteristics.md). 
However, some simplifications were adopted and here we want to explain what we would need to implement to make this production ready.


### API gateway
In a real-world deployment, we could implement an API gateway, to allow better decoupling between frontend and backend; moreover, it would also allow us to more easily incorporate [authentication](#authentication) and error handling.


### Authentication
Currently, there is no authentication at all; authentication is needed for admin and possibly also for staff, depending on specific security requirements of the company and airports.


### User interface to handle airports and planes
In the version delivered, the backend has the functionalities to handle airports; however, it is not possible to handle them via the frontend since this has strong implications with respect to the price estimation service.
Moreover, a better user experience can be achieved, by implementing a search for flights on the admin side. \
All these kinds of details should be discussed with the company to create an ad hoc implementation.


### Caching offers
Since the application receives a lot of requests in terms of available flights for a given route and date, it makes sense to cache offers for a certain time window in order to lower the amount of workload the price estimator has to do, with the aim of improving performance.


### Invalidation of tickets
Another feature that is currently not available is the invalidation of tickets by staff members. 
This can come in handy when the airline company wants to withdraw the validity of a ticket because there are no conditions to admit the passenger on the flight.


### Mail service
In the current version of the project, the emails are sent from the [update service](#update-service) by using a dummy mail server; in a real production environment, it is necessary to use a valid mail service that allows the delivery of emails to customers.


### Keys rotations
To ensure bulletproof security of digital signatures, it is a common practice to perform key rotation. 
This is easy to achieve, but still not implemented in our application. \
However, notice that our GRPC method that returns the validation key supports it by returning a list of keys and not just a single key.


### Law compliance
Before delivering the product to the company, we would assess the compliance of our product with all applicable laws (such as GDPR) with the help of an expert.
