# Processing PostgreSQL Events on AWS
Processing PostgreSQL events on AWS ETL to obtain events (UPDATE, INSERT, DELETE) from PostgreSQL databases, and process them for later loading into other systems/databases.

1) FUNCTIONAL DIAGRAM
The figure below represents the suggested diagram for the process.








Itens 1 and 2: Triggers on database accounts and certificates are configured to fire a custom event when an INSERT, UPDATE or DELETE operation is performed;

Item 3: The first Lambda receives the events, and based on the type received, executes a specific programming logic.
This first Lambda can send the transformed data directly to another system (S3 / DB / etc), or forward the payload to another Lambda, for the purpose of separating the processing into layers.
- In the case of forwarding, still within the first lambda, the final payload will be used to build the Postgres query. This Lambda writes to the Postgres Database, and forwards the payload to the second Lambda.

Item 4: The second Lambda receives the payload from the first, processes the content, and based on the clause type (INSERT or DELETE), produces the respective query and writes it to Redshift.

Item 5: The 2 lambdas, and all logs of the actions performed are available for monitoring on CloudWatch. 


2) ACCESS POLICIES / CONFIGURATION
For communication between DBs / Lambda / external access tools (e.g. PgAdmin), a security-group was created, with the following configuration:

- The purpose of the inbound rules is to:
  Allow POSTGRESQL traffic to any IPv4/IPv6 destination for communication between DBs;
  Allow SSH access for use of query tools and job scheduling (e.g. PgAdmin, DBeaver, etc.).
- The purpose of the outbound rule is to:
  Allow communication using IPv4.


Enabling the pg_notify function
pg_notify is an internal PostgreSQL function that allows you to send asynchronous notifications from a database to a connected client.
	Checking if pg_notify is enabled:

SELECT proname
FROM pg_proc
WHERE proname = 'pg_notify';


3) LAMBDA ENVIRONMENT

For communication between events generated by a database and AWS Lambda, we must consider the following steps:

- Creating the IAM Role
- In IAM, we must click on
- Create new rule
- Select the source service (in this case, RDS)
- Select the target service, with the appropriate permissions (in this case, AWSLambdaFullAcess)

Application in RDS

In “instance settings”, under connected compute resources, add the corresponding Lambda function.
In in the instance settings, apply the created IAM role

4) RDS Proxy
In order to improve response times to the account and certificate databases, 2 proxies were created, which keep the connection active, and substantially improve the total process time. Below are details:

Lambda configuration

In Lambda, left menu (RDS Databases):
Connect to an RDS Database -> using an existing database in the same VPC.
Then, in the same menu, the respective proxy must be added to the database

5) Execution in the POSTGRESQL environment

Triggers pg_notify:

These are custom triggers to send an RDS Postgres event of type INSERT, DELETE or UPDATE to the first Lambda. The payload has been configured to send the following information:
Event type;
Payload;
Source.
