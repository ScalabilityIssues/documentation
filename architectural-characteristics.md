# Architectural Characteristics
We analyze the architecture characteristics the system would like to achieve, listed in order of importance:

## Reliability
- Measure: units of failure that can be tolerated by the system per unit of time
- Reference to requirements: 1, 3, 5<br/>
    Every failure implies a loss of income for the company and worse user experience
- Quantification: the system should admit up to 10 unconventional behaviours per month and 99.5% of uptime

## Modularity
- Measure: degree to which the software is composed of discrete (logical) components
- Reference to requirement: 2, 3, 4, 5 <br/>
    These requirements define the different domains of the application (flight management, ticket sales, user interactions, and boarding)
- Quantification: our system should be designed to have high cohesion and low coupling by dividing it in 4 services according to the domains defined above

## Fault tolerance
- Measure: the system operates as intended despite hardware or software faults
- Reference to requirement: 5 <br/>
    Ticket validation is critical to time sensitive operations, you should be able to validate them even if the system fails
- Quantification: boarding should be possible even if the system is down for up to 24hours

## Scalability
- Measure: ability to increase the number of computational units over log unit of time
- Reference to requirement: 3, 4, 5 <br/>
    The system should be able to handle an increasing number of users and tickets as the company grows
- Quantification: scalable up to 5x the original number of customers 
