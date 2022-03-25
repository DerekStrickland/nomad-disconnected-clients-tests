# Standard Bench Test sequence for Nomad Disconnected Clients feature

## Environment Setup

- 1 server
- 2 clients
- My tests use the `Vagrantfile` from the Nomad repo
- Server IP `192.168.56.11`
- `curl` installed on development host
- `iptables` available on client machine
- `jq` installed on development host
- Nomad cluster running
</details>

## Check status of cluster

```shell
nomad status && nomad node status
No running jobs
ID        DC   Name            Class   Drain  Eligibility  Status
b02c891c  dc1  nomad-client02  <none>  false  eligible     ready
16a35141  dc1  nomad-client01  <none>  false  eligible     ready
```

## Run job

Several test jobs have been included in this repository as a convenience for different bench test cases. The list of jobs to test cases is in the link table below. I've also included links to specific instruction per test scenario if the test deviates from these instructions.

- Healthy task that has not expired
  - [spread.nomad](spread.nomad)
  - This file serves as the test instructions for this scenario.
- Failed task
  - [fail.nomad jopspec](fail.nomad)
  - [Test instructions](failed.md)
- Job version updated
  - [spread.nomad jopspec](spread.nomad)
  - [job-version.nomad jopspec](job-version.nomad)
  - [Test instructions](job-version.md)
- No replacement task
  - [no-replace.nomad jopspec](no-replace.nomad)
  - [Test instructions](no-replace.md)
- Expired task
  - [expired.nomad jopspec](expired.nomad)
  - [Test instructions](expired.md)
- Failed replacement task
  - [spread.nomad jopspec](spread.nomad)
  - [Test instructions](failed-replacement.md)


### Example Job HCL with configured setting

<details>
  <summary>Example jobspec</summary>
  
```hcl
job "spread" {
  datacenters = ["dc1"]

  group "cache" {
    count = 2
  
    # This is the setting that controls the disconnected client behavior
    max_client_disconnect = "1h"

    spread {
      attribute = "${node.datacenter}"
    }

    network {
      port "db" {
        to = 6379
      }
    }

    task "redis" {
      driver = "docker"

      config {
        image = "redis:3.2"

        ports = ["db"]
      }

      resources {
        cpu    = 500
        memory = 256
      }
    }
  }
}
```

</details>

### Run job

```shell
nomad run spread.nomad
==> 2022-03-04T11:39:32-05:00: Monitoring evaluation "515b17fe"
    2022-03-04T11:39:32-05:00: Evaluation triggered by job "spread"
    2022-03-04T11:39:32-05:00: Evaluation within deployment: "3ca056fc"
    2022-03-04T11:39:32-05:00: Allocation "e976191a" created: node "b02c891c", group "cache"
    2022-03-04T11:39:32-05:00: Allocation "8692e086" created: node "16a35141", group "cache"
    2022-03-04T11:39:32-05:00: Evaluation status changed: "pending" -> "complete"
==> 2022-03-04T11:39:32-05:00: Evaluation "515b17fe" finished with status "complete"
==> 2022-03-04T11:39:32-05:00: Monitoring deployment "3ca056fc"
  ✓ Deployment "3ca056fc" successful

    2022-03-04T11:39:44-05:00
    ID          = 3ca056fc
    Job ID      = spread
    Job Version = 0
    Status      = successful
    Description = Deployment completed successfully

    Deployed
    Task Group  Desired  Placed  Healthy  Unhealthy  Progress Deadline
    cache       2        2       2        0          2022-03-04T16:49:42Z
```

### Check job definition from HTTP API

```shell
curl $NOMAD_ADDR/v1/job/spread | jq | grep MaxClientDisconnect
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2567    0  2567    0     0  2506k      0 --:--:-- --:--:-- --:--:-- 2506k
      "MaxClientDisconnect": 3600000000000,
```

### Observe job

In a new shell

```shell
watch nomad status spread
...TRUNCATED
Allocations
ID        Node ID   Task Group  Version  Desired  Status   Created    Modified
8692e086  16a35141  cache       0        run      running  7m55s ago  7m45s ago
e976191a  b02c891c  cache       0        run      running  7m55s ago  7m45s ago
```

Make sure you have 2 allocs and they are on *different* clients

## Simulate client disconnect

From one of the clients 

```shell
sudo iptables -I INPUT -s 192.168.56.11 -j DROP
```

Watch session should show one of the allocs transition to `unknown` and a new alloc spun up on the other client.

```shell
Allocations
ID        Node ID   Task Group  Version  Desired  Status   Created    Modified
81104bcc  16a35141  cache       0        run      running  33s ago    32s ago
8692e086  16a35141  cache       0        run      running  9m12s ago  9m2s ago
e976191a  b02c891c  cache       0        run      unknown  9m12s ago  33s ago
```

## Check the metrics output

```shell
curl $NOMAD_ADDR/v1/metrics | jq | grep unknown
...TRUNCATED
{
      "Labels": {
        "task_group": "cache",
        "namespace": "default",
        "host": "nomad-server01",
        "job": "spread"
      },
      "Name": "nomad.nomad.job_summary.unknown",
      "Value": 1
    },
...TRUNCATED
```

## Simulate client reconnect 

### Reconnect network

From one of the clients 

```shell
sudo iptables -D INPUT -s 192.168.56.11 -j DROP
```

### Check job status

Switch to watch shell session and wait for `unknown` alloc to transition to `running` and the replacement alloc to
transition to `complete`.

```shell
Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created     Modified
81104bcc  16a35141  cache       0        stop     complete  3m48s ago   1m8s ago
8692e086  16a35141  cache       0        run      running   12m27s ago  12m17s ago
e976191a  b02c891c  cache       0        run      running   12m27s ago  1m8s ago
```

### Check alloc status for the reconnected alloc to see reconnect event

```
nomad alloc status <alloc-guid>
Recent Events:
Time                       Type         Description
2022-03-04T11:50:51-05:00  Reconnected  Client reconnected
2022-03-04T11:39:32-05:00  Started      Task started by client
2022-03-04T11:39:32-05:00  Task Setup   Building Task Directory
2022-03-04T11:39:32-05:00  Received     Task received by client
```

## Check the metrics output

```shell
curl $NOMAD_ADDR/v1/metrics | jq
...TRUNCATED
    {
      "Labels": {
        "job": "spread",
        "task_group": "cache",
        "namespace": "default",
        "host": "nomad-server01"
      },
      "Name": "nomad.nomad.job_summary.unknown",
      "Value": 0
    },
... TRUNCATED    
```
