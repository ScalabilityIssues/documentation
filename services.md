# Services' duties

## Sale service

- Handle the discover and purchase flow
  - Generate ticket offers (JWT with flight details and price)
  - Payments simulated
- Afterwards creates ticket


## Ticket service

- Keep track of sold tickets and free seats of the flight
- Send new or modified tickets to broker in order to nofify the client


## Boarding service

- handle the cryptographic signing of tickets through a rpc method
- handle the keys to decrypt the tickets


## Flight management service

- Manage the flight creation, update, delation done by the staff
- Interact with queueing system to send updates on the flights (publish)


## Update service

- Listen for flight changes (subscribe) and sends emails to ticket holders


## Frontend
- Public side
  - flight search
  - ticket purchase
  - ticket visualization
- Staff side
  - validation
- Admin side
  - flight management: create flights, create planes