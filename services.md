# Services' duties

## Sale service

- Handle the ticket's discover and purchase flow
  - Generate ticket offers (JWT with flight details and price)
  - Payments are simulated
- Afterwards creates ticket

## Price estimation service
- Retrieve flight information and price data from the web
- Train a Machine Learining model that estimate the price of a flight given some characteristics
- Predict the prices given unseen new data 

## Ticket service

- Manage tickets creation, update and delete
- Before a ticket creation check that a seat is available for that flight
- After a ticket creation or update, publish on the broker to notify the client


## Validation service
- Handle the cryptographic signing of tickets through a rpc method
- When a ticket is signed, a qr code is generated containing the ticket and the signature; this is necessary to check the validity of the ticket
- Handle the keys to verify tickets


## Flight management service
- Manage airports creation, update and delete
- Manage planes creation, update and delete
- Manage flights creation, update and delete
- After a flight modification, publish on the broker to notify clients that have a ticket on that flight


## Update service
- Listen from the broker for changes on flights and send emails to users that have a ticket for that flight
- Listen from the broker for changes on tickets and send an email to the user owner of the ticket


## Frontend
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