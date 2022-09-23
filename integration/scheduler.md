---
layout: page
title: Fully Custom Scheduler
subtitle: Integrating a Custom Scheduler with the Observatory Control System
---

# Custom Scheduler Integration
For users who wish to implement their own scheduler, this section describes that workflow.

The role of the scheduler is to turn observation requests into scheduled observations for a telescope to perform. A scheduler brings together the current set of schedulable requests, site telemetry and telescope availability to generate a schedule of observations.

At its most basic level, a scheduler should be able to: 
* Retrieve all schedulable requests for a given time window
* Take a set of observation requests and turn it into a set of observations to be executed on a telescope
* Determine if a target is observable within the time window specified by a request, and if it is, consider it for scheduling. The [OCS rise_set library](https://github.com/observatorycontrolsystem/rise_set) can be helpful in determining observability of a target from a given location.
* Create an observation for each scheduled request with a specific site, enclosure, telescope, instrument, and start and end time
* If a schedule exists, cancel the existing schedule, excluding any running observations, so it can be replaced with a new set of observations. (HTTP POST `/api/observations/cancel`)
* Submit the newly-generated observations (HTTP POST `/api/observations`)

## Determining the Set of Schedulable Requests
The first step to scheduling observations is for the scheduler to retrieve the set of schedulable requests from the observation portal by querying the `/api/requestgroups/schedulable_requests` endpoint.

A schedulable request is defined as a request which is not in a terminal state, is part of an active proposal with enough time allocated to accommodate the request, and has a window within the current semester.

For more information, see the [API Documentation](https://observatorycontrolsystem.github.io/assets/html/observation-portal.html#operation/listSchedulableRequests)

## Observation Requests Constrain the Scheduler

Recall that an observation request has the following structure, and is defined as a single independent set of observing parameters to be scheduled in one contiguous block on a resource

![Request Model](/assets/images/request.png)

The job of the scheduler is to consider a set of observation requests and create a set of observations that fulfill the requests. There are a set of parameters that help to constrain the scheduling of an observation request:

* **Location** - The location for an observation, which constrains the location in which an observation is allowed to be performed. This is specified by a site, enclosure, telescope and instrument type. 
* **Windows** - A set of acceptable time windows for an observation to be performed, as specified by the user
* **Configurations** - A set of individual configurations for an observation to be performed. Among the constraints specified by the configurations are:
* **Target** - The target to be observed
* **Observing Constraints** - Maximum airmass, minimum lunar distance, maximum lunar phase, maximum seeing, minimum sky transparency

### Location
The scheduler should honor the location defined within an observing request. It is particularly useful for any scheduler implementation to be able to understand the status of the observatory and its available enclosures and telescopes so that it does not schedule an observation where it cannot be performed. For an observatory using the OCS, the observatory resources are stored in the Configuration Database.

### Windows
The set of time windows specified in the observation request further constrains the observation to be scheduled during specific times. The scheduler should be able to use these windows, along with the other request constraints, to generate a specific start and end time for an observation that fulfills the request.

### Configurations
The configurations define the structure of each exposure. In particular the scheduler must ensure that the target is visible within the windows provided by the request, and that the observing constraints are honored. 

{% include notification.html message="Note: It is up to the scheduler to combine the location, window, and target information to ensure that a given target is adequately observable. The [OCS rise_set library](https://github.com/observatorycontrolsystem/rise_set) can be helpful in determining observability of a target from a given location." status="is-info" icon="fas fa-exclamation-triangle" %}

## A Simple Scheduler Implementation
For a single telescope, an implementation of a scheduler could be as simple as returning a single observation to be done in the very near future. The selected observation should be the one with the highest priority that is visible in the near future. If there are multiple observations that share the same priority, the scheduler could choose to schedule the request containing the target with the lower airmass.

In order to avoid gaps in the schedule after a scheduled observation completes, a scheduler could also attempt to schedule multiple observations “back-to-back,” or even fill an entire observing night prior to the start of observations for a night.

## Turning an Observation Request into an Observation
Given the constraining factors defined in the request, a scheduler can now create an observation to fulfill the request. 

The hierarchy of an Observation is shown below, but it is very similar to that of the Request except with a few additional pieces of information tacked on. 

![Observation Model](/assets/images/observation.png)

Once a request has been deemed schedulable at a site’s telescope, an observation must be created. This can be done by sending a POST request with a JSON payload to the `/api/observations` endpoint. 

To illustrate, the minimum JSON specification for an observation is presented:

```json
{
   "site": "mba",
   "enclosure": "enc1",
   "telescope": "tel1",
   "start": "2022-01-01T00:00:00.000",
   "end": "2022-01-01T00:30:00.000",
   "request": 4567, # ID of original observation request
   "configuration_statuses": [
       {
           "instrument_name": "ab12",
           "configuration": 3455 # ID of original request configuration
       },
       {
           "instrument_name": "ab12",
           "configuration": 3456
       },
       {
           "instrument_name": "ab12",
           "configuration": 3457
       },
   ],
}
```

See the [API documentation](https://observatorycontrolsystem.github.io/assets/html/observation-portal.html#operation/createObservation) for more information.

## Advanced Topics
Above and beyond the basic role of a scheduler, we’d like to touch on some advanced topics that an observatory should consider when implementing their own scheduler.

### Telescope Telemetry for Real-Time Availability
Any scheduler implementation should be able to understand the real-time status of observatory resources. An observatory’s Telescope Control System (TCS) can provide real-time updates about its schedulability status in order for the scheduler to understand whether to schedule observations on a given resource.

If real-time status is desired, a TCS should be able to regularly post telemetry data to a centralized storage space where it can be accessed by the scheduler. Las Cumbres Observatory (LCO) utilizes Elasticsearch for this purpose - and LCO’s scheduler regularly pulls data from an Elasticsearch index that the various TCS instances update.

The following telemetry data specification is sufficient to allow the scheduler to know when a resource is available/unavailable for scheduling:

**Datum Name**|**Value String**|**Timestamp**|**Site**|**Enclosure**|**Telescope**|
----------------|----------------|-------------|----------|------------|-----------|
Available for scheduling|True/False|ISO 8601 Date|Site ID|Enclosure ID|Telescope ID|


### Checking In-Progress Observations When Scheduling
Depending on an observatory’s TCS implementation, it may be important for the scheduler to understand when observations are in-progress so that when it produces a new schedule during an observing night, it does not schedule new observations over in-progress observations. If the TCS software receives a new schedule with an observation overlapping an in-progress observation, it may choose to abort the observation - which isn’t always ideal.

One way to avoid this issue is to take into account the time range(s) of currently-executing observation(s). 

To determine if an observation is currently running, check if its start time is before the current time, and its end time is greater than the projected end time of your scheduling run, and it is in a non-terminal state (PENDING or IN_PROGRESS). That observation's end time can then be used as the earliest start time for the next scheduled observation on that resource.

To retrieve all PENDING and IN_PROGRESS observations for a given start/end time:

```
GET /api/observations/?starts_after=<start_date>&ends_before=<end_date>&state=PENDING&state=IN_PROGRESS
```

An example of this is illustrated in the [Adaptive Scheduler](https://github.com/observatorycontrolsystem/adaptive_scheduler/blob/main/adaptive_scheduler/observations.py#L335) project.
