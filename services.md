# Services' duties
## Sale service
- Get the estimated price
- Manage sales (interact with user and look at flight management service)
- At the end of the process send a request yo ticket service


## Ticket service
- Keep track on sold tickets and disponibility of the airplain
- Interact with events related to flight management service
- Send to queueing system when ticket are sold an event (publish)

## Validation service
- Validate the clients tickets

## Flight management service
- Manage the flight creation, update, delation done by the staff
- Interact with queueing system to send updates on the flights (publish)

## Update service
- Listen for flight changes (subscribe) and schedule email jobs to to ticket holders (publish)

## Mail service
- Listen for emails to send (subscribe), emails send on creation, update or delete

## Auth service
- manage the authentication sessions

## Admin/Staff stervice
- Keep track on the admin/staff information
