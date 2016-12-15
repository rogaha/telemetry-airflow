# telemetry-airflow
Airflow is a platform to programmatically author, schedule and monitor workflows.

When workflows are defined as code, they become more maintainable, versionable,
testable, and collaborative.

Use Airflow to author workflows as directed acyclic graphs (DAGs) of tasks.
The Airflow scheduler executes your tasks on an array of workers while following
the specified dependencies. Rich command line utilities make performing complex
surgeries on DAGs a snap. The rich user interface makes it easy to visualize
pipelines running in production, monitor progress, and troubleshoot issues when
needed.

### Prerequisites

This app is built and deployed with
[docker](https://docs.docker.com/) and
[docker-compose](https://docs.docker.com/compose/).

### Build Container

An Airflow container can be built with

```bash
make build
```

### Testing

A single task, e.g. `spark`, of an Airflow dag, e.g. `example`, can be run with an execution date, e.g. `2016-01-01`, in the `dev` environment with:
```bash
AWS_SECRET_ACCESS_KEY=... AWS_ACCESS_KEY_ID=... \
make run COMMAND="test example spark 20160101"
```

The container will run the desired task to completion (or failure).
Note that if the container is stopped during the execution of a task,
the task will be aborted. In the example's case, the Spark job will be
terminated.

The logs of the task can be inspected in real-time with:
```bash
docker logs -f telemetryairflow_scheduler_1
```

### Local Deployment

Assuming you're using macOS and Docker for macOS, start the docker service,
click the docker icon in the menu bar, click on preferences and change the
available memory to 4GB.

To deploy the Airflow container on the docker engine, with its required dependencies, run:
```bash
make up
```

You can now connect to your local Airflow web console at
`http://localhost:8000/`.

### Production Setup

When deploying to production make sure to set up the following environment
variables:

- `AWS_ACCESS_KEY_ID` -- The AWS access key ID to spin up the Spark clusters
- `AWS_SECRET_ACCESS_KEY` -- The AWS secret access key
- `SPARK_BUCKET` -- The AWS S3 bucket where Spark related files are stored,
  e.g. `telemetry-spark-emr-2`
- `AIRFLOW_BUCKET` -- The AWS S3 bucket where airflow specific files are stored,
  e.g. `telemetry-airflow`
- `PUBLIC_OUTPUT_BUCKET` -- The AWS S3 bucket where public job results are
  stored in, e.g. `telemetry-public-analysis-2`
- `PRIVATE_OUTPUT_BUCKET` -- The AWS S3 bucket where private job results are
  stored in, e.g. `telemetry-parquet`
- `AIRFLOW_DATABASE_URL` -- The connection URI for the Airflow database, e.g.
  `postgres://username:password@hostname:port/password`
- `AIRFLOW_BROKER_URL` -- The connection URI for the Airflow worker queue, e.g.
  `redis://hostname:6379/0`

Also, please set

- `AIRFLOW_SECRET_KEY` -- A secret key for Airflow's Flask based webserver
- `AIRFLOW_FERNET_KEY` -- A secret key to save connection passwords in the db

Both values should be set by using the cryptography module's fernet tool that
we've wrapped in a docker-compose call:

    make secret

Run this for each key config variable, and **don't use the same for both!**

### Debugging

Some useful docker tricks for development and debugging:

```bash
# Stop all docker containers:
docker stop $(docker ps -aq)

# Remove any leftover docker volumes:
docker volume rm $(docker volume ls -qf dangling=true)
```

### Triggering a task to re-run within the Airflow UI

- Check if the task / run you want to re-run is visible in the DAG's Tree View UI
  - For example, [the `main_summary` DAG tree view](http://workflow.telemetry.mozilla.org/admin/airflow/tree?num_runs=25&root=&dag_id=main_summary).
  - Hover over the little squares to find the scheduled dag run you're looking for.
- If the dag run is not showing in the Dag Tree View UI (maybe deleted)
  - Browse -> Dag Runs
  - Create (you can look at another dag run of the same dag for example values too)
    - Dag Id: the name of the dag, for example `main_summary` or `crash_aggregates`
    - Execution Date: The date the dag should have run, for example `2016-07-14 00:00:00`
    - Start Date: Some date between the execution date and "now", for example `2016-07-20 00:00:05`
    - End Date: Leave it blank
    - State: success
    - Run Id: `scheduled__2016-07-14T00:00:00`
    - External Trigger: unchecked
  - Click Save
  - Click on the Graph view for the dag in question. From the main DAGs view, click the name of the DAG
  - Select the "Run Id" you just entered from the drop-down list
  - Click "Go"
  - Click each element of the DAG and "Mark Success"
  - The tasks should now show in the Tree View UI
- If the dag run is showing in the DAG's Tree View UI
  - Click on the small square for the task you want to re-run
  - **Uncheck** the "Downstream" toggle
  - Click the "Clear" button
  - Confirm that you want to clear it
  - The task should be scheduled to run again straight away.

### Triggering backfill tasks using the CLI

- SSH in to the ECS container instance
- List docker containers using `docker ps`
- Log in to one of the docker containers using `docker exec -it <container_id> bash`. The webserver instance is a good choice.
- Run the desired backfill command, something like `$ airflow backfill main_summary -s 2016-06-20 -e 2016-06-26`
