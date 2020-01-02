---
layout: post
title: Contextual Logging
---

# Rambling

When I first joined CareerBuilder, my team's project was a mess. I was told it began as a Hackathon project that was popular enough to be picked up by a development team as a replacement for their legacy solution. It was a data pipeline which accepted candidate resume and application data, verified some of the data, transformed it to match the client's ATS (applicant tracking system), then connected to that ATS and created/updated the data there. It was a microservice architecture composed of 20+ AWS Lambda functions.

Those functions were chained together with SNS topics and each Lambda was configured to know which SNS topic should receive its result. At the time, we weren't using any sort of metrics, error warnings, or really monitoring of any kind. When there was a problem, we relied on clients to report something like, "I know XYZ applied for job 123, but I can't find their resume in our ATS. What's going on?". At that point, we would attempt to repeatedly replay the candidate's data through our pipeline and then comb through CloudWatch logs for each Lambda and try to figure out where things went wrong. Once we were sure we had the right execution logs, we had to sort through log messages like, "Mapping candidate..." and "Getting configuration from event..." and "Delivery process encountered an error" then trace through the code to try to find where the logs stopped, what the code was doing after that point, and then guess what might've gone wrong. Typical causes were bad data, expired ATS credentials, unexplained Lambda timeouts that usually ended up being code defects, and sometimes just defects that weren't handling edge cases. At first, I thought that we didn't have the right tooling to efficiently diagnose these problems.

As I got familiar with the code and infrastructure, I began to realize that we had no idea what the scope of our problems were. Not only that, but we didn't have any transparency into the system's (mis)behavior. We weren't even exposing any information about the candidate data's trip through our pipeline to our callers; success returned an HTTP 201 CREATED response containing no information about how to follow up on the call's real result. We couldn't even give an estimated ratio of candidates successfully delivered vs completely dropped on the floor.

At this point, I sort of freaked out. Nothing big or crazy, just an intense introspective list of questions like, "What are we even doing?", "Why are we doing this at all?", "How do we know *any* of this even works?", "What if this is one of the reasons I had trouble finding a job (other than Amazon delivery driver or landscaper) through CareerBuilder when I was looking last year?", and "How many people could this be happening to?". I unconsciously made it my mission to make things better. I started looking into log aggregation (I knew that was a thing) and eventually found out that the company had the tools to aggregate and search logs, our team just wasn't storing them.

Around the time I was chatting with DevOps to figure out log aggregation, I started to realize that I was a pretty good engineer. There had been hints in past performance reviews and cool projects I was given and successfully completed, but it wasn't until I began to budget my time and replace my typical Reddit breaks with side work on logging that I understood I could *do* this. I could totally get something in place that could restore some sanity to my team even without official backing by my (awesome) team lead whose response to my ideas so far had actually been along the lines of, "YES, PLEASE. DO THAT."

I ended up looking into several logging libraries and gravitated towards [bunyan](https://github.com/trentm/node-bunyan). Their concept of [child loggers](https://github.com/trentm/node-bunyan#logchild) inspired me. All of these bland logs about mapping and auth and delivery needed **context**. We needed to know everything that the code knew about the candidate data and its current situation *every* time it was important enough to log anything at all. I didn't really give it a name at the time, but it's come a long way since then, and I've brought the idea to my current team and implemented it incredibly successfully. Now I call it, "Contextual Logging".

# Contextual Logging

The most important aspect of contextual logging is the ability to log context alongside status/diagnostic type messages throughout a program's execution without having to manually include it in messages or calls to a logger. When something goes wrong (e.g. candidate failing to map to the required structure) it's incredibly helpful to have *all* of the relevant data (e.g. candidate ID, requisition ID, missing fields) right there in that error message. At scale, logs that tell us things like "User mapped successfully" without context become noise that costs money to store and distracts from analysis and investigation. Beyond that, there are some important concepts that help ease investigation and aid in creating cool dashboards.

## Transaction IDs

"Transaction ID" is a loose term, but I generally use it to mean any ID which would be useful to link a chain of contextual log messages together. My current team's project is conversational, so each log message tends to have most of these transaction IDs:

* User ID: Indicates which consumer's actions triggered the logs.
* Conversation ID: Which conversation the consumer was engaged in, generated when a user starts a new conversation.
* Turn ID: A [conversation analysis](https://en.wikipedia.org/wiki/Conversation_analysis#Turn-taking_organization) concept which subdivides conversations into [turns](https://en.wikipedia.org/wiki/Turn-taking).
* Transaction ID: Typically generated, when execution begins, by the service which logged the given message.
* Lambda ID: Generated by the service when it cold boots, discarded when it is terminated.
* AWS Request ID: An ID passed in each request which can be used to find other AWS-collected information about that request.
* Name: The name of the service.
* File: The name of the file logging the given message. This may change during execution.

These types of IDs allow you to use log aggregation to answer basic questions such as these:
* Number of users per unit of time and/or over a period of time
* Most active users over a period of time
* Conversational volume and complexity
* Service traffic volume

## Choices and Results

Choices made, both by the user and the system, can be very revealing into a system's behavior and its users' satisfaction. Some helpful examples:

* An existing user returns
* A user is prompted for a decision
* The choice made (or lack of one, if user moves on to some other action)
* Feedback prompts
* Any success or failure when fulfilling a user's request

These types of logs let you display data like:

* How successful the system is at fulfilling user requests
* Portion of users selecting different paths through the system
* Users' tendencies
* Users' patience with the system
* The change of all of these aspects over time

## Timing

Knowing the amount of time individual services take can be *crucial* to solving performance problems. Without that kind of visibility, troubleshooting slowness, especially intermittent slowness, is impossible. To take it a step further, if each service logs the duration of its subprocesses, that information can be aggregated to find out which specific parts of specific services need attention.

At one point, using this kind of data in combination with various transaction IDs, my team was able to track and reduce response time my more than 50%. We used it to do a few things specifically:

* Average full-system response time
* Exceptionally long response times and what caused them
* How frequently a microservice was used and if it even warranted attention
* Determine where microservices were slow when interacting with infrastructure like data storage
* Where microservices needed to cache data retrieval
* When/If calls to other microservices could be cached

## Documentation

Something I've found extremely helpful is to expose the textual part of the logs in some way that is readable. The messages should be human readable and descriptive and detail what is happening in the system. Some simple examples would be things like, "A new user has been created" or "Required data could not be retrieved". Ideally, these messages should be abstracted out into a single file which (because it's code) is then updated as system functionality changes.

What you'll end up with is a big list of human-readable events that are important enough to show up in the logs. A list like this can give someone unfamiliar with the technical workings of the code (site reliability engineers, product managers, etc) insight and inspiration when making visualizations and dashboards using the log data. It may also prompt them to realize that some valuable logic flow isn't being tracked. This list can be provided as a direct link to the file in source control, or even updated (hopefully via automation) in some kind of separate documentation like a wiki. All of this should make tracking and discovery of [Key Performance Indicators](https://en.wikipedia.org/wiki/Performance_indicator) almost effortless.