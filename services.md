# Services' duties

## Sale service

- Handle the discover and purchase flow
  - Generate ticket offers (JWT with flight details and price)
  - Payments handled with a stub
- Handle the process of ticket details modification
- At the end of the process send an event to the broker; ticket service listen for that kind of events and manage them

## Ticket service

- Keep track of sold tickets and free seats of the flight
- Interact with events related to flight management service to create or modify tickets
- Send new or modified tickets to broker in order to nofify the client

## Boarding service

- has a db containing the validations
- has a db containing the invalid tickets
- handle the cryptographic signing of tickets through a rpc method
- handle the keys to decrypt the tickets
- maybe after the fly landed the validations can be moved to a different db with historical aggregated data (like flight_id: passanger_number)

## Flight management service

- Manage the flight creation, update, delation done by the staff
- Interact with queueing system to send updates on the flights (publish)

## Update service

- Listen for flight changes (subscribe) and schedule email jobs to to ticket holders (publish)

## Mail service

- Listen for emails to send (subscribe), emails send on creation, update or delete

## Auth service

- manage the authentication sessions

## Admin/Staff service

- Keep track on the admin/staff information
