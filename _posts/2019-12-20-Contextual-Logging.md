---
layout: post
title: Contextual Logging
---

# Rambling

When I first joined CareerBuilder, my team's project was a mess. I was told that it began as a Hackathon project that was popular enough to be picked up by a development team as a replacement for their legacy solution. It was a data pipeline which accepted candidate resume and application data, verified some of the data, transformed the data to match the client's ATS (applicant tracking system), then connected to that ATS and created/updated the data there. It was a microservice architecture composed of 20+ AWS Lambda functions.

Those functions were chained together with SNS topics and each Lambda was configured to know which SNS topic should receive its result. At the time, we weren't using any sort of metrics, error warnings, or really monitoring of any kind. When there was a problem, we relied on clients to report something like, "I know XYZ applied for job 123, but I can't find their resume in our ATS. What's going on?". At that point, we would attempt to repeatedly replay the candidate's data through our pipeline and then comb through CloudWatch logs for each Lambda and try to figure out where things went wrong. Once we were sure we had the right execution logs, we had to sort through log messages like, "Mapping candidate..." and "Getting configuration from event..." and "Delivery process encountered an error" and then trace through the code to try to find where the logs stopped, what the code was doing after that point, and then guess what might've gone wrong. Typical causes were bad data, expired ATS credentials, unexplained Lambda timeouts that usually ended up being code defects, and sometimes just defects that weren't handling edge cases. At first, I thought that we didn't have the right tooling to efficiently diagnose these problems.

As I got familiar with the code and infrastructure, I began to realize that we had no idea what the scope of our problem was. Not only that, but we didn't have any transparency into the system's (mis)behavior. We weren't even exposing any information about the candidate data's trip through our pipeline to our callers; success returned an HTTP 201 CREATED response containing no information about how to follow up on the call's real result. We simply couldn't give an estimated ratio of candidates successfully delivered vs completely dropped on the floor.

At this point, I sort of freaked out. Nothing big or crazy, just an intense introspective list of questions like, "What are we even doing?", "Why are we doing this at all?", "How do we know *any* of this even works?", "What if this is one of the reasons I had trouble finding a job (other than Amazon delivery driver or landscaper) through CareerBuilder when I was looking last year?", and "How many people could this be happening to?". I unconsciously made it my mission to make things better. I started looking into log aggregation (I knew that was a thing) and eventually found out that the company had the tools to aggregate and search logs, our team just wasn't storing them.

Around the time I was chatting with DevOps to figure out log aggregation, I started to come to terms with the realization that I was a pretty good engineer. There had been hints in past performance reviews and cool projects I was given and successfully completed, but it wasn't until I began to budget my time and replace my typical Reddit breaks with work on logging that I understood I could *do* this. I could totally get something in place that could restore some sanity to my team even without official backing by my (awesome) team lead whose response to my ideas so far had actually been along the lines of, "YES, PLEASE. DO THAT."

I ended up looking into several logging libraries, gravitating towards [bunyan](https://github.com/trentm/node-bunyan). When I came across their concept of [child loggers](https://github.com/trentm/node-bunyan#logchild) I had burst of inspiration. All of these bland logs about mapping and auth and delivery needed **context**. We needed to know everything the code knew about the candidate data and its current situation *every* time it was important enough to log anything at all. I didn't really give it a name at the time, but it's come a long way since then, and I've brought the idea to my current team and implemented it incredibly successfully. Now I call it, "Contextual Logging"

# Contextual Logging



## Transaction IDs

