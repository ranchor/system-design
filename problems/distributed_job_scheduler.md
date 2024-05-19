## Problem Statement
Design a job/task scheduler that runs jobs/task at a scheduled interval
## Requirements
### Functional Requirements
* **Submit tasks**: The system should allow the users to submit their tasks for execution.
* **Allocate resources**: The system should be able to allocate the required resources to each task.
* **Remove tasks:** The system should allow the users to cancel the submitted tasks.
* **Monitor task execution:** The task execution should be adequately monitored and rescheduled if the task fails to execute.
* **Release resources**: After successfully executing a task, the system should take back the resources assigned to the task.
* **Show task status**: The system should show the users the current status of the task.
* **Task can also have priority**.  Task with higher priority should be executed first than lower priority
### Non-Functional Requirements
* Availability:  system should always be available for users to add/view the task
* Scalability: Thousands or even millions of tasks can be scheduled and run per day
* Durability: Tasks must not get lost -> we need to persist tasks
* Reliability: Tasks must not be executed much later than expected or dropped -> we need a fault-tolerant system
* Jobs must not be executed multiple times (or such occurences should be kept to a minimum)
* Bounded waiting time: This is how long a task needs to wait before starting execution. We must not execute tasks much later than expected. Users shouldn’t be kept on waiting for an infinite time. 


## Back of Envelope Estimations/Capacity Estimation & Constraints
* Assumptions
    * Total submitted jobs daily = 100 M 
* Traffic Estimates
## High-level API design 
* Write Operations
    * submitJob(api_key, user_id, job_schedule_time, job_type, priority, result_location)
    * deleteJob(user_id, job_id)
* Read Operations
    * viewJob(api_key, user_id, job_id)
    * listJobs(api_key, user_id, pagination_token)

## Database Design
Table: JOB

+------------------------------+--------+
|          Attribute           |  Type  |
+------------------------------+--------+
| user_id (partition key)      | uuid   |
| task_id (sort key)            | uuid   |
| actual_job_execution_time    | date   |
| task_status                   | string |
| task_type                     | string |
| task_interval                 | int    |
| result_location              | string |
| current_retries              | int    |
| max_retries                  | int    |
| scheduled_job_execution_time | date   |
| execution_status             | string |
+------------------------------+--------+
## High Level System Design and Algorithm
## References
* https://www.linkedin.com/pulse/system-design-distributed-job-scheduler-keep-simple-stupid-ismail/
* https://towardsdatascience.com/ace-the-system-design-interview-job-scheduling-system-b25693817950